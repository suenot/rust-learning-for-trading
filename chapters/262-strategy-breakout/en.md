# Day 262: Strategy: Breakout

## Trading Analogy

Imagine you're watching Bitcoin trade in a tight range between $40,000 and $42,000 for several days. Traders are uncertain — buyers push the price up, but sellers resist at the upper boundary. Suddenly, a large buy order pushes the price above $42,000 with high volume. This is a **breakout** — the moment when price decisively moves beyond a key level, often signaling the start of a new trend.

Breakout trading is one of the most popular strategies in algorithmic trading. The idea is simple: identify key price levels (support and resistance), wait for the price to break through these levels, and enter a position in the direction of the breakout. The challenge lies in distinguishing real breakouts from false ones.

## Understanding Breakout Strategy

A breakout occurs when:
1. Price has been consolidating within a range
2. The range is bounded by support (lower) and resistance (upper) levels
3. Price moves decisively beyond one of these boundaries
4. Volume typically increases during a genuine breakout

Let's model this in Rust:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum BreakoutDirection {
    Bullish,  // Price breaks above resistance
    Bearish,  // Price breaks below support
}

#[derive(Debug, Clone)]
struct PriceRange {
    support: f64,      // Lower boundary
    resistance: f64,   // Upper boundary
    symbol: String,
}

impl PriceRange {
    fn new(symbol: &str, support: f64, resistance: f64) -> Self {
        PriceRange {
            support,
            resistance,
            symbol: symbol.to_string(),
        }
    }

    fn range_width(&self) -> f64 {
        self.resistance - self.support
    }

    fn is_within_range(&self, price: f64) -> bool {
        price >= self.support && price <= self.resistance
    }
}
```

## Detecting Breakouts with Pattern Matching

Rust's pattern matching is perfect for handling breakout detection logic:

```rust
#[derive(Debug)]
struct BreakoutSignal {
    direction: BreakoutDirection,
    entry_price: f64,
    target_price: f64,
    stop_loss: f64,
}

fn detect_breakout(
    range: &PriceRange,
    current_price: f64,
    previous_price: f64,
) -> Option<BreakoutSignal> {
    let range_width = range.range_width();

    match (current_price, previous_price) {
        // Bullish breakout: price crosses above resistance
        (curr, prev) if curr > range.resistance && prev <= range.resistance => {
            Some(BreakoutSignal {
                direction: BreakoutDirection::Bullish,
                entry_price: curr,
                target_price: curr + range_width,  // Target = entry + range width
                stop_loss: range.resistance - (range_width * 0.25),  // Stop below resistance
            })
        }
        // Bearish breakout: price crosses below support
        (curr, prev) if curr < range.support && prev >= range.support => {
            Some(BreakoutSignal {
                direction: BreakoutDirection::Bearish,
                entry_price: curr,
                target_price: curr - range_width,  // Target = entry - range width
                stop_loss: range.support + (range_width * 0.25),  // Stop above support
            })
        }
        // No breakout
        _ => None,
    }
}
```

## Volume Confirmation

Real breakouts are typically accompanied by increased trading volume. Let's add volume analysis:

```rust
#[derive(Debug, Clone)]
struct Candle {
    open: f64,
    high: f64,
    low: f64,
    close: f64,
    volume: f64,
    timestamp: u64,
}

struct BreakoutDetector {
    range: PriceRange,
    volume_threshold_multiplier: f64,
    average_volume: f64,
}

impl BreakoutDetector {
    fn new(range: PriceRange, average_volume: f64) -> Self {
        BreakoutDetector {
            range,
            volume_threshold_multiplier: 1.5,  // Require 150% of average volume
            average_volume,
        }
    }

    fn is_volume_confirmed(&self, current_volume: f64) -> bool {
        current_volume >= self.average_volume * self.volume_threshold_multiplier
    }

