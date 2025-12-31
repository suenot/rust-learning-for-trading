# Day 266: Trailing Stop — Dynamic Stop Loss

## Trading Analogy

Imagine you're a mountain climber ascending a peak. You have a safety rope that automatically tightens upward as you climb, but never loosens downward. If you slip, the rope catches you at the last height you reached.

**Trailing Stop** works exactly the same way in trading:
- When the price moves in your favor, the stop-loss automatically rises with it
- When the price falls, the stop stays in place
- If the price reaches the stop — the position is automatically closed

This allows you to **protect profits** and **limit losses** simultaneously, without requiring constant market monitoring.

## What is a Trailing Stop?

A Trailing Stop is a dynamic stop-loss that follows the price at a set distance. There are two types:

1. **Percentage-based** — stop at X% below the highest price
2. **Fixed amount** — stop at X points below the highest price

### Example

```
Buy BTC at $40,000 with 5% trailing stop

Price moves:
$40,000 → stop = $38,000 (5% below)
$42,000 → stop = $39,900 (stop moved up!)
$45,000 → stop = $42,750 (stop moved up again!)
$43,000 → stop = $42,750 (price fell, stop stayed)
$42,750 → TRIGGERED! Sold at $42,750
```

## Basic Trailing Stop Structure

```rust
#[derive(Debug, Clone)]
pub struct TrailingStop {
    /// Entry price
    entry_price: f64,
    /// Trail percentage from high (e.g., 0.05 = 5%)
    trail_percent: f64,
    /// Highest price reached
    highest_price: f64,
    /// Current stop level
    stop_price: f64,
    /// Position direction (Long/Short)
    direction: PositionDirection,
    /// Whether the stop has been triggered
    triggered: bool,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum PositionDirection {
    Long,  // Buy — protect against falling prices
    Short, // Sell — protect against rising prices
}

impl TrailingStop {
    /// Creates a new trailing stop for a long position
    pub fn new_long(entry_price: f64, trail_percent: f64) -> Self {
        let stop_price = entry_price * (1.0 - trail_percent);
        TrailingStop {
            entry_price,
            trail_percent,
            highest_price: entry_price,
            stop_price,
            direction: PositionDirection::Long,
            triggered: false,
        }
    }

    /// Creates a new trailing stop for a short position
    pub fn new_short(entry_price: f64, trail_percent: f64) -> Self {
        let stop_price = entry_price * (1.0 + trail_percent);
        TrailingStop {
            entry_price,
            trail_percent,
            highest_price: entry_price, // For short, this is lowest_price
            stop_price,
            direction: PositionDirection::Short,
            triggered: false,
        }
    }

    /// Updates the stop based on new price
    pub fn update(&mut self, current_price: f64) -> bool {
        if self.triggered {
            return true; // Already triggered
        }

        match self.direction {
            PositionDirection::Long => {
                // For long positions, track the maximum
                if current_price > self.highest_price {
                    self.highest_price = current_price;
                    self.stop_price = current_price * (1.0 - self.trail_percent);
                }

                // Check if stop was triggered
                if current_price <= self.stop_price {
                    self.triggered = true;
                }
            }
            PositionDirection::Short => {
                // For short positions, track the minimum
                if current_price < self.highest_price {
                    self.highest_price = current_price; // lowest_price for short
                    self.stop_price = current_price * (1.0 + self.trail_percent);
                }

                // Check if stop was triggered
                if current_price >= self.stop_price {
                    self.triggered = true;
                }
            }
        }

        self.triggered
    }

    /// Returns the current stop level
    pub fn get_stop_price(&self) -> f64 {
        self.stop_price
    }

    /// Checks if the stop has been triggered
    pub fn is_triggered(&self) -> bool {
        self.triggered
    }

    /// Returns the highest price reached
    pub fn get_highest_price(&self) -> f64 {
        self.highest_price
    }
}

fn main() {
    // Example: buy BTC at $40,000 with 5% trailing stop
    let mut stop = TrailingStop::new_long(40_000.0, 0.05);

    let prices = vec![
        40_000.0, 41_000.0, 42_000.0, 43_000.0, 45_000.0,
        44_000.0, 43_500.0, 43_000.0, 42_800.0, 42_750.0,
    ];

    println!("=== Trailing Stop Simulation ===");
    println!("Entry: ${:.2}, Trail: 5%\n", 40_000.0);

    for price in prices {
        let was_triggered = stop.is_triggered();
        stop.update(price);

        println!(
            "Price: ${:.2} | Stop: ${:.2} | High: ${:.2} | {}",
            price,
            stop.get_stop_price(),
            stop.get_highest_price(),
            if stop.is_triggered() && !was_triggered {
                "TRIGGERED!"
            } else if stop.is_triggered() {
                "already closed"
            } else {
                "active"
            }
        );
    }
}
```

