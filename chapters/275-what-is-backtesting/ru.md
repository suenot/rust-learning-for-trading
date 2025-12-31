# День 275: Что такое бэктестинг

## Аналогия из трейдинга

Представь, что ты придумал новую торговую стратегию: покупать биткоин каждый раз, когда цена падает на 5%, и продавать, когда она растёт на 10%. Звучит логично, но как узнать, сработает ли это на практике? Рисковать реальными деньгами сразу — плохая идея.

**Бэктестинг** (backtesting) — это тестирование торговой стратегии на исторических данных. Это как машина времени для трейдера: ты берёшь данные за прошлые годы и смотришь, как бы работала твоя стратегия, если бы ты начал торговать тогда.

В реальном трейдинге бэктестинг позволяет:
- Проверить гипотезу без риска потерять деньги
- Оценить потенциальную доходность и просадки
- Оптимизировать параметры стратегии
- Выявить слабые места до реальной торговли

## Что такое бэктестинг?

Бэктестинг — это процесс симуляции торговли на исторических данных. Основные компоненты:

1. **Исторические данные** — цены, объёмы, свечи (OHLCV)
2. **Торговая стратегия** — правила входа и выхода из позиции
3. **Симулятор** — движок, который исполняет сделки виртуально
4. **Метрики** — показатели эффективности (доходность, просадка, Sharpe ratio)

```
Исторические данные
        ↓
┌─────────────────────┐
│  Торговая стратегия │
│  ─────────────────  │
│  Правила покупки    │
│  Правила продажи    │
│  Управление риском  │
└─────────────────────┘
        ↓
    Симулятор
        ↓
   Результаты
   (P&L, метрики)
```

## Базовая структура для бэктестинга

```rust
use std::collections::VecDeque;

/// Свеча (OHLCV данные)
#[derive(Debug, Clone)]
struct Candle {
    timestamp: u64,
    open: f64,
    high: f64,
    low: f64,
    close: f64,
    volume: f64,
}

/// Тип ордера
#[derive(Debug, Clone, PartialEq)]
enum OrderSide {
    Buy,
    Sell,
}

/// Сделка
#[derive(Debug, Clone)]
struct Trade {
    timestamp: u64,
    side: OrderSide,
    price: f64,
    quantity: f64,
}

/// Позиция
#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    quantity: f64,
    entry_price: f64,
}

/// Состояние портфеля
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

## Простая стратегия: пересечение скользящих средних

```rust
/// Скользящее среднее
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

/// Сигнал стратегии
#[derive(Debug, Clone, PartialEq)]
enum Signal {
    Buy,
    Sell,
    Hold,
}

/// Стратегия пересечения скользящих средних
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
                // Быстрая MA пересекает медленную снизу вверх — покупка
                if pf <= ps && f > s {
                    Signal::Buy
                }
                // Быстрая MA пересекает медленную сверху вниз — продажа
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

## Движок бэктестинга

```rust
/// Результаты бэктестинга
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

/// Движок бэктестинга
struct Backtester {
    portfolio: Portfolio,
    symbol: String,
    position_size: f64,  // Размер позиции в долларах
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
                // Проверяем, нет ли уже открытой позиции
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
                // Закрываем позицию, если она есть
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

            // Записываем текущую стоимость портфеля
            let current_value = self.portfolio.total_value(
                &[(self.symbol.clone(), candle.close)]
            );
            equity_curve.push(current_value);
        }

        // Закрываем открытую позицию по последней цене
        if let Some(last_candle) = candles.last() {
            if !self.portfolio.positions.is_empty() {
                self.execute_signal(Signal::Sell, last_candle);
            }
        }

        // Вычисляем метрики
        let final_capital = self.portfolio.cash;
        let total_return = (final_capital - initial_capital) / initial_capital * 100.0;

        let (winning_trades, losing_trades) = self.calculate_win_loss();
        let max_drawdown = self.calculate_max_drawdown(&equity_curve);
        let sharpe_ratio = self.calculate_sharpe_ratio(&equity_curve);

        BacktestResult {
            initial_capital,
            final_capital,
            total_return,
            total_trades: self.portfolio.trades.len() / 2, // Пары buy/sell
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

        // Вычисляем дневные доходности
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
            // Аннуализированный Sharpe ratio (предполагаем 252 торговых дня)
            (mean_return / std_dev) * (252.0_f64).sqrt()
        }
    }
}
```

## Полный пример бэктестинга

