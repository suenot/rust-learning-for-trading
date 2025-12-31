# Day 272: Drawdown: Portfolio Decline

## Trading Analogy

Imagine you're managing a trading portfolio that started with $100,000. Over several months, your portfolio grows to $150,000 — a fantastic 50% gain! But then the market turns against you. Your portfolio drops to $120,000. Even though you're still up 20% from your starting point, you've experienced a **drawdown** of $30,000 (or 20%) from your peak.

This is exactly what **drawdown** measures — the decline from a peak to a trough in the value of a portfolio. In trading, drawdown is one of the most critical risk metrics because:
- It shows the worst-case scenario an investor experienced
- It helps set realistic expectations for strategy performance
- It's essential for position sizing and risk management
- Large drawdowns can wipe out years of gains

## What is Drawdown?

Drawdown measures the peak-to-trough decline during a specific period of an investment. There are several types:

1. **Absolute Drawdown** — the difference between initial capital and the lowest point
2. **Maximum Drawdown (MDD)** — the largest peak-to-trough decline in portfolio value
3. **Relative Drawdown** — drawdown expressed as a percentage of the peak value
4. **Drawdown Duration** — how long it takes to recover from a drawdown

```
Portfolio Value Over Time:

$150,000  ────────●─────────────────────────────────── Peak
                   \
                    \
$130,000             \                    ●────────── Recovery
                      \                  /
                       \                /
$120,000                ●──────────────●               Trough
                        |<-- Drawdown -->|
                        |   Duration     |
```

## Simple Drawdown Calculation

```rust
/// Calculates the current drawdown from the peak value
fn calculate_drawdown(peak: f64, current: f64) -> f64 {
    if peak <= 0.0 {
        return 0.0;
    }

    let drawdown = (peak - current) / peak * 100.0;
    drawdown.max(0.0) // Drawdown can't be negative
}

/// Calculates the maximum drawdown from a series of portfolio values
fn calculate_max_drawdown(values: &[f64]) -> (f64, usize, usize) {
    if values.is_empty() {
        return (0.0, 0, 0);
    }

    let mut max_drawdown = 0.0;
    let mut peak = values[0];
    let mut peak_index = 0;
    let mut trough_index = 0;
    let mut max_peak_index = 0;
    let mut max_trough_index = 0;

    for (i, &value) in values.iter().enumerate() {
        if value > peak {
            peak = value;
            peak_index = i;
        }

        let drawdown = (peak - value) / peak * 100.0;

        if drawdown > max_drawdown {
            max_drawdown = drawdown;
            max_peak_index = peak_index;
            max_trough_index = i;
        }
    }

    (max_drawdown, max_peak_index, max_trough_index)
}

fn main() {
    // Simulated portfolio values over time
    let portfolio_values = vec![
        100_000.0, 105_000.0, 110_000.0, 108_000.0, 115_000.0,
        120_000.0, 118_000.0, 125_000.0, 130_000.0, 128_000.0,
        122_000.0, 115_000.0, 110_000.0, 112_000.0, 118_000.0,
        125_000.0, 132_000.0, 140_000.0, 138_000.0, 145_000.0,
    ];

    let (max_dd, peak_idx, trough_idx) = calculate_max_drawdown(&portfolio_values);

    println!("Portfolio Analysis:");
    println!("  Starting value: ${:.2}", portfolio_values[0]);
    println!("  Final value: ${:.2}", portfolio_values.last().unwrap());
    println!();
    println!("Maximum Drawdown:");
    println!("  Drawdown: {:.2}%", max_dd);
    println!("  Peak value: ${:.2} (day {})", portfolio_values[peak_idx], peak_idx + 1);
    println!("  Trough value: ${:.2} (day {})", portfolio_values[trough_idx], trough_idx + 1);

    // Calculate current drawdown
    let current_peak = portfolio_values.iter().cloned().fold(0.0_f64, f64::max);
    let current_value = *portfolio_values.last().unwrap();
    let current_dd = calculate_drawdown(current_peak, current_value);

    println!();
    println!("Current Drawdown: {:.2}%", current_dd);
}
```

## Visualizing Drawdowns