    fn analyze_candle(&self, candle: &Candle, prev_close: f64) -> Option<BreakoutSignal> {
        // First check for basic breakout
        let signal = detect_breakout(&self.range, candle.close, prev_close)?;

        // Confirm with volume
        if self.is_volume_confirmed(candle.volume) {
            Some(signal)
        } else {
            println!(
                "Potential breakout rejected: volume {} below threshold {}",
                candle.volume,
                self.average_volume * self.volume_threshold_multiplier
            );
            None
        }
    }
}
```

## Practical Example: Complete Breakout Trading System

Let's build a complete breakout trading system that tracks positions and calculates P&L:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    direction: BreakoutDirection,
    entry_price: f64,
    quantity: f64,
    stop_loss: f64,
    target_price: f64,
}

impl Position {
    fn unrealized_pnl(&self, current_price: f64) -> f64 {
        let price_diff = current_price - self.entry_price;
        match self.direction {
            BreakoutDirection::Bullish => price_diff * self.quantity,
            BreakoutDirection::Bearish => -price_diff * self.quantity,
        }
    }

    fn should_close(&self, current_price: f64) -> bool {
        match self.direction {
            BreakoutDirection::Bullish => {
                current_price <= self.stop_loss || current_price >= self.target_price
            }
            BreakoutDirection::Bearish => {
                current_price >= self.stop_loss || current_price <= self.target_price
            }
        }
    }
}

struct BreakoutTradingSystem {
    positions: HashMap<String, Position>,
    detectors: HashMap<String, BreakoutDetector>,
    capital: f64,
    risk_per_trade: f64,  // Percentage of capital to risk per trade
    realized_pnl: f64,
}

impl BreakoutTradingSystem {
    fn new(capital: f64, risk_per_trade: f64) -> Self {
        BreakoutTradingSystem {
            positions: HashMap::new(),
            detectors: HashMap::new(),
            capital,
            risk_per_trade,
            realized_pnl: 0.0,
        }
    }

    fn add_detector(&mut self, symbol: &str, range: PriceRange, avg_volume: f64) {
        let detector = BreakoutDetector::new(range, avg_volume);
        self.detectors.insert(symbol.to_string(), detector);
    }

    fn calculate_position_size(&self, entry: f64, stop_loss: f64) -> f64 {
        let risk_amount = self.capital * self.risk_per_trade;
        let risk_per_unit = (entry - stop_loss).abs();
        risk_amount / risk_per_unit
    }

    fn process_candle(&mut self, symbol: &str, candle: &Candle, prev_close: f64) {
        // Check existing position
        if let Some(position) = self.positions.get(symbol) {
            if position.should_close(candle.close) {
                let pnl = position.unrealized_pnl(candle.close);
                println!(
                    "[{}] Closing position: {:?} at {} | P&L: {:.2}",
                    symbol, position.direction, candle.close, pnl
                );
                self.realized_pnl += pnl;
                self.capital += pnl;
                self.positions.remove(symbol);
            }
            return;
        }

        // Look for new breakout
        if let Some(detector) = self.detectors.get(symbol) {
            if let Some(signal) = detector.analyze_candle(candle, prev_close) {
                let quantity = self.calculate_position_size(
                    signal.entry_price,
                    signal.stop_loss,
                );

                let position = Position {
                    symbol: symbol.to_string(),
                    direction: signal.direction,
                    entry_price: signal.entry_price,
                    quantity,
                    stop_loss: signal.stop_loss,
                    target_price: signal.target_price,
                };

                println!(
                    "[{}] Opening {:?} position: entry={:.2}, qty={:.4}, stop={:.2}, target={:.2}",
                    symbol,
                    position.direction,
                    position.entry_price,
                    position.quantity,
                    position.stop_loss,
                    position.target_price
                );

                self.positions.insert(symbol.to_string(), position);
            }
        }
    }

    fn portfolio_status(&self) {
        println!("\n=== Portfolio Status ===");
        println!("Capital: ${:.2}", self.capital);
        println!("Realized P&L: ${:.2}", self.realized_pnl);
        println!("Open Positions: {}", self.positions.len());
        for (symbol, pos) in &self.positions {
            println!("  {} - {:?} @ {:.2}", symbol, pos.direction, pos.entry_price);
        }
    }
}

fn main() {
    // Initialize trading system with $100,000 capital, 1% risk per trade
    let mut system = BreakoutTradingSystem::new(100_000.0, 0.01);

    // Set up breakout detection for BTC
    let btc_range = PriceRange::new("BTC", 40_000.0, 42_000.0);
    system.add_detector("BTC", btc_range, 1000.0);

    // Set up breakout detection for ETH
    let eth_range = PriceRange::new("ETH", 2_200.0, 2_400.0);
    system.add_detector("ETH", eth_range, 5000.0);

    // Simulate price data - BTC bullish breakout
    let btc_candles = vec![
        Candle { open: 41_500.0, high: 41_800.0, low: 41_200.0, close: 41_700.0, volume: 900.0, timestamp: 1 },
        Candle { open: 41_700.0, high: 42_500.0, low: 41_600.0, close: 42_300.0, volume: 1800.0, timestamp: 2 },
        Candle { open: 42_300.0, high: 43_500.0, low: 42_100.0, close: 43_200.0, volume: 2200.0, timestamp: 3 },
        Candle { open: 43_200.0, high: 44_500.0, low: 43_000.0, close: 44_200.0, volume: 1500.0, timestamp: 4 },
    ];

    // Process BTC candles
    println!("Processing BTC price action...\n");
    let mut prev_close = 41_500.0;
    for candle in &btc_candles {
        system.process_candle("BTC", candle, prev_close);
        prev_close = candle.close;
    }

    system.portfolio_status();
}
```

