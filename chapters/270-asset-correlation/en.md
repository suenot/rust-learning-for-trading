# Day 270: Asset Correlation

## Trading Analogy

Imagine you're managing a cryptocurrency portfolio. You've noticed an interesting pattern: when Bitcoin rises, Ethereum usually rises too. When oil falls, airline stocks often go up. This relationship between the price movements of different assets is called **correlation**.

Correlation is a statistical measure showing how synchronously two assets move:
- **+1** — perfect positive correlation (assets move together)
- **0** — no correlation (movements are independent)
- **-1** — perfect negative correlation (assets move in opposite directions)

In real trading, correlation helps with:
- Portfolio diversification (selecting uncorrelated assets)
- Risk hedging (using negatively correlated assets)
- Finding arbitrage opportunities (when correlation temporarily breaks down)
- Pairs trading (trading the spread between correlated assets)

## What is Pearson Correlation?

The Pearson correlation coefficient is the most common way to measure linear dependence between two variables:

```
r = Σ((x - x̄)(y - ȳ)) / √(Σ(x - x̄)² × Σ(y - ȳ)²)
```

Where:
- `x̄` and `ȳ` — mean values of the series
- Numerator — covariance
- Denominator — product of standard deviations

## Basic Correlation Calculation in Rust

```rust
/// Calculates the mean of a vector
fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

/// Calculates Pearson correlation coefficient between two series
fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let n = x.len() as f64;
    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None; // One of the series is constant
    }

    Some(covariance / denominator)
}

fn main() {
    // Daily returns of BTC and ETH (in percentages)
    let btc_returns = vec![2.1, -1.5, 0.8, 3.2, -0.5, 1.2, -2.1, 0.3, 1.8, -1.0];
    let eth_returns = vec![2.8, -1.2, 1.1, 4.1, -0.3, 1.5, -1.8, 0.5, 2.2, -0.8];

    match pearson_correlation(&btc_returns, &eth_returns) {
        Some(corr) => {
            println!("BTC/ETH Correlation: {:.4}", corr);

            if corr > 0.7 {
                println!("High positive correlation — assets move together");
            } else if corr > 0.3 {
                println!("Moderate positive correlation");
            } else if corr > -0.3 {
                println!("Weak correlation — assets are relatively independent");
            } else if corr > -0.7 {
                println!("Moderate negative correlation");
            } else {
                println!("High negative correlation — good hedging instrument");
            }
        }
        None => println!("Unable to calculate correlation"),
    }
}
```

## Structure for Portfolio Correlation Analysis

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Asset {
    symbol: String,
    returns: Vec<f64>,
}

#[derive(Debug)]
struct CorrelationMatrix {
    assets: Vec<String>,
    matrix: Vec<Vec<f64>>,
}

impl Asset {
    fn new(symbol: &str, prices: &[f64]) -> Self {
        // Convert prices to returns
        let returns = prices
            .windows(2)
            .map(|w| (w[1] - w[0]) / w[0] * 100.0)
            .collect();

        Asset {
            symbol: symbol.to_string(),
            returns,
        }
    }

    fn from_returns(symbol: &str, returns: Vec<f64>) -> Self {
        Asset {
            symbol: symbol.to_string(),
            returns,
        }
    }
}

impl CorrelationMatrix {
    fn calculate(assets: &[Asset]) -> Option<Self> {
        if assets.is_empty() {
            return None;
        }

        let n = assets.len();
        let mut matrix = vec![vec![0.0; n]; n];
        let symbols: Vec<String> = assets.iter().map(|a| a.symbol.clone()).collect();

        for i in 0..n {
            for j in 0..n {
                if i == j {
                    matrix[i][j] = 1.0; // Correlation with itself = 1
                } else if i < j {
                    let corr = pearson_correlation(&assets[i].returns, &assets[j].returns)
                        .unwrap_or(0.0);
                    matrix[i][j] = corr;
                    matrix[j][i] = corr; // Matrix is symmetric
                }
            }
        }

        Some(CorrelationMatrix {
            assets: symbols,
            matrix,
        })
    }

