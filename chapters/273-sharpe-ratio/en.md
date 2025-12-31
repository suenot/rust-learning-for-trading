# Day 273: Sharpe Ratio: Risk-Adjusted Returns

## Trading Analogy

Imagine two traders: Trader A made 50% return last year, while Trader B made only 30%. At first glance, Trader A seems better. But wait — Trader A's portfolio swung wildly between -20% and +80% monthly, causing sleepless nights. Trader B's returns were steady, ranging from +2% to +4% monthly. Who is the better trader?

This is where the **Sharpe Ratio** comes in. Named after Nobel laureate William Sharpe, this metric measures **risk-adjusted returns** — how much return you get for each unit of risk taken. It's like comparing two cars not just by their top speed, but by how smoothly they get there.

In real trading, the Sharpe Ratio helps you:
- Compare strategies with different risk profiles
- Evaluate if extra risk is worth the extra return
- Build portfolios that maximize return per unit of risk

## What is the Sharpe Ratio?

The Sharpe Ratio formula is:

```
Sharpe Ratio = (Rp - Rf) / σp
```

Where:
- **Rp** = Portfolio return (average return of your strategy)
- **Rf** = Risk-free rate (e.g., Treasury bills, typically 2-5% annually)
- **σp** = Standard deviation of portfolio returns (volatility/risk)

### Interpreting Sharpe Ratio Values

| Sharpe Ratio | Interpretation |
|--------------|----------------|
| < 0 | Strategy loses money or underperforms risk-free rate |
| 0 - 1.0 | Suboptimal risk-adjusted return |
| 1.0 - 2.0 | Good — acceptable for most strategies |
| 2.0 - 3.0 | Very good — excellent risk-adjusted performance |
| > 3.0 | Exceptional — rare and should be verified |

## Basic Sharpe Ratio Implementation

```rust
/// Calculate the mean (average) of a slice of f64 values
fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

/// Calculate the standard deviation of a slice of f64 values
fn std_dev(data: &[f64]) -> f64 {
    if data.len() < 2 {
        return 0.0;
    }

    let avg = mean(data);
    let variance = data.iter()
        .map(|x| (x - avg).powi(2))
        .sum::<f64>() / (data.len() - 1) as f64;

    variance.sqrt()
}

/// Calculate the Sharpe Ratio
///
/// # Arguments
/// * `returns` - Slice of periodic returns (e.g., daily, monthly)
/// * `risk_free_rate` - Risk-free rate for the same period
///
/// # Returns
/// The Sharpe Ratio, or 0.0 if calculation is not possible
fn sharpe_ratio(returns: &[f64], risk_free_rate: f64) -> f64 {
    if returns.len() < 2 {
        return 0.0;
    }

    // Calculate excess returns (returns above risk-free rate)
    let excess_returns: Vec<f64> = returns.iter()
        .map(|r| r - risk_free_rate)
        .collect();

    let avg_excess = mean(&excess_returns);
    let volatility = std_dev(&excess_returns);

    if volatility == 0.0 {
        return 0.0;
    }

    avg_excess / volatility
}

fn main() {
    // Example: Monthly returns for a trading strategy
    let monthly_returns = vec![
        0.02, 0.03, -0.01, 0.04, 0.01, 0.02,
        0.03, -0.02, 0.05, 0.02, 0.01, 0.03
    ];

    // Monthly risk-free rate (annual 3% / 12 months)
    let monthly_rf = 0.03 / 12.0;

    let sharpe = sharpe_ratio(&monthly_returns, monthly_rf);

    println!("Monthly Returns Analysis:");
    println!("  Average return: {:.2}%", mean(&monthly_returns) * 100.0);
    println!("  Volatility (std dev): {:.2}%", std_dev(&monthly_returns) * 100.0);
    println!("  Sharpe Ratio: {:.2}", sharpe);

    // Annualize the Sharpe Ratio (multiply by sqrt of periods per year)
    let annualized_sharpe = sharpe * (12.0_f64).sqrt();
    println!("  Annualized Sharpe Ratio: {:.2}", annualized_sharpe);
}
```

