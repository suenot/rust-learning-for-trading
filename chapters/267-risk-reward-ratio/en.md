# Day 267: Risk/Reward Ratio

## Trading Analogy

Imagine you're a trader evaluating a potential trade. You spot an opportunity to buy BTC at $40,000 with a target of $44,000 and a stop-loss at $38,000. Before entering, you calculate: if the trade works, you gain $4,000 per BTC; if it fails, you lose $2,000 per BTC. This gives you a **risk/reward ratio** of 1:2 — you're risking 1 unit to potentially gain 2 units.

The risk/reward ratio is one of the most critical metrics in trading:
- A 1:2 ratio means you only need to win 34% of trades to break even
- A 1:3 ratio means you only need to win 25% of trades to break even
- Professional traders typically seek ratios of 1:2 or better

In algorithmic trading, we can systematically calculate and enforce risk/reward requirements before any trade is executed.

## What is Risk/Reward Ratio?

The risk/reward ratio (R:R or RRR) measures the potential profit of a trade relative to its potential loss:

```
Risk/Reward Ratio = (Entry Price - Stop Loss) / (Take Profit - Entry Price)
```

Or equivalently:
```
Reward/Risk Ratio = Potential Reward / Potential Risk
```

A ratio of 1:3 means for every $1 risked, the potential reward is $3.

## Basic Risk/Reward Calculator

```rust
#[derive(Debug, Clone)]
struct TradeSetup {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    position_size: f64,
}

#[derive(Debug)]
struct RiskRewardAnalysis {
    risk_amount: f64,
    reward_amount: f64,
    risk_reward_ratio: f64,
    reward_risk_ratio: f64,
    risk_percent: f64,
    reward_percent: f64,
    breakeven_winrate: f64,
}

impl TradeSetup {
    fn new(symbol: &str, entry: f64, stop: f64, target: f64, size: f64) -> Self {
        TradeSetup {
            symbol: symbol.to_string(),
            entry_price: entry,
            stop_loss: stop,
            take_profit: target,
            position_size: size,
        }
    }

    fn analyze(&self) -> RiskRewardAnalysis {
        let risk_per_unit = (self.entry_price - self.stop_loss).abs();
        let reward_per_unit = (self.take_profit - self.entry_price).abs();

        let risk_amount = risk_per_unit * self.position_size;
        let reward_amount = reward_per_unit * self.position_size;

        let risk_reward_ratio = if reward_per_unit > 0.0 {
            risk_per_unit / reward_per_unit
        } else {
            f64::INFINITY
        };

        let reward_risk_ratio = if risk_per_unit > 0.0 {
            reward_per_unit / risk_per_unit
        } else {
            f64::INFINITY
        };

        let risk_percent = (risk_per_unit / self.entry_price) * 100.0;
        let reward_percent = (reward_per_unit / self.entry_price) * 100.0;

        // Breakeven win rate = Risk / (Risk + Reward)
        let breakeven_winrate = if risk_amount + reward_amount > 0.0 {
            (risk_amount / (risk_amount + reward_amount)) * 100.0
        } else {
            50.0
        };

        RiskRewardAnalysis {
            risk_amount,
            reward_amount,
            risk_reward_ratio,
            reward_risk_ratio,
            risk_percent,
            reward_percent,
            breakeven_winrate,
        }
    }

    fn is_long(&self) -> bool {
        self.take_profit > self.entry_price
    }
}

fn main() {
    // Long trade example: Buy BTC
    let long_trade = TradeSetup::new(
        "BTC/USDT",
        40000.0,  // Entry
        38000.0,  // Stop loss
        46000.0,  // Take profit
        0.5,      // Position size (0.5 BTC)
    );

    let analysis = long_trade.analyze();

    println!("=== Trade Analysis: {} ===", long_trade.symbol);
    println!("Direction: {}", if long_trade.is_long() { "LONG" } else { "SHORT" });
    println!("Entry: ${:.2}", long_trade.entry_price);
    println!("Stop Loss: ${:.2}", long_trade.stop_loss);
    println!("Take Profit: ${:.2}", long_trade.take_profit);
    println!();
    println!("Risk: ${:.2} ({:.2}%)", analysis.risk_amount, analysis.risk_percent);
    println!("Reward: ${:.2} ({:.2}%)", analysis.reward_amount, analysis.reward_percent);
    println!("Risk:Reward = 1:{:.2}", analysis.reward_risk_ratio);
    println!("Breakeven Win Rate: {:.1}%", analysis.breakeven_winrate);

    // Short trade example
    let short_trade = TradeSetup::new(
        "ETH/USDT",
        2500.0,  // Entry
        2650.0,  // Stop loss (above for short)
        2200.0,  // Take profit (below for short)
        2.0,     // Position size (2 ETH)
    );

    println!("\n=== Trade Analysis: {} ===", short_trade.symbol);
    let short_analysis = short_trade.analyze();
    println!("Direction: SHORT");
    println!("Risk:Reward = 1:{:.2}", short_analysis.reward_risk_ratio);
}
```

