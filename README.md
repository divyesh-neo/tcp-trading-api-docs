# TCP Trading Gateway API Documentation

## Overview

The gateway exposes **two TCP connections**. Every byte on both connections is a
fixed-size framed packet whose first field is a `type` enum. The receiver reads
the type, then reads the rest of the union to get the full payload. No dynamic
allocation, no length-prefix parsing.

| Connection | Direction | Purpose |
| ---------- | --------- | ------- |
| **TCP 1 – Market Data** | Bidirectional | Subscribe / unsubscribe tokens; receive live depth snapshots |
| **TCP 2 – Order & Portfolio** | Bidirectional | Place / modify / cancel orders (send); receive order updates, ACKs, and portfolio run/stop commands (recv) |

Testing Zip: [Download Testing ZIP](https://raw.githubusercontent.com/divyesh-neo/tcp-trading-api-docs/Download/Downloads/Alphabots.zip)

*Note: This ZIP is for testing purposes only. When you subscribe to a token, the system generates random market data for that token. The data is not realistic and should not be used for strategy validation. You can use any `pf_id` from `0` to `5000` for testing.*


---

# Common Inner Structs

These structs are embedded unchanged inside the union payloads on both connections.

```cpp
#pragma pack(push, 1)

// ── Enums ──────────────────────────────────────────────────────────────────

enum class Side : uint8_t {
    BUY  = 0,
    SELL = 1
};

enum class OrderType : uint8_t {
    LIMIT = 0,
    IOC   = 1
};

typedef enum : uint8_t {
    NewOms           = 0,
    NewExchange      = 1,
    ModifyOms        = 2,
    ModifyExchange   = 3,
    CancelOms        = 4,
    CancelExchange   = 5,
    ExchnageRejected = 6,
    Fill             = 7,
    PartialFill      = 8,
    ModifyReject     = 9,
    CancelReject     = 10,
    RequestFailed    = 11,
    ModifyFailed     = 12,
    CancelFailed     = 13,
    RMSReject        = 14,
    Unknown          = 15
} OrderEventType;

// ── Inner Structs ──────────────────────────────────────────────────────────

struct Leg {
    uint32_t           symbol_id;
    uint32_t           price;
    uint32_t           qty;
    Side               side;
    unsigned long long start_time;
    uint32_t           oms_order_id;
};

struct SubscribeData {
    uint32_t token;
    bool     is_subscribe;          // true = subscribe, false = unsubscribe
};

struct SubscribeAckData {
    bool     success;
    uint32_t token;
};

struct MarketDepthData {
    uint32_t token;
    uint32_t bids[5];
    uint32_t asks[5];
    uint32_t bids_qty[5];
    uint32_t asks_qty[5];
};

struct NewOrderData {
    uint32_t  strategy_order_id;    // client-assigned; echoed in ACK
    uint16_t  portfolio_id;
    OrderType type;
    uint8_t   num_legs;
    Leg       legs[3];
};

struct NewOrderAckData {
    uint32_t strategy_order_id;     // echoed from NewOrderData
    uint32_t oms_order_id;          // gateway-assigned; use for all future ops
};

struct ModifyOrderData {
    uint32_t oms_order_id;
    Leg      new_leg;
};

struct CancelOrderData {
    uint32_t oms_order_id;
};

struct OrderUpdateData {
    uint32_t       oms_order_id;
    uint64_t       exchange_order_id;
    uint32_t       token;
    uint8_t        side;
    OrderEventType order_event_type;
    int64_t        ordered_price;
    int32_t        ordered_qty;
    int32_t        filled_qty;
    int32_t        filled_qty_sum;
    int64_t        last_fill_price;
};

struct PortfolioData {
    uint16_t portfolio_id;
    bool     is_running;            // true = PORTFOLIO_RUN, false = PORTFOLIO_STOP
};

#pragma pack(pop)
```

---

# TCP 1 — Market Data Connection

## Purpose

* Subscribe and unsubscribe instrument tokens
* Receive live order book snapshots (streamed every 1 second per subscribed token)

## Framed Packet

Every message on this connection is a `MarketDataTCPPacket`. Read the `type`
field first, then dispatch on it to interpret `payload`.

```cpp
#pragma pack(push, 1)

enum class MDPacketType : uint8_t {
    Subscribe    = 0,   // Client → Gateway : subscribe/unsubscribe a token
    SubscribeAck = 1,   // Gateway → Client : response to Subscribe
    MarketDepth  = 2    // Gateway → Client : live depth snapshot
};

struct MarketDataTCPPacket {
    MDPacketType type;

    union {
        SubscribeData    subscribe;      // type == Subscribe
        SubscribeAckData subscribe_ack;  // type == SubscribeAck
        MarketDepthData  market_depth;   // type == MarketDepth
    } payload;
};

#pragma pack(pop)
```

## Packet Type Reference

| `type`        | Value | Direction         | Description |
| ------------- | ----- | ----------------- | ----------- |
| `Subscribe`   | 0     | Client → Gateway  | Subscribe or unsubscribe a token. Set `payload.subscribe.is_subscribe = true` to subscribe, `false` to unsubscribe |
| `SubscribeAck`| 1     | Gateway → Client  | Confirms or rejects a subscribe request. Check `payload.subscribe_ack.success` |
| `MarketDepth` | 2     | Gateway → Client  | Live top-5 bid/ask snapshot for a subscribed token, delivered every 1 second |

## `MarketDepthData` Fields

| Field        | Description              |
| ------------ | ------------------------ |
| `token`      | Instrument token         |
| `bids[5]`    | Top 5 bid prices         |
| `asks[5]`    | Top 5 ask prices         |
| `bids_qty[5]`| Quantity at each bid level |
| `asks_qty[5]`| Quantity at each ask level |

## Subscribe / Receive Example

```cpp
// ── Subscribe ──────────────────────────────────────────────────────────────
MarketDataTCPPacket pkt{};
pkt.type                        = MDPacketType::Subscribe;
pkt.payload.subscribe.token     = 12345;
pkt.payload.subscribe.is_subscribe = true;

send(md_fd, &pkt, sizeof(pkt), 0);

// ── Receive loop ───────────────────────────────────────────────────────────
while (running) {
    MarketDataTCPPacket in{};
    if (recv(md_fd, &in, sizeof(in), MSG_WAITALL) != sizeof(in)) break;

    switch (in.type) {
        case MDPacketType::SubscribeAck:
            printf("[MD] token=%u  ack=%s\n",
                   in.payload.subscribe_ack.token,
                   in.payload.subscribe_ack.success ? "OK" : "FAIL");
            break;

        case MDPacketType::MarketDepth:
            printf("[MD] token=%u  best_bid=%u  best_ask=%u\n",
                   in.payload.market_depth.token,
                   in.payload.market_depth.bids[0],
                   in.payload.market_depth.asks[0]);
            break;

        default:
            break;
    }
}
```

---

# TCP 2 — Order & Portfolio Connection

## Purpose

**Send (Client → Gateway):**

* Place new orders (`NewOrder`)
* Modify open orders (`ModifyOrder`)
* Cancel open orders (`CancelOrder`)

**Receive (Gateway → Client):**

* New order ACK — maps `strategy_order_id` → `oms_order_id` (`NewOrderAck`)
* Order lifecycle updates — fills, rejections, cancels, etc. (`OrderUpdate`)
* Portfolio run/stop commands — **sent by the GUI/server, received by the strategy** (`PortfolioUpdate`)

> **Portfolio run/stop is inbound only on this connection.** The strategy never
> sends `PortfolioData`; it arrives from the GUI/server side. The strategy
> reads it in its recv loop and adjusts its behaviour accordingly.

## Framed Packet

Every message on this connection is an `OrderTCPPacket`.

```cpp
#pragma pack(push, 1)

enum class OrdPacketType : uint8_t {
    // ── Outbound (Client → Gateway) ───────────────────────────────────────
    NewOrder     = 0,   // place a new order
    ModifyOrder  = 1,   // modify price/qty of an open order
    CancelOrder  = 2,   // cancel an open order

    // ── Inbound (Gateway → Client) ────────────────────────────────────────
    NewOrderAck  = 3,   // immediate ACK: strategy_order_id → oms_order_id
    OrderUpdate  = 4,   // order lifecycle event (fills, rejects, cancels…)
    PortfolioUpdate = 5 // portfolio run/stop command from GUI/server
};

struct OrderTCPPacket {
    OrdPacketType type;

    union {
        // Outbound
        NewOrderData    new_order;       // type == NewOrder
        ModifyOrderData modify_order;    // type == ModifyOrder
        CancelOrderData cancel_order;    // type == CancelOrder

        // Inbound
        NewOrderAckData new_order_ack;   // type == NewOrderAck
        OrderUpdateData order_update;    // type == OrderUpdate
        PortfolioData   portfolio_update;// type == PortfolioUpdate
    } payload;
};

#pragma pack(pop)
```

## Packet Type Reference

| `type`            | Value | Direction        | Description |
| ----------------- | ----- | ---------------- | ----------- |
| `NewOrder`        | 0     | Client → Gateway | Place a new order. Fill `payload.new_order`. Assign a unique `strategy_order_id`. |
| `ModifyOrder`     | 1     | Client → Gateway | Modify price/qty. Fill `payload.modify_order` with `oms_order_id` and `new_leg`. |
| `CancelOrder`     | 2     | Client → Gateway | Cancel an open order. Fill `payload.cancel_order.oms_order_id`. |
| `NewOrderAck`     | 3     | Gateway → Client | Immediate response to `NewOrder`. Maps `strategy_order_id → oms_order_id`. All further references use `oms_order_id`. |
| `OrderUpdate`     | 4     | Gateway → Client | Order lifecycle event. Inspect `payload.order_update.order_event_type`. |
| `PortfolioUpdate` | 5     | Gateway → Client | Portfolio run/stop from the GUI/server. `payload.portfolio_update.is_running = true` means RUN, `false` means STOP. |

## Portfolio Run / Stop — Inbound Behaviour

`PortfolioUpdate` packets arrive on the **recv side** of this connection,
pushed by the GUI or server. The strategy never sends them.

**Rules enforced by the gateway:**

* A portfolio starts in the **stopped** state.
* Any `NewOrder` packet whose `portfolio_id` maps to a **stopped** portfolio is
  **silently dropped** — no `NewOrderAck` is returned, the order never reaches
  the exchange.
* `ModifyOrder` and `CancelOrder` are **not** filtered by portfolio state (they
  reference `oms_order_id` directly).
* `PortfolioUpdate` with `is_running = false` does **not** cancel live exchange
  orders; those continue to generate `OrderUpdate` events.
* The strategy must track portfolio state locally and **must not** send
  `NewOrder` for a stopped portfolio — doing so causes `recv()` to block
  indefinitely waiting for an ACK that will never arrive.

**Timeline:**

```text
[GUI/Server → Gateway → Strategy recv]

PortfolioUpdate(portfolio_id=42, is_running=true)   → portfolio 42 = RUNNING
  NewOrder(portfolio_id=42) sent by strategy         → ACCEPTED → exchange
  NewOrder(portfolio_id=42) sent by strategy         → ACCEPTED → exchange

PortfolioUpdate(portfolio_id=42, is_running=false)  → portfolio 42 = STOPPED
  NewOrder(portfolio_id=42) sent by strategy         → DROPPED  (no ACK)
  NewOrder(portfolio_id=42) sent by strategy         → DROPPED  (no ACK)

PortfolioUpdate(portfolio_id=42, is_running=true)   → portfolio 42 = RUNNING
  NewOrder(portfolio_id=42) sent by strategy         → ACCEPTED → exchange
```

## `OrderUpdateData` Fields

| Field               | Type           | Description                              |
| ------------------- | -------------- | ---------------------------------------- |
| `oms_order_id`      | uint32_t       | Gateway-assigned order ID                |
| `exchange_order_id` | uint64_t       | Exchange-assigned order ID               |
| `token`             | uint32_t       | Instrument token                         |
| `side`              | uint8_t        | 0 = Buy, 1 = Sell                        |
| `order_event_type`  | OrderEventType | Current event (see Order Event Reference)|
| `ordered_price`     | int64_t        | Original order price                     |
| `ordered_qty`       | int32_t        | Original order quantity                  |
| `filled_qty`        | int32_t        | Quantity filled in **this** event        |
| `filled_qty_sum`    | int32_t        | Cumulative total filled so far           |
| `last_fill_price`   | int64_t        | Price of the latest fill                 |

## Send / Receive Example

```cpp
// ── Place a new order ──────────────────────────────────────────────────────
OrderTCPPacket out{};
out.type                                  = OrdPacketType::NewOrder;
out.payload.new_order.strategy_order_id   = 1001;
out.payload.new_order.portfolio_id        = 42;
out.payload.new_order.type                = OrderType::LIMIT;
out.payload.new_order.num_legs            = 1;
out.payload.new_order.legs[0].symbol_id   = 12345;
out.payload.new_order.legs[0].price       = 98500;
out.payload.new_order.legs[0].qty         = 10;
out.payload.new_order.legs[0].side        = Side::BUY;
out.payload.new_order.legs[0].start_time  = 0;

send(ord_fd, &out, sizeof(out), 0);

// ── Cancel an order ────────────────────────────────────────────────────────
OrderTCPPacket cancel{};
cancel.type                             = OrdPacketType::CancelOrder;
cancel.payload.cancel_order.oms_order_id = 5;

send(ord_fd, &cancel, sizeof(cancel), 0);

// ── Receive loop ───────────────────────────────────────────────────────────
while (running) {
    OrderTCPPacket in{};
    if (recv(ord_fd, &in, sizeof(in), MSG_WAITALL) != sizeof(in)) break;

    switch (in.type) {

        case OrdPacketType::NewOrderAck: {
            // Immediately maps client ID → OMS ID.
            // Store this mapping; all further events use oms_order_id.
            auto& a = in.payload.new_order_ack;
            order_map[a.strategy_order_id] = a.oms_order_id;
            printf("[ACK]  sid=%-6u  →  oms=%u\n",
                   a.strategy_order_id, a.oms_order_id);
            break;
        }

        case OrdPacketType::OrderUpdate: {
            auto& u = in.payload.order_update;
            handle_order_update(u);   // dispatch on u.order_event_type
            break;
        }

        case OrdPacketType::PortfolioUpdate: {
            // Sent by GUI/server — the strategy must react immediately.
            auto& p = in.payload.portfolio_update;
            set_portfolio_running(p.portfolio_id, p.is_running);
            printf("[PORTFOLIO] id=%u  state=%s\n",
                   p.portfolio_id, p.is_running ? "RUNNING" : "STOPPED");
            break;
        }

        default:
            break;
    }
}
```

---

# Order Event Reference

## Order Event Type Enum

```cpp
typedef enum : uint8_t {
    NewOms           = 0,
    NewExchange      = 1,
    ModifyOms        = 2,
    ModifyExchange   = 3,
    CancelOms        = 4,
    CancelExchange   = 5,
    ExchnageRejected = 6,
    Fill             = 7,
    PartialFill      = 8,
    ModifyReject     = 9,
    CancelReject     = 10,
    RequestFailed    = 11,
    ModifyFailed     = 12,
    CancelFailed     = 13,
    RMSReject        = 14,
    Unknown          = 15
} OrderEventType;
```

## Detailed Event Descriptions

### `NewOms` (0)

**Trigger:** Sent immediately after `NewOrderAck` once the OMS has accepted and queued the order for forwarding to the exchange.

**What it means:** The order exists inside the OMS. It has **not** yet reached the exchange.

**Action:** Mark order as `PENDING`. Do not assume a fill is imminent.

---

### `NewExchange` (1)

**Trigger:** Exchange confirmed the order is booked in its order book.

**What it means:** The order is now **live**. Matching can begin.

**Action:** Mark order as `OPEN`. Safe to send `ModifyOrder` or `CancelOrder` from this point.

---

### `ModifyOms` (2)

**Trigger:** OMS accepted the modify and forwarded it to the exchange.

**What it means:** Modify is in flight. Original parameters still active until `ModifyExchange` arrives.

**Action:** Record the pending modification; do not update working price/qty yet.

---

### `ModifyExchange` (3)

**Trigger:** Exchange confirmed the modification.

**What it means:** New price/qty from `ModifyOrderData.new_leg` is now live on the book.

**Action:** Update local order cache with new parameters.

---

### `CancelOms` (4)

**Trigger:** OMS accepted the cancel and forwarded it to the exchange.

**What it means:** Cancel is in flight. The order is still technically open — a fill can still arrive before the cancel is confirmed.

**Action:** Mark as `CANCEL_PENDING`. Continue handling `PartialFill` or `Fill` events that may still arrive.

---

### `CancelExchange` (5)

**Trigger:** Exchange confirmed the order is removed from the book.

**What it means:** Order is fully cancelled. No further fills will arrive. **TERMINAL.**

**Action:** Mark order as `CANCELLED`. Free associated resources.

---

### `ExchnageRejected` (6)

**Trigger:** Exchange refused to accept the order after it was forwarded by the OMS.

**What it means:** Common causes: price outside circuit-breaker limits, invalid instrument state, duplicate exchange order ID. **TERMINAL.**

**Action:** Mark order as `REJECTED`. Log for investigation. Do not retry without correcting the underlying cause.

---

### `Fill` (7)

**Trigger:** Entire ordered quantity has been matched.

**What it means:** Order is completely filled. `filled_qty_sum == ordered_qty`. **TERMINAL.**

**Action:** Mark order as `FILLED`. Update position and P&L.

---

### `PartialFill` (8)

**Trigger:** Some quantity matched; remainder stays open.

**What it means:** `filled_qty` = qty in this event. `filled_qty_sum` = cumulative total filled. `last_fill_price` = price of this fill. Order remains `OPEN`.

**Action:** Update position. Continue monitoring for further fills or cancels.

---

### `ModifyReject` (9)

**Trigger:** Exchange refused the modify request.

**What it means:** Common cause: order already filled or cancelled at the exchange. Original parameters remain active.

**Action:** Revert local order state to pre-modify parameters. Log for investigation.

---

### `CancelReject` (10)

**Trigger:** Exchange refused the cancel request.

**What it means:** Most common cause: order was already filled or cancelled before the cancel arrived (race condition). Order may already be in a terminal state.

**Action:** Check if `Fill` or `CancelExchange` has already arrived for this `oms_order_id`. If so, the order is closed. Otherwise investigate.

---

### `RequestFailed` (11)

**Trigger:** New order could not be forwarded to the exchange due to a connectivity or internal OMS failure.

**What it means:** The order never reached the exchange. **TERMINAL.**

**Action:** Mark order as `FAILED`. Retry with a new `strategy_order_id` after verifying system state.

---

### `ModifyFailed` (12)

**Trigger:** Modify could not be forwarded to the exchange due to a connectivity or internal OMS failure.

**What it means:** Modify never reached the exchange. Original parameters remain active.

**Action:** Revert local order state to pre-modify parameters. Retry if appropriate.

---

### `CancelFailed` (13)

**Trigger:** Cancel could not be forwarded to the exchange due to a connectivity or internal OMS failure.

**What it means:** Cancel never reached the exchange. Order is still `OPEN`.

**Action:** Retry `CancelOrder` if cancellation is still intended.

---

### `RMSReject` (14)

**Trigger:** Pre-trade risk check blocked the order before it was forwarded to the exchange.

**What it means:** Order violated a risk limit (position size, order value, margin, etc.). **TERMINAL.**

**Action:** Mark order as `REJECTED`. Do not retry without addressing the risk breach.

---

### `Unknown` (15)

**Trigger:** OMS received an exchange response it could not classify.

**What it means:** Should not occur under normal operation.

**Action:** Log the full `OrderUpdateData` for investigation. Do not change order state.

---

## Order Event Summary Table

| Value | Name               | Terminal? | Description                                        |
| ----- | ------------------ | --------- | -------------------------------------------------- |
| 0     | `NewOms`           | No        | OMS accepted, queued for exchange                  |
| 1     | `NewExchange`      | No        | Order live in exchange book                        |
| 2     | `ModifyOms`        | No        | Modify forwarded to exchange                       |
| 3     | `ModifyExchange`   | No        | Exchange confirmed modification                    |
| 4     | `CancelOms`        | No        | Cancel forwarded to exchange                       |
| 5     | `CancelExchange`   | **Yes**   | Exchange confirmed cancellation                    |
| 6     | `ExchnageRejected` | **Yes**   | Exchange rejected the order                        |
| 7     | `Fill`             | **Yes**   | Order fully filled                                 |
| 8     | `PartialFill`      | No        | Partial fill; remainder open                       |
| 9     | `ModifyReject`     | No        | Exchange rejected the modify                       |
| 10    | `CancelReject`     | No        | Exchange rejected the cancel                       |
| 11    | `RequestFailed`    | **Yes**   | New order never reached exchange                   |
| 12    | `ModifyFailed`     | No        | Modify never reached exchange                      |
| 13    | `CancelFailed`     | No        | Cancel never reached exchange                      |
| 14    | `RMSReject`        | **Yes**   | Blocked by pre-trade risk                          |
| 15    | `Unknown`          | No        | Unclassified exchange response                     |

> **Terminal** — no further `OrderUpdate` events will arrive for that `oms_order_id` after a terminal event.

---

# Typical Lifecycles

**Happy path — full fill:**

```text
Client sends  OrderTCPPacket{NewOrder,      strategy_order_id=1001, portfolio_id=42}
Gateway sends OrderTCPPacket{NewOrderAck,   strategy_order_id=1001, oms_order_id=5}
              ──────────────────────────── all events below use oms_order_id=5 ────
Gateway sends OrderTCPPacket{OrderUpdate,   NewOms}
Gateway sends OrderTCPPacket{OrderUpdate,   NewExchange}
Gateway sends OrderTCPPacket{OrderUpdate,   PartialFill,  filled_qty=5,  sum=5}
Gateway sends OrderTCPPacket{OrderUpdate,   Fill,         filled_qty=5,  sum=10}  ✓ TERMINAL
```

**Cancel path:**

```text
…NewExchange received…
Client sends  OrderTCPPacket{CancelOrder,   oms_order_id=5}
Gateway sends OrderTCPPacket{OrderUpdate,   CancelOms}
Gateway sends OrderTCPPacket{OrderUpdate,   CancelExchange}  ✓ TERMINAL
```

**Portfolio stop → order dropped:**

```text
GUI/Server sends  OrderTCPPacket{PortfolioUpdate, portfolio_id=42, is_running=false}
  → strategy recv loop updates local state: portfolio 42 = STOPPED

Client sends  OrderTCPPacket{NewOrder, portfolio_id=42}
  → gateway drops silently — no NewOrderAck, no OrderUpdate
```

---

# Strategy Lifecycle Example

Full multi-threaded C++ example demonstrating both TCP connections.

```cpp
#include <sys/socket.h>
#include <unistd.h>
#include <thread>
#include <atomic>
#include <mutex>
#include <unordered_map>
#include <unordered_set>
#include <cstdio>

// ── Shared state ─────────────────────────────────────────────────────────────
std::atomic<bool> app_running{false};

// Latest market snapshot (updated by market data thread)
MarketDepthData   latest_depth{};
std::mutex        depth_mutex;

// strategy_order_id → oms_order_id (populated on NewOrderAck)
std::unordered_map<uint32_t, uint32_t> order_map;
std::mutex                              order_map_mutex;

// Portfolio running state – mirrors gateway state, updated from PortfolioUpdate
std::unordered_set<uint16_t> running_portfolios;
std::mutex                   portfolio_mutex;

std::atomic<uint32_t> next_sid{1};   // monotonic strategy_order_id counter

inline bool is_running(uint16_t pid) {
    std::lock_guard<std::mutex> lk(portfolio_mutex);
    return running_portfolios.count(pid) > 0;
}

inline void set_running(uint16_t pid, bool run) {
    std::lock_guard<std::mutex> lk(portfolio_mutex);
    if (run) running_portfolios.insert(pid);
    else     running_portfolios.erase(pid);
}

// ── Thread 1: Market Data recv ───────────────────────────────────────────────
void market_data_thread(int md_fd, uint32_t token)
{
    // Subscribe
    MarketDataTCPPacket sub{};
    sub.type                        = MDPacketType::Subscribe;
    sub.payload.subscribe.token     = token;
    sub.payload.subscribe.is_subscribe = true;
    send(md_fd, &sub, sizeof(sub), 0);

    while (app_running.load()) {
        MarketDataTCPPacket in{};
        if (recv(md_fd, &in, sizeof(in), MSG_WAITALL) != sizeof(in)) break;

        switch (in.type) {
            case MDPacketType::SubscribeAck:
                printf("[MD] token=%u  ack=%s\n",
                       in.payload.subscribe_ack.token,
                       in.payload.subscribe_ack.success ? "OK" : "FAIL");
                break;

            case MDPacketType::MarketDepth: {
                std::lock_guard<std::mutex> lk(depth_mutex);
                latest_depth = in.payload.market_depth;
                break;
            }

            default: break;
        }
    }

    // Unsubscribe
    sub.payload.subscribe.is_subscribe = false;
    send(md_fd, &sub, sizeof(sub), 0);
}

// ── Thread 2: Order & Portfolio recv ─────────────────────────────────────────
void order_recv_thread(int ord_fd)
{
    while (app_running.load()) {
        OrderTCPPacket in{};
        if (recv(ord_fd, &in, sizeof(in), MSG_WAITALL) != sizeof(in)) break;

        switch (in.type) {

            case OrdPacketType::NewOrderAck: {
                auto& a = in.payload.new_order_ack;
                {
                    std::lock_guard<std::mutex> lk(order_map_mutex);
                    order_map[a.strategy_order_id] = a.oms_order_id;
                }
                printf("[ACK]  sid=%-6u  →  oms=%u\n",
                       a.strategy_order_id, a.oms_order_id);
                break;
            }

            case OrdPacketType::OrderUpdate: {
                auto& u = in.payload.order_update;
                switch (u.order_event_type) {
                    case NewOms:
                        printf("[UPD] oms=%u  NewOms\n", u.oms_order_id); break;
                    case NewExchange:
                        printf("[UPD] oms=%u  NewExchange – LIVE\n", u.oms_order_id); break;
                    case CancelOms:
                        printf("[UPD] oms=%u  CancelOms – cancel in flight\n", u.oms_order_id); break;
                    case CancelExchange:
                        printf("[UPD] oms=%u  CancelExchange – CANCELLED ✓\n", u.oms_order_id); break;
                    case PartialFill:
                        printf("[UPD] oms=%u  PartialFill  qty=%d  sum=%d  px=%ld\n",
                               u.oms_order_id, u.filled_qty, u.filled_qty_sum, u.last_fill_price); break;
                    case Fill:
                        printf("[UPD] oms=%u  Fill  sum=%d  px=%ld ✓\n",
                               u.oms_order_id, u.filled_qty_sum, u.last_fill_price); break;
                    case ExchnageRejected:
                        printf("[UPD] oms=%u  ExchangeRejected ✗\n", u.oms_order_id); break;
                    case RMSReject:
                        printf("[UPD] oms=%u  RMSReject ✗\n", u.oms_order_id); break;
                    case RequestFailed:
                        printf("[UPD] oms=%u  RequestFailed ✗\n", u.oms_order_id); break;
                    default:
                        printf("[UPD] oms=%u  event=%d\n", u.oms_order_id, u.order_event_type); break;
                }
                break;
            }

            case OrdPacketType::PortfolioUpdate: {
                // Pushed by GUI/server — react immediately
                auto& p = in.payload.portfolio_update;
                set_running(p.portfolio_id, p.is_running);
                printf("[PORTFOLIO] id=%u  →  %s\n",
                       p.portfolio_id, p.is_running ? "RUNNING" : "STOPPED");
                break;
            }

            default: break;
        }
    }
}

// ── Thread 3: Strategy / Order send ──────────────────────────────────────────
void strategy_thread(int ord_fd, uint16_t portfolio_id, uint32_t token)
{
    // Wait until GUI sends PORTFOLIO_RUN for our portfolio
    while (app_running.load() && !is_running(portfolio_id))
        std::this_thread::sleep_for(std::chrono::milliseconds(10));

    printf("[STRATEGY] Portfolio %u is RUNNING – starting trade loop\n", portfolio_id);

    uint32_t open_oms_id = 0;

    while (app_running.load()) {

        // If GUI stopped the portfolio, pause order placement
        if (!is_running(portfolio_id)) {
            printf("[STRATEGY] Portfolio %u STOPPED – pausing\n", portfolio_id);
            while (app_running.load() && !is_running(portfolio_id))
                std::this_thread::sleep_for(std::chrono::milliseconds(10));
            printf("[STRATEGY] Portfolio %u RUNNING again\n", portfolio_id);
        }

        MarketDepthData md{};
        {
            std::lock_guard<std::mutex> lk(depth_mutex);
            md = latest_depth;
        }

        if (md.token == 0 || open_oms_id != 0) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            continue;
        }

        const uint32_t THRESHOLD = 100000;
        if (md.asks[0] > 0 && md.asks[0] < THRESHOLD) {

            // Final guard: portfolio must still be running at send time
            if (!is_running(portfolio_id)) continue;

            uint32_t sid = next_sid.fetch_add(1);

            OrderTCPPacket out{};
            out.type                                 = OrdPacketType::NewOrder;
            out.payload.new_order.strategy_order_id  = sid;
            out.payload.new_order.portfolio_id       = portfolio_id;
            out.payload.new_order.type               = OrderType::LIMIT;
            out.payload.new_order.num_legs           = 1;
            out.payload.new_order.legs[0].symbol_id  = token;
            out.payload.new_order.legs[0].price      = md.asks[0];
            out.payload.new_order.legs[0].qty        = 10;
            out.payload.new_order.legs[0].side       = Side::BUY;
            out.payload.new_order.legs[0].start_time = 0;

            send(ord_fd, &out, sizeof(out), 0);
            printf("[STRATEGY] NewOrder sid=%u  px=%u\n", sid, md.asks[0]);
            // oms_order_id arrives asynchronously in order_recv_thread via NewOrderAck
            open_oms_id = sid;   // placeholder until ACK maps it
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

// ── Main ──────────────────────────────────────────────────────────────────────
int main()
{
    int md_fd  = /* connect_tcp("md-host",  MD_PORT)  */ 0;
    int ord_fd = /* connect_tcp("ord-host", ORD_PORT) */ 0;

    const uint16_t PORTFOLIO_ID = 42;
    const uint32_t TOKEN        = 12345;

    app_running.store(true);

    std::thread t1(market_data_thread, md_fd,  TOKEN);
    std::thread t2(order_recv_thread,  ord_fd);
    std::thread t3(strategy_thread,    ord_fd, PORTFOLIO_ID, TOKEN);

    std::this_thread::sleep_for(std::chrono::seconds(60));
    app_running.store(false);

    t1.join(); t2.join(); t3.join();
    close(md_fd); close(ord_fd);
    return 0;
}
```

---

# Binary Protocol Notes

All structures use `#pragma pack(push, 1)` / `#pragma pack(pop)`.
The union inside each TCP packet is **fixed-size** — it is always as large
as the largest member. Both sides always send and receive `sizeof(MarketDataTCPPacket)`
or `sizeof(OrderTCPPacket)` exactly; no variable-length reads are needed.

---

# Transport Notes

| Property   | Value         |
| ---------- | ------------- |
| Protocol   | TCP           |
| Encoding   | Packed binary |
| Byte Order | Little Endian |

---

# Recommended Socket Options

```cpp
TCP_NODELAY   // disable Nagle – critical for low latency
SO_REUSEADDR
SO_KEEPALIVE
```

---

# Recommended Architecture

```text
Thread 1 – Market Data recv      (TCP 1)
Thread 2 – Order & Portfolio recv (TCP 2)
Thread 3 – Strategy / Order send  (TCP 2)
```

---

# Best Practices

* Always read `sizeof(MarketDataTCPPacket)` or `sizeof(OrderTCPPacket)` in one `recv` — the fixed union size means this is always safe.
* The strategy must **never** send a `NewOrder` for a stopped portfolio. No ACK will arrive and `recv` will block indefinitely.
* Update local portfolio state **in the recv thread** the moment `PortfolioUpdate` arrives — do not defer.
* Use a monotonically increasing `strategy_order_id`; never reuse IDs within a session.
* Store the `strategy_order_id → oms_order_id` mapping immediately on `NewOrderAck`; all subsequent operations use `oms_order_id`.
* `CancelOms` is not confirmation — fills can still arrive until `CancelExchange` is received.
* `PORTFOLIO_STOP` does **not** cancel live exchange orders; manage open positions explicitly.
* Use dedicated threads per connection; avoid sharing sockets across threads.
* Avoid heap allocations in the hot recv path — all packets are fixed-size stack structs.

---

# Version Information

| Field            | Value                 |
| ---------------- | --------------------- |
| Protocol Version | 1.1                   |
| Transport        | TCP                   |
| Encoding         | Packed binary structs |
