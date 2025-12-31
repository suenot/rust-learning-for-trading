# Day 275: What is Backtesting

## Trading Analogy

Imagine you've devised a new trading strategy: buy Bitcoin every time the price drops by 5%, and sell when it rises by 10%. Sounds logical, but how do you know if it will work in practice? Risking real money right away is a bad idea.

**Backtesting** is testing a trading strategy on historical data. It's like a time machine for traders: you take data from past years and see how your strategy would have performed if you had started trading back then.

In real trading, backtesting allows you to:
- Test a hypothesis without risking money
- Evaluate potential returns and drawdowns
- Optimize strategy parameters
- Identify weaknesses before live trading

## What is Backtesting?

Backtesting is the process of simulating trades on historical data. Key components include:

1. **Historical data** — prices, volumes, candles (OHLCV)
2. **Trading strategy** — rules for entering and exiting positions
3. **Simulator** — engine that executes virtual trades
4. **Metrics** — performance indicators (returns, drawdown, Sharpe ratio)

```
Historical Data
        ↓
┌─────────────────────┐
│  Trading Strategy   │
│  ─────────────────  │
│  Entry Rules        │
│  Exit Rules         │
│  Risk Management    │
└─────────────────────┘
        ↓
    Simulator
        ↓
   Results
   (P&L, Metrics)
```

## Basic Structure for Backtesting

```rust
use std::collections::VecDeque;

/// Candle (OHLCV data)
#[derive(Debug, Clone)]
struct Candle {
    timestamp: u64,
    open: f64,
    high: f64,
    low: f64,
    close: f64,
    volume: f64,
}

/// Order type
#[derive(Debug, Clone, PartialEq)]
enum OrderSide {
    Buy,
    Sell,
}

/// Trade
#[derive(Debug, Clone)]
struct Trade {
    timestamp: u64,
    side: OrderSide,
    price: f64,
    quantity: f64,
}

/// Position
#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    quantity: f64,
    entry_price: f64,
}

/// Portfolio state
#[derive(Debug)]
struct Portfolio {
    cash: f64,
    positions: Vec<Position>,
    trades: Vec<Trade>,
}

impl Portfolio {
    fn new(initial_cash: f64) -> Self {
        Portfolio {
            cash: initial_cash,
            positions: Vec::new(),
            trades: Vec::new(),
        }
    }

    fn total_value(&self, current_prices: &[(String, f64)]) -> f64 {
        let positions_value: f64 = self.positions.iter().map(|pos| {
            current_prices
                .iter()
                .find(|(symbol, _)| symbol == &pos.symbol)
                .map(|(_, price)| pos.quantity * price)
                .unwrap_or(0.0)
        }).sum();

        self.cash + positions_value
    }
}
```

## Simple Strategy: Moving Average Crossover

```rust
/// Moving Average
struct MovingAverage {
    period: usize,
    values: VecDeque<f64>,
}

impl MovingAverage {
    fn new(period: usize) -> Self {
        MovingAverage {
            period,
            values: VecDeque::with_capacity(period),
        }
    }

    fn update(&mut self, value: f64) -> Option<f64> {
        self.values.push_back(value);

        if self.values.len() > self.period {
            self.values.pop_front();
        }

        if self.values.len() == self.period {
            Some(self.values.iter().sum::<f64>() / self.period as f64)
        } else {
            None
        }
    }
}

/// Strategy signal
#[derive(Debug, Clone, PartialEq)]
enum Signal {
    Buy,
    Sell,
    Hold,
}

/// Moving Average Crossover Strategy
struct MACrossStrategy {
    fast_ma: MovingAverage,
    slow_ma: MovingAverage,
    prev_fast: Option<f64>,
    prev_slow: Option<f64>,
}

impl MACrossStrategy {
    fn new(fast_period: usize, slow_period: usize) -> Self {
        MACrossStrategy {
            fast_ma: MovingAverage::new(fast_period),
            slow_ma: MovingAverage::new(slow_period),
            prev_fast: None,
            prev_slow: None,
        }
    }

    fn update(&mut self, price: f64) -> Signal {
        let fast = self.fast_ma.update(price);
        let slow = self.slow_ma.update(price);

        let signal = match (fast, slow, self.prev_fast, self.prev_slow) {
            (Some(f), Some(s), Some(pf), Some(ps)) => {
                // Fast MA crosses above slow MA — buy signal
                if pf <= ps && f > s {
                    Signal::Buy
                }
                // Fast MA crosses below slow MA — sell signal
                else if pf >= ps && f < s {
                    Signal::Sell
                } else {
                    Signal::Hold
                }
            }
            _ => Signal::Hold,
        };

        self.prev_fast = fast;
        self.prev_slow = slow;

        signal
    }
}
```