    fn print(&self) {
        // Header
        print!("{:>10}", "");
        for symbol in &self.assets {
            print!("{:>10}", symbol);
        }
        println!();

        // Data
        for (i, symbol) in self.assets.iter().enumerate() {
            print!("{:>10}", symbol);
            for j in 0..self.assets.len() {
                print!("{:>10.4}", self.matrix[i][j]);
            }
            println!();
        }
    }

    fn get_correlation(&self, asset1: &str, asset2: &str) -> Option<f64> {
        let i = self.assets.iter().position(|a| a == asset1)?;
        let j = self.assets.iter().position(|a| a == asset2)?;
        Some(self.matrix[i][j])
    }

    /// Finds pairs with lowest correlation for diversification
    fn find_diversification_pairs(&self, threshold: f64) -> Vec<(String, String, f64)> {
        let mut pairs = Vec::new();
        let n = self.assets.len();

        for i in 0..n {
            for j in (i + 1)..n {
                if self.matrix[i][j].abs() < threshold {
                    pairs.push((
                        self.assets[i].clone(),
                        self.assets[j].clone(),
                        self.matrix[i][j],
                    ));
                }
            }
        }

        pairs.sort_by(|a, b| a.2.abs().partial_cmp(&b.2.abs()).unwrap());
        pairs
    }

    /// Finds pairs for hedging (negative correlation)
    fn find_hedging_pairs(&self, threshold: f64) -> Vec<(String, String, f64)> {
        let mut pairs = Vec::new();
        let n = self.assets.len();

        for i in 0..n {
            for j in (i + 1)..n {
                if self.matrix[i][j] < -threshold {
                    pairs.push((
                        self.assets[i].clone(),
                        self.assets[j].clone(),
                        self.matrix[i][j],
                    ));
                }
            }
        }

        pairs.sort_by(|a, b| a.2.partial_cmp(&b.2).unwrap());
        pairs
    }
}

fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None;
    }

    Some(covariance / denominator)
}

fn main() {
    // Historical asset prices (sample data)
    let btc_prices = vec![
        42000.0, 42500.0, 41800.0, 43200.0, 44000.0,
        43500.0, 44200.0, 43800.0, 45000.0, 44500.0,
    ];

    let eth_prices = vec![
        2800.0, 2850.0, 2780.0, 2900.0, 2980.0,
        2920.0, 3000.0, 2950.0, 3100.0, 3050.0,
    ];

    let sol_prices = vec![
        100.0, 105.0, 98.0, 110.0, 115.0,
        108.0, 120.0, 112.0, 125.0, 118.0,
    ];

    let gold_prices = vec![
        1950.0, 1945.0, 1960.0, 1940.0, 1935.0,
        1955.0, 1930.0, 1965.0, 1920.0, 1970.0,
    ];

    // Create assets
    let assets = vec![
        Asset::new("BTC", &btc_prices),
        Asset::new("ETH", &eth_prices),
        Asset::new("SOL", &sol_prices),
        Asset::new("GOLD", &gold_prices),
    ];

    // Calculate correlation matrix
    if let Some(corr_matrix) = CorrelationMatrix::calculate(&assets) {
        println!("=== Correlation Matrix ===\n");
        corr_matrix.print();

        println!("\n=== Diversification Pairs (|r| < 0.3) ===");
        for (a1, a2, corr) in corr_matrix.find_diversification_pairs(0.3) {
            println!("{}/{}: {:.4}", a1, a2, corr);
        }

        println!("\n=== Hedging Pairs (r < -0.5) ===");
        for (a1, a2, corr) in corr_matrix.find_hedging_pairs(0.5) {
            println!("{}/{}: {:.4}", a1, a2, corr);
        }

        // Check specific pair
        if let Some(corr) = corr_matrix.get_correlation("BTC", "ETH") {
            println!("\nBTC/ETH Correlation: {:.4}", corr);
        }
    }
}
```

## Rolling Correlation

In real trading, correlation between assets changes over time. Rolling correlation helps track these changes:

```rust
#[derive(Debug)]
struct RollingCorrelation {
    window_size: usize,
    x_buffer: Vec<f64>,
    y_buffer: Vec<f64>,
}

