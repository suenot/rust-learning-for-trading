# Day 271: Volatility: Measuring Volatility

## Trading Analogy

Imagine you're a sailor planning a voyage. Before setting sail, you want to know how rough the seas have been lately. A calm sea with small, predictable waves is very different from a stormy ocean with massive, unpredictable swells. **Volatility** in trading is exactly this — it measures how "rough" the market seas are for a particular asset.

A stock like a utility company might be a calm pond — prices move slowly and predictably. A cryptocurrency might be a stormy ocean — prices can swing 10% in a single hour. Understanding and measuring volatility helps traders:
- Size their positions appropriately (smaller positions in stormy markets)
- Set stop-losses at reasonable distances
- Price options and other derivatives
- Identify trading opportunities (high volatility = more potential profit, but also more risk)

## What is Volatility?

**Volatility** is a statistical measure of the dispersion of returns for a given asset. In simpler terms, it tells us how much the price typically moves from its average.

### Types of Volatility

1. **Historical Volatility** — calculated from past price data
2. **Implied Volatility** — derived from options prices, represents market expectations
3. **Realized Volatility** — actual volatility that occurred over a period

In this chapter, we'll focus on **Historical Volatility** as it's the foundation for understanding market risk.

## The Mathematics of Volatility

The most common way to measure volatility is through **standard deviation** of returns:

1. Calculate returns: `r_t = (P_t - P_{t-1}) / P_{t-1}` or `r_t = ln(P_t / P_{t-1})`
2. Calculate mean return: `μ = sum(r) / n`
3. Calculate variance: `σ² = sum((r - μ)²) / (n-1)`
4. Volatility is the standard deviation: `σ = sqrt(σ²)`

To annualize daily volatility: `σ_annual = σ_daily * sqrt(252)` (252 trading days per year)

## Basic Volatility Calculator in Rust

Let's build a volatility calculator from scratch:

```rust
/// Represents a single price point with timestamp
#[derive(Debug, Clone)]
struct PricePoint {
    timestamp: u64,
    price: f64,
}

/// Volatility calculation results
#[derive(Debug)]
struct VolatilityMetrics {
    daily_volatility: f64,
    annualized_volatility: f64,
    mean_return: f64,
    max_return: f64,
    min_return: f64,
}

/// Calculate logarithmic returns from price data
fn calculate_returns(prices: &[f64]) -> Vec<f64> {
    if prices.len() < 2 {
        return vec![];
    }

    prices
        .windows(2)
        .map(|window| (window[1] / window[0]).ln())
        .collect()
}

/// Calculate mean of a slice
fn mean(values: &[f64]) -> f64 {
    if values.is_empty() {
        return 0.0;
    }
    values.iter().sum::<f64>() / values.len() as f64
}

/// Calculate standard deviation (sample)
fn std_deviation(values: &[f64]) -> f64 {
    if values.len() < 2 {
        return 0.0;
    }

    let avg = mean(values);
    let variance = values
        .iter()
        .map(|x| (x - avg).powi(2))
        .sum::<f64>()
        / (values.len() - 1) as f64;

    variance.sqrt()
}

/// Calculate comprehensive volatility metrics
fn calculate_volatility(prices: &[f64]) -> Option<VolatilityMetrics> {
    let returns = calculate_returns(prices);

    if returns.is_empty() {
        return None;
    }

    let daily_vol = std_deviation(&returns);
    let annualized_vol = daily_vol * (252.0_f64).sqrt();

    Some(VolatilityMetrics {
        daily_volatility: daily_vol,
        annualized_volatility: annualized_vol,
        mean_return: mean(&returns),
        max_return: returns.iter().cloned().fold(f64::NEG_INFINITY, f64::max),
        min_return: returns.iter().cloned().fold(f64::INFINITY, f64::min),
    })
}

fn main() {
    // Simulated BTC daily closing prices
    let btc_prices = vec![
        42000.0, 42500.0, 41800.0, 43200.0, 44100.0,
        43800.0, 45000.0, 44200.0, 46000.0, 45500.0,
        47000.0, 46200.0, 48000.0, 47500.0, 49000.0,
    ];

    // Simulated stable stock prices
    let stable_stock_prices = vec![
        100.0, 100.2, 100.1, 100.3, 100.4,
        100.3, 100.5, 100.4, 100.6, 100.5,
        100.7, 100.6, 100.8, 100.7, 100.9,
    ];

    println!("=== Volatility Analysis ===\n");

    if let Some(btc_vol) = calculate_volatility(&btc_prices) {
        println!("Bitcoin (BTC):");
        println!("  Daily Volatility:     {:.4} ({:.2}%)",
            btc_vol.daily_volatility, btc_vol.daily_volatility * 100.0);
        println!("  Annualized Volatility: {:.4} ({:.2}%)",
            btc_vol.annualized_volatility, btc_vol.annualized_volatility * 100.0);
        println!("  Mean Daily Return:    {:.4} ({:.2}%)",
            btc_vol.mean_return, btc_vol.mean_return * 100.0);
        println!("  Max Single Day Return: {:.2}%", btc_vol.max_return * 100.0);
        println!("  Min Single Day Return: {:.2}%", btc_vol.min_return * 100.0);
    }

    println!();

    if let Some(stock_vol) = calculate_volatility(&stable_stock_prices) {
        println!("Stable Stock:");
        println!("  Daily Volatility:     {:.4} ({:.2}%)",
            stock_vol.daily_volatility, stock_vol.daily_volatility * 100.0);
        println!("  Annualized Volatility: {:.4} ({:.2}%)",
            stock_vol.annualized_volatility, stock_vol.annualized_volatility * 100.0);
        println!("  Mean Daily Return:    {:.4} ({:.2}%)",
            stock_vol.mean_return, stock_vol.mean_return * 100.0);
    }
}
```

