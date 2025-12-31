# День 267: Соотношение риска и прибыли (Risk/Reward Ratio)

## Аналогия из трейдинга

Представь, что ты трейдер и оцениваешь потенциальную сделку. Ты видишь возможность купить BTC по $40,000 с целью $44,000 и стоп-лоссом на $38,000. Перед входом ты считаешь: если сделка сработает, ты заработаешь $4,000 на каждый BTC; если нет — потеряешь $2,000 на каждый BTC. Это даёт тебе **соотношение риска и прибыли** 1:2 — ты рискуешь 1 единицей, чтобы потенциально получить 2 единицы.

Соотношение риска и прибыли — одна из важнейших метрик в трейдинге:
- При соотношении 1:2 достаточно выигрывать 34% сделок, чтобы выйти в безубыток
- При соотношении 1:3 достаточно выигрывать 25% сделок, чтобы выйти в безубыток
- Профессиональные трейдеры обычно ищут соотношение 1:2 или лучше

В алгоритмическом трейдинге мы можем систематически рассчитывать и проверять требования к соотношению риска/прибыли перед любой сделкой.

## Что такое Risk/Reward Ratio?

Соотношение риска и прибыли (R:R или RRR) измеряет потенциальную прибыль сделки относительно потенциального убытка:

```
Соотношение Риск/Прибыль = (Цена входа - Стоп-лосс) / (Тейк-профит - Цена входа)
```

Или эквивалентно:
```
Соотношение Прибыль/Риск = Потенциальная прибыль / Потенциальный риск
```

Соотношение 1:3 означает, что на каждый $1 риска потенциальная прибыль составляет $3.

## Базовый калькулятор Risk/Reward

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

        // Безубыточный винрейт = Риск / (Риск + Прибыль)
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
    // Пример лонг-сделки: покупка BTC
    let long_trade = TradeSetup::new(
        "BTC/USDT",
        40000.0,  // Вход
        38000.0,  // Стоп-лосс
        46000.0,  // Тейк-профит
        0.5,      // Размер позиции (0.5 BTC)
    );

    let analysis = long_trade.analyze();

    println!("=== Анализ сделки: {} ===", long_trade.symbol);
    println!("Направление: {}", if long_trade.is_long() { "ЛОНГ" } else { "ШОРТ" });
    println!("Вход: ${:.2}", long_trade.entry_price);
    println!("Стоп-лосс: ${:.2}", long_trade.stop_loss);
    println!("Тейк-профит: ${:.2}", long_trade.take_profit);
    println!();
    println!("Риск: ${:.2} ({:.2}%)", analysis.risk_amount, analysis.risk_percent);
    println!("Прибыль: ${:.2} ({:.2}%)", analysis.reward_amount, analysis.reward_percent);
    println!("Риск:Прибыль = 1:{:.2}", analysis.reward_risk_ratio);
    println!("Безубыточный винрейт: {:.1}%", analysis.breakeven_winrate);

    // Пример шорт-сделки
    let short_trade = TradeSetup::new(
        "ETH/USDT",
        2500.0,  // Вход
        2650.0,  // Стоп-лосс (выше для шорта)
        2200.0,  // Тейк-профит (ниже для шорта)
        2.0,     // Размер позиции (2 ETH)
    );

    println!("\n=== Анализ сделки: {} ===", short_trade.symbol);
    let short_analysis = short_trade.analyze();
    println!("Направление: ШОРТ");
    println!("Риск:Прибыль = 1:{:.2}", short_analysis.reward_risk_ratio);
}
```

## Фильтр сделок на основе Risk/Reward

```rust
#[derive(Debug, Clone)]
struct TradingRules {
    min_reward_risk_ratio: f64,
    max_risk_percent: f64,
    max_position_risk: f64,  // Макс. риск в $ на сделку
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

        // Проверка соотношения прибыль/риск
        if analysis.reward_risk_ratio < self.min_reward_risk_ratio {
            errors.push(format!(
                "R:R соотношение {:.2} ниже минимального {:.2}",
                analysis.reward_risk_ratio,
                self.min_reward_risk_ratio
            ));
        }

        // Проверка процента риска
        if analysis.risk_percent > self.max_risk_percent {
            errors.push(format!(
                "Риск {:.2}% превышает максимум {:.2}%",
                analysis.risk_percent,
                self.max_risk_percent
            ));
        }

