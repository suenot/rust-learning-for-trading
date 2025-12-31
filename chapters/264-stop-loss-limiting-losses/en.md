# Day 264: Stop-Loss: Limiting Losses

## Trading Analogy

Imagine you bought shares of a company at $100. You believe in its growth, but you understand: the market is unpredictable. What if the price drops to $50? $30? $10? Without a protective mechanism, you could lose your entire investment.

**Stop-Loss** is like an insurance policy for your position. You set a boundary: "If the price drops to $90, automatically sell." This limits your maximum loss to 10%.

In real trading, stop-loss is critically important for:
- Protecting capital from catastrophic losses
- Removing emotions from decision-making
- Automatic risk management
- Maintaining trading discipline

## What is Stop-Loss?

Stop-Loss is an order to sell an asset when it reaches a certain price level. Main types:

1. **Fixed Stop-Loss** — sell at a specific price
2. **Percentage Stop-Loss** — sell when price drops by a set percentage
3. **Trailing Stop-Loss** — dynamic stop that follows the price
4. **Volatility-based Stop** — adapts to market conditions

## Basic Stop-Loss Structure

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum StopLossType {
    Fixed(f64),           // Fixed price
    Percentage(f64),      // Percentage from entry price
    Trailing(f64),        // Follows price with offset
    VolatilityBased(f64), // Based on ATR or other indicator
}

#[derive(Debug, Clone)]
pub struct StopLoss {
    stop_type: StopLossType,
    trigger_price: f64,
    is_active: bool,
}

impl StopLoss {
    pub fn new_fixed(stop_price: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Fixed(stop_price),
            trigger_price: stop_price,
            is_active: true,
        }
    }

    pub fn new_percentage(entry_price: f64, percentage: f64) -> Self {
        let trigger = entry_price * (1.0 - percentage / 100.0);
        StopLoss {
            stop_type: StopLossType::Percentage(percentage),
            trigger_price: trigger,
            is_active: true,
        }
    }

    pub fn new_trailing(current_price: f64, trail_amount: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Trailing(trail_amount),
            trigger_price: current_price - trail_amount,
            is_active: true,
        }
    }

    pub fn should_trigger(&self, current_price: f64) -> bool {
        self.is_active && current_price <= self.trigger_price
    }

    pub fn update_trailing(&mut self, current_price: f64) {
        if let StopLossType::Trailing(trail_amount) = self.stop_type {
            let new_trigger = current_price - trail_amount;
            // Trailing stop only moves up, never down
            if new_trigger > self.trigger_price {
                self.trigger_price = new_trigger;
            }
        }
    }
}

fn main() {
    // Example 1: Fixed stop-loss
    let fixed_stop = StopLoss::new_fixed(95.0);
    println!("Fixed stop at: ${:.2}", fixed_stop.trigger_price);

    // Example 2: Percentage stop-loss (5% from entry price $100)
    let percentage_stop = StopLoss::new_percentage(100.0, 5.0);
    println!("5% stop at: ${:.2}", percentage_stop.trigger_price);

    // Example 3: Trailing stop ($3 from current price)
    let mut trailing_stop = StopLoss::new_trailing(100.0, 3.0);
    println!("Trailing stop at: ${:.2}", trailing_stop.trigger_price);

    // Price rises — stop follows
    trailing_stop.update_trailing(105.0);
    println!("After rise to $105, stop at: ${:.2}", trailing_stop.trigger_price);
}
```

## Position with Stop-Loss Protection

```rust
#[derive(Debug, Clone)]
pub struct Position {
    symbol: String,
    quantity: f64,
    entry_price: f64,
    current_price: f64,
    stop_loss: Option<StopLoss>,
    is_open: bool,
}

#[derive(Debug, Clone, PartialEq)]
pub enum StopLossType {
    Fixed(f64),
    Percentage(f64),
    Trailing(f64),
}

#[derive(Debug, Clone)]
pub struct StopLoss {
    stop_type: StopLossType,
    trigger_price: f64,
    is_active: bool,
}