## Backtesting Engine

```rust
/// Backtesting results
#[derive(Debug)]
struct BacktestResult {
    initial_capital: f64,
    final_capital: f64,
    total_return: f64,
    total_trades: usize,
    winning_trades: usize,
    losing_trades: usize,
    max_drawdown: f64,
    sharpe_ratio: f64,
}

impl BacktestResult {
    fn win_rate(&self) -> f64 {
        if self.total_trades == 0 {
            0.0
        } else {
            self.winning_trades as f64 / self.total_trades as f64 * 100.0
        }
    }
}

/// Backtesting engine
struct Backtester {
    portfolio: Portfolio,
    symbol: String,
    position_size: f64,  // Position size in dollars
}

impl Backtester {
    fn new(initial_capital: f64, symbol: &str, position_size: f64) -> Self {
        Backtester {
            portfolio: Portfolio::new(initial_capital),
            symbol: symbol.to_string(),
            position_size,
        }
    }

    fn execute_signal(&mut self, signal: Signal, candle: &Candle) {
        match signal {
            Signal::Buy => {
                // Check if we don't have an open position
                if self.portfolio.positions.is_empty() && self.portfolio.cash >= self.position_size {
                    let quantity = self.position_size / candle.close;
                    self.portfolio.cash -= self.position_size;
                    self.portfolio.positions.push(Position {
                        symbol: self.symbol.clone(),
                        quantity,
                        entry_price: candle.close,
                    });
                    self.portfolio.trades.push(Trade {
                        timestamp: candle.timestamp,
                        side: OrderSide::Buy,
                        price: candle.close,
                        quantity,
                    });
                    println!(
                        "BUY: {} @ ${:.2}, qty: {:.4}",
                        self.symbol, candle.close, quantity
                    );
                }
            }
            Signal::Sell => {
                // Close position if we have one
                if let Some(position) = self.portfolio.positions.pop() {
                    let revenue = position.quantity * candle.close;
                    self.portfolio.cash += revenue;
                    self.portfolio.trades.push(Trade {
                        timestamp: candle.timestamp,
                        side: OrderSide::Sell,
                        price: candle.close,
                        quantity: position.quantity,
                    });

                    let pnl = revenue - (position.quantity * position.entry_price);
                    println!(
                        "SELL: {} @ ${:.2}, P&L: ${:.2}",
                        self.symbol, candle.close, pnl
                    );
                }
            }
            Signal::Hold => {}
        }
    }

    fn run(&mut self, candles: &[Candle], strategy: &mut MACrossStrategy) -> BacktestResult {
        let initial_capital = self.portfolio.cash;
        let mut equity_curve: Vec<f64> = Vec::new();

        for candle in candles {
            let signal = strategy.update(candle.close);
            self.execute_signal(signal, candle);

            // Record current portfolio value
            let current_value = self.portfolio.total_value(
                &[(self.symbol.clone(), candle.close)]
            );
            equity_curve.push(current_value);
        }

        // Close any open position at the last price
        if let Some(last_candle) = candles.last() {
            if !self.portfolio.positions.is_empty() {
                self.execute_signal(Signal::Sell, last_candle);
            }
        }

        // Calculate metrics
        let final_capital = self.portfolio.cash;
        let total_return = (final_capital - initial_capital) / initial_capital * 100.0;

        let (winning_trades, losing_trades) = self.calculate_win_loss();
        let max_drawdown = self.calculate_max_drawdown(&equity_curve);
        let sharpe_ratio = self.calculate_sharpe_ratio(&equity_curve);

        BacktestResult {
            initial_capital,
            final_capital,
            total_return,
            total_trades: self.portfolio.trades.len() / 2, // Buy/sell pairs
            winning_trades,
            losing_trades,
            max_drawdown,
            sharpe_ratio,
        }
    }

    fn calculate_win_loss(&self) -> (usize, usize) {
        let mut wins = 0;
        let mut losses = 0;

        let trades = &self.portfolio.trades;
        for i in (0..trades.len()).step_by(2) {
            if i + 1 < trades.len() {
                let buy = &trades[i];
                let sell = &trades[i + 1];
                if sell.price > buy.price {
                    wins += 1;
                } else {
                    losses += 1;
                }
            }
        }

        (wins, losses)
    }

    fn calculate_max_drawdown(&self, equity_curve: &[f64]) -> f64 {
        let mut max_value = equity_curve.first().copied().unwrap_or(0.0);
        let mut max_drawdown = 0.0;

        for &value in equity_curve {
            if value > max_value {
                max_value = value;
            }
            let drawdown = (max_value - value) / max_value * 100.0;
            if drawdown > max_drawdown {
                max_drawdown = drawdown;
            }
        }

        max_drawdown
    }

    fn calculate_sharpe_ratio(&self, equity_curve: &[f64]) -> f64 {
        if equity_curve.len() < 2 {
            return 0.0;
        }

        // Calculate daily returns
        let returns: Vec<f64> = equity_curve
            .windows(2)
            .map(|w| (w[1] - w[0]) / w[0])
            .collect();

        if returns.is_empty() {
            return 0.0;
        }

        let mean_return = returns.iter().sum::<f64>() / returns.len() as f64;
        let variance = returns.iter()
            .map(|r| (r - mean_return).powi(2))
            .sum::<f64>() / returns.len() as f64;
        let std_dev = variance.sqrt();

        if std_dev == 0.0 {
            0.0
        } else {
            // Annualized Sharpe ratio (assuming 252 trading days)
            (mean_return / std_dev) * (252.0_f64).sqrt()
        }
    }
}
```