## Comparing Trading Strategies

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct TradingStrategy {
    name: String,
    returns: Vec<f64>,
}

impl TradingStrategy {
    fn new(name: &str, returns: Vec<f64>) -> Self {
        TradingStrategy {
            name: name.to_string(),
            returns,
        }
    }

    fn mean_return(&self) -> f64 {
        if self.returns.is_empty() {
            return 0.0;
        }
        self.returns.iter().sum::<f64>() / self.returns.len() as f64
    }

    fn volatility(&self) -> f64 {
        if self.returns.len() < 2 {
            return 0.0;
        }
        let avg = self.mean_return();
        let variance = self.returns.iter()
            .map(|x| (x - avg).powi(2))
            .sum::<f64>() / (self.returns.len() - 1) as f64;
        variance.sqrt()
    }

    fn sharpe_ratio(&self, risk_free_rate: f64) -> f64 {
        let volatility = self.volatility();
        if volatility == 0.0 {
            return 0.0;
        }
        (self.mean_return() - risk_free_rate) / volatility
    }

    fn total_return(&self) -> f64 {
        self.returns.iter()
            .fold(1.0, |acc, r| acc * (1.0 + r)) - 1.0
    }

    fn max_drawdown(&self) -> f64 {
        let mut peak = 1.0;
        let mut max_dd = 0.0;
        let mut value = 1.0;

        for r in &self.returns {
            value *= 1.0 + r;
            if value > peak {
                peak = value;
            }
            let drawdown = (peak - value) / peak;
            if drawdown > max_dd {
                max_dd = drawdown;
            }
        }
        max_dd
    }
}

fn main() {
    // Define three different trading strategies
    let strategies = vec![
        TradingStrategy::new("Momentum", vec![
            0.05, 0.08, -0.03, 0.06, -0.02, 0.10,
            -0.04, 0.07, 0.03, -0.05, 0.09, 0.04
        ]),
        TradingStrategy::new("Mean Reversion", vec![
            0.02, 0.03, 0.01, 0.02, 0.03, 0.01,
            0.02, 0.02, 0.03, 0.01, 0.02, 0.03
        ]),
        TradingStrategy::new("Trend Following", vec![
            0.01, 0.02, 0.04, 0.06, 0.08, 0.03,
            -0.02, -0.03, 0.05, 0.07, 0.04, 0.02
        ]),
    ];

    let monthly_rf = 0.03 / 12.0; // 3% annual risk-free rate

    println!("Strategy Comparison:");
    println!("{:-<70}", "");
    println!("{:<20} {:>10} {:>12} {:>12} {:>12}",
             "Strategy", "Return", "Volatility", "Sharpe", "Max DD");
    println!("{:-<70}", "");

    for strategy in &strategies {
        let sharpe = strategy.sharpe_ratio(monthly_rf);
        let annualized_sharpe = sharpe * (12.0_f64).sqrt();

        println!("{:<20} {:>9.1}% {:>11.1}% {:>12.2} {:>11.1}%",
                 strategy.name,
                 strategy.total_return() * 100.0,
                 strategy.volatility() * 100.0,
                 annualized_sharpe,
                 strategy.max_drawdown() * 100.0);
    }
    println!("{:-<70}", "");

    // Find the best strategy by Sharpe Ratio
    let best = strategies.iter()
        .max_by(|a, b| {
            a.sharpe_ratio(monthly_rf)
                .partial_cmp(&b.sharpe_ratio(monthly_rf))
                .unwrap()
        })
        .unwrap();

    println!("\nBest risk-adjusted strategy: {}", best.name);
}
```

## Rolling Sharpe Ratio for Real-Time Analysis

```rust
use std::collections::VecDeque;

struct RollingSharpe {
    window_size: usize,
    returns: VecDeque<f64>,
    risk_free_rate: f64,
}

