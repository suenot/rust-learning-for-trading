# Day 265: Take-Profit: Locking Profits

## Trading Analogy

Imagine you bought Bitcoin at $40,000 and it started rising. The price reached $44,000 — you have a 10% profit! But what if you don't lock in the profit and the price reverses back to $40,000? All your paper profit evaporates.

**Take-Profit (TP)** is an automatic order to close a position when a target price is reached. It's like insurance against greed: you decide in advance at what profit level you'll exit the trade, and the system does it automatically.

In real trading, take-profit helps:
- Lock in profits without emotions
- Protect against price reversals
- Discipline your trading strategy
- Free up capital for new trades

## Basic Take-Profit Order Structure

```rust
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum OrderSide {
    Buy,
    Sell,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum OrderStatus {
    Pending,
    Triggered,
    Filled,
    Cancelled,
}

#[derive(Debug, Clone)]
pub struct TakeProfitOrder {
    pub id: u64,
    pub symbol: String,
    pub side: OrderSide,           // Direction to close (opposite of position)
    pub quantity: f64,             // Quantity to close
    pub trigger_price: f64,        // Take-profit activation price
    pub status: OrderStatus,
    pub created_at: u64,           // Creation timestamp
    pub triggered_at: Option<u64>, // Trigger timestamp
}

impl TakeProfitOrder {
    pub fn new(
        id: u64,
        symbol: &str,
        side: OrderSide,
        quantity: f64,
        trigger_price: f64,
        created_at: u64,
    ) -> Self {
        TakeProfitOrder {
            id,
            symbol: symbol.to_string(),
            side,
            quantity,
            trigger_price,
            status: OrderStatus::Pending,
            created_at,
            triggered_at: None,
        }
    }

    /// Check if take-profit should trigger at current price
    pub fn should_trigger(&self, current_price: f64) -> bool {
        if self.status != OrderStatus::Pending {
            return false;
        }

        match self.side {
            // For long position: sell when price >= trigger_price
            OrderSide::Sell => current_price >= self.trigger_price,
            // For short position: buy when price <= trigger_price
            OrderSide::Buy => current_price <= self.trigger_price,
        }
    }

    /// Activate the take-profit
    pub fn trigger(&mut self, timestamp: u64) {
        self.status = OrderStatus::Triggered;
        self.triggered_at = Some(timestamp);
    }

    /// Mark as filled
    pub fn fill(&mut self) {
        self.status = OrderStatus::Filled;
    }
}

impl fmt::Display for TakeProfitOrder {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "TP#{} {} {} {} @ {} [{:?}]",
            self.id,
            self.symbol,
            match self.side {
                OrderSide::Buy => "BUY",
                OrderSide::Sell => "SELL",
            },
            self.quantity,
            self.trigger_price,
            self.status
        )
    }
}

fn main() {
    // Create take-profit for a long BTC position
    // Bought at $40,000, want to lock profit at $44,000
    let mut tp_order = TakeProfitOrder::new(
        1,
        "BTC/USDT",
        OrderSide::Sell,
        0.5,
        44000.0,
        1000,
    );

    println!("Created order: {}", tp_order);

    // Price movement simulation
    let prices = [41000.0, 42500.0, 43800.0, 44200.0, 44500.0];

    for (i, &price) in prices.iter().enumerate() {
        println!("\nTick {}: Price = ${}", i + 1, price);

        if tp_order.should_trigger(price) {
            tp_order.trigger(1000 + i as u64);
            println!(">>> Take-Profit triggered! {}", tp_order);
            tp_order.fill();
            println!(">>> Order filled: {}", tp_order);
            break;
        } else {
            println!("   Take-Profit waiting (need >= ${})", tp_order.trigger_price);
        }
    }
}
```

## Calculating Take-Profit Price

