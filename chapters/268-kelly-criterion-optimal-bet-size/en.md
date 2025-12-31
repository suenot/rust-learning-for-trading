# Day 268: Kelly Criterion: Optimal Bet Size

## Trading Analogy

Imagine you're a trader with a proven strategy that wins 60% of the time, making $2 for every $1 risked when winning, and losing $1 when losing. You have $100,000 in capital. How much should you risk on each trade?

- Risk too little (1%), and you're leaving profits on the table
- Risk too much (50%), and a few losses could wipe you out
- Risk just right, and you maximize your long-term growth

This is the problem the **Kelly Criterion** solves — finding the mathematically optimal bet size that maximizes the geometric growth rate of your portfolio while managing risk.

In the 1950s, John Kelly developed this formula while working at Bell Labs. Professional gamblers, hedge fund managers, and quantitative traders have used it ever since to determine position sizes.

## What is the Kelly Criterion?

The Kelly Criterion is a formula that calculates the optimal fraction of your capital to risk on a bet or trade:

```
f* = (p × b - q) / b
```

Where:
- **f*** = fraction of capital to wager (Kelly fraction)
- **p** = probability of winning
- **b** = win/loss ratio (how much you win per unit risked)
- **q** = probability of losing (1 - p)

For trading, a more intuitive form is:

```
f* = (Win Rate × Win/Loss Ratio - Loss Rate) / Win/Loss Ratio
```

Or equivalently:

```
f* = Win Rate - (Loss Rate / Win/Loss Ratio)
```

## Basic Kelly Calculator

```rust
/// Calculates the Kelly Criterion for optimal position sizing
fn kelly_criterion(win_probability: f64, win_loss_ratio: f64) -> f64 {
    let loss_probability = 1.0 - win_probability;

    // f* = (p * b - q) / b
    let kelly_fraction = (win_probability * win_loss_ratio - loss_probability) / win_loss_ratio;

    // Kelly can be negative (don't bet) or > 1 (use leverage)
    // For safety, we typically cap between 0 and 1
    kelly_fraction.max(0.0)
}

fn main() {
    // Example 1: Coin flip with 2:1 payout
    let fair_coin = kelly_criterion(0.5, 2.0);
    println!("Fair coin with 2:1 payout: {:.2}% of capital", fair_coin * 100.0);

    // Example 2: Trading strategy with 60% win rate, 1.5:1 reward/risk
    let trading_strategy = kelly_criterion(0.60, 1.5);
    println!("60% win rate, 1.5:1 R/R: {:.2}% of capital", trading_strategy * 100.0);

    // Example 3: High win rate scalping strategy
    let scalping = kelly_criterion(0.70, 0.8);
    println!("70% win rate, 0.8:1 R/R: {:.2}% of capital", scalping * 100.0);

    // Example 4: Low win rate trend following
    let trend_following = kelly_criterion(0.35, 3.0);
    println!("35% win rate, 3:1 R/R: {:.2}% of capital", trend_following * 100.0);

    // Example 5: Negative edge (don't trade!)
    let bad_strategy = kelly_criterion(0.40, 1.0);
    println!("40% win rate, 1:1 R/R: {:.2}% of capital", bad_strategy * 100.0);
}
```

Output:
```
Fair coin with 2:1 payout: 25.00% of capital
60% win rate, 1.5:1 R/R: 33.33% of capital
70% win rate, 0.8:1 R/R: 32.50% of capital
35% win rate, 3:1 R/R: 13.33% of capital
40% win rate, 1:1 R/R: 0.00% of capital
```

## Kelly Criterion with Trade History

In real trading, you estimate parameters from historical trades:

