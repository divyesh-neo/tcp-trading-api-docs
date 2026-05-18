# TCP Trading Gateway API Documentation

## Overview

This document describes the TCP-based trading gateway interface for:

1. **Market Data Connection**
2. **Order Management Connection**

   * New Order Placement
   * Modify Order
   * Cancel Order
   * Order Updates

The API uses packed binary structures for ultra-low-latency communication.

---

# Common Packet Definitions

```cpp
#pragma pack(push, 1)

enum class Side : uint8_t {
    BUY = 0,
    SELL = 1
};

enum class OrderType : uint8_t {
    LIMIT = 0,
    IOC = 1
};

typedef enum : uint8_t {
    NewOms = 0,
    NewExchange = 1,
    ModifyOms = 2,
    ModifyExchange = 3,
    CancelExchange = 4,
    ExchnageRejected = 5,
    Fill = 6,
    PartialFill = 7,
    ModifyReject = 8,
    CancelReject = 9,
    RequestFailed = 10,
    ModifyFailed = 11,
    CancelFailed = 12,
    RMSReject = 13,
    Unknown = 14
} OrderEventType;

struct Leg {
    uint32_t symbol_id;
    uint32_t price;
    uint32_t qty;
    Side side;
    unsigned long long start_time;
    uint32_t oms_order_id;
};

#pragma pack(pop)
```

---

# 1. Market Data TCP Connection

## Purpose

Used for:

* Token subscription
* Receiving live market data

Market data is streamed every **1 second** for subscribed tokens.

---

# Market Data Subscription Packet

```cpp
#pragma pack(push, 1)

struct SubscribePacket {
    uint32_t token;
    bool is_subsribe;
};

struct SubscribeResponsePacket {
    bool success;
    uint32_t token;
};

#pragma pack(pop)
```

---

# Subscribe Request

| Field       | Type     | Description                           |
| ----------- | -------- | ------------------------------------- |
| token       | uint32_t | Exchange token / symbol token         |
| is_subsribe | bool     | true = subscribe, false = unsubscribe |

---

# Subscribe Example

```cpp
SubscribePacket sub{};

sub.token = 12345;
sub.is_subsribe = true;

send(socket_fd, &sub, sizeof(sub), 0);
```

---

# Frontend Response Packet

```cpp
#pragma pack(push, 1)

struct FrontendResponsePacket {
    bool success;
    uint32_t token;
};

#pragma pack(pop)
```

---

# Market Data Packet

```cpp
#pragma pack(push, 1)

struct MarketDataPacket {
    uint32_t token;

    uint32_t bids[5];
    uint32_t asks[5];

    uint32_t bids_qty[5];
    uint32_t asks_qty[5];
};

#pragma pack(pop)
```

---

# Market Data Packet Description

| Field       | Description             |
| ----------- | ----------------------- |
| token       | Instrument token        |
| bids[5]     | Top 5 bid prices        |
| asks[5]     | Top 5 ask prices        |
| bids_qty[5] | Quantity for bid levels |
| asks_qty[5] | Quantity for ask levels |

---

# 2. Order Management TCP Connection

## Purpose

Used for:

* Place Order
* Modify Order
* Cancel Order
* Receive Order Updates

---

# New Order Packet

```cpp
#pragma pack(push, 1)

struct NewOrderPacket {
    uint16_t portfolio_id;
    OrderType type;
    uint8_t num_legs;
    Leg legs[3];
};

#pragma pack(pop)
```

---

# New Order Packet Fields

| Field        | Type      | Description                   |
| ------------ | --------- | ----------------------------- |
| portfolio_id | uint16_t  | Strategy/portfolio identifier |
| type         | OrderType | LIMIT or IOC                  |
| num_legs     | uint8_t   | Number of active legs         |
| legs         | Leg[3]    | Maximum 3 legs                |

---

# Modify Order Packet

```cpp
#pragma pack(push, 1)

struct ModifyOrderPacket {
    uint32_t oms_order_id;
    Leg new_leg;
};

#pragma pack(pop)
```