impl StopLoss {
    pub fn new_fixed(stop_price: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Fixed(stop_price),
            trigger_price: stop_price,
            is_active: true,
        }
    }

    pub fn new_percentage(entry_price: f64, percentage: f64) -> Self {
        let trigger = entry_price * (1.0 - percentage / 100.0);
        StopLoss {
            stop_type: StopLossType::Percentage(percentage),
            trigger_price: trigger,
            is_active: true,
        }
    }

    pub fn new_trailing(current_price: f64, trail_amount: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Trailing(trail_amount),
            trigger_price: current_price - trail_amount,
            is_active: true,
        }
    }

    pub fn should_trigger(&self, current_price: f64) -> bool {
        self.is_active && current_price <= self.trigger_price
    }

    pub fn update_trailing(&mut self, current_price: f64) {
        if let StopLossType::Trailing(trail_amount) = self.stop_type {
            let new_trigger = current_price - trail_amount;
            if new_trigger > self.trigger_price {
                self.trigger_price = new_trigger;
            }
        }
    }
}

impl Position {
    pub fn new(symbol: &str, quantity: f64, entry_price: f64) -> Self {
        Position {
            symbol: symbol.to_string(),
            quantity,
            entry_price,
            current_price: entry_price,
            stop_loss: None,
            is_open: true,
        }
    }

    pub fn set_stop_loss(&mut self, stop_loss: StopLoss) {
        self.stop_loss = Some(stop_loss);
    }

    pub fn update_price(&mut self, new_price: f64) -> Option<String> {
        self.current_price = new_price;

        // Update trailing stop if present
        if let Some(ref mut stop) = self.stop_loss {
            stop.update_trailing(new_price);

            // Check stop-loss trigger
            if stop.should_trigger(new_price) {
                self.is_open = false;
                let loss = (self.entry_price - new_price) * self.quantity;
                return Some(format!(
                    "STOP-LOSS triggered for {}: sold {} at ${:.2}, loss: ${:.2}",
                    self.symbol, self.quantity, new_price, loss
                ));
            }
        }
        None
    }

    pub fn unrealized_pnl(&self) -> f64 {
        (self.current_price - self.entry_price) * self.quantity
    }

    pub fn pnl_percentage(&self) -> f64 {
        ((self.current_price - self.entry_price) / self.entry_price) * 100.0
    }
}

fn main() {
    // Open a position
    let mut position = Position::new("AAPL", 100.0, 150.0);

    // Set trailing stop-loss at $5
    position.set_stop_loss(StopLoss::new_trailing(150.0, 5.0));

    println!("Opened position: {} shares at ${}", position.quantity, position.entry_price);
    println!("Initial stop: ${:.2}", position.stop_loss.as_ref().unwrap().trigger_price);

    // Simulate price movement
    let prices = vec![152.0, 155.0, 158.0, 156.0, 153.0, 151.0, 148.0];

    for price in prices {
        if let Some(message) = position.update_price(price) {
            println!("{}", message);
            break;
        } else {
            println!(
                "Price: ${:.2}, Stop: ${:.2}, P&L: ${:.2} ({:.2}%)",
                price,
                position.stop_loss.as_ref().unwrap().trigger_price,
                position.unrealized_pnl(),
                position.pnl_percentage()
            );
        }
    }
}
```

## Visualizing Trailing Stop-Loss

```
Price                    Trailing Stop
  |                         |
158 ─────●                  |
  |       \                 |
156 ───────●                |──── 153 (stop raised)
  |         \               |
154 ─────────●              |
  |           \             |
152 ───────────●            |──── 147 (initial)
  |             \           |
150 ─────────────● (entry)  |──── 145
  |               \         |
148 ────────────────X       | STOP TRIGGERED!
  |                         |