## Trade Filter Based on Risk/Reward

```rust
#[derive(Debug, Clone)]
struct TradingRules {
    min_reward_risk_ratio: f64,
    max_risk_percent: f64,
    max_position_risk: f64,  // Max $ risk per trade
}

impl TradingRules {
    fn new(min_rr: f64, max_risk_pct: f64, max_pos_risk: f64) -> Self {
        TradingRules {
            min_reward_risk_ratio: min_rr,
            max_risk_percent: max_risk_pct,
            max_position_risk: max_pos_risk,
        }
    }

    fn validate_trade(&self, setup: &TradeSetup) -> Result<(), Vec<String>> {
        let analysis = setup.analyze();
        let mut errors = Vec::new();

        // Check reward/risk ratio
        if analysis.reward_risk_ratio < self.min_reward_risk_ratio {
            errors.push(format!(
                "R:R ratio {:.2} below minimum {:.2}",
                analysis.reward_risk_ratio,
                self.min_reward_risk_ratio
            ));
        }

        // Check risk percentage
        if analysis.risk_percent > self.max_risk_percent {
            errors.push(format!(
                "Risk {:.2}% exceeds maximum {:.2}%",
                analysis.risk_percent,
                self.max_risk_percent
            ));
        }

        // Check absolute risk
        if analysis.risk_amount > self.max_position_risk {
            errors.push(format!(
                "Position risk ${:.2} exceeds maximum ${:.2}",
                analysis.risk_amount,
                self.max_position_risk
            ));
        }

        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
}

fn main() {
    let rules = TradingRules::new(
        2.0,     // Minimum 1:2 R:R
        5.0,     // Max 5% risk per trade
        1000.0,  // Max $1000 risk per trade
    );

    let trades = vec![
        TradeSetup::new("BTC/USDT", 40000.0, 38000.0, 46000.0, 0.5),  // Good R:R
        TradeSetup::new("ETH/USDT", 2500.0, 2400.0, 2550.0, 10.0),   // Poor R:R
        TradeSetup::new("SOL/USDT", 100.0, 85.0, 145.0, 50.0),       // Good R:R
    ];

    println!("=== Trade Validation ===\n");

    for trade in &trades {
        let analysis = trade.analyze();
        print!("{}: ", trade.symbol);

        match rules.validate_trade(trade) {
            Ok(()) => {
                println!("APPROVED (R:R = 1:{:.2})", analysis.reward_risk_ratio);
            }
            Err(errors) => {
                println!("REJECTED");
                for error in errors {
                    println!("  - {}", error);
                }
            }
        }
    }
}
```

## Position Sizing Based on Risk