## Rolling Window Volatility

In real trading, we often want to track how volatility changes over time using a **rolling window**:

```rust
/// Calculate rolling volatility with a specified window size
fn rolling_volatility(prices: &[f64], window_size: usize) -> Vec<f64> {
    if prices.len() < window_size + 1 {
        return vec![];
    }

    let returns = calculate_returns(prices);

    returns
        .windows(window_size)
        .map(|window| std_deviation(window))
        .collect()
}

/// Volatility regime classification
#[derive(Debug, Clone, PartialEq)]
enum VolatilityRegime {
    Low,
    Normal,
    High,
    Extreme,
}

impl VolatilityRegime {
    fn from_volatility(vol: f64, historical_avg: f64) -> Self {
        let ratio = vol / historical_avg;
        match ratio {
            r if r < 0.5 => VolatilityRegime::Low,
            r if r < 1.5 => VolatilityRegime::Normal,
            r if r < 2.5 => VolatilityRegime::High,
            _ => VolatilityRegime::Extreme,
        }
    }

    fn position_size_multiplier(&self) -> f64 {
        match self {
            VolatilityRegime::Low => 1.5,      // Can take larger positions
            VolatilityRegime::Normal => 1.0,   // Standard position size
            VolatilityRegime::High => 0.5,     // Reduce position size
            VolatilityRegime::Extreme => 0.25, // Minimal positions
        }
    }
}

fn main() {
    let prices = vec![
        100.0, 102.0, 101.0, 103.0, 105.0, 104.0, 106.0, 108.0,
        107.0, 110.0, 108.0, 112.0, 115.0, 113.0, 118.0, 120.0,
        118.0, 122.0, 125.0, 123.0, 128.0, 130.0, 127.0, 132.0,
    ];

    let window = 5;
    let rolling_vol = rolling_volatility(&prices, window);

    println!("Rolling {}-day volatility:", window);
    for (i, vol) in rolling_vol.iter().enumerate() {
        let annualized = vol * (252.0_f64).sqrt();
        println!("  Day {}: {:.4} (annualized: {:.2}%)",
            i + window + 1, vol, annualized * 100.0);
    }

    // Calculate average volatility for regime detection
    let avg_vol = mean(&rolling_vol);
    println!("\nAverage rolling volatility: {:.4}", avg_vol);

    // Classify current regime
    if let Some(&current_vol) = rolling_vol.last() {
        let regime = VolatilityRegime::from_volatility(current_vol, avg_vol);
        println!("Current regime: {:?}", regime);
        println!("Recommended position size multiplier: {:.2}x",
            regime.position_size_multiplier());
    }
}
```

