# Day 90: Project — Order Book

## Introduction

This is a **project chapter** that combines the month's knowledge into a practical mini-project. We'll create a complete implementation of an **Order Book** — one of the key data structures in algorithmic trading.

## What is an Order Book?

An **Order Book** is a data structure that stores all active buy and sell orders for a specific trading instrument.

```
         ASKS (sell orders)
         ┌─────────────────────────┐
         │ $42,150  │  2.5 BTC    │ ← Best ask (lowest)
         │ $42,200  │  1.8 BTC    │
         │ $42,300  │  3.2 BTC    │
         │ $42,500  │  5.0 BTC    │
         └─────────────────────────┘
                   SPREAD
         ┌─────────────────────────┐
         │ $42,100  │  1.5 BTC    │ ← Best bid (highest)
         │ $42,050  │  2.0 BTC    │
         │ $42,000  │  4.2 BTC    │
         │ $41,900  │  3.8 BTC    │
         └─────────────────────────┘
         BIDS (buy orders)
```

### Key Concepts

- **Bid** — buy order (buyer is willing to buy at this price)
- **Ask** — sell order (seller is willing to sell at this price)
- **Spread** — difference between the best ask and best bid
- **Depth** — number of price levels in the order book
- **Liquidity** — total volume on each side of the book

## Part 1: Basic Order Structure

Let's start by defining a structure for an individual order:

```rust
use std::cmp::Ordering;

/// Order side
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side {
    Bid,  // Buy
    Ask,  // Sell
}

/// Individual order in the order book
#[derive(Debug, Clone)]
pub struct Order {
    pub id: u64,
    pub price: f64,
    pub quantity: f64,
    pub side: Side,
    pub timestamp: u64,
}

impl Order {
    pub fn new(id: u64, price: f64, quantity: f64, side: Side, timestamp: u64) -> Self {
        Order {
            id,
            price,
            quantity,
            side,
            timestamp,
        }
    }

    /// Total order value
    pub fn total_value(&self) -> f64 {
        self.price * self.quantity
    }
}

fn main() {
    let bid = Order::new(1, 42100.0, 1.5, Side::Bid, 1000);
    let ask = Order::new(2, 42150.0, 2.0, Side::Ask, 1001);

    println!("Bid: {:?}", bid);
    println!("Ask: {:?}", ask);
    println!("Bid value: ${:.2}", bid.total_value());
}
```

## Part 2: Price Level

Group orders by price levels:

```rust
use std::collections::VecDeque;

/// Price level — all orders at one price
#[derive(Debug, Clone)]
pub struct PriceLevel {
    pub price: f64,
    pub orders: VecDeque<Order>,
}

impl PriceLevel {
    pub fn new(price: f64) -> Self {
        PriceLevel {
            price,
            orders: VecDeque::new(),
        }
    }

    /// Add order to the level
    pub fn add_order(&mut self, order: Order) {
        self.orders.push_back(order);
    }

    /// Total quantity at this level
    pub fn total_quantity(&self) -> f64 {
        self.orders.iter().map(|o| o.quantity).sum()
    }

    /// Number of orders at this level
    pub fn order_count(&self) -> usize {
        self.orders.len()
    }

    /// Remove order by ID
    pub fn remove_order(&mut self, order_id: u64) -> Option<Order> {
        if let Some(pos) = self.orders.iter().position(|o| o.id == order_id) {
            self.orders.remove(pos)
        } else {
            None
        }
    }
}

fn main() {
    let mut level = PriceLevel::new(42100.0);

    level.add_order(Order::new(1, 42100.0, 1.5, Side::Bid, 1000));
    level.add_order(Order::new(2, 42100.0, 2.0, Side::Bid, 1001));
    level.add_order(Order::new(3, 42100.0, 0.8, Side::Bid, 1002));

    println!("Price level: ${}", level.price);
    println!("Total quantity: {} BTC", level.total_quantity());
    println!("Order count: {}", level.order_count());
}
```

## Part 3: Complete Order Book Implementation