```rust
#[derive(Debug, Clone, Copy)]
pub enum TakeProfitStrategy {
    /// Fixed percentage gain
    PercentageGain(f64),
    /// Fixed points/pips
    FixedPoints(f64),
    /// Risk/reward ratio (relative to stop-loss)
    RiskRewardRatio { stop_loss: f64, ratio: f64 },
    /// Resistance/support level
    PriceLevel(f64),
}

pub struct TakeProfitCalculator;

impl TakeProfitCalculator {
    /// Calculate take-profit price for a long position
    pub fn calculate_long(
        entry_price: f64,
        strategy: TakeProfitStrategy,
    ) -> f64 {
        match strategy {
            TakeProfitStrategy::PercentageGain(percent) => {
                entry_price * (1.0 + percent / 100.0)
            }
            TakeProfitStrategy::FixedPoints(points) => {
                entry_price + points
            }
            TakeProfitStrategy::RiskRewardRatio { stop_loss, ratio } => {
                let risk = entry_price - stop_loss;
                entry_price + (risk * ratio)
            }
            TakeProfitStrategy::PriceLevel(level) => level,
        }
    }

    /// Calculate take-profit price for a short position
    pub fn calculate_short(
        entry_price: f64,
        strategy: TakeProfitStrategy,
    ) -> f64 {
        match strategy {
            TakeProfitStrategy::PercentageGain(percent) => {
                entry_price * (1.0 - percent / 100.0)
            }
            TakeProfitStrategy::FixedPoints(points) => {
                entry_price - points
            }
            TakeProfitStrategy::RiskRewardRatio { stop_loss, ratio } => {
                let risk = stop_loss - entry_price;
                entry_price - (risk * ratio)
            }
            TakeProfitStrategy::PriceLevel(level) => level,
        }
    }
}

fn main() {
    let entry_price = 40000.0;
    let stop_loss = 38000.0;

    println!("=== Take-Profit Calculation for Long Position ===");
    println!("Entry price: ${}", entry_price);
    println!("Stop-Loss: ${}", stop_loss);
    println!();

    // Different strategies
    let strategies = [
        ("5% profit", TakeProfitStrategy::PercentageGain(5.0)),
        ("10% profit", TakeProfitStrategy::PercentageGain(10.0)),
        ("+$3000", TakeProfitStrategy::FixedPoints(3000.0)),
        ("R:R 1:2", TakeProfitStrategy::RiskRewardRatio {
            stop_loss: 38000.0,
            ratio: 2.0,
        }),
        ("R:R 1:3", TakeProfitStrategy::RiskRewardRatio {
            stop_loss: 38000.0,
            ratio: 3.0,
        }),
        ("Level $45000", TakeProfitStrategy::PriceLevel(45000.0)),
    ];

    for (name, strategy) in strategies {
        let tp_price = TakeProfitCalculator::calculate_long(entry_price, strategy);
        let profit_percent = ((tp_price - entry_price) / entry_price) * 100.0;
        println!(
            "{}: TP = ${:.2} (+{:.2}%)",
            name, tp_price, profit_percent
        );
    }
}
```

## Multiple Take-Profit Levels

Professional traders often use multiple take-profit levels for gradual profit-taking:

```rust
use std::collections::BTreeMap;

#[derive(Debug, Clone)]
pub struct MultiLevelTakeProfit {
    pub symbol: String,
    pub total_quantity: f64,
    /// Levels: price -> percentage of position
    pub levels: BTreeMap<u64, TakeProfitLevel>,
    pub filled_quantity: f64,
}

#[derive(Debug, Clone)]
pub struct TakeProfitLevel {
    pub price: f64,
    pub percentage: f64,  // Percentage of total position
    pub quantity: f64,    // Calculated quantity
    pub is_filled: bool,
}

impl MultiLevelTakeProfit {
    pub fn new(symbol: &str, total_quantity: f64) -> Self {
        MultiLevelTakeProfit {
            symbol: symbol.to_string(),
            total_quantity,
            levels: BTreeMap::new(),
            filled_quantity: 0.0,
        }
    }

    /// Add a take-profit level
    pub fn add_level(&mut self, price: f64, percentage: f64) -> Result<(), String> {
        // Check that total percentage doesn't exceed 100%
        let current_total: f64 = self.levels.values()
            .map(|l| l.percentage)
            .sum();

        if current_total + percentage > 100.0 {
            return Err(format!(
                "Exceeds 100%: current {:.1}% + new {:.1}% = {:.1}%",
                current_total, percentage, current_total + percentage
            ));
        }

        let quantity = self.total_quantity * (percentage / 100.0);
        let price_key = (price * 100.0) as u64; // For sorting

        self.levels.insert(price_key, TakeProfitLevel {
            price,
            percentage,
            quantity,
            is_filled: false,
        });

        Ok(())
    }

    /// Check and fill levels at current price
    pub fn check_and_fill(&mut self, current_price: f64) -> Vec<(f64, f64)> {
        let mut filled = Vec::new();

        for level in self.levels.values_mut() {
            if !level.is_filled && current_price >= level.price {
                level.is_filled = true;
                self.filled_quantity += level.quantity;
                filled.push((level.price, level.quantity));
            }
        }

        filled
    }

    /// Get remaining quantity
    pub fn remaining_quantity(&self) -> f64 {
        self.total_quantity - self.filled_quantity
    }

    /// Print status
    pub fn print_status(&self) {
        println!("\n=== Take-Profit Status for {} ===", self.symbol);
        println!("Total quantity: {}", self.total_quantity);
        println!("Filled: {}", self.filled_quantity);
        println!("Remaining: {}", self.remaining_quantity());
        println!("\nLevels:");

        for level in self.levels.values() {
            let status = if level.is_filled { "[FILLED]" } else { "[PENDING]" };
            println!(
                "  ${:.2}: {:.1}% ({} units) {}",
                level.price, level.percentage, level.quantity, status
            );
        }
    }
}

fn main() {
    // Create multi-level take-profit
    let mut tp = MultiLevelTakeProfit::new("BTC/USDT", 1.0);

    // Add levels: gradual profit-taking
    tp.add_level(42000.0, 25.0).unwrap();  // 25% at $42,000
    tp.add_level(44000.0, 35.0).unwrap();  // 35% at $44,000
    tp.add_level(48000.0, 40.0).unwrap();  // 40% at $48,000

    tp.print_status();

    // Price movement simulation
    let prices = [41000.0, 42500.0, 43000.0, 44100.0, 46000.0, 48500.0];

    for price in prices {
        println!("\n--- Price: ${} ---", price);
        let filled = tp.check_and_fill(price);

        for (tp_price, qty) in filled {
            println!(
                ">>> TP triggered at ${}: sold {} BTC",
                tp_price, qty
            );
        }
    }

    tp.print_status();
}
```

## Trailing Take-Profit

Trailing take-profit allows you to "trail" the profit-taking level behind the price:

```rust
#[derive(Debug, Clone)]
pub struct TrailingTakeProfit {
    pub symbol: String,
    pub quantity: f64,
    pub activation_price: f64,    // Price to activate trailing
    pub trailing_percent: f64,    // Trailing offset in percent
    pub highest_price: f64,       // Maximum price reached
    pub current_tp_price: f64,    // Current TP level
    pub is_activated: bool,       // Is trailing activated
    pub is_filled: bool,
}

impl TrailingTakeProfit {
    pub fn new(
        symbol: &str,
        quantity: f64,
        activation_price: f64,
        trailing_percent: f64,
    ) -> Self {
        TrailingTakeProfit {
            symbol: symbol.to_string(),
            quantity,
            activation_price,
            trailing_percent,
            highest_price: 0.0,
            current_tp_price: 0.0,
            is_activated: false,
            is_filled: false,
        }
    }

    /// Update on new price
    pub fn update(&mut self, current_price: f64) -> Option<f64> {
        if self.is_filled {
            return None;
        }

        // Check activation
        if !self.is_activated && current_price >= self.activation_price {
            self.is_activated = true;
            self.highest_price = current_price;
            self.current_tp_price = current_price * (1.0 - self.trailing_percent / 100.0);
            println!(
                "[Trailing TP] Activated at ${:.2}, initial TP: ${:.2}",
                current_price, self.current_tp_price
            );
        }

        if !self.is_activated {
            return None;
        }

        // Update highest price
        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.current_tp_price = current_price * (1.0 - self.trailing_percent / 100.0);
            println!(
                "[Trailing TP] New high ${:.2}, TP moved to ${:.2}",
                current_price, self.current_tp_price
            );
        }

        // Check trigger
        if current_price <= self.current_tp_price {
            self.is_filled = true;
            println!(
                "[Trailing TP] FILLED at ${:.2}! Profit locked",
                current_price
            );
            return Some(self.quantity);
        }

        None
    }

    pub fn status(&self) -> String {
        if self.is_filled {
            format!("FILLED at ${:.2}", self.current_tp_price)
        } else if self.is_activated {
            format!(
                "Active: high ${:.2}, TP ${:.2}",
                self.highest_price, self.current_tp_price
            )
        } else {
            format!("Awaiting activation at ${:.2}", self.activation_price)
        }
    }
}

fn main() {
    // Create trailing take-profit
    // Activates at $42,000, 3% trailing offset
    let mut trailing_tp = TrailingTakeProfit::new(
        "BTC/USDT",
        0.5,
        42000.0,
        3.0,
    );

    println!("=== Trailing Take-Profit ===");
    println!("Activation: $42,000");
    println!("Trailing: 3%");
    println!();

    // Simulation: price rises, then falls
    let prices = [
        40000.0, 41000.0, 42000.0, 43000.0, 44500.0,
        46000.0, 47000.0, 46500.0, 45000.0, 44000.0,
    ];

    for (i, &price) in prices.iter().enumerate() {
        println!("\nTick {}: Price = ${}", i + 1, price);
        trailing_tp.update(price);
        println!("Status: {}", trailing_tp.status());

        if trailing_tp.is_filled {
            break;
        }
    }

    // Calculate final profit
    if trailing_tp.is_filled {
        let entry = 40000.0;
        let exit = trailing_tp.current_tp_price;
        let profit = exit - entry;
        let profit_percent = (profit / entry) * 100.0;
        println!("\n=== Summary ===");
        println!("Entry: ${}", entry);
        println!("Exit: ${:.2}", exit);
        println!("Profit: ${:.2} ({:.2}%)", profit, profit_percent);
    }
}
```