```rust
#[derive(Debug)]
struct Portfolio {
    balance: f64,
    risk_per_trade_percent: f64,
}

impl Portfolio {
    fn new(balance: f64, risk_percent: f64) -> Self {
        Portfolio {
            balance,
            risk_per_trade_percent: risk_percent,
        }
    }

    fn max_risk_amount(&self) -> f64 {
        self.balance * (self.risk_per_trade_percent / 100.0)
    }

    fn calculate_position_size(&self, entry: f64, stop_loss: f64) -> f64 {
        let risk_per_unit = (entry - stop_loss).abs();
        if risk_per_unit == 0.0 {
            return 0.0;
        }

        let max_risk = self.max_risk_amount();
        max_risk / risk_per_unit
    }

    fn calculate_full_trade(
        &self,
        symbol: &str,
        entry: f64,
        stop_loss: f64,
        take_profit: f64,
    ) -> TradeSetup {
        let position_size = self.calculate_position_size(entry, stop_loss);

        TradeSetup::new(symbol, entry, stop_loss, take_profit, position_size)
    }
}

#[derive(Debug, Clone)]
struct TradeSetup {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    position_size: f64,
}

impl TradeSetup {
    fn new(symbol: &str, entry: f64, stop: f64, target: f64, size: f64) -> Self {
        TradeSetup {
            symbol: symbol.to_string(),
            entry_price: entry,
            stop_loss: stop,
            take_profit: target,
            position_size: size,
        }
    }

    fn risk_amount(&self) -> f64 {
        (self.entry_price - self.stop_loss).abs() * self.position_size
    }

    fn reward_amount(&self) -> f64 {
        (self.take_profit - self.entry_price).abs() * self.position_size
    }

    fn reward_risk_ratio(&self) -> f64 {
        let risk = (self.entry_price - self.stop_loss).abs();
        let reward = (self.take_profit - self.entry_price).abs();
        if risk > 0.0 { reward / risk } else { 0.0 }
    }
}

fn main() {
    let portfolio = Portfolio::new(50000.0, 2.0);  // $50k, 2% risk per trade

    println!("Portfolio Balance: ${:.2}", portfolio.balance);
    println!("Risk Per Trade: {:.1}% = ${:.2}\n",
        portfolio.risk_per_trade_percent,
        portfolio.max_risk_amount()
    );

    // Calculate position sizes for different setups
    let setups = vec![
        ("BTC/USDT", 42000.0, 40000.0, 48000.0),  // $2000 stop distance
        ("ETH/USDT", 2500.0, 2350.0, 2800.0),    // $150 stop distance
        ("SOL/USDT", 100.0, 92.0, 120.0),         // $8 stop distance
    ];

    for (symbol, entry, stop, target) in setups {
        let trade = portfolio.calculate_full_trade(symbol, entry, stop, target);

        println!("=== {} ===", symbol);
        println!("Entry: ${:.2}, Stop: ${:.2}, Target: ${:.2}",
            trade.entry_price, trade.stop_loss, trade.take_profit);
        println!("Position Size: {:.4} units", trade.position_size);
        println!("Position Value: ${:.2}", trade.position_size * trade.entry_price);
        println!("Risk: ${:.2}", trade.risk_amount());
        println!("Potential Reward: ${:.2}", trade.reward_amount());
        println!("R:R = 1:{:.2}", trade.reward_risk_ratio());
        println!();
    }
}
```

## Multi-Target Trade Management

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct MultiTargetTrade {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    targets: Vec<(f64, f64)>,  // (price, percentage of position)
    position_size: f64,
}

#[derive(Debug)]
struct TargetAnalysis {
    target_price: f64,
    position_percent: f64,
    units_to_sell: f64,
    profit_at_target: f64,
    rr_at_target: f64,
}

impl MultiTargetTrade {
    fn new(symbol: &str, entry: f64, stop: f64, targets: Vec<(f64, f64)>, size: f64) -> Self {
        MultiTargetTrade {
            symbol: symbol.to_string(),
            entry_price: entry,
            stop_loss: stop,
            targets,
            position_size: size,
        }
    }

    fn analyze_targets(&self) -> Vec<TargetAnalysis> {
        let risk_per_unit = (self.entry_price - self.stop_loss).abs();

        self.targets.iter().map(|(price, percent)| {
            let units = self.position_size * (percent / 100.0);
            let profit_per_unit = (price - self.entry_price).abs();
            let profit = profit_per_unit * units;
            let rr = if risk_per_unit > 0.0 {
                profit_per_unit / risk_per_unit
            } else {
                0.0
            };

            TargetAnalysis {
                target_price: *price,
                position_percent: *percent,
                units_to_sell: units,
                profit_at_target: profit,
                rr_at_target: rr,
            }
        }).collect()
    }

    fn total_risk(&self) -> f64 {
        (self.entry_price - self.stop_loss).abs() * self.position_size
    }

    fn expected_reward(&self) -> f64 {
        self.analyze_targets().iter().map(|t| t.profit_at_target).sum()
    }

    fn average_rr(&self) -> f64 {
        let analyses = self.analyze_targets();
        let total_percent: f64 = analyses.iter().map(|t| t.position_percent).sum();
        let weighted_rr: f64 = analyses.iter()
            .map(|t| t.rr_at_target * t.position_percent)
            .sum();

        if total_percent > 0.0 {
            weighted_rr / total_percent
        } else {
            0.0
        }
    }
}