```rust
use std::collections::{BTreeMap, VecDeque};

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side {
    Bid,
    Ask,
}

#[derive(Debug, Clone)]
pub struct Order {
    pub id: u64,
    pub price: f64,
    pub quantity: f64,
    pub side: Side,
    pub timestamp: u64,
}

impl Order {
    pub fn new(id: u64, price: f64, quantity: f64, side: Side, timestamp: u64) -> Self {
        Order { id, price, quantity, side, timestamp }
    }
}

#[derive(Debug, Clone)]
pub struct PriceLevel {
    pub price: f64,
    pub orders: VecDeque<Order>,
}

impl PriceLevel {
    pub fn new(price: f64) -> Self {
        PriceLevel { price, orders: VecDeque::new() }
    }

    pub fn add_order(&mut self, order: Order) {
        self.orders.push_back(order);
    }

    pub fn total_quantity(&self) -> f64 {
        self.orders.iter().map(|o| o.quantity).sum()
    }

    pub fn is_empty(&self) -> bool {
        self.orders.is_empty()
    }
}

/// Complete order book
#[derive(Debug)]
pub struct OrderBook {
    pub symbol: String,
    // Bids: key = price * -1 for reverse sorting
    bids: BTreeMap<i64, PriceLevel>,
    // Asks: key = price (normal sorting)
    asks: BTreeMap<i64, PriceLevel>,
    next_order_id: u64,
    price_precision: u32,
}

impl OrderBook {
    pub fn new(symbol: &str, price_precision: u32) -> Self {
        OrderBook {
            symbol: symbol.to_string(),
            bids: BTreeMap::new(),
            asks: BTreeMap::new(),
            next_order_id: 1,
            price_precision,
        }
    }

    /// Convert price to integer key
    fn price_to_key(&self, price: f64) -> i64 {
        (price * 10_f64.powi(self.price_precision as i32)) as i64
    }

    /// Add order to the book
    pub fn add_order(&mut self, price: f64, quantity: f64, side: Side) -> u64 {
        let order_id = self.next_order_id;
        self.next_order_id += 1;

        let timestamp = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_millis() as u64;

        let order = Order::new(order_id, price, quantity, side, timestamp);

        match side {
            Side::Bid => {
                let key = -self.price_to_key(price); // Negative for reverse sorting
                self.bids
                    .entry(key)
                    .or_insert_with(|| PriceLevel::new(price))
                    .add_order(order);
            }
            Side::Ask => {
                let key = self.price_to_key(price);
                self.asks
                    .entry(key)
                    .or_insert_with(|| PriceLevel::new(price))
                    .add_order(order);
            }
        }

        order_id
    }

    /// Best bid (highest buy price)
    pub fn best_bid(&self) -> Option<(f64, f64)> {
        self.bids.values().next().map(|level| {
            (level.price, level.total_quantity())
        })
    }

    /// Best ask (lowest sell price)
    pub fn best_ask(&self) -> Option<(f64, f64)> {
        self.asks.values().next().map(|level| {
            (level.price, level.total_quantity())
        })
    }

    /// Spread (difference between best ask and bid)
    pub fn spread(&self) -> Option<f64> {
        match (self.best_bid(), self.best_ask()) {
            (Some((bid, _)), Some((ask, _))) => Some(ask - bid),
            _ => None,
        }
    }

    /// Percentage spread
    pub fn spread_percent(&self) -> Option<f64> {
        match (self.best_bid(), self.best_ask()) {
            (Some((bid, _)), Some((ask, _))) => {
                let mid_price = (bid + ask) / 2.0;
                Some((ask - bid) / mid_price * 100.0)
            }
            _ => None,
        }
    }

    /// Mid price
    pub fn mid_price(&self) -> Option<f64> {
        match (self.best_bid(), self.best_ask()) {
            (Some((bid, _)), Some((ask, _))) => Some((bid + ask) / 2.0),
            _ => None,
        }
    }

    /// Get N best bid levels
    pub fn top_bids(&self, n: usize) -> Vec<(f64, f64)> {
        self.bids
            .values()
            .take(n)
            .map(|level| (level.price, level.total_quantity()))
            .collect()
    }

    /// Get N best ask levels
    pub fn top_asks(&self, n: usize) -> Vec<(f64, f64)> {
        self.asks
            .values()
            .take(n)
            .map(|level| (level.price, level.total_quantity()))
            .collect()
    }

    /// Total bid side volume
    pub fn total_bid_volume(&self) -> f64 {
        self.bids.values().map(|l| l.total_quantity()).sum()
    }

    /// Total ask side volume
    pub fn total_ask_volume(&self) -> f64 {
        self.asks.values().map(|l| l.total_quantity()).sum()
    }

    /// Order imbalance (bid to ask ratio)
    pub fn order_imbalance(&self) -> f64 {
        let bid_vol = self.total_bid_volume();
        let ask_vol = self.total_ask_volume();

        if ask_vol == 0.0 {
            return if bid_vol > 0.0 { 1.0 } else { 0.0 };
        }

        bid_vol / (bid_vol + ask_vol)
    }

    /// Pretty print the order book
    pub fn display(&self, depth: usize) {
        println!("\n╔══════════════════════════════════════════╗");
        println!("║         ORDER BOOK: {}               ║", self.symbol);
        println!("╠══════════════════════════════════════════╣");

        // Asks (in reverse order for display)
        let asks: Vec<_> = self.top_asks(depth);
        println!("║              ASKS (Sell)                 ║");
        println!("║──────────────────────────────────────────║");
        for (price, qty) in asks.iter().rev() {
            println!("║   {:>10.2}  │  {:>10.4}               ║", price, qty);
        }

        // Spread
        println!("║══════════════════════════════════════════║");
        if let Some(spread) = self.spread() {
            println!("║   SPREAD: ${:.2} ({:.4}%)                 ║",
                spread, self.spread_percent().unwrap_or(0.0));
        }
        println!("║══════════════════════════════════════════║");

        // Bids
        println!("║              BIDS (Buy)                  ║");
        println!("║──────────────────────────────────────────║");
        for (price, qty) in self.top_bids(depth) {
            println!("║   {:>10.2}  │  {:>10.4}               ║", price, qty);
        }

        println!("╚══════════════════════════════════════════╝");

        // Statistics
        println!("\n📊 Statistics:");
        println!("   Mid Price: ${:.2}", self.mid_price().unwrap_or(0.0));
        println!("   Bid Volume: {:.4}", self.total_bid_volume());
        println!("   Ask Volume: {:.4}", self.total_ask_volume());
        println!("   Imbalance: {:.2}% (bids)", self.order_imbalance() * 100.0);
    }
}

fn main() {
    let mut book = OrderBook::new("BTC/USDT", 2);

    // Add buy orders (bids)
    book.add_order(42100.0, 1.5, Side::Bid);
    book.add_order(42050.0, 2.0, Side::Bid);
    book.add_order(42000.0, 3.5, Side::Bid);
    book.add_order(41950.0, 1.8, Side::Bid);
    book.add_order(41900.0, 4.2, Side::Bid);

    // Add sell orders (asks)
    book.add_order(42150.0, 1.2, Side::Ask);
    book.add_order(42200.0, 2.5, Side::Ask);
    book.add_order(42250.0, 1.8, Side::Ask);
    book.add_order(42300.0, 3.0, Side::Ask);
    book.add_order(42400.0, 2.2, Side::Ask);

    // Display the order book
    book.display(5);
}
```