impl RollingCorrelation {
    fn new(window_size: usize) -> Self {
        RollingCorrelation {
            window_size,
            x_buffer: Vec::with_capacity(window_size),
            y_buffer: Vec::with_capacity(window_size),
        }
    }

    fn update(&mut self, x: f64, y: f64) -> Option<f64> {
        self.x_buffer.push(x);
        self.y_buffer.push(y);

        // Remove old data if buffer is full
        if self.x_buffer.len() > self.window_size {
            self.x_buffer.remove(0);
            self.y_buffer.remove(0);
        }

        // Calculate correlation only if buffer is full
        if self.x_buffer.len() == self.window_size {
            pearson_correlation(&self.x_buffer, &self.y_buffer)
        } else {
            None
        }
    }

    fn current_correlation(&self) -> Option<f64> {
        if self.x_buffer.len() == self.window_size {
            pearson_correlation(&self.x_buffer, &self.y_buffer)
        } else {
            None
        }
    }

    fn reset(&mut self) {
        self.x_buffer.clear();
        self.y_buffer.clear();
    }
}

fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None;
    }

    Some(covariance / denominator)
}

fn main() {
    // Simulate stream of BTC and ETH returns
    let btc_returns = vec![
        1.2, -0.8, 0.5, 2.1, -1.5, 0.3, 1.8, -0.4, 0.9, -1.2,
        2.5, -0.9, 0.7, 1.4, -2.0, 0.6, 1.1, -0.3, 0.8, -1.0,
    ];

    let eth_returns = vec![
        1.5, -0.6, 0.8, 2.8, -1.2, 0.5, 2.2, -0.2, 1.2, -0.9,
        3.0, -0.7, 1.0, 1.8, -1.5, 0.9, 1.4, -0.1, 1.1, -0.7,
    ];

    let mut rolling = RollingCorrelation::new(10); // 10-period window

    println!("=== Rolling Correlation BTC/ETH (window = 10) ===\n");
    println!("{:>6} {:>10} {:>10} {:>12}", "Day", "BTC", "ETH", "Correlation");
    println!("{}", "-".repeat(42));

    for (i, (btc, eth)) in btc_returns.iter().zip(eth_returns.iter()).enumerate() {
        let corr = rolling.update(*btc, *eth);

        match corr {
            Some(c) => println!(
                "{:>6} {:>10.2} {:>10.2} {:>12.4}",
                i + 1, btc, eth, c
            ),
            None => println!(
                "{:>6} {:>10.2} {:>10.2} {:>12}",
                i + 1, btc, eth, "insufficient"
            ),
        }
    }
}
```

## Correlation-Based Trading Strategy

```rust
use std::collections::VecDeque;

#[derive(Debug, Clone, Copy, PartialEq)]
enum Signal {
    Long,        // Open long position
    Short,       // Open short position
    Close,       // Close position
    Hold,        // Hold current position
}

#[derive(Debug)]
struct PairsTradingStrategy {
    window_size: usize,
    entry_threshold: f64,      // Threshold for entry (deviation from norm)
    exit_threshold: f64,       // Threshold for exit
    normal_correlation: f64,   // "Normal" correlation for the pair
    x_buffer: VecDeque<f64>,
    y_buffer: VecDeque<f64>,
    in_position: bool,
    position_type: Option<Signal>,
}

impl PairsTradingStrategy {
    fn new(
        window_size: usize,
        entry_threshold: f64,
        exit_threshold: f64,
        normal_correlation: f64,
    ) -> Self {
        PairsTradingStrategy {
            window_size,
            entry_threshold,
            exit_threshold,
            normal_correlation,
            x_buffer: VecDeque::with_capacity(window_size),
            y_buffer: VecDeque::with_capacity(window_size),
            in_position: false,
            position_type: None,
        }
    }