## Complete Backtesting Example

```rust
fn main() {
    // Generate test data (in reality, loaded from file/API)
    let candles = generate_sample_data();

    println!("=== MA Crossover Strategy Backtest ===\n");
    println!("Symbol: BTC/USDT");
    println!("Period: {} candles", candles.len());
    println!("Initial Capital: $10,000");
    println!("Position Size: $1,000\n");

    // Create strategy: fast MA(10), slow MA(30)
    let mut strategy = MACrossStrategy::new(10, 30);

    // Create backtester
    let mut backtester = Backtester::new(10_000.0, "BTC/USDT", 1_000.0);

    // Run backtest
    println!("--- Trades ---");
    let result = backtester.run(&candles, &mut strategy);

    // Print results
    println!("\n--- Results ---");
    println!("Initial Capital: ${:.2}", result.initial_capital);
    println!("Final Capital: ${:.2}", result.final_capital);
    println!("Total Return: {:.2}%", result.total_return);
    println!("Total Trades: {}", result.total_trades);
    println!("Winning Trades: {}", result.winning_trades);
    println!("Losing Trades: {}", result.losing_trades);
    println!("Win Rate: {:.1}%", result.win_rate());
    println!("Max Drawdown: {:.2}%", result.max_drawdown);
    println!("Sharpe Ratio: {:.2}", result.sharpe_ratio);
}

fn generate_sample_data() -> Vec<Candle> {
    use std::f64::consts::PI;

    let mut candles = Vec::new();
    let mut price = 40000.0;

    for i in 0..500 {
        // Simulate price movement with trend and volatility
        let trend = (i as f64 * 0.01).sin() * 5000.0;
        let noise = ((i as f64 * 0.1).sin() + (i as f64 * 0.05).cos()) * 500.0;

        price = 40000.0 + trend + noise;
        let volatility = 0.02;

        candles.push(Candle {
            timestamp: 1700000000 + i * 3600,
            open: price * (1.0 - volatility / 2.0),
            high: price * (1.0 + volatility),
            low: price * (1.0 - volatility),
            close: price,
            volume: 1000.0 + (i as f64 * 0.5).sin() * 500.0,
        });
    }

    candles
}
```

## Important Aspects of Backtesting

### 1. Avoid Overfitting

```rust
/// Split data into training and testing sets
fn split_data(candles: &[Candle], train_ratio: f64) -> (&[Candle], &[Candle]) {
    let split_index = (candles.len() as f64 * train_ratio) as usize;
    (&candles[..split_index], &candles[split_index..])
}

fn validate_strategy() {
    let all_candles = generate_sample_data();

    // 70% for training, 30% for testing
    let (train_data, test_data) = split_data(&all_candles, 0.7);

    println!("Training set: {} candles", train_data.len());
    println!("Test set: {} candles", test_data.len());

    // Optimize parameters on training set
    let best_params = optimize_on_train(train_data);

    // Validate on test set
    let mut strategy = MACrossStrategy::new(best_params.0, best_params.1);
    let mut backtester = Backtester::new(10_000.0, "BTC/USDT", 1_000.0);
    let result = backtester.run(test_data, &mut strategy);

    println!("\nResults on test data:");
    println!("Return: {:.2}%", result.total_return);
}

fn optimize_on_train(data: &[Candle]) -> (usize, usize) {
    // Simple parameter grid search
    let mut best_return = f64::MIN;
    let mut best_params = (5, 20);

    for fast in [5, 10, 15].iter() {
        for slow in [20, 30, 50].iter() {
            if fast >= slow {
                continue;
            }

            let mut strategy = MACrossStrategy::new(*fast, *slow);
            let mut backtester = Backtester::new(10_000.0, "BTC/USDT", 1_000.0);
            let result = backtester.run(data, &mut strategy);

            if result.total_return > best_return {
                best_return = result.total_return;
                best_params = (*fast, *slow);
            }
        }
    }

    println!("Best parameters: fast={}, slow={}", best_params.0, best_params.1);
    println!("Return on training data: {:.2}%", best_return);

    best_params
}
```