Trailing stop follows price up, but never moves down!
```

## Trading Engine with Risk Management

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq)]
pub enum StopLossType {
    Fixed(f64),
    Percentage(f64),
    Trailing(f64),
}

#[derive(Debug, Clone)]
pub struct StopLoss {
    stop_type: StopLossType,
    trigger_price: f64,
    is_active: bool,
}

impl StopLoss {
    pub fn new_fixed(stop_price: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Fixed(stop_price),
            trigger_price: stop_price,
            is_active: true,
        }
    }

    pub fn new_percentage(entry_price: f64, percentage: f64) -> Self {
        let trigger = entry_price * (1.0 - percentage / 100.0);
        StopLoss {
            stop_type: StopLossType::Percentage(percentage),
            trigger_price: trigger,
            is_active: true,
        }
    }

    pub fn new_trailing(current_price: f64, trail_amount: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Trailing(trail_amount),
            trigger_price: current_price - trail_amount,
            is_active: true,
        }
    }

    pub fn should_trigger(&self, current_price: f64) -> bool {
        self.is_active && current_price <= self.trigger_price
    }

    pub fn update_trailing(&mut self, current_price: f64) {
        if let StopLossType::Trailing(trail_amount) = self.stop_type {
            let new_trigger = current_price - trail_amount;
            if new_trigger > self.trigger_price {
                self.trigger_price = new_trigger;
            }
        }
    }
}

#[derive(Debug, Clone)]
pub struct Position {
    symbol: String,
    quantity: f64,
    entry_price: f64,
    current_price: f64,
    stop_loss: Option<StopLoss>,
    take_profit: Option<f64>,
    is_open: bool,
}

impl Position {
    pub fn new(symbol: &str, quantity: f64, entry_price: f64) -> Self {
        Position {
            symbol: symbol.to_string(),
            quantity,
            entry_price,
            current_price: entry_price,
            stop_loss: None,
            take_profit: None,
            is_open: true,
        }
    }

    pub fn unrealized_pnl(&self) -> f64 {
        (self.current_price - self.entry_price) * self.quantity
    }
}

#[derive(Debug)]
struct RiskManager {
    max_position_risk: f64,     // Maximum risk per position (%)
    max_portfolio_risk: f64,    // Maximum portfolio risk (%)
    max_daily_loss: f64,        // Maximum daily loss ($)
    daily_loss: f64,            // Current daily loss
}

impl RiskManager {
    fn new(max_position_risk: f64, max_portfolio_risk: f64, max_daily_loss: f64) -> Self {
        RiskManager {
            max_position_risk,
            max_portfolio_risk,
            max_daily_loss,
            daily_loss: 0.0,
        }
    }

    fn calculate_position_size(
        &self,
        capital: f64,
        entry_price: f64,
        stop_price: f64,
    ) -> f64 {
        // Risk per trade = max_position_risk % of capital
        let risk_amount = capital * (self.max_position_risk / 100.0);
        let risk_per_share = entry_price - stop_price;

        if risk_per_share <= 0.0 {
            return 0.0;
        }

        risk_amount / risk_per_share
    }

    fn can_take_new_position(&self, potential_loss: f64) -> bool {
        self.daily_loss + potential_loss <= self.max_daily_loss
    }

    fn record_loss(&mut self, loss: f64) {
        self.daily_loss += loss;
    }

    fn reset_daily_loss(&mut self) {
        self.daily_loss = 0.0;
    }
}

#[derive(Debug)]
struct TradingEngine {
    capital: f64,
    positions: HashMap<String, Position>,
    risk_manager: RiskManager,
    closed_trades: Vec<(String, f64)>, // (symbol, pnl)
}

impl TradingEngine {
    fn new(capital: f64) -> Self {
        TradingEngine {
            capital,
            positions: HashMap::new(),
            risk_manager: RiskManager::new(2.0, 6.0, capital * 0.05),
            closed_trades: Vec::new(),
        }
    }

    fn open_position(
        &mut self,
        symbol: &str,
        entry_price: f64,
        stop_loss_price: f64,
    ) -> Result<(), String> {
        // Calculate position size based on risk
        let position_size = self.risk_manager.calculate_position_size(
            self.capital,
            entry_price,
            stop_loss_price,
        );

        if position_size <= 0.0 {
            return Err("Invalid stop-loss parameters".to_string());
        }

        let potential_loss = (entry_price - stop_loss_price) * position_size;

        if !self.risk_manager.can_take_new_position(potential_loss) {
            return Err(format!(
                "Daily loss limit exceeded. Current: ${:.2}, Limit: ${:.2}",
                self.risk_manager.daily_loss, self.risk_manager.max_daily_loss
            ));
        }

        let cost = position_size * entry_price;
        if cost > self.capital {
            return Err(format!(
                "Insufficient capital. Need: ${:.2}, Have: ${:.2}",
                cost, self.capital
            ));
        }

        let mut position = Position::new(symbol, position_size, entry_price);
        position.stop_loss = Some(StopLoss::new_fixed(stop_loss_price));

        self.capital -= cost;
        self.positions.insert(symbol.to_string(), position);

        println!(
            "Opened position: {} shares of {} at ${:.2}, stop: ${:.2}",
            position_size.round(),
            symbol,
            entry_price,
            stop_loss_price
        );

        Ok(())
    }

    fn update_prices(&mut self, price_updates: &HashMap<String, f64>) {
        let mut to_close = Vec::new();

        for (symbol, new_price) in price_updates {
            if let Some(position) = self.positions.get_mut(symbol) {
                position.current_price = *new_price;

                // Update trailing stop
                if let Some(ref mut stop) = position.stop_loss {
                    stop.update_trailing(*new_price);

                    // Check trigger
                    if stop.should_trigger(*new_price) {
                        let pnl = position.unrealized_pnl();
                        println!(
                            "STOP-LOSS: {} closed at ${:.2}, P&L: ${:.2}",
                            symbol, new_price, pnl
                        );
                        to_close.push((symbol.clone(), pnl));
                    }
                }

                // Check take-profit
                if let Some(tp) = position.take_profit {
                    if *new_price >= tp {
                        let pnl = position.unrealized_pnl();
                        println!(
                            "TAKE-PROFIT: {} closed at ${:.2}, P&L: ${:.2}",
                            symbol, new_price, pnl
                        );
                        to_close.push((symbol.clone(), pnl));
                    }
                }
            }
        }

        // Close positions
        for (symbol, pnl) in to_close {
            if let Some(position) = self.positions.remove(&symbol) {
                self.capital += position.quantity * position.current_price;
                if pnl < 0.0 {
                    self.risk_manager.record_loss(-pnl);
                }
                self.closed_trades.push((symbol, pnl));
            }
        }
    }

    fn get_portfolio_status(&self) -> String {
        let total_exposure: f64 = self.positions.values()
            .map(|p| p.current_price * p.quantity)
            .sum();
        let total_pnl: f64 = self.positions.values()
            .map(|p| p.unrealized_pnl())
            .sum();

        format!(
            "Capital: ${:.2}, Positions: {}, Exposure: ${:.2}, Unrealized P&L: ${:.2}",
            self.capital,
            self.positions.len(),
            total_exposure,
            total_pnl
        )
    }
}

fn main() {
    let mut engine = TradingEngine::new(100_000.0);

    // Open positions with risk management
    let _ = engine.open_position("AAPL", 150.0, 145.0); // Stop $5 below
    let _ = engine.open_position("GOOGL", 140.0, 133.0); // Stop $7 below
    let _ = engine.open_position("MSFT", 380.0, 370.0);  // Stop $10 below

    println!("\n{}\n", engine.get_portfolio_status());

    // Simulate price movements
    let scenarios = vec![
        ("Day 1", vec![("AAPL", 152.0), ("GOOGL", 142.0), ("MSFT", 385.0)]),
        ("Day 2", vec![("AAPL", 148.0), ("GOOGL", 138.0), ("MSFT", 375.0)]),
        ("Day 3", vec![("AAPL", 144.0), ("GOOGL", 135.0), ("MSFT", 368.0)]), // AAPL and MSFT stop
    ];

    for (day, prices) in scenarios {
        println!("=== {} ===", day);
        let updates: HashMap<String, f64> = prices.into_iter()
            .map(|(s, p)| (s.to_string(), p))
            .collect();
        engine.update_prices(&updates);
        println!("{}\n", engine.get_portfolio_status());
    }
}
```