    fn update(&mut self, x_return: f64, y_return: f64) -> Signal {
        self.x_buffer.push_back(x_return);
        self.y_buffer.push_back(y_return);

        if self.x_buffer.len() > self.window_size {
            self.x_buffer.pop_front();
            self.y_buffer.pop_front();
        }

        if self.x_buffer.len() < self.window_size {
            return Signal::Hold;
        }

        let x_vec: Vec<f64> = self.x_buffer.iter().cloned().collect();
        let y_vec: Vec<f64> = self.y_buffer.iter().cloned().collect();

        let current_corr = match pearson_correlation(&x_vec, &y_vec) {
            Some(c) => c,
            None => return Signal::Hold,
        };

        let deviation = current_corr - self.normal_correlation;

        if self.in_position {
            // Check exit condition
            if deviation.abs() < self.exit_threshold {
                self.in_position = false;
                self.position_type = None;
                return Signal::Close;
            }
            Signal::Hold
        } else {
            // Check entry condition
            if deviation > self.entry_threshold {
                // Correlation above normal — expect reversion to norm
                self.in_position = true;
                self.position_type = Some(Signal::Short);
                Signal::Short
            } else if deviation < -self.entry_threshold {
                // Correlation below normal — expect reversion to norm
                self.in_position = true;
                self.position_type = Some(Signal::Long);
                Signal::Long
            } else {
                Signal::Hold
            }
        }
    }

    fn get_current_correlation(&self) -> Option<f64> {
        if self.x_buffer.len() < self.window_size {
            return None;
        }
        let x_vec: Vec<f64> = self.x_buffer.iter().cloned().collect();
        let y_vec: Vec<f64> = self.y_buffer.iter().cloned().collect();
        pearson_correlation(&x_vec, &y_vec)
    }
}

fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None;
    }

    Some(covariance / denominator)
}

fn main() {
    // BTC/ETH pairs trading strategy
    let mut strategy = PairsTradingStrategy::new(
        10,    // Correlation window
        0.15,  // Entry threshold (deviation of 0.15 from norm)
        0.05,  // Exit threshold
        0.85,  // Normal BTC/ETH correlation
    );

    // Simulated returns
    let btc_returns = vec![
        1.2, -0.8, 0.5, 2.1, -1.5, 0.3, 1.8, -0.4, 0.9, -1.2,
        2.5, -0.9, 0.7, 1.4, -2.0, 0.6, 1.1, -0.3, 0.8, -1.0,
        3.0, -1.5, 0.2, 1.0, -0.5,
    ];

    let eth_returns = vec![
        1.5, -0.6, 0.8, 2.8, -1.2, 0.5, 2.2, -0.2, 1.2, -0.9,
        1.0, 0.5, -0.3, 0.2, 0.8,  // Temporary correlation breakdown
        0.9, 1.4, -0.1, 1.1, -0.7,
        2.8, -1.2, 0.5, 1.3, -0.3,
    ];

    println!("=== BTC/ETH Pairs Trading ===\n");
    println!("{:>4} {:>8} {:>8} {:>10} {:>10}",
             "Day", "BTC", "ETH", "Corr.", "Signal");
    println!("{}", "-".repeat(46));

    for (i, (btc, eth)) in btc_returns.iter().zip(eth_returns.iter()).enumerate() {
        let signal = strategy.update(*btc, *eth);
        let corr = strategy.get_current_correlation();

        let corr_str = match corr {
            Some(c) => format!("{:.4}", c),
            None => "N/A".to_string(),
        };

        let signal_str = match signal {
            Signal::Long => "LONG",
            Signal::Short => "SHORT",
            Signal::Close => "CLOSE",
            Signal::Hold => "HOLD",
        };

        println!(
            "{:>4} {:>8.2} {:>8.2} {:>10} {:>10}",
            i + 1, btc, eth, corr_str, signal_str
        );
    }
}
```

## Correlation Heatmap Visualization

```rust
/// Prints a text-based correlation heatmap
fn print_correlation_heatmap(matrix: &[Vec<f64>], symbols: &[String]) {
    println!("\n=== Correlation Heatmap ===\n");

    // Symbols for visualizing correlation levels
    fn corr_to_symbol(corr: f64) -> &'static str {
        if corr > 0.8 { "##" }
        else if corr > 0.6 { "**" }
        else if corr > 0.3 { "++" }
        else if corr > -0.3 { ".." }
        else if corr > -0.6 { "++" }
        else if corr > -0.8 { "**" }
        else { "##" }
    }

    fn corr_to_sign(corr: f64) -> &'static str {
        if corr > 0.3 { "+" }
        else if corr < -0.3 { "-" }
        else { " " }
    }

    // Header
    print!("{:>8}", "");
    for symbol in symbols {
        print!("{:>6}", symbol);
    }
    println!();

    // Data
    for (i, symbol) in symbols.iter().enumerate() {
        print!("{:>8}", symbol);
        for j in 0..symbols.len() {
            let corr = matrix[i][j];
            print!(" {}{}", corr_to_sign(corr), corr_to_symbol(corr));
        }
        println!();
    }

    // Legend
    println!("\nLegend:");
    println!("  +## : strong positive (> 0.8)");
    println!("  +** : moderate positive (0.6 - 0.8)");
    println!("  +++ : weak positive (0.3 - 0.6)");
    println!("   .. : no correlation (-0.3 - 0.3)");
    println!("  -++ : weak negative (-0.6 - -0.3)");
    println!("  -** : moderate negative (-0.8 - -0.6)");
    println!("  -## : strong negative (< -0.8)");
}