impl RollingSharpe {
    fn new(window_size: usize, risk_free_rate: f64) -> Self {
        RollingSharpe {
            window_size,
            returns: VecDeque::with_capacity(window_size),
            risk_free_rate,
        }
    }

    fn add_return(&mut self, ret: f64) {
        if self.returns.len() >= self.window_size {
            self.returns.pop_front();
        }
        self.returns.push_back(ret);
    }

    fn calculate(&self) -> Option<f64> {
        if self.returns.len() < 2 {
            return None;
        }

        let returns_vec: Vec<f64> = self.returns.iter().cloned().collect();

        // Calculate excess returns
        let excess: Vec<f64> = returns_vec.iter()
            .map(|r| r - self.risk_free_rate)
            .collect();

        let mean = excess.iter().sum::<f64>() / excess.len() as f64;
        let variance = excess.iter()
            .map(|x| (x - mean).powi(2))
            .sum::<f64>() / (excess.len() - 1) as f64;
        let std_dev = variance.sqrt();

        if std_dev == 0.0 {
            return None;
        }

        Some(mean / std_dev)
    }

    fn is_ready(&self) -> bool {
        self.returns.len() >= self.window_size
    }
}

fn main() {
    // Simulate daily returns from a trading bot
    let daily_returns = vec![
        0.005, 0.008, -0.003, 0.012, -0.002, 0.007, 0.003,
        -0.004, 0.009, 0.002, -0.006, 0.011, 0.004, -0.001,
        0.006, 0.003, -0.008, 0.010, 0.005, -0.003, 0.007,
        0.002, 0.008, -0.002, 0.006, 0.004, -0.005, 0.009,
    ];

    // 10-day rolling Sharpe with daily risk-free rate
    let daily_rf = 0.03 / 252.0; // 252 trading days
    let mut rolling = RollingSharpe::new(10, daily_rf);

    println!("Rolling 10-Day Sharpe Ratio:");
    println!("{:-<40}", "");

    for (day, &ret) in daily_returns.iter().enumerate() {
        rolling.add_return(ret);

        if rolling.is_ready() {
            if let Some(sharpe) = rolling.calculate() {
                let status = if sharpe > 2.0 {
                    "Excellent"
                } else if sharpe > 1.0 {
                    "Good"
                } else if sharpe > 0.0 {
                    "Suboptimal"
                } else {
                    "Poor"
                };

                println!("Day {:2}: Sharpe = {:>6.2} ({})",
                         day + 1, sharpe, status);
            }
        }
    }
}
```

## Portfolio Optimization Using Sharpe Ratio

```rust
use std::f64::consts::PI;

#[derive(Debug, Clone)]
struct Asset {
    name: String,
    expected_return: f64,  // Annual expected return
    volatility: f64,       // Annual volatility (std dev)
}

#[derive(Debug)]
struct Portfolio {
    assets: Vec<Asset>,
    weights: Vec<f64>,
    correlation_matrix: Vec<Vec<f64>>,
}

impl Portfolio {
    fn new(assets: Vec<Asset>, correlation_matrix: Vec<Vec<f64>>) -> Self {
        let n = assets.len();
        let weights = vec![1.0 / n as f64; n]; // Equal weights initially
        Portfolio {
            assets,
            weights,
            correlation_matrix,
        }
    }

    fn set_weights(&mut self, weights: Vec<f64>) {
        assert_eq!(weights.len(), self.assets.len());
        let sum: f64 = weights.iter().sum();
        self.weights = weights.iter().map(|w| w / sum).collect();
    }

    fn expected_return(&self) -> f64 {
        self.assets.iter()
            .zip(self.weights.iter())
            .map(|(asset, weight)| asset.expected_return * weight)
            .sum()
    }

    fn portfolio_volatility(&self) -> f64 {
        let n = self.assets.len();
        let mut variance = 0.0;

        for i in 0..n {
            for j in 0..n {
                variance += self.weights[i] * self.weights[j]
                    * self.assets[i].volatility * self.assets[j].volatility
                    * self.correlation_matrix[i][j];
            }
        }

        variance.sqrt()
    }