```rust
fn main() {
    // Генерируем тестовые данные (в реальности загружаются из файла/API)
    let candles = generate_sample_data();

    println!("=== Бэктестинг стратегии MA Crossover ===\n");
    println!("Символ: BTC/USDT");
    println!("Период: {} свечей", candles.len());
    println!("Начальный капитал: $10,000");
    println!("Размер позиции: $1,000\n");

    // Создаём стратегию: быстрая MA(10), медленная MA(30)
    let mut strategy = MACrossStrategy::new(10, 30);

    // Создаём бэктестер
    let mut backtester = Backtester::new(10_000.0, "BTC/USDT", 1_000.0);

    // Запускаем бэктест
    println!("--- Сделки ---");
    let result = backtester.run(&candles, &mut strategy);

    // Выводим результаты
    println!("\n--- Результаты ---");
    println!("Начальный капитал: ${:.2}", result.initial_capital);
    println!("Конечный капитал: ${:.2}", result.final_capital);
    println!("Общая доходность: {:.2}%", result.total_return);
    println!("Всего сделок: {}", result.total_trades);
    println!("Прибыльных: {}", result.winning_trades);
    println!("Убыточных: {}", result.losing_trades);
    println!("Win Rate: {:.1}%", result.win_rate());
    println!("Макс. просадка: {:.2}%", result.max_drawdown);
    println!("Sharpe Ratio: {:.2}", result.sharpe_ratio);
}

fn generate_sample_data() -> Vec<Candle> {
    use std::f64::consts::PI;

    let mut candles = Vec::new();
    let mut price = 40000.0;

    for i in 0..500 {
        // Симулируем движение цены с трендом и волатильностью
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

## Важные аспекты бэктестинга

### 1. Избегайте переобучения (Overfitting)

```rust
/// Разделение данных на обучающую и тестовую выборки
fn split_data(candles: &[Candle], train_ratio: f64) -> (&[Candle], &[Candle]) {
    let split_index = (candles.len() as f64 * train_ratio) as usize;
    (&candles[..split_index], &candles[split_index..])
}

fn validate_strategy() {
    let all_candles = generate_sample_data();

    // 70% для обучения, 30% для тестирования
    let (train_data, test_data) = split_data(&all_candles, 0.7);

    println!("Обучающая выборка: {} свечей", train_data.len());
    println!("Тестовая выборка: {} свечей", test_data.len());

    // Оптимизируем параметры на обучающей выборке
    let best_params = optimize_on_train(train_data);

    // Проверяем на тестовой выборке
    let mut strategy = MACrossStrategy::new(best_params.0, best_params.1);
    let mut backtester = Backtester::new(10_000.0, "BTC/USDT", 1_000.0);
    let result = backtester.run(test_data, &mut strategy);

    println!("\nРезультаты на тестовых данных:");
    println!("Доходность: {:.2}%", result.total_return);
}

fn optimize_on_train(data: &[Candle]) -> (usize, usize) {
    // Простой перебор параметров
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

    println!("Лучшие параметры: fast={}, slow={}", best_params.0, best_params.1);
    println!("Доходность на обучающих данных: {:.2}%", best_return);

    best_params
}
```

### 2. Учёт комиссий и проскальзывания

```rust
struct BacktesterWithCosts {
    portfolio: Portfolio,
    symbol: String,
    position_size: f64,
    commission_rate: f64,  // Комиссия (например, 0.001 = 0.1%)
    slippage: f64,         // Проскальзывание (например, 0.0005 = 0.05%)
}

impl BacktesterWithCosts {
    fn execute_with_costs(&mut self, signal: Signal, candle: &Candle) {
        match signal {
            Signal::Buy => {
                if self.portfolio.positions.is_empty() {
                    // Цена с учётом проскальзывания (покупаем дороже)
                    let exec_price = candle.close * (1.0 + self.slippage);
                    let quantity = self.position_size / exec_price;

                    // Комиссия
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
                    // Цена с учётом проскальзывания (продаём дешевле)
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

### 3. Управление рисками

```rust
/// Стоп-лосс и тейк-профит
struct RiskManager {
    stop_loss_pct: f64,   // Стоп-лосс в процентах
    take_profit_pct: f64, // Тейк-профит в процентах
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

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Бэктестинг | Тестирование стратегии на исторических данных |
| OHLCV | Open, High, Low, Close, Volume — формат свечных данных |
| Скользящее среднее | Индикатор для сглаживания ценовых колебаний |
| MA Crossover | Стратегия на пересечении быстрой и медленной MA |
| Equity Curve | Кривая капитала — динамика стоимости портфеля |
| Max Drawdown | Максимальная просадка от пика до минимума |
| Sharpe Ratio | Отношение доходности к риску (волатильности) |
| Переобучение | Оптимизация под исторические данные без обобщения |
| Проскальзывание | Разница между ожидаемой и фактической ценой исполнения |

## Домашнее задание

1. **Новая стратегия**: Реализуй стратегию на основе RSI (Relative Strength Index):
   - RSI > 70 — продажа (перекупленность)
   - RSI < 30 — покупка (перепроданность)
   - Протестируй на исторических данных

2. **Оптимизация параметров**: Создай функцию для grid search оптимизации:
   - Перебери различные комбинации параметров стратегии
   - Найди оптимальные значения по критерию Sharpe Ratio
   - Визуализируй результаты в виде таблицы

3. **Множественные активы**: Расширь бэктестер для работы с несколькими активами:
   - Добавь возможность торговать BTC, ETH и SOL одновременно
   - Реализуй ребалансировку портфеля
   - Вычисли корреляцию между активами

4. **Walk-Forward анализ**: Реализуй walk-forward тестирование:
   - Раздели данные на последовательные окна
   - Оптимизируй на каждом окне, тестируй на следующем
   - Сравни результаты с обычным бэктестингом

## Навигация

[← Предыдущий день](../274-strategy-pattern-trading/ru.md) | [Следующий день →](../276-historical-data-formats/ru.md)