```
Day:  1    5    10   15   20   25   30
      |    |    |    |    |    |    |
$150k ─────────────────●─────────────── Peak
                      /│\
$140k ───────────────/ │ \────────────
                    /  │  \
$130k ─────────────●   │   \────●─────
                  /    │    \  /
$120k ────────●──/     │     \/
             /         │
$110k ──────/          │      ●──────── Trough
           /           │      │
$100k ────●            │      │
          Start        │      │
                       |------|
                       Drawdown: 26.7%
                       ($150k → $110k)
```

## Portfolio Tracker with Drawdown Monitoring

```rust
use std::collections::VecDeque;

#[derive(Debug, Clone)]
struct DrawdownInfo {
    current_drawdown: f64,
    max_drawdown: f64,
    peak_value: f64,
    trough_value: f64,
    days_in_drawdown: u32,
    recovery_needed: f64,
}

#[derive(Debug)]
struct PortfolioTracker {
    values: VecDeque<f64>,
    peak_value: f64,
    max_drawdown: f64,
    trough_value: f64,
    days_since_peak: u32,
    max_history: usize,
}

impl PortfolioTracker {
    fn new(initial_value: f64, max_history: usize) -> Self {
        let mut values = VecDeque::with_capacity(max_history);
        values.push_back(initial_value);

        PortfolioTracker {
            values,
            peak_value: initial_value,
            max_drawdown: 0.0,
            trough_value: initial_value,
            days_since_peak: 0,
            max_history,
        }
    }

    fn update(&mut self, new_value: f64) {
        // Add new value to history
        self.values.push_back(new_value);
        if self.values.len() > self.max_history {
            self.values.pop_front();
        }

        // Update peak tracking
        if new_value > self.peak_value {
            self.peak_value = new_value;
            self.days_since_peak = 0;
        } else {
            self.days_since_peak += 1;
        }

        // Calculate current drawdown
        let current_dd = (self.peak_value - new_value) / self.peak_value * 100.0;

        // Update max drawdown if this is worse
        if current_dd > self.max_drawdown {
            self.max_drawdown = current_dd;
            self.trough_value = new_value;
        }
    }

    fn get_drawdown_info(&self) -> DrawdownInfo {
        let current_value = *self.values.back().unwrap_or(&0.0);
        let current_drawdown = (self.peak_value - current_value) / self.peak_value * 100.0;

        // Calculate recovery needed: if portfolio dropped 20%, need 25% gain to recover
        // Formula: (peak / current) - 1
        let recovery_needed = if current_value > 0.0 {
            (self.peak_value / current_value - 1.0) * 100.0
        } else {
            100.0
        };

        DrawdownInfo {
            current_drawdown: current_drawdown.max(0.0),
            max_drawdown: self.max_drawdown,
            peak_value: self.peak_value,
            trough_value: self.trough_value,
            days_in_drawdown: self.days_since_peak,
            recovery_needed: recovery_needed.max(0.0),
        }
    }

    fn get_underwater_chart(&self) -> Vec<f64> {
        let mut peak = self.values[0];
        self.values.iter().map(|&value| {
            if value > peak {
                peak = value;
            }
            -((peak - value) / peak * 100.0)
        }).collect()
    }
}

fn main() {
    let mut tracker = PortfolioTracker::new(100_000.0, 100);

    // Simulate daily portfolio values
    let daily_changes = [
        1.02, 1.01, 0.98, 1.03, 1.02, 0.97, 0.95, 1.01, 1.04, 1.02,
        0.96, 0.94, 0.98, 1.03, 1.05, 1.02, 0.99, 1.01, 1.03, 1.02,
    ];

    let mut current_value = 100_000.0;

    println!("Daily Portfolio Updates:");
    println!("{:>4} {:>12} {:>10} {:>10} {:>12}",
        "Day", "Value", "Daily%", "Drawdown%", "Max DD%");
    println!("{}", "-".repeat(52));

    for (day, &change) in daily_changes.iter().enumerate() {
        let prev_value = current_value;
        current_value *= change;
        tracker.update(current_value);

        let info = tracker.get_drawdown_info();
        let daily_pct = (current_value / prev_value - 1.0) * 100.0;

        println!("{:>4} {:>12.2} {:>+10.2} {:>10.2} {:>12.2}",
            day + 1, current_value, daily_pct, info.current_drawdown, info.max_drawdown);
    }

    let final_info = tracker.get_drawdown_info();
    println!();
    println!("Final Analysis:");
    println!("  Peak Value: ${:.2}", final_info.peak_value);
    println!("  Current Drawdown: {:.2}%", final_info.current_drawdown);
    println!("  Maximum Drawdown: {:.2}%", final_info.max_drawdown);
    println!("  Days in Drawdown: {}", final_info.days_in_drawdown);
    println!("  Recovery Needed: {:.2}%", final_info.recovery_needed);

    // Show underwater chart
    let underwater = tracker.get_underwater_chart();
    println!();
    println!("Underwater Chart (drawdown over time):");
    for (i, &dd) in underwater.iter().enumerate() {
        let bar_len = (-dd * 2.0) as usize;
        let bar: String = "█".repeat(bar_len.min(40));
        println!("Day {:>2}: {:>6.2}% {}", i + 1, dd, bar);
    }
}
```