## Fixed Amount Trailing Stop

```rust
#[derive(Debug, Clone)]
pub struct FixedTrailingStop {
    /// Fixed trail amount in price units
    trail_amount: f64,
    /// Highest price reached
    highest_price: f64,
    /// Current stop level
    stop_price: f64,
    /// Whether the stop has been triggered
    triggered: bool,
}

impl FixedTrailingStop {
    pub fn new(entry_price: f64, trail_amount: f64) -> Self {
        FixedTrailingStop {
            trail_amount,
            highest_price: entry_price,
            stop_price: entry_price - trail_amount,
            triggered: false,
        }
    }

    pub fn update(&mut self, current_price: f64) -> bool {
        if self.triggered {
            return true;
        }

        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.stop_price = current_price - self.trail_amount;
        }

        if current_price <= self.stop_price {
            self.triggered = true;
        }

        self.triggered
    }

    pub fn get_stop_price(&self) -> f64 {
        self.stop_price
    }
}

fn main() {
    // Trailing stop with $100 trail
    let mut stop = FixedTrailingStop::new(1000.0, 100.0);

    let prices = [1000.0, 1050.0, 1100.0, 1080.0, 1000.0, 990.0];

    println!("=== Fixed Trailing Stop ($100) ===\n");

    for price in prices {
        let triggered = stop.update(price);
        println!(
            "Price: ${:.2} | Stop: ${:.2} | {}",
            price,
            stop.get_stop_price(),
            if triggered { "TRIGGERED!" } else { "active" }
        );
    }
}
```

## Advanced Trailing Stop with History

```rust
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
pub struct StopUpdate {
    timestamp: u64,
    price: f64,
    stop_level: f64,
    highest: f64,
}

#[derive(Debug)]
pub struct AdvancedTrailingStop {
    trail_percent: f64,
    highest_price: f64,
    stop_price: f64,
    triggered: bool,
    trigger_price: Option<f64>,
    history: Vec<StopUpdate>,
}

impl AdvancedTrailingStop {
    pub fn new(entry_price: f64, trail_percent: f64) -> Self {
        let stop_price = entry_price * (1.0 - trail_percent);
        let mut stop = AdvancedTrailingStop {
            trail_percent,
            highest_price: entry_price,
            stop_price,
            triggered: false,
            trigger_price: None,
            history: Vec::new(),
        };

        stop.record_update(entry_price);
        stop
    }

    fn get_timestamp() -> u64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_millis() as u64
    }

    fn record_update(&mut self, price: f64) {
        self.history.push(StopUpdate {
            timestamp: Self::get_timestamp(),
            price,
            stop_level: self.stop_price,
            highest: self.highest_price,
        });
    }

    pub fn update(&mut self, current_price: f64) -> bool {
        if self.triggered {
            return true;
        }

        let old_stop = self.stop_price;

        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.stop_price = current_price * (1.0 - self.trail_percent);
        }

        if current_price <= self.stop_price {
            self.triggered = true;
            self.trigger_price = Some(current_price);
        }

        // Record only if stop changed or triggered
        if self.stop_price != old_stop || self.triggered {
            self.record_update(current_price);
        }

        self.triggered
    }

    /// Returns profit/loss in percentage
    pub fn get_pnl_percent(&self, entry_price: f64) -> Option<f64> {
        self.trigger_price.map(|exit| {
            ((exit - entry_price) / entry_price) * 100.0
        })
    }

    /// Returns the history of stop updates
    pub fn get_history(&self) -> &[StopUpdate] {
        &self.history
    }

    /// Returns the number of times the stop was adjusted
    pub fn get_adjustment_count(&self) -> usize {
        self.history.len().saturating_sub(1)
    }
}

fn main() {
    let entry = 50_000.0;
    let mut stop = AdvancedTrailingStop::new(entry, 0.03); // 3% trailing

    let prices = vec![
        50_000.0, 51_000.0, 52_000.0, 53_000.0, 54_000.0,
        53_500.0, 53_000.0, 52_500.0, 52_380.0,
    ];

    println!("=== Advanced Trailing Stop ===");
    println!("Entry: ${:.2}, Trail: 3%\n", entry);

    for price in prices {
        stop.update(price);
    }

    println!("Stop adjustment history:");
    for (i, update) in stop.get_history().iter().enumerate() {
        println!(
            "  {}. Price: ${:.2} | Stop: ${:.2} | High: ${:.2}",
            i + 1,
            update.price,
            update.stop_level,
            update.highest
        );
    }

    println!("\nStatistics:");
    println!("  Stop adjustments: {}", stop.get_adjustment_count());

    if let Some(pnl) = stop.get_pnl_percent(entry) {
        println!("  P&L: {:.2}%", pnl);
    }
}
```