## Part 4: Market Order Execution

Let's add functionality to execute market orders:

```rust
/// Order execution result
#[derive(Debug)]
pub struct ExecutionResult {
    pub filled_quantity: f64,
    pub average_price: f64,
    pub fills: Vec<Fill>,
}

#[derive(Debug)]
pub struct Fill {
    pub price: f64,
    pub quantity: f64,
    pub order_id: u64,
}

impl OrderBook {
    /// Execute market buy order
    pub fn execute_market_buy(&mut self, quantity: f64) -> ExecutionResult {
        let mut remaining = quantity;
        let mut fills = Vec::new();
        let mut total_cost = 0.0;
        let mut keys_to_remove = Vec::new();

        for (&key, level) in self.asks.iter_mut() {
            if remaining <= 0.0 {
                break;
            }

            while let Some(mut order) = level.orders.pop_front() {
                if remaining <= 0.0 {
                    level.orders.push_front(order);
                    break;
                }

                let fill_qty = remaining.min(order.quantity);
                fills.push(Fill {
                    price: order.price,
                    quantity: fill_qty,
                    order_id: order.id,
                });

                total_cost += fill_qty * order.price;
                remaining -= fill_qty;
                order.quantity -= fill_qty;

                if order.quantity > 0.0 {
                    level.orders.push_front(order);
                    break;
                }
            }

            if level.is_empty() {
                keys_to_remove.push(key);
            }
        }

        // Remove empty levels
        for key in keys_to_remove {
            self.asks.remove(&key);
        }

        let filled_quantity = quantity - remaining;
        let average_price = if filled_quantity > 0.0 {
            total_cost / filled_quantity
        } else {
            0.0
        };

        ExecutionResult {
            filled_quantity,
            average_price,
            fills,
        }
    }

    /// Execute market sell order
    pub fn execute_market_sell(&mut self, quantity: f64) -> ExecutionResult {
        let mut remaining = quantity;
        let mut fills = Vec::new();
        let mut total_revenue = 0.0;
        let mut keys_to_remove = Vec::new();

        for (&key, level) in self.bids.iter_mut() {
            if remaining <= 0.0 {
                break;
            }

            while let Some(mut order) = level.orders.pop_front() {
                if remaining <= 0.0 {
                    level.orders.push_front(order);
                    break;
                }

                let fill_qty = remaining.min(order.quantity);
                fills.push(Fill {
                    price: order.price,
                    quantity: fill_qty,
                    order_id: order.id,
                });

                total_revenue += fill_qty * order.price;
                remaining -= fill_qty;
                order.quantity -= fill_qty;

                if order.quantity > 0.0 {
                    level.orders.push_front(order);
                    break;
                }
            }

            if level.is_empty() {
                keys_to_remove.push(key);
            }
        }

        for key in keys_to_remove {
            self.bids.remove(&key);
        }

        let filled_quantity = quantity - remaining;
        let average_price = if filled_quantity > 0.0 {
            total_revenue / filled_quantity
        } else {
            0.0
        };

        ExecutionResult {
            filled_quantity,
            average_price,
            fills,
        }
    }
}
```