fn main() {
    // Sample correlation matrix
    let symbols = vec![
        "BTC".to_string(),
        "ETH".to_string(),
        "SOL".to_string(),
        "GOLD".to_string(),
        "USD".to_string(),
    ];

    let matrix = vec![
        vec![1.00,  0.85,  0.75,  0.10, -0.20],  // BTC
        vec![0.85,  1.00,  0.80,  0.05, -0.15],  // ETH
        vec![0.75,  0.80,  1.00, -0.10, -0.25],  // SOL
        vec![0.10,  0.05, -0.10,  1.00,  0.30],  // GOLD
        vec![-0.20, -0.15, -0.25, 0.30,  1.00],  // USD
    ];

    print_correlation_heatmap(&matrix, &symbols);
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Correlation | Statistical measure of relationship between assets |
| Pearson Coefficient | Value from -1 to +1 measuring linear dependence |
| Correlation Matrix | Table of correlations between all asset pairs |
| Rolling Correlation | Correlation calculated on a sliding window of data |
| Diversification | Using uncorrelated assets to reduce risk |
| Hedging | Using negatively correlated assets for protection |
| Pairs Trading | Strategy based on correlation deviation from norm |

## Homework

1. **Spearman Correlation**: Implement a function to calculate Spearman correlation (rank correlation). It's more robust to outliers. Compare results with Pearson correlation on data with extreme values.

2. **Regime Change Detector**: Create a `CorrelationRegimeDetector` struct that:
   - Tracks rolling correlation between two assets
   - Determines the "normal" correlation range
   - Generates a signal when correlation moves outside the normal range
   - Supports multiple regimes: "normal", "high correlation", "low correlation"

3. **Portfolio Optimizer**: Implement an `optimize_portfolio` function that:
   - Takes a list of assets and their historical returns
   - Calculates the correlation matrix
   - Finds a combination of assets with minimum average correlation
   - Returns asset weights for a diversified portfolio

4. **Real-Time Correlation Monitor**: Create a `CorrelationMonitor` struct with methods:
   - `add_asset(symbol, prices)` — add an asset
   - `update_price(symbol, price)` — update price
   - `get_correlation_matrix()` — current correlation matrix
   - `get_alerts()` — alerts about significant correlation changes

## Navigation

[← Previous day](../269-portfolio-variance/en.md) | [Next day →](../271-covariance-matrix/en.md)