## Exponentially Weighted Moving Average (EWMA) Volatility

EWMA gives more weight to recent observations, making it more responsive to current market conditions:

```rust
/// Calculate EWMA volatility
/// lambda: decay factor (typically 0.94 for daily data per RiskMetrics)
fn ewma_volatility(returns: &[f64], lambda: f64) -> Vec<f64> {
    if returns.is_empty() {
        return vec![];
    }

    let mut ewma_var = vec![0.0; returns.len()];

    // Initialize with first return squared
    ewma_var[0] = returns[0].powi(2);

    // Calculate EWMA variance recursively
    for i in 1..returns.len() {
        ewma_var[i] = lambda * ewma_var[i - 1] + (1.0 - lambda) * returns[i].powi(2);
    }

    // Return volatility (sqrt of variance)
    ewma_var.iter().map(|v| v.sqrt()).collect()
}

/// Compare different volatility estimation methods
fn compare_volatility_methods(prices: &[f64]) {
    let returns = calculate_returns(prices);

    if returns.len() < 5 {
        println!("Not enough data for comparison");
        return;
    }

    // Simple standard deviation
    let simple_vol = std_deviation(&returns);

    // EWMA with different decay factors
    let ewma_094 = ewma_volatility(&returns, 0.94);
    let ewma_097 = ewma_volatility(&returns, 0.97);

    println!("Volatility Method Comparison:");
    println!("  Simple Std Dev:    {:.4}", simple_vol);
    println!("  EWMA (λ=0.94):     {:.4}", ewma_094.last().unwrap_or(&0.0));
    println!("  EWMA (λ=0.97):     {:.4}", ewma_097.last().unwrap_or(&0.0));
}

fn main() {
    // Simulated prices with a volatility spike in the middle
    let prices: Vec<f64> = vec![
        100.0, 101.0, 100.5, 101.5, 102.0,  // Calm period
        102.5, 108.0, 103.0, 110.0, 105.0,  // Volatile period
        106.0, 106.5, 107.0, 107.5, 108.0,  // Calm again
    ];

    compare_volatility_methods(&prices);

    let returns = calculate_returns(&prices);
    let ewma_vol = ewma_volatility(&returns, 0.94);

    println!("\nEWMA Volatility Evolution:");
    for (i, vol) in ewma_vol.iter().enumerate() {
        let bar = "█".repeat((vol * 200.0) as usize);
        println!("  Day {:2}: {:.4} {}", i + 2, vol, bar);
    }
}
```

## Practical Trading Application: Volatility-Based Position Sizing

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct Asset {
    symbol: String,
    prices: Vec<f64>,
    current_price: f64,
}

#[derive(Debug)]
struct PositionSizer {
    total_capital: f64,
    risk_per_trade: f64,  // As a fraction (e.g., 0.02 for 2%)
    volatility_window: usize,
}

impl PositionSizer {
    fn new(capital: f64, risk_fraction: f64, window: usize) -> Self {
        PositionSizer {
            total_capital: capital,
            risk_per_trade: risk_fraction,
            volatility_window: window,
        }
    }

    /// Calculate position size based on volatility
    fn calculate_position(&self, asset: &Asset) -> Option<PositionResult> {
        if asset.prices.len() < self.volatility_window + 1 {
            return None;
        }

        let returns = calculate_returns(&asset.prices);
        let recent_returns: Vec<f64> = returns
            .iter()
            .rev()
            .take(self.volatility_window)
            .copied()
            .collect();

        let volatility = std_deviation(&recent_returns);
        let atr_estimate = volatility * asset.current_price;

        // Risk amount in dollars
        let risk_amount = self.total_capital * self.risk_per_trade;

        // Position size: risk_amount / (volatility * price)
        // This means we're risking the same dollar amount regardless of volatility
        let stop_distance = 2.0 * atr_estimate; // 2x volatility for stop
        let shares = risk_amount / stop_distance;
        let position_value = shares * asset.current_price;

        Some(PositionResult {
            symbol: asset.symbol.clone(),
            volatility,
            annualized_volatility: volatility * (252.0_f64).sqrt(),
            shares: shares.floor(),
            position_value,
            stop_loss_price: asset.current_price - stop_distance,
            risk_amount,
        })
    }
}