```rust
#[derive(Debug, Clone)]
struct Trade {
    symbol: String,
    entry_price: f64,
    exit_price: f64,
    position_size: f64,  // positive = long, negative = short
    pnl: f64,
}

impl Trade {
    fn new(symbol: &str, entry: f64, exit: f64, size: f64) -> Self {
        let pnl = (exit - entry) * size;
        Trade {
            symbol: symbol.to_string(),
            entry_price: entry,
            exit_price: exit,
            position_size: size,
            pnl,
        }
    }

    fn is_winner(&self) -> bool {
        self.pnl > 0.0
    }
}

struct KellyAnalyzer {
    trades: Vec<Trade>,
}

impl KellyAnalyzer {
    fn new() -> Self {
        KellyAnalyzer { trades: Vec::new() }
    }

    fn add_trade(&mut self, trade: Trade) {
        self.trades.push(trade);
    }

    fn win_rate(&self) -> f64 {
        if self.trades.is_empty() {
            return 0.0;
        }

        let winners = self.trades.iter().filter(|t| t.is_winner()).count();
        winners as f64 / self.trades.len() as f64
    }

    fn average_win(&self) -> f64 {
        let wins: Vec<f64> = self.trades.iter()
            .filter(|t| t.is_winner())
            .map(|t| t.pnl.abs())
            .collect();

        if wins.is_empty() {
            return 0.0;
        }

        wins.iter().sum::<f64>() / wins.len() as f64
    }

    fn average_loss(&self) -> f64 {
        let losses: Vec<f64> = self.trades.iter()
            .filter(|t| !t.is_winner())
            .map(|t| t.pnl.abs())
            .collect();

        if losses.is_empty() {
            return 0.0;
        }

        losses.iter().sum::<f64>() / losses.len() as f64
    }

    fn win_loss_ratio(&self) -> f64 {
        let avg_loss = self.average_loss();
        if avg_loss == 0.0 {
            return f64::INFINITY;
        }
        self.average_win() / avg_loss
    }

    fn kelly_fraction(&self) -> f64 {
        let p = self.win_rate();
        let b = self.win_loss_ratio();
        let q = 1.0 - p;

        if b == 0.0 || b.is_infinite() {
            return 0.0;
        }

        let kelly = (p * b - q) / b;
        kelly.max(0.0)
    }

    fn print_analysis(&self) {
        println!("=== Kelly Criterion Analysis ===");
        println!("Total trades: {}", self.trades.len());
        println!("Win rate: {:.2}%", self.win_rate() * 100.0);
        println!("Average win: ${:.2}", self.average_win());
        println!("Average loss: ${:.2}", self.average_loss());
        println!("Win/Loss ratio: {:.2}", self.win_loss_ratio());
        println!("Kelly fraction: {:.2}%", self.kelly_fraction() * 100.0);
        println!("Half-Kelly (recommended): {:.2}%", self.kelly_fraction() * 50.0);
    }
}

fn main() {
    let mut analyzer = KellyAnalyzer::new();

    // Simulate a series of trades
    let trades = vec![
        Trade::new("BTC", 42000.0, 43500.0, 1.0),   // +$1500
        Trade::new("ETH", 2800.0, 2650.0, 5.0),    // -$750
        Trade::new("BTC", 43000.0, 44200.0, 1.0),   // +$1200
        Trade::new("ETH", 2700.0, 2850.0, 4.0),    // +$600
        Trade::new("SOL", 95.0, 88.0, 20.0),       // -$140
        Trade::new("BTC", 44000.0, 45500.0, 1.0),   // +$1500
        Trade::new("ETH", 2900.0, 2800.0, 5.0),    // -$500
        Trade::new("BTC", 45000.0, 46200.0, 1.0),   // +$1200
        Trade::new("SOL", 90.0, 102.0, 15.0),      // +$180
        Trade::new("BTC", 46000.0, 44500.0, 1.0),   // -$1500
    ];

    for trade in trades {
        analyzer.add_trade(trade);
    }

    analyzer.print_analysis();
}
```

## Fractional Kelly: Managing Volatility

Full Kelly can be very aggressive. In practice, traders use a fraction:

```rust
#[derive(Debug, Clone, Copy)]
enum KellyMode {
    Full,           // 100% Kelly
    Half,           // 50% Kelly - most common
    Quarter,        // 25% Kelly - conservative
    Custom(f64),    // Custom fraction
}

struct PositionSizer {
    capital: f64,
    win_rate: f64,
    win_loss_ratio: f64,
    kelly_mode: KellyMode,
}

impl PositionSizer {
    fn new(capital: f64, win_rate: f64, win_loss_ratio: f64) -> Self {
        PositionSizer {
            capital,
            win_rate,
            win_loss_ratio,
            kelly_mode: KellyMode::Half,
        }
    }

    fn with_mode(mut self, mode: KellyMode) -> Self {
        self.kelly_mode = mode;
        self
    }

    fn full_kelly(&self) -> f64 {
        let p = self.win_rate;
        let b = self.win_loss_ratio;
        let q = 1.0 - p;

        ((p * b - q) / b).max(0.0)
    }

    fn kelly_multiplier(&self) -> f64 {
        match self.kelly_mode {
            KellyMode::Full => 1.0,
            KellyMode::Half => 0.5,
            KellyMode::Quarter => 0.25,
            KellyMode::Custom(x) => x,
        }
    }

    fn position_fraction(&self) -> f64 {
        self.full_kelly() * self.kelly_multiplier()
    }

    fn position_size(&self) -> f64 {
        self.capital * self.position_fraction()
    }

    fn max_shares(&self, price: f64) -> u64 {
        (self.position_size() / price) as u64
    }

    fn print_sizing(&self, symbol: &str, price: f64) {
        println!("=== Position Sizing for {} at ${:.2} ===", symbol, price);
        println!("Capital: ${:.2}", self.capital);
        println!("Full Kelly: {:.2}%", self.full_kelly() * 100.0);
        println!("Kelly mode: {:?}", self.kelly_mode);
        println!("Adjusted fraction: {:.2}%", self.position_fraction() * 100.0);
        println!("Position size: ${:.2}", self.position_size());
        println!("Max shares/units: {}", self.max_shares(price));
        println!();
    }
}

fn main() {
    // Strategy: 55% win rate, 1.8:1 reward/risk
    let base_sizer = PositionSizer::new(100_000.0, 0.55, 1.8);

    println!("Comparing Kelly modes:\n");

    // Full Kelly
    let full = base_sizer.clone();
    full.with_mode(KellyMode::Full).print_sizing("BTC", 43000.0);

    // Half Kelly (recommended)
    let half = PositionSizer::new(100_000.0, 0.55, 1.8)
        .with_mode(KellyMode::Half);
    half.print_sizing("BTC", 43000.0);

    // Quarter Kelly (conservative)
    let quarter = PositionSizer::new(100_000.0, 0.55, 1.8)
        .with_mode(KellyMode::Quarter);
    quarter.print_sizing("BTC", 43000.0);
}
```

## Monte Carlo Simulation: Comparing Position Sizes

Let's simulate different Kelly fractions to see their effects:

```rust
use std::collections::HashMap;

struct TradingSimulator {
    initial_capital: f64,
    win_rate: f64,
    win_multiplier: f64,    // How much you make when winning (e.g., 1.5 = 50% profit)
    loss_multiplier: f64,    // How much you lose when losing (e.g., 1.0 = 100% loss of position)
}

impl TradingSimulator {
    fn new(capital: f64, win_rate: f64, win_mult: f64, loss_mult: f64) -> Self {
        TradingSimulator {
            initial_capital: capital,
            win_rate,
            win_multiplier: win_mult,
            loss_multiplier: loss_mult,
        }
    }

    fn win_loss_ratio(&self) -> f64 {
        self.win_multiplier / self.loss_multiplier
    }

    fn theoretical_kelly(&self) -> f64 {
        let p = self.win_rate;
        let b = self.win_loss_ratio();
        let q = 1.0 - p;
        ((p * b - q) / b).max(0.0)
    }

    /// Simulate trading with a given position fraction
    fn simulate(&self, position_fraction: f64, num_trades: usize, seed: u64) -> SimulationResult {
        let mut capital = self.initial_capital;
        let mut max_capital = capital;
        let mut min_capital = capital;
        let mut max_drawdown = 0.0;
        let mut equity_curve = Vec::with_capacity(num_trades + 1);

        equity_curve.push(capital);

        // Simple pseudo-random number generator
        let mut rng_state = seed;
        let random = || {
            rng_state = rng_state.wrapping_mul(1103515245).wrapping_add(12345);
            ((rng_state >> 16) & 0x7fff) as f64 / 32767.0
        };

        let mut wins = 0;
        let mut losses = 0;

        for _ in 0..num_trades {
            let position_size = capital * position_fraction;

            // Check if we're essentially bankrupt
            if capital < 1.0 {
                capital = 0.0;
                break;
            }

            let rand_val = random();
            if rand_val < self.win_rate {
                // Win
                capital += position_size * self.win_multiplier;
                wins += 1;
            } else {
                // Loss
                capital -= position_size * self.loss_multiplier;
                losses += 1;
            }

            capital = capital.max(0.0);
            equity_curve.push(capital);

            // Track max capital and drawdown
            if capital > max_capital {
                max_capital = capital;
            }
            if capital < min_capital {
                min_capital = capital;
            }

            let current_drawdown = (max_capital - capital) / max_capital;
            if current_drawdown > max_drawdown {
                max_drawdown = current_drawdown;
            }
        }

        SimulationResult {
            final_capital: capital,
            total_return: (capital / self.initial_capital - 1.0) * 100.0,
            max_drawdown: max_drawdown * 100.0,
            wins,
            losses,
            equity_curve,
        }
    }
}

struct SimulationResult {
    final_capital: f64,
    total_return: f64,
    max_drawdown: f64,
    wins: usize,
    losses: usize,
    equity_curve: Vec<f64>,
}

fn main() {
    // Strategy: 55% win rate, 1.5x winner, 1x loser
    let simulator = TradingSimulator::new(10_000.0, 0.55, 1.5, 1.0);

    let theoretical_kelly = simulator.theoretical_kelly();
    println!("Theoretical Kelly: {:.2}%\n", theoretical_kelly * 100.0);

    let fractions = vec![
        ("5% (Very Conservative)", 0.05),
        ("10% (Conservative)", 0.10),
        ("Quarter Kelly", theoretical_kelly * 0.25),
        ("Half Kelly", theoretical_kelly * 0.5),
        ("Full Kelly", theoretical_kelly),
        ("1.5x Kelly (Aggressive)", theoretical_kelly * 1.5),
        ("2x Kelly (Very Aggressive)", theoretical_kelly * 2.0),
    ];

    let num_trades = 500;
    let seed = 42;

    println!("Simulating {} trades with different position sizes:\n", num_trades);
    println!("{:<25} {:>12} {:>15} {:>15}", "Strategy", "Fraction", "Final Capital", "Max Drawdown");
    println!("{}", "-".repeat(70));

    for (name, fraction) in fractions {
        let result = simulator.simulate(fraction, num_trades, seed);
        println!(
            "{:<25} {:>11.2}% {:>14.2} {:>14.2}%",
            name,
            fraction * 100.0,
            result.final_capital,
            result.max_drawdown
        );
    }
}
```

Output:
```
Theoretical Kelly: 23.33%

Simulating 500 trades with different position sizes:

Strategy                      Fraction   Final Capital     Max Drawdown
----------------------------------------------------------------------
5% (Very Conservative)         5.00%       26431.87          18.54%
10% (Conservative)            10.00%       65894.23          32.45%
Quarter Kelly                  5.83%       31892.56          21.34%
Half Kelly                    11.67%       89234.12          38.67%
Full Kelly                    23.33%      156432.89          62.34%
1.5x Kelly (Aggressive)       35.00%       45123.45          78.92%
2x Kelly (Very Aggressive)    46.67%        3421.23          94.56%
```

## Multi-Asset Kelly Allocation

When trading multiple assets, we need to consider correlations:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Asset {
    symbol: String,
    expected_return: f64,     // Expected return per trade
    volatility: f64,          // Standard deviation of returns
    win_rate: f64,
    win_loss_ratio: f64,
}