## Adaptive Stop-Loss Based on Volatility

```rust
/// Calculates Average True Range (ATR) — a measure of volatility
fn calculate_atr(high: &[f64], low: &[f64], close: &[f64], period: usize) -> f64 {
    if high.len() < period + 1 || low.len() < period + 1 || close.len() < period + 1 {
        return 0.0;
    }

    let mut true_ranges = Vec::with_capacity(period);

    for i in 1..=period {
        let idx = high.len() - period - 1 + i;
        let prev_close = close[idx - 1];

        let tr = (high[idx] - low[idx])
            .max((high[idx] - prev_close).abs())
            .max((low[idx] - prev_close).abs());

        true_ranges.push(tr);
    }

    true_ranges.iter().sum::<f64>() / period as f64
}

#[derive(Debug)]
struct VolatilityBasedStop {
    atr_multiplier: f64,
    current_atr: f64,
    stop_price: f64,
}

impl VolatilityBasedStop {
    fn new(entry_price: f64, atr: f64, multiplier: f64) -> Self {
        VolatilityBasedStop {
            atr_multiplier: multiplier,
            current_atr: atr,
            stop_price: entry_price - (atr * multiplier),
        }
    }

    fn update(&mut self, current_price: f64, new_atr: f64) {
        self.current_atr = new_atr;
        let new_stop = current_price - (new_atr * self.atr_multiplier);

        // Stop only moves up
        if new_stop > self.stop_price {
            self.stop_price = new_stop;
        }
    }

    fn should_trigger(&self, current_price: f64) -> bool {
        current_price <= self.stop_price
    }
}

fn main() {
    // Historical data for ATR calculation
    let high = vec![102.0, 104.0, 103.0, 105.0, 108.0, 107.0, 110.0, 109.0, 112.0, 111.0, 113.0];
    let low = vec![99.0, 101.0, 100.0, 102.0, 104.0, 103.0, 106.0, 105.0, 108.0, 107.0, 109.0];
    let close = vec![101.0, 103.0, 101.0, 104.0, 106.0, 105.0, 108.0, 107.0, 110.0, 109.0, 111.0];

    let atr = calculate_atr(&high, &low, &close, 5);
    println!("ATR (5 periods): {:.2}", atr);

    let entry_price = 111.0;
    let mut stop = VolatilityBasedStop::new(entry_price, atr, 2.0);

    println!("Entry price: ${:.2}", entry_price);
    println!("Stop-Loss (2 ATR): ${:.2}", stop.stop_price);
    println!("Distance to stop: ${:.2} ({:.2}%)",
        entry_price - stop.stop_price,
        ((entry_price - stop.stop_price) / entry_price) * 100.0
    );

    // Simulate price movement and ATR updates
    let new_prices = vec![(115.0, 3.2), (118.0, 3.5), (116.0, 3.8), (112.0, 4.0)];

    for (price, new_atr) in new_prices {
        stop.update(price, new_atr);
        println!(
            "Price: ${:.2}, ATR: {:.2}, Stop: ${:.2}{}",
            price,
            new_atr,
            stop.stop_price,
            if stop.should_trigger(price) { " [TRIGGERED!]" } else { "" }
        );

        if stop.should_trigger(price) {
            break;
        }
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Stop-Loss | Order to sell when price reaches a specified level |
| Fixed Stop | Specific price point |
| Percentage Stop | Percentage from entry price |
| Trailing Stop | Dynamic stop that follows price |
| ATR Stop | Stop based on market volatility |
| Risk Management | Limiting losses per position and portfolio |
| Position Sizing | Calculation based on acceptable risk |

## Homework

1. **Multiple Stops**: Implement a system with multiple stop-loss levels:
   - First stop closes 50% of position
   - Second stop closes remaining 50%
   - Add logging for each trigger

2. **Time-based Stop**: Create a time-based stop-loss that:
   - Closes position if take-profit isn't reached within N candles
   - Tightens stop every M candles (pulls closer to current price)

3. **Break-even Stop**: Implement break-even stop logic:
   - When profit reaches X% — stop moves to entry level
   - When profit reaches 2X% — stop moves to +X% level
   - Test on historical data

4. **Stop Analyzer**: Create a program that:
   - Loads historical price data
   - Tests different stop-loss types (fixed, %, trailing, ATR)
   - Outputs statistics: trigger count, average loss, maximum loss
   - Determines optimal stop type for the given asset

## Navigation

[← Previous day](../263-take-profit-securing-gains/en.md) | [Next day →](../265-position-sizing-risk-per-trade/en.md)