---

# Modify Order Packet Fields

| Field        | Type     | Description              |
| ------------ | -------- | ------------------------ |
| oms_order_id | uint32_t | Existing OMS order ID    |
| new_leg      | Leg      | Updated order parameters |

---

# Cancel Order Packet

```cpp
#pragma pack(push, 1)

struct CancelOrderPacket {
    uint32_t oms_order_id;
};

#pragma pack(pop)
```

---

# Cancel Order Packet Fields

| Field        | Type     | Description            |
| ------------ | -------- | ---------------------- |
| oms_order_id | uint32_t | OMS order ID to cancel |

---

# Order Update Packet

```cpp
#pragma pack(push, 1)

struct OrderUpdatePacket {
    uint32_t oms_order_id;
    uint64_t exchange_order_id;
    uint32_t token;

    uint8_t side;

    OrderEventType order_event_type;

    int64_t ordered_price;
    int32_t ordered_qty;

    int32_t filled_qty;
    int32_t filled_qty_sum;

    int64_t last_fill_price;
};

#pragma pack(pop)
```

---

# Order Update Packet Fields

| Field             | Type           | Description                      |
| ----------------- | -------------- | -------------------------------- |
| oms_order_id      | uint32_t       | OMS generated order ID           |
| exchange_order_id | uint64_t       | Exchange assigned order ID       |
| token             | uint32_t       | Instrument token                 |
| side              | uint8_t        | 0 = Buy, 1 = Sell                |
| order_event_type  | OrderEventType | Current order state              |
| ordered_price     | int64_t        | Original order price             |
| ordered_qty       | int32_t        | Original order quantity          |
| filled_qty        | int32_t        | Current fill quantity            |
| filled_qty_sum    | int32_t        | Total cumulative filled quantity |
| last_fill_price   | int64_t        | Latest fill price                |

---

# Order Event Types

```cpp
#pragma pack(push, 1)

typedef enum : uint8_t {
    NewOms = 0,
    NewExchange = 1,
    ModifyOms = 2,
    ModifyExchange = 3,
    CancelExchange = 4,
    ExchnageRejected = 5,
    Fill = 6,
    PartialFill = 7,
    ModifyReject = 8,
    CancelReject = 9,
    RequestFailed = 10,
    ModifyFailed = 11,
    CancelFailed = 12,
    RMSReject = 13,
    Unknown = 14
} OrderEventType;

#pragma pack(pop)
```

---

# Typical Order Lifecycle

```text
Client Sends New Order
        |
        v

NewOms
        |
        v

NewExchange
        |
        v

PartialFill
        |
        v

Fill
```

---

# Binary Protocol Notes

## Packing

All structures are packed using:

```cpp
#pragma pack(push, 1)
...
#pragma pack(pop)
```

---

# Transport Notes

| Property   | Value         |
| ---------- | ------------- |
| Protocol   | TCP           |
| Encoding   | Binary        |
| Byte Order | Little Endian |

---

# Recommended Socket Options

```cpp
TCP_NODELAY
SO_REUSEADDR
SO_KEEPALIVE
```

---

# Recommended Architecture

```text
Market Data Thread
Order Placement Thread
Order Update Thread
Strategy Thread
```

---

# Sample Workflow

## Market Data Flow

```text
Connect
    ->
Subscribe Token
    ->
Receive MarketDataPacket continuously
```

---

## Order Flow

```text
Connect
    ->
Send NewOrderPacket
    ->
Receive OrderUpdatePacket
    ->
Modify/Cancel if required
```

---

# Best Practices

* Use non-blocking sockets
* Use dedicated threads
* Validate packet sizes
* Avoid heap allocations in hot path
* Maintain local order cache
* Handle reconnect logic properly

---

# Version Information

| Field            | Value                 |
| ---------------- | --------------------- |
| Protocol Version | 1.0                   |
| Transport        | TCP                   |
| Encoding         | Packed Binary Structs |