    fn sharpe_ratio(&self, risk_free_rate: f64) -> f64 {
        let vol = self.portfolio_volatility();
        if vol == 0.0 {
            return 0.0;
        }
        (self.expected_return() - risk_free_rate) / vol
    }
}

fn optimize_portfolio(assets: Vec<Asset>, correlation: Vec<Vec<f64>>,
                      risk_free_rate: f64) -> (Vec<f64>, f64) {
    let mut portfolio = Portfolio::new(assets, correlation);
    let n = portfolio.assets.len();

    let mut best_weights = portfolio.weights.clone();
    let mut best_sharpe = portfolio.sharpe_ratio(risk_free_rate);

    // Simple grid search optimization
    // In practice, use quadratic programming or gradient descent
    let steps = 20;

    if n == 2 {
        for i in 0..=steps {
            let w1 = i as f64 / steps as f64;
            let w2 = 1.0 - w1;

            portfolio.set_weights(vec![w1, w2]);
            let sharpe = portfolio.sharpe_ratio(risk_free_rate);

            if sharpe > best_sharpe {
                best_sharpe = sharpe;
                best_weights = portfolio.weights.clone();
            }
        }
    } else if n == 3 {
        for i in 0..=steps {
            for j in 0..=(steps - i) {
                let w1 = i as f64 / steps as f64;
                let w2 = j as f64 / steps as f64;
                let w3 = 1.0 - w1 - w2;

                if w3 >= 0.0 {
                    portfolio.set_weights(vec![w1, w2, w3]);
                    let sharpe = portfolio.sharpe_ratio(risk_free_rate);

                    if sharpe > best_sharpe {
                        best_sharpe = sharpe;
                        best_weights = portfolio.weights.clone();
                    }
                }
            }
        }
    }

    (best_weights, best_sharpe)
}

fn main() {
    let assets = vec![
        Asset {
            name: "BTC".to_string(),
            expected_return: 0.50,  // 50% expected annual return
            volatility: 0.80,       // 80% annual volatility
        },
        Asset {
            name: "ETH".to_string(),
            expected_return: 0.60,  // 60% expected annual return
            volatility: 0.90,       // 90% annual volatility
        },
        Asset {
            name: "Stable Strategy".to_string(),
            expected_return: 0.15,  // 15% expected annual return
            volatility: 0.20,       // 20% annual volatility
        },
    ];

    // Correlation matrix
    let correlation = vec![
        vec![1.0, 0.7, 0.2],   // BTC correlations
        vec![0.7, 1.0, 0.3],   // ETH correlations
        vec![0.2, 0.3, 1.0],   // Stable correlations
    ];

    let risk_free_rate = 0.05; // 5% annual

    println!("Portfolio Optimization for Maximum Sharpe Ratio");
    println!("{:-<60}", "");

    // Show individual asset Sharpe ratios
    println!("\nIndividual Asset Sharpe Ratios:");
    for asset in &assets {
        let sharpe = (asset.expected_return - risk_free_rate) / asset.volatility;
        println!("  {}: {:.2}", asset.name, sharpe);
    }

    // Find optimal weights
    let (optimal_weights, optimal_sharpe) =
        optimize_portfolio(assets.clone(), correlation.clone(), risk_free_rate);

    println!("\nOptimal Portfolio Allocation:");
    for (asset, weight) in assets.iter().zip(optimal_weights.iter()) {
        println!("  {}: {:.1}%", asset.name, weight * 100.0);
    }

    println!("\nOptimal Portfolio Sharpe Ratio: {:.2}", optimal_sharpe);

    // Create and show optimal portfolio stats
    let mut optimal_portfolio = Portfolio::new(assets, correlation);
    optimal_portfolio.set_weights(optimal_weights);

    println!("\nOptimal Portfolio Metrics:");
    println!("  Expected Return: {:.1}%",
             optimal_portfolio.expected_return() * 100.0);
    println!("  Volatility: {:.1}%",
             optimal_portfolio.portfolio_volatility() * 100.0);
}
```

## Practical Trading Application: Strategy Monitor

```rust
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
struct Trade {
    timestamp: u64,
    symbol: String,
    pnl_percent: f64,
}