## Risk Management with Drawdown Limits

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum RiskLevel {
    Normal,
    Elevated,
    High,
    Critical,
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum TradingAction {
    FullPosition,
    ReducedPosition(f64), // Percentage of normal position
    StopTrading,
}

#[derive(Debug)]
struct DrawdownRiskManager {
    max_allowed_drawdown: f64,
    warning_threshold: f64,
    critical_threshold: f64,
    current_drawdown: f64,
    peak_value: f64,
}

impl DrawdownRiskManager {
    fn new(max_allowed_drawdown: f64) -> Self {
        DrawdownRiskManager {
            max_allowed_drawdown,
            warning_threshold: max_allowed_drawdown * 0.5,
            critical_threshold: max_allowed_drawdown * 0.8,
            current_drawdown: 0.0,
            peak_value: 0.0,
        }
    }

    fn update(&mut self, portfolio_value: f64) {
        if portfolio_value > self.peak_value {
            self.peak_value = portfolio_value;
        }

        self.current_drawdown = if self.peak_value > 0.0 {
            (self.peak_value - portfolio_value) / self.peak_value * 100.0
        } else {
            0.0
        };
    }

    fn get_risk_level(&self) -> RiskLevel {
        if self.current_drawdown >= self.max_allowed_drawdown {
            RiskLevel::Critical
        } else if self.current_drawdown >= self.critical_threshold {
            RiskLevel::High
        } else if self.current_drawdown >= self.warning_threshold {
            RiskLevel::Elevated
        } else {
            RiskLevel::Normal
        }
    }

    fn get_trading_action(&self) -> TradingAction {
        match self.get_risk_level() {
            RiskLevel::Normal => TradingAction::FullPosition,
            RiskLevel::Elevated => TradingAction::ReducedPosition(0.75),
            RiskLevel::High => TradingAction::ReducedPosition(0.50),
            RiskLevel::Critical => TradingAction::StopTrading,
        }
    }

    fn calculate_position_size(&self, base_position: f64) -> f64 {
        match self.get_trading_action() {
            TradingAction::FullPosition => base_position,
            TradingAction::ReducedPosition(pct) => base_position * pct,
            TradingAction::StopTrading => 0.0,
        }
    }

    fn get_status(&self) -> String {
        let risk_level = self.get_risk_level();
        let action = self.get_trading_action();

        format!(
            "Drawdown: {:.2}% | Risk: {:?} | Action: {:?}",
            self.current_drawdown, risk_level, action
        )
    }
}

fn main() {
    // Max allowed drawdown of 20%
    let mut risk_manager = DrawdownRiskManager::new(20.0);

    let portfolio_values = [
        100_000.0, 105_000.0, 110_000.0, 108_000.0, 103_000.0,
        98_000.0, 95_000.0, 92_000.0, 89_000.0, 88_000.0,
        90_000.0, 94_000.0, 98_000.0, 102_000.0, 107_000.0,
    ];

    let base_position_size = 10_000.0;

    println!("Drawdown Risk Management Simulation");
    println!("Max Allowed Drawdown: {:.2}%", 20.0);
    println!();
    println!("{:>4} {:>12} {:>10} {:>10} {:>12}",
        "Day", "Portfolio", "DD%", "Risk", "Position");
    println!("{}", "-".repeat(54));

    for (day, &value) in portfolio_values.iter().enumerate() {
        risk_manager.update(value);

        let risk = risk_manager.get_risk_level();
        let position = risk_manager.calculate_position_size(base_position_size);

        println!("{:>4} {:>12.2} {:>10.2} {:>10?} {:>12.2}",
            day + 1, value, risk_manager.current_drawdown, risk, position);
    }
}
```

## Strategy Performance Analysis

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct StrategyMetrics {
    total_return: f64,
    max_drawdown: f64,
    calmar_ratio: f64,     // Annual Return / Max Drawdown
    recovery_factor: f64,   // Net Profit / Max Drawdown
    longest_drawdown_days: u32,
    number_of_drawdowns: u32,
    average_drawdown: f64,
}

#[derive(Debug)]
struct StrategyAnalyzer {
    equity_curve: Vec<f64>,
    drawdowns: Vec<f64>,
    drawdown_periods: Vec<(usize, usize, f64)>, // (start, end, magnitude)
}

impl StrategyAnalyzer {
    fn new() -> Self {
        StrategyAnalyzer {
            equity_curve: Vec::new(),
            drawdowns: Vec::new(),
            drawdown_periods: Vec::new(),
        }
    }

    fn analyze(&mut self, equity_values: &[f64]) -> StrategyMetrics {
        self.equity_curve = equity_values.to_vec();
        self.calculate_drawdowns();
        self.identify_drawdown_periods();

        let total_return = if !equity_values.is_empty() {
            (equity_values.last().unwrap() / equity_values[0] - 1.0) * 100.0
        } else {
            0.0
        };

        let max_drawdown = self.drawdowns.iter().cloned().fold(0.0_f64, f64::max);

        // Assume 252 trading days per year for annualization
        let trading_days = equity_values.len() as f64;
        let annual_return = total_return * (252.0 / trading_days);

        let calmar_ratio = if max_drawdown > 0.0 {
            annual_return / max_drawdown
        } else {
            0.0
        };

        let net_profit = equity_values.last().unwrap_or(&0.0) - equity_values.first().unwrap_or(&0.0);
        let max_dd_absolute = max_drawdown / 100.0 * equity_values.iter().cloned().fold(0.0_f64, f64::max);

        let recovery_factor = if max_dd_absolute > 0.0 {
            net_profit / max_dd_absolute
        } else {
            0.0
        };

        let longest_drawdown = self.drawdown_periods.iter()
            .map(|(start, end, _)| (end - start) as u32)
            .max()
            .unwrap_or(0);

        let avg_drawdown = if !self.drawdown_periods.is_empty() {
            self.drawdown_periods.iter().map(|(_, _, mag)| mag).sum::<f64>()
                / self.drawdown_periods.len() as f64
        } else {
            0.0
        };

        StrategyMetrics {
            total_return,
            max_drawdown,
            calmar_ratio,
            recovery_factor,
            longest_drawdown_days: longest_drawdown,
            number_of_drawdowns: self.drawdown_periods.len() as u32,
            average_drawdown: avg_drawdown,
        }
    }

    fn calculate_drawdowns(&mut self) {
        self.drawdowns.clear();
        let mut peak = self.equity_curve[0];

        for &value in &self.equity_curve {
            if value > peak {
                peak = value;
            }
            let dd = (peak - value) / peak * 100.0;
            self.drawdowns.push(dd);
        }
    }

    fn identify_drawdown_periods(&mut self) {
        self.drawdown_periods.clear();

        let mut in_drawdown = false;
        let mut start_idx = 0;
        let mut max_dd_in_period = 0.0;

        for (i, &dd) in self.drawdowns.iter().enumerate() {
            if dd > 0.0 && !in_drawdown {
                // Starting a new drawdown
                in_drawdown = true;
                start_idx = i;
                max_dd_in_period = dd;
            } else if dd > 0.0 && in_drawdown {
                // Continuing drawdown
                max_dd_in_period = max_dd_in_period.max(dd);
            } else if dd == 0.0 && in_drawdown {
                // Recovered from drawdown
                self.drawdown_periods.push((start_idx, i, max_dd_in_period));
                in_drawdown = false;
            }
        }

        // Handle case where we're still in drawdown at the end
        if in_drawdown {
            self.drawdown_periods.push((start_idx, self.drawdowns.len(), max_dd_in_period));
        }
    }
}

fn main() {
    // Simulate equity curves for different strategies
    let mut strategies: HashMap<&str, Vec<f64>> = HashMap::new();

    // Conservative strategy: steady growth, small drawdowns
    strategies.insert("Conservative", vec![
        100000.0, 100500.0, 101000.0, 100800.0, 101200.0,
        101800.0, 102300.0, 102100.0, 102500.0, 103000.0,
        103500.0, 103300.0, 103800.0, 104200.0, 104800.0,
    ]);

    // Aggressive strategy: higher returns, larger drawdowns
    strategies.insert("Aggressive", vec![
        100000.0, 103000.0, 106000.0, 102000.0, 108000.0,
        112000.0, 105000.0, 110000.0, 118000.0, 115000.0,
        108000.0, 115000.0, 122000.0, 118000.0, 125000.0,
    ]);

    // Volatile strategy: big swings
    strategies.insert("Volatile", vec![
        100000.0, 110000.0, 95000.0, 105000.0, 90000.0,
        100000.0, 115000.0, 100000.0, 120000.0, 105000.0,
        95000.0, 110000.0, 100000.0, 115000.0, 120000.0,
    ]);

    println!("Strategy Performance Comparison");
    println!("{}", "=".repeat(70));

    let mut analyzer = StrategyAnalyzer::new();

    for (name, equity) in &strategies {
        let metrics = analyzer.analyze(equity);

        println!();
        println!("Strategy: {}", name);
        println!("{}", "-".repeat(40));
        println!("  Total Return:        {:>8.2}%", metrics.total_return);
        println!("  Max Drawdown:        {:>8.2}%", metrics.max_drawdown);
        println!("  Calmar Ratio:        {:>8.2}", metrics.calmar_ratio);
        println!("  Recovery Factor:     {:>8.2}", metrics.recovery_factor);
        println!("  Longest DD (days):   {:>8}", metrics.longest_drawdown_days);
        println!("  Number of DDs:       {:>8}", metrics.number_of_drawdowns);
        println!("  Average DD:          {:>8.2}%", metrics.average_drawdown);
    }
}
```

## Practical Example: Real-Time Portfolio Monitor

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

#[derive(Debug, Clone)]
struct Alert {
    timestamp: u64,
    message: String,
    level: AlertLevel,
}

#[derive(Debug, Clone, Copy)]
enum AlertLevel {
    Info,
    Warning,
    Critical,
}

#[derive(Debug)]
struct RealTimePortfolioMonitor {
    portfolio_value: Arc<Mutex<f64>>,
    peak_value: Arc<Mutex<f64>>,
    max_drawdown: Arc<Mutex<f64>>,
    alerts: Arc<Mutex<Vec<Alert>>>,
    drawdown_threshold: f64,
    critical_threshold: f64,
    tick_counter: Arc<Mutex<u64>>,
}

impl RealTimePortfolioMonitor {
    fn new(initial_value: f64, drawdown_threshold: f64, critical_threshold: f64) -> Self {
        RealTimePortfolioMonitor {
            portfolio_value: Arc::new(Mutex::new(initial_value)),
            peak_value: Arc::new(Mutex::new(initial_value)),
            max_drawdown: Arc::new(Mutex::new(0.0)),
            alerts: Arc::new(Mutex::new(Vec::new())),
            drawdown_threshold,
            critical_threshold,
            tick_counter: Arc::new(Mutex::new(0)),
        }
    }

    fn update_price(&self, new_value: f64) {
        let mut value = self.portfolio_value.lock().unwrap();
        let mut peak = self.peak_value.lock().unwrap();
        let mut max_dd = self.max_drawdown.lock().unwrap();
        let mut alerts = self.alerts.lock().unwrap();
        let mut tick = self.tick_counter.lock().unwrap();

        *tick += 1;
        *value = new_value;

        // Update peak if new high
        if new_value > *peak {
            *peak = new_value;
        }

        // Calculate current drawdown
        let current_dd = (*peak - new_value) / *peak * 100.0;

        // Update max drawdown
        if current_dd > *max_dd {
            *max_dd = current_dd;
        }

        // Generate alerts based on thresholds
        if current_dd >= self.critical_threshold {
            alerts.push(Alert {
                timestamp: *tick,
                message: format!(
                    "CRITICAL: Drawdown at {:.2}% exceeds critical threshold of {:.2}%!",
                    current_dd, self.critical_threshold
                ),
                level: AlertLevel::Critical,
            });
        } else if current_dd >= self.drawdown_threshold {
            alerts.push(Alert {
                timestamp: *tick,
                message: format!(
                    "WARNING: Drawdown at {:.2}% exceeds threshold of {:.2}%",
                    current_dd, self.drawdown_threshold
                ),
                level: AlertLevel::Warning,
            });
        }
    }

    fn get_status(&self) -> String {
        let value = *self.portfolio_value.lock().unwrap();
        let peak = *self.peak_value.lock().unwrap();
        let max_dd = *self.max_drawdown.lock().unwrap();

        let current_dd = (peak - value) / peak * 100.0;

        format!(
            "Portfolio: ${:.2} | Peak: ${:.2} | Current DD: {:.2}% | Max DD: {:.2}%",
            value, peak, current_dd, max_dd
        )
    }

    fn get_alerts(&self) -> Vec<Alert> {
        self.alerts.lock().unwrap().clone()
    }
}

fn main() {
    let monitor = Arc::new(RealTimePortfolioMonitor::new(100_000.0, 5.0, 10.0));

    // Simulate market price updates
    let price_updates = [
        100_000.0, 102_000.0, 104_000.0, 103_000.0, 101_000.0,
        99_000.0, 97_000.0, 95_000.0, 93_000.0, 91_000.0,
        93_000.0, 96_000.0, 98_000.0, 101_000.0, 104_000.0,
    ];

    println!("Real-Time Portfolio Monitoring");
    println!("Drawdown Threshold: 5.0% | Critical Threshold: 10.0%");
    println!("{}", "=".repeat(70));
    println!();

    for (i, &price) in price_updates.iter().enumerate() {
        monitor.update_price(price);

        println!("Tick {}: {}", i + 1, monitor.get_status());

        // Check for new alerts
        let alerts = monitor.get_alerts();
        for alert in alerts.iter().filter(|a| a.timestamp == (i + 1) as u64) {
            match alert.level {
                AlertLevel::Critical => println!("  [!!!] {}", alert.message),
                AlertLevel::Warning => println!("  [!] {}", alert.message),
                AlertLevel::Info => println!("  [i] {}", alert.message),
            }
        }

        thread::sleep(Duration::from_millis(100));
    }

    println!();
    println!("Final Status: {}", monitor.get_status());
    println!();
    println!("All Alerts Generated:");
    for alert in monitor.get_alerts() {
        println!("  Tick {}: {:?} - {}", alert.timestamp, alert.level, alert.message);
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Drawdown | Peak-to-trough decline in portfolio value |
| Maximum Drawdown | Largest drop from peak during entire period |
| Recovery Factor | Net profit divided by max drawdown |
| Calmar Ratio | Annual return divided by max drawdown |
| Underwater Chart | Visual representation of drawdown over time |
| Drawdown Duration | Time taken to recover from a drawdown |

## Homework

1. **Drawdown Calculator**: Create a `DrawdownCalculator` struct that:
   - Accepts a vector of daily portfolio values
   - Calculates all types of drawdown (absolute, relative, maximum)
   - Finds all drawdown periods with their start/end dates
   - Returns the average drawdown duration

2. **Risk-Adjusted Returns**: Implement a function that compares multiple trading strategies using:
   - Sharpe Ratio (return per unit of risk)
   - Calmar Ratio (return per unit of drawdown)
   - Sortino Ratio (return per unit of downside risk)
   Print a comparison table ranking strategies by each metric.

3. **Drawdown Alert System**: Build a multi-threaded monitoring system that:
   - Receives real-time price updates via a channel
   - Monitors drawdown in real-time
   - Sends alerts at different severity levels (warning, critical, emergency)
   - Automatically reduces position size when drawdown exceeds thresholds

4. **Monte Carlo Drawdown Analysis**: Create a simulation that:
   - Takes historical daily returns
   - Runs 1000 random simulations of possible future paths
   - Calculates the probability distribution of maximum drawdowns
   - Reports the 95th percentile worst-case drawdown

## Navigation

[← Previous day](../271-portfolio-variance/en.md) | [Next day →](../273-sharpe-ratio/en.md)