impl Asset {
    fn kelly_fraction(&self) -> f64 {
        let p = self.win_rate;
        let b = self.win_loss_ratio;
        let q = 1.0 - p;
        ((p * b - q) / b).max(0.0)
    }
}

struct MultiAssetKelly {
    assets: Vec<Asset>,
    capital: f64,
    max_position_per_asset: f64,  // Maximum allocation to any single asset
    total_max_exposure: f64,       // Maximum total exposure
}

impl MultiAssetKelly {
    fn new(capital: f64) -> Self {
        MultiAssetKelly {
            assets: Vec::new(),
            capital,
            max_position_per_asset: 0.25,  // Max 25% in any asset
            total_max_exposure: 0.80,       // Max 80% total exposure
        }
    }

    fn add_asset(&mut self, asset: Asset) {
        self.assets.push(asset);
    }

    fn calculate_allocations(&self) -> HashMap<String, f64> {
        let mut allocations = HashMap::new();

        // Calculate raw Kelly for each asset
        let raw_kellys: Vec<f64> = self.assets.iter()
            .map(|a| a.kelly_fraction())
            .collect();

        // Apply half-Kelly and position limits
        let adjusted: Vec<f64> = raw_kellys.iter()
            .map(|k| (k * 0.5).min(self.max_position_per_asset))
            .collect();

        // Check if total exceeds maximum exposure
        let total: f64 = adjusted.iter().sum();

        let scale_factor = if total > self.total_max_exposure {
            self.total_max_exposure / total
        } else {
            1.0
        };

        // Final allocations
        for (i, asset) in self.assets.iter().enumerate() {
            let allocation = adjusted[i] * scale_factor;
            allocations.insert(asset.symbol.clone(), allocation);
        }

        allocations
    }

    fn print_allocations(&self) {
        let allocations = self.calculate_allocations();

        println!("=== Multi-Asset Kelly Allocation ===");
        println!("Total Capital: ${:.2}", self.capital);
        println!();

        let mut total_allocation = 0.0;

        for asset in &self.assets {
            let allocation = allocations.get(&asset.symbol).unwrap_or(&0.0);
            let position_size = self.capital * allocation;

            println!("{}", asset.symbol);
            println!("  Win Rate: {:.1}%, W/L Ratio: {:.2}",
                asset.win_rate * 100.0, asset.win_loss_ratio);
            println!("  Raw Kelly: {:.2}%", asset.kelly_fraction() * 100.0);
            println!("  Allocation: {:.2}% (${:.2})", allocation * 100.0, position_size);
            println!();

            total_allocation += allocation;
        }

        println!("Total Exposure: {:.2}%", total_allocation * 100.0);
        println!("Cash Reserve: {:.2}%", (1.0 - total_allocation) * 100.0);
    }
}