        // Проверка абсолютного риска
        if analysis.risk_amount > self.max_position_risk {
            errors.push(format!(
                "Риск позиции ${:.2} превышает максимум ${:.2}",
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
        2.0,     // Минимум 1:2 R:R
        5.0,     // Макс 5% риска на сделку
        1000.0,  // Макс $1000 риска на сделку
    );

    let trades = vec![
        TradeSetup::new("BTC/USDT", 40000.0, 38000.0, 46000.0, 0.5),  // Хороший R:R
        TradeSetup::new("ETH/USDT", 2500.0, 2400.0, 2550.0, 10.0),   // Плохой R:R
        TradeSetup::new("SOL/USDT", 100.0, 85.0, 145.0, 50.0),       // Хороший R:R
    ];

    println!("=== Валидация сделок ===\n");

    for trade in &trades {
        let analysis = trade.analyze();
        print!("{}: ", trade.symbol);

        match rules.validate_trade(trade) {
            Ok(()) => {
                println!("ОДОБРЕНО (R:R = 1:{:.2})", analysis.reward_risk_ratio);
            }
            Err(errors) => {
                println!("ОТКЛОНЕНО");
                for error in errors {
                    println!("  - {}", error);
                }
            }
        }
    }
}
```

## Расчёт размера позиции на основе риска

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
    let portfolio = Portfolio::new(50000.0, 2.0);  // $50к, 2% риска на сделку

    println!("Баланс портфеля: ${:.2}", portfolio.balance);
    println!("Риск на сделку: {:.1}% = ${:.2}\n",
        portfolio.risk_per_trade_percent,
        portfolio.max_risk_amount()
    );

    // Расчёт размеров позиций для разных сетапов
    let setups = vec![
        ("BTC/USDT", 42000.0, 40000.0, 48000.0),  // Дистанция стопа $2000
        ("ETH/USDT", 2500.0, 2350.0, 2800.0),    // Дистанция стопа $150
        ("SOL/USDT", 100.0, 92.0, 120.0),         // Дистанция стопа $8
    ];

    for (symbol, entry, stop, target) in setups {
        let trade = portfolio.calculate_full_trade(symbol, entry, stop, target);

        println!("=== {} ===", symbol);
        println!("Вход: ${:.2}, Стоп: ${:.2}, Цель: ${:.2}",
            trade.entry_price, trade.stop_loss, trade.take_profit);
        println!("Размер позиции: {:.4} единиц", trade.position_size);
        println!("Стоимость позиции: ${:.2}", trade.position_size * trade.entry_price);
        println!("Риск: ${:.2}", trade.risk_amount());
        println!("Потенциальная прибыль: ${:.2}", trade.reward_amount());
        println!("R:R = 1:{:.2}", trade.reward_risk_ratio());
        println!();
    }
}
```

## Управление сделкой с несколькими целями

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct MultiTargetTrade {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    targets: Vec<(f64, f64)>,  // (цена, процент позиции)
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
    // Сделка с несколькими уровнями тейк-профита
    let trade = MultiTargetTrade::new(
        "BTC/USDT",
        40000.0,  // Вход
        38000.0,  // Стоп-лосс
        vec![
            (42000.0, 33.0),  // Цель 1: $42к, продать 33%
            (44000.0, 33.0),  // Цель 2: $44к, продать 33%
            (48000.0, 34.0),  // Цель 3: $48к, продать оставшиеся 34%
        ],
        1.0,  // Позиция 1 BTC
    );

    println!("=== Сделка с несколькими целями: {} ===\n", trade.symbol);
    println!("Вход: ${:.2}", trade.entry_price);
    println!("Стоп-лосс: ${:.2}", trade.stop_loss);
    println!("Позиция: {} единиц", trade.position_size);
    println!("Общий риск: ${:.2}\n", trade.total_risk());

    println!("Цели:");
    for (i, analysis) in trade.analyze_targets().iter().enumerate() {
        println!(
            "  Ц{}: ${:.2} ({:.0}% = {:.4} единиц) | Прибыль: ${:.2} | R:R 1:{:.2}",
            i + 1,
            analysis.target_price,
            analysis.position_percent,
            analysis.units_to_sell,
            analysis.profit_at_target,
            analysis.rr_at_target
        );
    }

    println!("\nОбщая ожидаемая прибыль: ${:.2}", trade.expected_reward());
    println!("Средневзвешенный R:R: 1:{:.2}", trade.average_rr());
}
```

## Журнал сделок с отслеживанием Risk/Reward

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
enum TradeResult {
    Win(f64),   // Фактическая прибыль
    Loss(f64),  // Фактический убыток (положительное число)
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

    // Добавляем завершённые сделки
    journal.add_trade(CompletedTrade {
        symbol: "BTC/USDT".to_string(),
        entry_price: 40000.0,
        stop_loss: 38000.0,
        take_profit: 46000.0,
        exit_price: 45500.0,
        position_size: 0.5,
        result: TradeResult::Win(2750.0),  // Достигнута почти цель
    });

    journal.add_trade(CompletedTrade {
        symbol: "ETH/USDT".to_string(),
        entry_price: 2500.0,
        stop_loss: 2350.0,
        take_profit: 2800.0,
        exit_price: 2355.0,
        position_size: 4.0,
        result: TradeResult::Loss(580.0),  // Сработал стоп
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

    println!("=== Статистика журнала сделок ===\n");
    println!("Всего сделок: {}", stats.total_trades);
    println!("Прибыльных: {} | Убыточных: {} | Безубыток: {}",
        stats.wins, stats.losses, stats.breakevens);
    println!("Винрейт: {:.1}%\n", stats.win_rate);

    println!("Общая прибыль: ${:.2}", stats.total_profit);
    println!("Общий убыток: ${:.2}", stats.total_loss);
    println!("Чистый P&L: ${:.2}\n", stats.net_pnl);

    println!("Средняя прибыль: ${:.2}", stats.avg_win);
    println!("Средний убыток: ${:.2}", stats.avg_loss);
    println!("Фактический R:R: 1:{:.2}", stats.actual_rr);
    println!("Планируемый R:R (средн.): 1:{:.2}", stats.expected_rr_avg);
    println!("Профит-фактор: {:.2}", stats.profit_factor);
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Risk/Reward Ratio | Сравнение потенциального убытка с потенциальной прибылью |
| Расчёт размера позиции | Вычисление размера сделки на основе допустимого риска |
| Безубыточный винрейт | Минимальный процент побед для прибыльности |
| Несколько целей | Масштабирование выхода из позиции на разных уровнях |
| Профит-фактор | Общая прибыль, делённая на общий убыток |
| Валидация сделок | Фильтрация сделок по требованиям R:R |

## Упражнения

1. **Калькулятор R:R**: Создай функцию, которая принимает цены входа, стоп-лосса и тейк-профита и возвращает детальный анализ, включая соотношение R:R, безубыточный винрейт и информацию о соответствии минимальному требованию 1:2.

2. **Динамический расчёт позиции**: Реализуй калькулятор размера позиции, который:
   - Принимает баланс счёта и процент риска
   - Рассчитывает размер позиции для любого входа и стоп-лосса
   - Гарантирует, что максимальный долларовый риск никогда не превышен

3. **Скринер сделок**: Построй систему скрининга сделок, которая:
   - Принимает несколько потенциальных торговых сетапов
   - Отфильтровывает сделки с R:R ниже настраиваемого порога
   - Ранжирует оставшиеся сделки по их R:R
   - Возвращает топ N лучших возможностей

4. **Риск-скорректированная доходность**: Создай функцию, которая рассчитывает математическое ожидание сделки на основе:
   - Соотношения риск/прибыль
   - Исторического винрейта
   - Возвращает ожидаемую прибыль/убыток на сделку

## Домашнее задание

1. **Полная торговая система**: Построй комплексную торговую систему, которая:
   - Валидирует сделки по настраиваемым правилам R:R
   - Рассчитывает размеры позиций на основе риска портфеля
   - Поддерживает несколько уровней тейк-профита
   - Отслеживает все сделки в журнале
   - Рассчитывает статистику производительности, включая профит-фактор и фактический vs планируемый R:R

2. **Симуляция Монте-Карло**: Используя концепцию журнала сделок, создай симуляцию Монте-Карло, которая:
   - Принимает исторический винрейт и средний R:R как входные данные
   - Симулирует 1000 последовательностей по 100 сделок каждая
   - Рассчитывает вероятность роста счёта vs просадки
   - Визуализирует распределение исходов

3. **Трейлинг-стоп с R:R**: Реализуй систему трейлинг-стопа, которая:
   - Переводит стоп-лосс в безубыток после достижения прибыли в 1R
   - Подтягивает стоп на настраиваемое расстояние при движении цены в нужную сторону
   - Пересчитывает эффективный R:R при каждой корректировке
   - Логирует все перемещения стопа с временными метками

4. **Риск-менеджер портфеля**: Создай менеджер риска на уровне портфеля, который:
   - Отслеживает общую экспозицию риска по всем открытым позициям
   - Запрещает новые сделки, если общий риск портфеля превышает порог (например, 6%)
   - Рассчитывает риск с учётом корреляции для связанных активов
   - Предлагает уменьшение размера позиций при слишком высоком риске

## Навигация

[← Предыдущий день](../266-stop-loss-take-profit/ru.md) | [Следующий день →](../268-expectancy-calculation/ru.md)