## False Breakout Protection

False breakouts are a common challenge. Here's how to add protection:

```rust
struct BreakoutFilter {
    min_close_beyond_level: f64,  // Minimum % price must close beyond level
    confirmation_candles: usize,   // Number of candles to confirm breakout
    breakout_history: Vec<(u64, f64)>,  // (timestamp, close_price)
}

impl BreakoutFilter {
    fn new(min_close_pct: f64, confirmation_candles: usize) -> Self {
        BreakoutFilter {
            min_close_beyond_level: min_close_pct,
            confirmation_candles,
            breakout_history: Vec::new(),
        }
    }

    fn validate_bullish_breakout(
        &mut self,
        resistance: f64,
        candle: &Candle,
    ) -> bool {
        // Check if close is far enough beyond resistance
        let breakout_pct = (candle.close - resistance) / resistance * 100.0;
        if breakout_pct < self.min_close_beyond_level {
            return false;
        }

        // Track for confirmation
        self.breakout_history.push((candle.timestamp, candle.close));

        // Clean old entries
        self.breakout_history.retain(|(ts, _)| {
            candle.timestamp - ts <= self.confirmation_candles as u64
        });

        // Check if we have enough confirming candles above resistance
        let confirmed_candles = self.breakout_history
            .iter()
            .filter(|(_, close)| *close > resistance)
            .count();

        confirmed_candles >= self.confirmation_candles
    }
}
```

## Identifying Support and Resistance Levels

To make the system more dynamic, we can automatically identify key levels:

```rust
fn find_support_resistance(prices: &[f64], lookback: usize) -> Option<PriceRange> {
    if prices.len() < lookback {
        return None;
    }

    let recent_prices = &prices[prices.len() - lookback..];

    let high = recent_prices
        .iter()
        .cloned()
        .fold(f64::NEG_INFINITY, f64::max);

    let low = recent_prices
        .iter()
        .cloned()
        .fold(f64::INFINITY, f64::min);

    // Calculate range width as percentage
    let range_pct = (high - low) / low * 100.0;

    // Only return range if it's a consolidation (tight range)
    if range_pct < 10.0 {  // Less than 10% range = consolidation
        Some(PriceRange::new("AUTO", low, high))
    } else {
        None
    }
}

fn calculate_average_volume(candles: &[Candle], periods: usize) -> f64 {
    let slice = if candles.len() > periods {
        &candles[candles.len() - periods..]
    } else {
        candles
    };

    let total: f64 = slice.iter().map(|c| c.volume).sum();
    total / slice.len() as f64
}
```

## What We Learned

| Concept | Trading Application |
|---------|---------------------|
| Enums | Model breakout direction (Bullish/Bearish) |
| Pattern Matching | Detect and classify breakout conditions |
| Option<T> | Handle cases where no breakout occurs |
| Structs | Represent price ranges, positions, and signals |
| HashMap | Track multiple positions and detectors |
| Iterators | Analyze price and volume data |
| Methods | Encapsulate trading logic in types |
| f64 operations | Calculate targets, stops, and P&L |

## Homework

1. **Enhanced Breakout Filter**: Implement a "retest" filter that waits for price to break out, pull back to the breakout level, and then continue in the breakout direction before entering a position.

2. **Multi-Timeframe Confirmation**: Modify the `BreakoutDetector` to require breakout confirmation on multiple timeframes (e.g., 1-hour and 4-hour charts must both show breakout).

3. **Trailing Stop Implementation**: Add a trailing stop mechanism to the `Position` struct that moves the stop loss in the direction of profit as the trade progresses.

4. **Breakout Statistics Tracker**: Create a `BreakoutStats` struct that tracks historical breakout performance including win rate, average profit/loss, and maximum drawdown for the strategy.

## Navigation

[← Previous day](../261-strategy-mean-reversion/en.md) | [Next day →](../263-strategy-momentum/en.md)