fn main() {
    // Trade with multiple take-profit levels
    let trade = MultiTargetTrade::new(
        "BTC/USDT",
        40000.0,  // Entry
        38000.0,  // Stop loss
        vec![
            (42000.0, 33.0),  // Target 1: $42k, sell 33%
            (44000.0, 33.0),  // Target 2: $44k, sell 33%
            (48000.0, 34.0),  // Target 3: $48k, sell remaining 34%
        ],
        1.0,  // 1 BTC position
    );

    println!("=== Multi-Target Trade: {} ===\n", trade.symbol);
    println!("Entry: ${:.2}", trade.entry_price);
    println!("Stop Loss: ${:.2}", trade.stop_loss);
    println!("Position: {} units", trade.position_size);
    println!("Total Risk: ${:.2}\n", trade.total_risk());

    println!("Targets:");
    for (i, analysis) in trade.analyze_targets().iter().enumerate() {
        println!(
            "  T{}: ${:.2} ({:.0}% = {:.4} units) | Profit: ${:.2} | R:R 1:{:.2}",
            i + 1,
            analysis.target_price,
            analysis.position_percent,
            analysis.units_to_sell,
            analysis.profit_at_target,
            analysis.rr_at_target
        );
    }

    println!("\nTotal Expected Reward: ${:.2}", trade.expected_reward());
    println!("Weighted Average R:R: 1:{:.2}", trade.average_rr());
}
```

## Trade Journal with Risk/Reward Tracking

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
enum TradeResult {
    Win(f64),   // Actual profit
    Loss(f64),  // Actual loss (positive number)
    BreakEven,
}

#[derive(Debug, Clone)]
struct CompletedTrade {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    exit_price: f64,
    position_size: f64,
    result: TradeResult,
}

#[derive(Debug)]
struct TradeJournal {
    trades: Vec<CompletedTrade>,
}

#[derive(Debug)]
struct JournalStats {
    total_trades: usize,
    wins: usize,
    losses: usize,
    breakevens: usize,
    win_rate: f64,
    total_profit: f64,
    total_loss: f64,
    net_pnl: f64,
    avg_win: f64,
    avg_loss: f64,
    actual_rr: f64,
    expected_rr_avg: f64,
    profit_factor: f64,
}

impl CompletedTrade {
    fn planned_risk(&self) -> f64 {
        (self.entry_price - self.stop_loss).abs() * self.position_size
    }

    fn planned_reward(&self) -> f64 {
        (self.take_profit - self.entry_price).abs() * self.position_size
    }

    fn planned_rr(&self) -> f64 {
        let risk = (self.entry_price - self.stop_loss).abs();
        let reward = (self.take_profit - self.entry_price).abs();
        if risk > 0.0 { reward / risk } else { 0.0 }
    }

    fn actual_pnl(&self) -> f64 {
        match &self.result {
            TradeResult::Win(profit) => *profit,
            TradeResult::Loss(loss) => -loss,
            TradeResult::BreakEven => 0.0,
        }
    }
}

impl TradeJournal {
    fn new() -> Self {
        TradeJournal { trades: Vec::new() }
    }

    fn add_trade(&mut self, trade: CompletedTrade) {
        self.trades.push(trade);
    }

    fn calculate_stats(&self) -> JournalStats {
        let total_trades = self.trades.len();

        let wins: Vec<_> = self.trades.iter()
            .filter(|t| matches!(t.result, TradeResult::Win(_)))
            .collect();

        let losses: Vec<_> = self.trades.iter()
            .filter(|t| matches!(t.result, TradeResult::Loss(_)))
            .collect();

        let breakevens = self.trades.iter()
            .filter(|t| matches!(t.result, TradeResult::BreakEven))
            .count();

        let win_count = wins.len();
        let loss_count = losses.len();

        let total_profit: f64 = wins.iter()
            .map(|t| t.actual_pnl())
            .sum();

        let total_loss: f64 = losses.iter()
            .map(|t| t.actual_pnl().abs())
            .sum();

        let avg_win = if win_count > 0 {
            total_profit / win_count as f64
        } else { 0.0 };

        let avg_loss = if loss_count > 0 {
            total_loss / loss_count as f64
        } else { 0.0 };

        let actual_rr = if avg_loss > 0.0 {
            avg_win / avg_loss
        } else { 0.0 };

        let expected_rr_avg = if total_trades > 0 {
            self.trades.iter().map(|t| t.planned_rr()).sum::<f64>() / total_trades as f64
        } else { 0.0 };

        let profit_factor = if total_loss > 0.0 {
            total_profit / total_loss
        } else if total_profit > 0.0 {
            f64::INFINITY
        } else { 0.0 };

        JournalStats {
            total_trades,
            wins: win_count,
            losses: loss_count,
            breakevens,
            win_rate: if total_trades > 0 {
                (win_count as f64 / total_trades as f64) * 100.0
            } else { 0.0 },
            total_profit,
            total_loss,
            net_pnl: total_profit - total_loss,
            avg_win,
            avg_loss,
            actual_rr,
            expected_rr_avg,
            profit_factor,
        }
    }
}

fn main() {
    let mut journal = TradeJournal::new();

    // Add some completed trades
    journal.add_trade(CompletedTrade {
        symbol: "BTC/USDT".to_string(),
        entry_price: 40000.0,
        stop_loss: 38000.0,
        take_profit: 46000.0,
        exit_price: 45500.0,
        position_size: 0.5,
        result: TradeResult::Win(2750.0),  // Hit near target
    });

    journal.add_trade(CompletedTrade {
        symbol: "ETH/USDT".to_string(),
        entry_price: 2500.0,
        stop_loss: 2350.0,
        take_profit: 2800.0,
        exit_price: 2355.0,
        position_size: 4.0,
        result: TradeResult::Loss(580.0),  // Stopped out
    });

    journal.add_trade(CompletedTrade {
        symbol: "SOL/USDT".to_string(),
        entry_price: 100.0,
        stop_loss: 92.0,
        take_profit: 120.0,
        exit_price: 118.0,
        position_size: 50.0,
        result: TradeResult::Win(900.0),
    });

    journal.add_trade(CompletedTrade {
        symbol: "BTC/USDT".to_string(),
        entry_price: 42000.0,
        stop_loss: 40500.0,
        take_profit: 45000.0,
        exit_price: 40600.0,
        position_size: 0.3,
        result: TradeResult::Loss(420.0),
    });

    let stats = journal.calculate_stats();

    println!("=== Trade Journal Statistics ===\n");
    println!("Total Trades: {}", stats.total_trades);
    println!("Wins: {} | Losses: {} | Breakeven: {}",
        stats.wins, stats.losses, stats.breakevens);
    println!("Win Rate: {:.1}%\n", stats.win_rate);

    println!("Total Profit: ${:.2}", stats.total_profit);
    println!("Total Loss: ${:.2}", stats.total_loss);
    println!("Net P&L: ${:.2}\n", stats.net_pnl);

    println!("Average Win: ${:.2}", stats.avg_win);
    println!("Average Loss: ${:.2}", stats.avg_loss);
    println!("Actual R:R: 1:{:.2}", stats.actual_rr);
    println!("Planned R:R (avg): 1:{:.2}", stats.expected_rr_avg);
    println!("Profit Factor: {:.2}", stats.profit_factor);
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Risk/Reward Ratio | Comparison of potential loss to potential gain |
| Position Sizing | Calculating trade size based on acceptable risk |
| Breakeven Win Rate | Minimum win rate needed to be profitable |
| Multi-Target Exits | Scaling out of positions at different price levels |
| Profit Factor | Total profits divided by total losses |
| Trade Validation | Filtering trades based on R:R requirements |

## Exercises

1. **R:R Calculator**: Create a function that takes entry, stop-loss, and take-profit prices and returns a detailed analysis including the R:R ratio, breakeven win rate, and whether the trade meets a minimum 1:2 R:R requirement.

2. **Dynamic Position Sizer**: Implement a position sizing calculator that:
   - Takes account balance and risk percentage as input
   - Calculates position size for any given entry and stop-loss
   - Ensures the maximum dollar risk is never exceeded

3. **Trade Screener**: Build a trade screening system that:
   - Accepts multiple potential trade setups
   - Filters out trades with R:R below a configurable threshold
   - Ranks remaining trades by their R:R ratio
   - Returns the top N best opportunities

4. **Risk-Adjusted Returns**: Create a function that calculates the expected value of a trade given:
   - The risk/reward ratio
   - Historical win rate
   - Return the expected profit/loss per trade

## Homework

1. **Complete Trading System**: Build a comprehensive trading system that:
   - Validates trades against configurable R:R rules
   - Calculates position sizes based on portfolio risk
   - Supports multiple take-profit levels
   - Tracks all trades in a journal
   - Calculates performance statistics including profit factor and actual vs. planned R:R

2. **Monte Carlo Simulation**: Using the trade journal concept, create a Monte Carlo simulation that:
   - Takes historical win rate and average R:R as inputs
   - Simulates 1000 sequences of 100 trades each
   - Calculates the probability of account growth vs. drawdown
   - Visualizes the distribution of outcomes

3. **Trailing Stop R:R**: Implement a trailing stop system that:
   - Moves the stop-loss to breakeven after 1R profit is reached
   - Trails the stop at a configurable distance as price moves favorably
   - Recalculates the effective R:R at each adjustment
   - Logs all stop movements with timestamps

4. **Portfolio Risk Manager**: Create a portfolio-level risk manager that:
   - Tracks total risk exposure across all open positions
   - Prevents new trades if total portfolio risk exceeds a threshold (e.g., 6%)
   - Calculates correlation-adjusted risk for related assets
   - Suggests position size reductions when risk is too high

## Navigation

[← Previous day](../266-stop-loss-take-profit/en.md) | [Next day →](../268-expectancy-calculation/en.md)