## Trailing Stop Manager for Multiple Positions

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct Position {
    pub symbol: String,
    pub entry_price: f64,
    pub quantity: f64,
    pub trailing_stop: TrailingStopState,
}

#[derive(Debug, Clone)]
pub struct TrailingStopState {
    pub trail_percent: f64,
    pub highest_price: f64,
    pub stop_price: f64,
    pub triggered: bool,
}

impl TrailingStopState {
    pub fn new(entry_price: f64, trail_percent: f64) -> Self {
        TrailingStopState {
            trail_percent,
            highest_price: entry_price,
            stop_price: entry_price * (1.0 - trail_percent),
            triggered: false,
        }
    }

    pub fn update(&mut self, price: f64) -> bool {
        if self.triggered {
            return true;
        }

        if price > self.highest_price {
            self.highest_price = price;
            self.stop_price = price * (1.0 - self.trail_percent);
        }

        if price <= self.stop_price {
            self.triggered = true;
        }

        self.triggered
    }
}

#[derive(Debug)]
pub struct TrailingStopManager {
    positions: HashMap<String, Position>,
    closed_positions: Vec<Position>,
}

impl TrailingStopManager {
    pub fn new() -> Self {
        TrailingStopManager {
            positions: HashMap::new(),
            closed_positions: Vec::new(),
        }
    }

    /// Opens a new position with trailing stop
    pub fn open_position(
        &mut self,
        symbol: &str,
        entry_price: f64,
        quantity: f64,
        trail_percent: f64,
    ) {
        let position = Position {
            symbol: symbol.to_string(),
            entry_price,
            quantity,
            trailing_stop: TrailingStopState::new(entry_price, trail_percent),
        };

        self.positions.insert(symbol.to_string(), position);
        println!(
            "Opened position: {} | Price: ${:.2} | Qty: {} | Trail: {:.1}%",
            symbol, entry_price, quantity, trail_percent * 100.0
        );
    }

    /// Updates prices and checks stops
    pub fn update_prices(&mut self, prices: &HashMap<String, f64>) {
        let mut to_close = Vec::new();

        for (symbol, position) in &mut self.positions {
            if let Some(&price) = prices.get(symbol) {
                if position.trailing_stop.update(price) {
                    to_close.push(symbol.clone());
                }
            }
        }

        // Close triggered positions
        for symbol in to_close {
            if let Some(position) = self.positions.remove(&symbol) {
                let pnl = (position.trailing_stop.stop_price - position.entry_price)
                    * position.quantity;
                let pnl_percent = ((position.trailing_stop.stop_price - position.entry_price)
                    / position.entry_price)
                    * 100.0;

                println!(
                    "CLOSED position: {} | Exit: ${:.2} | P&L: ${:.2} ({:.2}%)",
                    symbol, position.trailing_stop.stop_price, pnl, pnl_percent
                );

                self.closed_positions.push(position);
            }
        }
    }

    /// Prints status of all positions
    pub fn print_status(&self) {
        println!("\n=== Portfolio Status ===");

        if self.positions.is_empty() {
            println!("No open positions");
        } else {
            for (symbol, pos) in &self.positions {
                println!(
                    "{}: Entry ${:.2} | High ${:.2} | Stop ${:.2}",
                    symbol,
                    pos.entry_price,
                    pos.trailing_stop.highest_price,
                    pos.trailing_stop.stop_price
                );
            }
        }

        if !self.closed_positions.is_empty() {
            println!("\nClosed positions: {}", self.closed_positions.len());
        }
    }

    /// Returns total P&L of closed positions
    pub fn get_total_pnl(&self) -> f64 {
        self.closed_positions
            .iter()
            .map(|p| {
                (p.trailing_stop.stop_price - p.entry_price) * p.quantity
            })
            .sum()
    }
}