#[derive(Debug)]
struct PositionResult {
    symbol: String,
    volatility: f64,
    annualized_volatility: f64,
    shares: f64,
    position_value: f64,
    stop_loss_price: f64,
    risk_amount: f64,
}

fn main() {
    let sizer = PositionSizer::new(100_000.0, 0.02, 20);

    // High volatility asset (crypto)
    let btc = Asset {
        symbol: "BTC".to_string(),
        prices: vec![
            40000.0, 41000.0, 39500.0, 42000.0, 43500.0,
            42000.0, 44000.0, 43000.0, 45000.0, 44500.0,
            46000.0, 45000.0, 47000.0, 46500.0, 48000.0,
            47000.0, 49000.0, 48000.0, 50000.0, 49500.0,
            51000.0,
        ],
        current_price: 51000.0,
    };

    // Low volatility asset (stable stock)
    let stable = Asset {
        symbol: "STABLE".to_string(),
        prices: vec![
            100.0, 100.2, 100.1, 100.3, 100.4,
            100.3, 100.5, 100.4, 100.6, 100.5,
            100.7, 100.6, 100.8, 100.7, 100.9,
            100.8, 101.0, 100.9, 101.1, 101.0,
            101.2,
        ],
        current_price: 101.2,
    };

    println!("=== Volatility-Based Position Sizing ===");
    println!("Capital: ${:.0}", sizer.total_capital);
    println!("Risk per trade: {:.1}%\n", sizer.risk_per_trade * 100.0);

    for asset in [&btc, &stable] {
        if let Some(result) = sizer.calculate_position(asset) {
            println!("{}:", result.symbol);
            println!("  Current Price:        ${:.2}", asset.current_price);
            println!("  Daily Volatility:     {:.2}%", result.volatility * 100.0);
            println!("  Annualized Volatility: {:.2}%", result.annualized_volatility * 100.0);
            println!("  Recommended Shares:   {:.0}", result.shares);
            println!("  Position Value:       ${:.2}", result.position_value);
            println!("  Stop Loss Price:      ${:.2}", result.stop_loss_price);
            println!("  Risk Amount:          ${:.2}", result.risk_amount);
            println!();
        }
    }
}
```

## Volatility Indicators for Trading Signals

```rust
/// Bollinger Bands calculation
#[derive(Debug)]
struct BollingerBands {
    upper: f64,
    middle: f64,
    lower: f64,
    bandwidth: f64,
}

fn calculate_bollinger_bands(prices: &[f64], period: usize, num_std: f64) -> Option<BollingerBands> {
    if prices.len() < period {
        return None;
    }

    let recent: Vec<f64> = prices.iter().rev().take(period).copied().collect();
    let sma = mean(&recent);
    let std = std_deviation(&recent);

    let upper = sma + num_std * std;
    let lower = sma - num_std * std;
    let bandwidth = (upper - lower) / sma;

    Some(BollingerBands {
        upper,
        middle: sma,
        lower,
        bandwidth,
    })
}

/// Trading signal based on volatility
#[derive(Debug, Clone)]
enum VolatilitySignal {
    Squeeze,           // Low volatility, breakout expected
    Expansion,         // High volatility, trend in progress
    MeanReversion,     // Price at band extremes
    Neutral,
}

fn analyze_volatility_signal(
    current_price: f64,
    bands: &BollingerBands,
    historical_bandwidth: f64,
) -> VolatilitySignal {
    // Squeeze: bandwidth is significantly below average
    if bands.bandwidth < historical_bandwidth * 0.5 {
        return VolatilitySignal::Squeeze;
    }

    // Expansion: bandwidth is significantly above average
    if bands.bandwidth > historical_bandwidth * 1.5 {
        return VolatilitySignal::Expansion;
    }

    // Mean reversion: price is at band extremes
    let position = (current_price - bands.lower) / (bands.upper - bands.lower);
    if position > 0.95 || position < 0.05 {
        return VolatilitySignal::MeanReversion;
    }

    VolatilitySignal::Neutral
}