## Integrating Take-Profit into a Trading System

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct Position {
    pub id: u64,
    pub symbol: String,
    pub side: OrderSide,
    pub quantity: f64,
    pub entry_price: f64,
    pub take_profit: Option<f64>,
    pub stop_loss: Option<f64>,
}

#[derive(Debug)]
pub struct TradingSystem {
    positions: HashMap<u64, Position>,
    tp_orders: HashMap<u64, TakeProfitOrder>,
    next_position_id: u64,
    next_order_id: u64,
    current_timestamp: u64,
}

impl TradingSystem {
    pub fn new() -> Self {
        TradingSystem {
            positions: HashMap::new(),
            tp_orders: HashMap::new(),
            next_position_id: 1,
            next_order_id: 1,
            current_timestamp: 0,
        }
    }

    /// Open a position with take-profit
    pub fn open_position(
        &mut self,
        symbol: &str,
        side: OrderSide,
        quantity: f64,
        entry_price: f64,
        take_profit_percent: Option<f64>,
    ) -> u64 {
        let position_id = self.next_position_id;
        self.next_position_id += 1;

        // Calculate take-profit
        let take_profit = take_profit_percent.map(|percent| {
            match side {
                OrderSide::Buy => entry_price * (1.0 + percent / 100.0),
                OrderSide::Sell => entry_price * (1.0 - percent / 100.0),
            }
        });

        let position = Position {
            id: position_id,
            symbol: symbol.to_string(),
            side,
            quantity,
            entry_price,
            take_profit,
            stop_loss: None,
        };

        println!(
            "[System] Opened position #{}: {} {} {} @ ${}",
            position_id,
            match side { OrderSide::Buy => "LONG", OrderSide::Sell => "SHORT" },
            quantity,
            symbol,
            entry_price
        );

        // Create take-profit order
        if let Some(tp_price) = take_profit {
            let tp_side = match side {
                OrderSide::Buy => OrderSide::Sell,
                OrderSide::Sell => OrderSide::Buy,
            };

            let tp_order = TakeProfitOrder::new(
                self.next_order_id,
                symbol,
                tp_side,
                quantity,
                tp_price,
                self.current_timestamp,
            );

            println!(
                "[System] Created Take-Profit #{} at ${}",
                self.next_order_id, tp_price
            );

            self.tp_orders.insert(self.next_order_id, tp_order);
            self.next_order_id += 1;
        }

        self.positions.insert(position_id, position);
        position_id
    }