## Part 5: Liquidity Analysis

```rust
impl OrderBook {
    /// Calculate price for buying a specific quantity
    pub fn price_for_buy(&self, quantity: f64) -> Option<f64> {
        let mut remaining = quantity;
        let mut total_cost = 0.0;

        for level in self.asks.values() {
            if remaining <= 0.0 {
                break;
            }

            let available = level.total_quantity();
            let fill_qty = remaining.min(available);
            total_cost += fill_qty * level.price;
            remaining -= fill_qty;
        }

        if remaining > 0.0 {
            None // Insufficient liquidity
        } else {
            Some(total_cost / quantity)
        }
    }

    /// Calculate price for selling a specific quantity
    pub fn price_for_sell(&self, quantity: f64) -> Option<f64> {
        let mut remaining = quantity;
        let mut total_revenue = 0.0;

        for level in self.bids.values() {
            if remaining <= 0.0 {
                break;
            }

            let available = level.total_quantity();
            let fill_qty = remaining.min(available);
            total_revenue += fill_qty * level.price;
            remaining -= fill_qty;
        }

        if remaining > 0.0 {
            None
        } else {
            Some(total_revenue / quantity)
        }
    }

    /// Calculate slippage for buy
    pub fn buy_slippage(&self, quantity: f64) -> Option<f64> {
        let exec_price = self.price_for_buy(quantity)?;
        let best_ask = self.best_ask()?.0;
        Some((exec_price - best_ask) / best_ask * 100.0)
    }

    /// Calculate slippage for sell
    pub fn sell_slippage(&self, quantity: f64) -> Option<f64> {
        let exec_price = self.price_for_sell(quantity)?;
        let best_bid = self.best_bid()?.0;
        Some((best_bid - exec_price) / best_bid * 100.0)
    }

    /// Market depth to a specific price
    pub fn depth_to_price(&self, side: Side, target_price: f64) -> f64 {
        match side {
            Side::Bid => {
                self.bids
                    .values()
                    .filter(|l| l.price >= target_price)
                    .map(|l| l.total_quantity())
                    .sum()
            }
            Side::Ask => {
                self.asks
                    .values()
                    .filter(|l| l.price <= target_price)
                    .map(|l| l.total_quantity())
                    .sum()
            }
        }
    }
}

fn main() {
    let mut book = OrderBook::new("BTC/USDT", 2);

    // Fill the book
    for i in 0..10 {
        book.add_order(42000.0 - (i as f64 * 50.0), 1.0 + (i as f64 * 0.5), Side::Bid);
        book.add_order(42100.0 + (i as f64 * 50.0), 1.0 + (i as f64 * 0.5), Side::Ask);
    }

    book.display(5);

    // Liquidity analysis
    println!("\n📈 Liquidity Analysis:");

    for qty in [1.0, 5.0, 10.0, 20.0] {
        if let Some(price) = book.price_for_buy(qty) {
            let slippage = book.buy_slippage(qty).unwrap_or(0.0);
            println!("   Buy {} BTC @ ${:.2} (slippage: {:.4}%)", qty, price, slippage);
        } else {
            println!("   Buy {} BTC: Insufficient liquidity!", qty);
        }
    }
}
```

## Exercises

### Exercise 1: Order Cancellation

Implement a `cancel_order` method that removes an order by its ID:

```rust
impl OrderBook {
    pub fn cancel_order(&mut self, order_id: u64) -> Option<Order> {
        // Your code here
        // Hint: need to search both sides (bids and asks)
        todo!()
    }
}
```

### Exercise 2: Order Modification

Implement a method to change the quantity of an order:

```rust
impl OrderBook {
    pub fn modify_order(&mut self, order_id: u64, new_quantity: f64) -> bool {
        // Your code here
        // Hint: find the order and modify its quantity
        todo!()
    }
}
```

### Exercise 3: Aggregated Order Book

Create a method that returns aggregated order book data:

```rust
#[derive(Debug)]
pub struct AggregatedLevel {
    pub price: f64,
    pub quantity: f64,
    pub order_count: usize,
    pub cumulative_quantity: f64,  // Cumulative volume
}

impl OrderBook {
    pub fn aggregated_book(&self, depth: usize) -> (Vec<AggregatedLevel>, Vec<AggregatedLevel>) {
        // Returns (bids, asks) with cumulative volume
        todo!()
    }
}
```

### Exercise 4: VWAP Calculator

Calculate VWAP (Volume Weighted Average Price) for a given quantity:

```rust
impl OrderBook {
    pub fn vwap_buy(&self, quantity: f64) -> Option<f64> {
        // Volume weighted average price for buying quantity
        todo!()
    }

    pub fn vwap_sell(&self, quantity: f64) -> Option<f64> {
        // Volume weighted average price for selling quantity
        todo!()
    }
}
```

## Homework

### 1. Add Limit Order Support with Matching

Implement logic where a new order automatically matches against the opposite side if prices cross:

```rust
pub fn add_limit_order(&mut self, price: f64, quantity: f64, side: Side)
    -> (u64, Vec<Fill>) {
    // If buy order with price >= best ask — partially or fully execute
    // If sell order with price <= best bid — partially or fully execute
    // Add remainder to the book
}
```

### 2. Order Book Snapshots and Delta Updates

Implement a system for creating snapshots and applying delta updates:

```rust
#[derive(Debug, Clone)]
pub struct OrderBookSnapshot {
    pub symbol: String,
    pub bids: Vec<(f64, f64)>,
    pub asks: Vec<(f64, f64)>,
    pub timestamp: u64,
}

#[derive(Debug)]
pub enum DeltaUpdate {
    Add { side: Side, price: f64, quantity: f64 },
    Remove { side: Side, price: f64 },
    Update { side: Side, price: f64, quantity: f64 },
}

impl OrderBook {
    pub fn snapshot(&self, depth: usize) -> OrderBookSnapshot { todo!() }
    pub fn apply_delta(&mut self, delta: DeltaUpdate) { todo!() }
}
```

### 3. Market Depth Visualization

Create an ASCII visualization of market depth:

```
DEPTH CHART: BTC/USDT
    ASK ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░  42400: 15.5 BTC
        ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░  42300: 10.2 BTC
        ▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░  42200:  6.5 BTC
    ────────────────────────────  SPREAD: $100
    BID ▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░  42100:  8.0 BTC
        ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░  42000: 12.3 BTC
        ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░  41900: 18.7 BTC
```

### 4. Market Microstructure Analysis

Implement market microstructure metrics calculation:

```rust
pub struct MicrostructureMetrics {
    pub bid_ask_spread: f64,
    pub spread_bps: f64,        // Spread in basis points
    pub order_imbalance: f64,
    pub depth_imbalance: f64,   // Imbalance at first N levels
    pub weighted_mid_price: f64, // Weighted mid price
}

impl OrderBook {
    pub fn microstructure_metrics(&self, depth: usize) -> MicrostructureMetrics {
        todo!()
    }
}
```

## What We Learned

| Concept | Application |
|---------|-------------|
| `struct` | Defining Order, PriceLevel, OrderBook |
| `enum` | Side (Bid/Ask), operation results |
| `BTreeMap` | Sorted storage of price levels |
| `VecDeque` | FIFO queue of orders |
| `impl` | Methods for structs |
| Iterators | Processing collections |
| `Option` | Safe handling of missing data |

## Key Trading Concepts

- **Order Book** — the central data structure of an exchange
- **Spread** — indicator of market liquidity
- **Slippage** — actual cost of executing large orders
- **Order Imbalance** — short-term price movement predictor
- **Depth** — price resistance to large orders

## Navigation

[← Previous day](../089-btreemap-sorted-prices/en.md) | [Next day →](../091-vectors-dynamic-orders/en.md)