### 2. Account for Fees and Slippage

```rust
struct BacktesterWithCosts {
    portfolio: Portfolio,
    symbol: String,
    position_size: f64,
    commission_rate: f64,  // Commission (e.g., 0.001 = 0.1%)
    slippage: f64,         // Slippage (e.g., 0.0005 = 0.05%)
}

impl BacktesterWithCosts {
    fn execute_with_costs(&mut self, signal: Signal, candle: &Candle) {
        match signal {
            Signal::Buy => {
                if self.portfolio.positions.is_empty() {
                    // Price with slippage (buy at higher price)
                    let exec_price = candle.close * (1.0 + self.slippage);
                    let quantity = self.position_size / exec_price;

                    // Commission
                    let commission = self.position_size * self.commission_rate;
                    let total_cost = self.position_size + commission;

                    if self.portfolio.cash >= total_cost {
                        self.portfolio.cash -= total_cost;
                        self.portfolio.positions.push(Position {
                            symbol: self.symbol.clone(),
                            quantity,
                            entry_price: exec_price,
                        });

                        println!(
                            "BUY: {} @ ${:.2} (slip: ${:.2}, comm: ${:.2})",
                            self.symbol, exec_price,
                            exec_price - candle.close, commission
                        );
                    }
                }
            }
            Signal::Sell => {
                if let Some(position) = self.portfolio.positions.pop() {
                    // Price with slippage (sell at lower price)
                    let exec_price = candle.close * (1.0 - self.slippage);
                    let revenue = position.quantity * exec_price;
                    let commission = revenue * self.commission_rate;
                    let net_revenue = revenue - commission;

                    self.portfolio.cash += net_revenue;

                    let pnl = net_revenue - (position.quantity * position.entry_price);
                    println!(
                        "SELL: {} @ ${:.2} (slip: ${:.2}, comm: ${:.2}), P&L: ${:.2}",
                        self.symbol, exec_price,
                        candle.close - exec_price, commission, pnl
                    );
                }
            }
            Signal::Hold => {}
        }
    }
}
```

### 3. Risk Management

```rust
/// Stop-loss and take-profit
struct RiskManager {
    stop_loss_pct: f64,   // Stop-loss percentage
    take_profit_pct: f64, // Take-profit percentage
}

impl RiskManager {
    fn new(stop_loss_pct: f64, take_profit_pct: f64) -> Self {
        RiskManager {
            stop_loss_pct,
            take_profit_pct,
        }
    }

    fn check_exit(&self, position: &Position, current_price: f64) -> Signal {
        let pnl_pct = (current_price - position.entry_price) / position.entry_price * 100.0;

        if pnl_pct <= -self.stop_loss_pct {
            println!("STOP LOSS triggered at {:.2}%", pnl_pct);
            Signal::Sell
        } else if pnl_pct >= self.take_profit_pct {
            println!("TAKE PROFIT triggered at {:.2}%", pnl_pct);
            Signal::Sell
        } else {
            Signal::Hold
        }
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Backtesting | Testing a strategy on historical data |
| OHLCV | Open, High, Low, Close, Volume — candle data format |
| Moving Average | Indicator for smoothing price fluctuations |
| MA Crossover | Strategy based on fast and slow MA crossing |
| Equity Curve | Portfolio value dynamics over time |
| Max Drawdown | Maximum decline from peak to trough |
| Sharpe Ratio | Return-to-risk (volatility) ratio |
| Overfitting | Optimizing for historical data without generalization |
| Slippage | Difference between expected and actual execution price |

## Homework

1. **New Strategy**: Implement an RSI (Relative Strength Index) based strategy:
   - RSI > 70 — sell (overbought)
   - RSI < 30 — buy (oversold)
   - Test on historical data

2. **Parameter Optimization**: Create a grid search optimization function:
   - Try various combinations of strategy parameters
   - Find optimal values using Sharpe Ratio as the criterion
   - Visualize results in a table format

3. **Multiple Assets**: Extend the backtester to work with multiple assets:
   - Add ability to trade BTC, ETH, and SOL simultaneously
   - Implement portfolio rebalancing
   - Calculate correlation between assets

4. **Walk-Forward Analysis**: Implement walk-forward testing:
   - Split data into sequential windows
   - Optimize on each window, test on the next
   - Compare results with standard backtesting

## Navigation

[← Previous day](../274-strategy-pattern-trading/en.md) | [Next day →](../276-historical-data-formats/en.md)