    /// Process price tick
    pub fn on_price_update(&mut self, symbol: &str, price: f64) {
        self.current_timestamp += 1;

        // Check take-profit orders
        let mut triggered_orders = Vec::new();

        for (order_id, order) in &mut self.tp_orders {
            if order.symbol == symbol && order.should_trigger(price) {
                order.trigger(self.current_timestamp);
                order.fill();
                triggered_orders.push(*order_id);
                println!(
                    "[System] Take-Profit #{} filled at ${:.2}!",
                    order_id, price
                );
            }
        }

        // Close corresponding positions
        for order_id in triggered_orders {
            // In a real system, there would be order -> position mapping
            println!("[System] Profit locked");
        }
    }

    /// Show portfolio status
    pub fn print_portfolio(&self, current_prices: &HashMap<String, f64>) {
        println!("\n=== Portfolio ===");

        for position in self.positions.values() {
            let current_price = current_prices
                .get(&position.symbol)
                .copied()
                .unwrap_or(position.entry_price);

            let pnl = match position.side {
                OrderSide::Buy => (current_price - position.entry_price) * position.quantity,
                OrderSide::Sell => (position.entry_price - current_price) * position.quantity,
            };

            let pnl_percent = (pnl / (position.entry_price * position.quantity)) * 100.0;

            println!(
                "#{} {} {} {} @ ${} | Current: ${} | PnL: ${:.2} ({:.2}%)",
                position.id,
                match position.side { OrderSide::Buy => "LONG", OrderSide::Sell => "SHORT" },
                position.quantity,
                position.symbol,
                position.entry_price,
                current_price,
                pnl,
                pnl_percent
            );

            if let Some(tp) = position.take_profit {
                println!("   Take-Profit: ${}", tp);
            }
        }
    }
}

fn main() {
    let mut system = TradingSystem::new();

    // Open positions with take-profit
    system.open_position("BTC/USDT", OrderSide::Buy, 0.5, 40000.0, Some(10.0));
    system.open_position("ETH/USDT", OrderSide::Buy, 2.0, 2500.0, Some(15.0));

    println!("\n--- Market Simulation ---");

    // Price movement simulation
    system.on_price_update("BTC/USDT", 41000.0);
    system.on_price_update("BTC/USDT", 42500.0);
    system.on_price_update("BTC/USDT", 44500.0); // Take-Profit will trigger

    system.on_price_update("ETH/USDT", 2600.0);
    system.on_price_update("ETH/USDT", 2800.0);
    system.on_price_update("ETH/USDT", 2900.0); // TP not reached yet

    // Current prices
    let mut prices = HashMap::new();
    prices.insert("BTC/USDT".to_string(), 44500.0);
    prices.insert("ETH/USDT".to_string(), 2900.0);

    system.print_portfolio(&prices);
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Take-Profit (TP) | Automatic order to lock profits when target price is reached |
| Trigger Price | The price at which take-profit activates |
| Risk/Reward Ratio | Ratio of potential profit to risk (stop-loss) |
| Multi-level TP | Gradual profit-taking at multiple levels |
| Trailing TP | Dynamic take-profit that follows the price |

## Practical Exercises

1. **Basic Take-Profit**: Modify the `TakeProfitOrder` structure by adding a `limit_price` field — the minimum execution price after triggering (slippage protection).

2. **Dynamic Calculation**: Create a function that calculates optimal take-profit based on:
   - ATR (Average True Range) — asset volatility
   - Historical resistance levels
   - Specified risk/reward ratio

3. **Time-Decay TP**: Implement a take-profit that automatically lowers the target price if the position has been open too long (decay TP).

## Homework

1. **Partial Closing**: Extend `MultiLevelTakeProfit` to allow modifying percentages on levels after partial execution.

2. **Bracket Order**: Create a `BracketOrder` struct that contains:
   - Entry order (position entry)
   - Take-profit order
   - Stop-loss order

   Implement logic where triggering TP automatically cancels SL and vice versa.

3. **TP Strategy Backtesting**: Write a program that:
   - Loads historical prices (can hardcode an array)
   - Tests different take-profit strategies
   - Outputs statistics: win rate, average profit, maximum drawdown

4. **Stepped Trailing**: Implement a trailing take-profit that moves in steps rather than smoothly. For example, after every +5% price increase, the take-profit moves up by 4%.

## Navigation

[← Previous day](../264-stop-loss-protecting-capital/en.md) | [Next day →](../266-bracket-orders-entry-with-exits/en.md)