#[derive(Debug)]
struct StrategyMonitor {
    name: String,
    trades: Vec<Trade>,
    daily_returns: Vec<f64>,
    risk_free_rate: f64,
    min_sharpe_threshold: f64,
}

impl StrategyMonitor {
    fn new(name: &str, annual_rf: f64, min_sharpe: f64) -> Self {
        StrategyMonitor {
            name: name.to_string(),
            trades: Vec::new(),
            daily_returns: Vec::new(),
            risk_free_rate: annual_rf / 252.0, // Convert to daily
            min_sharpe_threshold: min_sharpe,
        }
    }

    fn record_trade(&mut self, symbol: &str, pnl_percent: f64) {
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();

        self.trades.push(Trade {
            timestamp,
            symbol: symbol.to_string(),
            pnl_percent,
        });
    }

    fn close_day(&mut self, daily_return: f64) {
        self.daily_returns.push(daily_return);
    }

    fn calculate_sharpe(&self) -> Option<f64> {
        if self.daily_returns.len() < 5 {
            return None; // Need at least 5 days of data
        }

        let excess: Vec<f64> = self.daily_returns.iter()
            .map(|r| r - self.risk_free_rate)
            .collect();

        let mean = excess.iter().sum::<f64>() / excess.len() as f64;
        let variance = excess.iter()
            .map(|x| (x - mean).powi(2))
            .sum::<f64>() / (excess.len() - 1) as f64;
        let std_dev = variance.sqrt();

        if std_dev == 0.0 {
            return None;
        }

        // Annualize: multiply by sqrt(252)
        Some((mean / std_dev) * (252.0_f64).sqrt())
    }

    fn get_status(&self) -> StrategyStatus {
        match self.calculate_sharpe() {
            None => StrategyStatus::InsufficientData,
            Some(sharpe) if sharpe >= self.min_sharpe_threshold => {
                StrategyStatus::Healthy(sharpe)
            }
            Some(sharpe) if sharpe > 0.0 => {
                StrategyStatus::Warning(sharpe)
            }
            Some(sharpe) => {
                StrategyStatus::Critical(sharpe)
            }
        }
    }

    fn generate_report(&self) -> String {
        let mut report = format!("Strategy Report: {}\n", self.name);
        report.push_str(&format!("{:-<50}\n", ""));

        report.push_str(&format!("Total trades: {}\n", self.trades.len()));
        report.push_str(&format!("Days tracked: {}\n", self.daily_returns.len()));

        if !self.daily_returns.is_empty() {
            let total_return: f64 = self.daily_returns.iter()
                .fold(1.0, |acc, r| acc * (1.0 + r)) - 1.0;
            report.push_str(&format!("Total return: {:.2}%\n", total_return * 100.0));
        }

        match self.get_status() {
            StrategyStatus::InsufficientData => {
                report.push_str("Status: Insufficient data for Sharpe calculation\n");
            }
            StrategyStatus::Healthy(sharpe) => {
                report.push_str(&format!("Status: HEALTHY\n"));
                report.push_str(&format!("Annualized Sharpe: {:.2}\n", sharpe));
            }
            StrategyStatus::Warning(sharpe) => {
                report.push_str(&format!("Status: WARNING - Below threshold\n"));
                report.push_str(&format!("Annualized Sharpe: {:.2}\n", sharpe));
            }
            StrategyStatus::Critical(sharpe) => {
                report.push_str(&format!("Status: CRITICAL - Negative Sharpe!\n"));
                report.push_str(&format!("Annualized Sharpe: {:.2}\n", sharpe));
            }
        }

        report
    }
}

#[derive(Debug)]
enum StrategyStatus {
    InsufficientData,
    Healthy(f64),
    Warning(f64),
    Critical(f64),
}