fn main() {
    let mut manager = TrailingStopManager::new();

    // Open several positions
    manager.open_position("BTC", 40_000.0, 0.5, 0.05);
    manager.open_position("ETH", 2_500.0, 4.0, 0.04);
    manager.open_position("SOL", 100.0, 50.0, 0.06);

    // Simulate price movements
    let price_updates = vec![
        HashMap::from([
            ("BTC".to_string(), 41_000.0),
            ("ETH".to_string(), 2_600.0),
            ("SOL".to_string(), 105.0),
        ]),
        HashMap::from([
            ("BTC".to_string(), 43_000.0),
            ("ETH".to_string(), 2_700.0),
            ("SOL".to_string(), 110.0),
        ]),
        HashMap::from([
            ("BTC".to_string(), 42_000.0),
            ("ETH".to_string(), 2_650.0),
            ("SOL".to_string(), 103.0), // Close to stop
        ]),
        HashMap::from([
            ("BTC".to_string(), 41_500.0),
            ("ETH".to_string(), 2_600.0),
            ("SOL".to_string(), 100.0), // Stop will trigger
        ]),
    ];

    println!("\n=== Trading Simulation ===\n");

    for (i, prices) in price_updates.iter().enumerate() {
        println!("--- Tick {} ---", i + 1);
        manager.update_prices(prices);
        manager.print_status();
        println!();
    }

    println!("=== Total P&L: ${:.2} ===", manager.get_total_pnl());
}
```

## Trailing Stop with Activation

Sometimes a trailing stop activates only after reaching a certain profit:

```rust
#[derive(Debug, Clone)]
pub struct ActivatedTrailingStop {
    entry_price: f64,
    /// Profit percentage for activation (e.g., 0.02 = 2%)
    activation_percent: f64,
    /// Trail percentage after activation
    trail_percent: f64,
    /// Whether trailing stop is activated
    activated: bool,
    highest_price: f64,
    stop_price: f64,
    triggered: bool,
}

impl ActivatedTrailingStop {
    pub fn new(entry_price: f64, activation_percent: f64, trail_percent: f64) -> Self {
        ActivatedTrailingStop {
            entry_price,
            activation_percent,
            trail_percent,
            activated: false,
            highest_price: entry_price,
            stop_price: 0.0, // No stop until activation
            triggered: false,
        }
    }

    pub fn update(&mut self, current_price: f64) -> TrailStatus {
        if self.triggered {
            return TrailStatus::Triggered;
        }

        // Check for activation
        if !self.activated {
            let profit_percent = (current_price - self.entry_price) / self.entry_price;

            if profit_percent >= self.activation_percent {
                self.activated = true;
                self.highest_price = current_price;
                self.stop_price = current_price * (1.0 - self.trail_percent);
                return TrailStatus::JustActivated;
            }

            return TrailStatus::WaitingActivation;
        }

        // Trailing stop is activated
        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.stop_price = current_price * (1.0 - self.trail_percent);
        }

        if current_price <= self.stop_price {
            self.triggered = true;
            return TrailStatus::Triggered;
        }

        TrailStatus::Active
    }

    pub fn is_activated(&self) -> bool {
        self.activated
    }

    pub fn get_stop_price(&self) -> Option<f64> {
        if self.activated {
            Some(self.stop_price)
        } else {
            None
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum TrailStatus {
    WaitingActivation,
    JustActivated,
    Active,
    Triggered,
}

fn main() {
    // Activates at +3% profit, then trails at 2%
    let mut stop = ActivatedTrailingStop::new(1000.0, 0.03, 0.02);

    let prices = [
        1000.0, 1010.0, 1020.0, 1030.0, // Activation at 1030
        1050.0, 1060.0, 1055.0, 1040.0, 1038.8, // Triggers around 1038.8
    ];

    println!("=== Activated Trailing Stop ===");
    println!("Entry: $1000 | Activation: +3% | Trail: 2%\n");

    for price in prices {
        let status = stop.update(price);

        let stop_str = match stop.get_stop_price() {
            Some(s) => format!("${:.2}", s),
            None => "---".to_string(),
        };

        println!(
            "Price: ${:.2} | Stop: {} | Status: {:?}",
            price, stop_str, status
        );
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Trailing Stop | Dynamic stop-loss that follows the price |
| Percentage stop | Stop at X% below the highest price |
| Fixed stop | Stop at X points below the highest price |
| Activated stop | Activates after reaching target profit |
| Long/Short | Different logic for long and short positions |
| Stop Manager | Managing stops for multiple positions |

## Exercises

1. **Basic trailing stop**: Implement a trailing stop that prints a message each time the stop moves to a new level.

2. **Bidirectional stop**: Add support for Short positions in `TrailingStopManager`, where the stop lowers as price falls.

3. **Stop with minimum step**: Modify the trailing stop so it only moves up if the new level is at least 0.5% higher than the previous one.

4. **Stop statistics**: Add a method to `TrailingStopManager` that returns statistics: average P&L, win rate percentage, maximum drawdown.

## Homework

1. **ATR-based Trailing Stop**: Implement a trailing stop that uses Average True Range (ATR) to determine the distance to the stop. ATR is the average price movement range over N periods.

2. **Chandelier Exit**: Create a trailing stop based on Chandelier Exit — stop is set at N * ATR distance from the highest high.

3. **Parabolic SAR**: Implement a simplified version of Parabolic SAR — a trailing stop that accelerates as price rises.

4. **Backtesting**: Write a function that tests different trailing stop parameters (1%, 2%, 3%, ..., 10%) on historical data and outputs a comparison table of results.

## Navigation

[← Previous day](../265-stop-loss-take-profit/en.md) | [Next day →](../267-position-sizing/en.md)