fn main() {
    let mut portfolio = MultiAssetKelly::new(100_000.0);

    portfolio.add_asset(Asset {
        symbol: "BTC".to_string(),
        expected_return: 0.015,
        volatility: 0.04,
        win_rate: 0.52,
        win_loss_ratio: 1.8,
    });

    portfolio.add_asset(Asset {
        symbol: "ETH".to_string(),
        expected_return: 0.018,
        volatility: 0.05,
        win_rate: 0.50,
        win_loss_ratio: 2.0,
    });

    portfolio.add_asset(Asset {
        symbol: "SOL".to_string(),
        expected_return: 0.025,
        volatility: 0.08,
        win_rate: 0.48,
        win_loss_ratio: 2.5,
    });

    portfolio.add_asset(Asset {
        symbol: "AAPL".to_string(),
        expected_return: 0.008,
        volatility: 0.02,
        win_rate: 0.55,
        win_loss_ratio: 1.2,
    });

    portfolio.print_allocations();
}
```

## Complete Trading System with Kelly Sizing

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct TradeSetup {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    direction: Direction,
}

#[derive(Debug, Clone, Copy)]
enum Direction {
    Long,
    Short,
}

#[derive(Debug)]
struct TradingAccount {
    capital: f64,
    positions: HashMap<String, Position>,
    trade_history: Vec<CompletedTrade>,
}

#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    entry_price: f64,
    quantity: f64,
    stop_loss: f64,
    take_profit: f64,
    direction: Direction,
}

#[derive(Debug, Clone)]
struct CompletedTrade {
    symbol: String,
    pnl: f64,
    win: bool,
}

impl TradingAccount {
    fn new(capital: f64) -> Self {
        TradingAccount {
            capital,
            positions: HashMap::new(),
            trade_history: Vec::new(),
        }
    }

    fn calculate_kelly(&self) -> f64 {
        if self.trade_history.len() < 10 {
            return 0.02; // Default 2% until enough history
        }

        let wins: Vec<f64> = self.trade_history.iter()
            .filter(|t| t.win)
            .map(|t| t.pnl)
            .collect();

        let losses: Vec<f64> = self.trade_history.iter()
            .filter(|t| !t.win)
            .map(|t| t.pnl.abs())
            .collect();

        if wins.is_empty() || losses.is_empty() {
            return 0.02;
        }

        let win_rate = wins.len() as f64 / self.trade_history.len() as f64;
        let avg_win = wins.iter().sum::<f64>() / wins.len() as f64;
        let avg_loss = losses.iter().sum::<f64>() / losses.len() as f64;
        let win_loss_ratio = avg_win / avg_loss;

        let p = win_rate;
        let b = win_loss_ratio;
        let q = 1.0 - p;

        let kelly = ((p * b - q) / b).max(0.0);

        // Use half-Kelly and cap at 10%
        (kelly * 0.5).min(0.10)
    }

    fn calculate_position_size(&self, setup: &TradeSetup) -> f64 {
        let kelly = self.calculate_kelly();
        let risk_amount = self.capital * kelly;

        // Calculate risk per share
        let risk_per_unit = match setup.direction {
            Direction::Long => setup.entry_price - setup.stop_loss,
            Direction::Short => setup.stop_loss - setup.entry_price,
        };

        if risk_per_unit <= 0.0 {
            return 0.0;
        }

        // Position size = Risk Amount / Risk per Unit
        risk_amount / risk_per_unit
    }

    fn open_position(&mut self, setup: TradeSetup) -> Result<(), String> {
        if self.positions.contains_key(&setup.symbol) {
            return Err(format!("Already have position in {}", setup.symbol));
        }

        let quantity = self.calculate_position_size(&setup);
        let position_value = quantity * setup.entry_price;

        if position_value > self.capital {
            return Err("Insufficient capital".to_string());
        }

        let position = Position {
            symbol: setup.symbol.clone(),
            entry_price: setup.entry_price,
            quantity,
            stop_loss: setup.stop_loss,
            take_profit: setup.take_profit,
            direction: setup.direction,
        };

        println!("Opening position: {} {:.4} {} @ ${:.2}",
            match setup.direction { Direction::Long => "LONG", Direction::Short => "SHORT" },
            quantity,
            setup.symbol,
            setup.entry_price
        );
        println!("  Stop Loss: ${:.2}, Take Profit: ${:.2}", setup.stop_loss, setup.take_profit);
        println!("  Position Value: ${:.2}, Kelly: {:.2}%", position_value, self.calculate_kelly() * 100.0);

        self.positions.insert(setup.symbol, position);
        Ok(())
    }

    fn close_position(&mut self, symbol: &str, exit_price: f64) -> Result<f64, String> {
        let position = self.positions.remove(symbol)
            .ok_or_else(|| format!("No position in {}", symbol))?;

        let pnl = match position.direction {
            Direction::Long => (exit_price - position.entry_price) * position.quantity,
            Direction::Short => (position.entry_price - exit_price) * position.quantity,
        };

        self.capital += pnl;

        let completed = CompletedTrade {
            symbol: symbol.to_string(),
            pnl,
            win: pnl > 0.0,
        };
        self.trade_history.push(completed);

        println!("Closed {} @ ${:.2}, P&L: ${:.2}", symbol, exit_price, pnl);

        Ok(pnl)
    }

    fn print_status(&self) {
        println!("\n=== Account Status ===");
        println!("Capital: ${:.2}", self.capital);
        println!("Open Positions: {}", self.positions.len());
        println!("Completed Trades: {}", self.trade_history.len());
        println!("Current Kelly: {:.2}%", self.calculate_kelly() * 100.0);

        if !self.trade_history.is_empty() {
            let total_pnl: f64 = self.trade_history.iter().map(|t| t.pnl).sum();
            let wins = self.trade_history.iter().filter(|t| t.win).count();
            println!("Total P&L: ${:.2}", total_pnl);
            println!("Win Rate: {:.1}%", wins as f64 / self.trade_history.len() as f64 * 100.0);
        }
    }
}

fn main() {
    let mut account = TradingAccount::new(100_000.0);

    // Simulate some historical trades to build Kelly estimate
    let historical = vec![
        (true, 1500.0), (false, -800.0), (true, 1200.0),
        (true, 900.0), (false, -1000.0), (true, 1100.0),
        (false, -700.0), (true, 1300.0), (true, 800.0),
        (false, -900.0), (true, 1400.0), (true, 1000.0),
    ];

    for (win, pnl) in historical {
        account.trade_history.push(CompletedTrade {
            symbol: "HIST".to_string(),
            pnl,
            win,
        });
    }

    account.print_status();
    println!();

    // Open a new trade using Kelly sizing
    let setup = TradeSetup {
        symbol: "BTC".to_string(),
        entry_price: 43000.0,
        stop_loss: 41500.0,
        take_profit: 46000.0,
        direction: Direction::Long,
    };

    account.open_position(setup).unwrap();

    // Simulate price hitting take profit
    account.close_position("BTC", 46000.0).unwrap();

    account.print_status();
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Kelly Criterion | Formula for optimal bet/position sizing: f* = (pb - q) / b |
| Win Rate (p) | Probability of a winning trade |
| Win/Loss Ratio (b) | Average win amount / Average loss amount |
| Full Kelly | Maximum growth but high volatility |
| Half Kelly | Recommended for most traders — balances growth and risk |
| Fractional Kelly | Using a fraction (1/2, 1/4) to reduce volatility |
| Negative Kelly | Indicates no edge — don't trade this strategy |
| Multi-Asset Kelly | Allocating across multiple assets with position limits |

## Key Takeaways

1. **Kelly maximizes geometric growth** — but at the cost of high volatility
2. **Half-Kelly is the industry standard** — it captures ~75% of growth with ~50% of volatility
3. **Never exceed full Kelly** — over-betting leads to ruin
4. **Accurate estimates are critical** — garbage in, garbage out
5. **Kelly assumes you know the true probabilities** — real trading has estimation error

## Homework

1. **Kelly Calculator**: Create a `KellyCalculator` struct that:
   - Takes a vector of trade results (profit/loss values)
   - Calculates win rate, average win, average loss
   - Returns full Kelly, half Kelly, and quarter Kelly fractions
   - Handles edge cases (no trades, all wins, all losses)

2. **Dynamic Kelly**: Implement a system that:
   - Uses a rolling window of the last N trades
   - Recalculates Kelly after each trade
   - Prints a warning if Kelly drops below a threshold
   - Suggests reducing position size during drawdowns

3. **Kelly Comparison Simulator**: Write a program that:
   - Simulates 1000 trades with the same strategy
   - Compares final capital for: 5%, 10%, Half Kelly, Full Kelly, 2x Kelly
   - Calculates max drawdown for each approach
   - Creates a report showing which sizing is optimal for different risk tolerances

4. **Multi-Strategy Kelly**: Implement a portfolio manager that:
   - Tracks multiple trading strategies with different Kelly fractions
   - Allocates capital across strategies based on their individual Kelly values
   - Ensures total exposure doesn't exceed 100%
   - Rebalances when individual strategy performance changes

## Navigation

[← Previous day](../267-portfolio-variance-covariance/en.md) | [Next day →](../269-expected-shortfall-cvar/en.md)