fn main() {
    let mut monitor = StrategyMonitor::new(
        "Crypto Momentum Bot",
        0.05,  // 5% annual risk-free rate
        1.5,   // Minimum acceptable Sharpe
    );

    // Simulate 20 trading days
    let simulated_daily_returns = vec![
        0.02, 0.01, -0.005, 0.015, 0.008,
        0.012, -0.01, 0.02, 0.005, 0.018,
        -0.008, 0.025, 0.01, -0.003, 0.015,
        0.007, 0.02, -0.012, 0.018, 0.01,
    ];

    println!("Simulating 20 trading days...\n");

    for (day, &daily_return) in simulated_daily_returns.iter().enumerate() {
        // Record some trades
        monitor.record_trade("BTC/USDT", daily_return * 0.6);
        monitor.record_trade("ETH/USDT", daily_return * 0.4);

        // Close the day
        monitor.close_day(daily_return);

        // Check status every 5 days
        if (day + 1) % 5 == 0 {
            println!("Day {} checkpoint:", day + 1);
            match monitor.get_status() {
                StrategyStatus::InsufficientData => {
                    println!("  Gathering data...\n");
                }
                StrategyStatus::Healthy(sharpe) => {
                    println!("  Sharpe: {:.2} - Strategy performing well\n", sharpe);
                }
                StrategyStatus::Warning(sharpe) => {
                    println!("  Sharpe: {:.2} - Consider adjusting parameters\n", sharpe);
                }
                StrategyStatus::Critical(sharpe) => {
                    println!("  Sharpe: {:.2} - STOP TRADING!\n", sharpe);
                }
            }
        }
    }

    println!("{}", monitor.generate_report());
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Sharpe Ratio | Measures excess return per unit of risk |
| Formula | (Portfolio Return - Risk-Free Rate) / Volatility |
| Annualization | Multiply by sqrt(periods per year) |
| Rolling Sharpe | Tracks strategy performance over time |
| Portfolio Optimization | Find weights that maximize Sharpe Ratio |
| Strategy Monitoring | Use Sharpe thresholds for risk management |

## Exercises

1. **Sharpe Calculator**: Implement a function that takes a CSV file of daily prices and calculates the annualized Sharpe Ratio. Handle edge cases like missing data and zero volatility.

2. **Strategy Comparison Dashboard**: Create a struct that holds multiple strategies and ranks them by Sharpe Ratio. Add methods to filter strategies by minimum Sharpe threshold.

3. **Dynamic Risk-Free Rate**: Modify the `StrategyMonitor` to accept a time series of risk-free rates instead of a single value. This reflects real-world scenarios where rates change over time.

4. **Sharpe Decay Detection**: Implement a function that detects when a strategy's rolling Sharpe Ratio has declined by more than 20% from its peak. This is useful for identifying strategies that may need reoptimization.

## Homework

1. **Modified Sharpe Ratio**: Research and implement the **Sortino Ratio**, which only penalizes downside volatility. Compare results between Sharpe and Sortino for a strategy with asymmetric returns.

2. **Monte Carlo Simulation**: Create a program that:
   - Generates 1000 random return series using a given mean and volatility
   - Calculates the Sharpe Ratio for each series
   - Plots a histogram of Sharpe Ratio distribution
   - Determines confidence intervals for the "true" Sharpe

3. **Multi-Asset Optimizer**: Extend the portfolio optimization example to:
   - Accept any number of assets
   - Use gradient descent instead of grid search
   - Add constraints (e.g., max 30% in any single asset)
   - Output an efficient frontier (risk vs. return chart)

4. **Real-Time Alert System**: Build a trading bot monitor that:
   - Tracks multiple strategies simultaneously
   - Calculates 30-day rolling Sharpe Ratios
   - Sends alerts when Sharpe drops below threshold
   - Automatically reduces position sizes for underperforming strategies

## Navigation

[← Previous day](../272-risk-parity-balancing/en.md) | [Next day →](../274-sortino-ratio-downside-risk/en.md)