fn main() {
    let prices = vec![
        100.0, 102.0, 101.0, 103.0, 104.0, 103.5, 105.0, 104.5,
        106.0, 105.5, 107.0, 106.5, 108.0, 107.5, 109.0, 108.5,
        110.0, 109.5, 111.0, 110.5,
    ];

    let period = 10;
    let num_std = 2.0;

    // Calculate Bollinger Bands for each point
    println!("Bollinger Bands Analysis (period={}, std={}):\n", period, num_std);

    let mut bandwidths = Vec::new();

    for i in period..=prices.len() {
        let slice = &prices[..i];
        if let Some(bands) = calculate_bollinger_bands(slice, period, num_std) {
            bandwidths.push(bands.bandwidth);
            let current_price = prices[i - 1];

            println!("Day {}:", i);
            println!("  Price:  ${:.2}", current_price);
            println!("  Upper:  ${:.2}", bands.upper);
            println!("  Middle: ${:.2}", bands.middle);
            println!("  Lower:  ${:.2}", bands.lower);
            println!("  Bandwidth: {:.4}", bands.bandwidth);

            if bandwidths.len() > 1 {
                let avg_bandwidth = mean(&bandwidths);
                let signal = analyze_volatility_signal(current_price, &bands, avg_bandwidth);
                println!("  Signal: {:?}", signal);
            }
            println!();
        }
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Volatility | Statistical measure of price dispersion |
| Standard Deviation | Most common volatility measure |
| Log Returns | `ln(P_t / P_{t-1})` — preferred for financial calculations |
| Annualization | Multiply daily vol by √252 for yearly estimate |
| Rolling Volatility | Track volatility changes over time |
| EWMA | Gives more weight to recent observations |
| Position Sizing | Adjust position size inversely with volatility |
| Bollinger Bands | Price channels based on volatility |

## Exercises

1. **Basic Volatility Calculator**: Implement a function that takes a vector of prices and returns both simple and log returns volatility. Test it with at least 3 different assets.

2. **Volatility Comparison**: Create a program that compares the volatility of two assets and determines which one is riskier. Include visualization using ASCII charts.

3. **ATR Implementation**: Implement the Average True Range (ATR) indicator, which uses high, low, and close prices:
   ```
   TR = max(high - low, |high - prev_close|, |low - prev_close|)
   ATR = rolling_mean(TR, period)
   ```

4. **Volatility Alert System**: Build a system that monitors volatility and triggers alerts when:
   - Volatility exceeds 2x the 30-day average
   - Volatility drops below 0.5x the 30-day average
   - Volatility changes by more than 50% in a single day

## Homework

1. **Parkinson Volatility**: Implement the Parkinson volatility estimator which uses high and low prices instead of just close prices:
   ```
   σ² = (1/4ln(2)) * mean((ln(high/low))²)
   ```
   Compare its results with standard deviation on the same dataset.

2. **Volatility Targeting Strategy**: Create a trading system that:
   - Targets a specific portfolio volatility (e.g., 15% annualized)
   - Adjusts position sizes daily based on rolling volatility
   - Tracks the realized volatility to see how close you get to the target

3. **Volatility Clustering**: Implement a detector for volatility clustering (GARCH effect) — the phenomenon where high volatility days tend to be followed by high volatility days. Calculate the autocorrelation of squared returns.

4. **Multi-Asset Volatility Dashboard**: Build a dashboard that tracks volatility for multiple assets and shows:
   - Current volatility vs 30-day average
   - Volatility percentile (where current vol ranks historically)
   - Correlation between assets' volatilities
   - Suggested portfolio allocation based on inverse volatility weighting

## Navigation

[← Previous day](../270-risk-metrics-foundations/en.md) | [Next day →](../272-volatility-models/en.md)
