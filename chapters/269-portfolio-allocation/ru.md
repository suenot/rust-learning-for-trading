# День 269: Распределение портфеля (Portfolio Allocation)

## Аналогия из трейдинга

Представь, что ты управляешь инвестиционным портфелем. У тебя есть определённая сумма денег, и ты хочешь распределить её между разными активами: акциями, криптовалютами, облигациями. Как опытный трейдер не кладёт все яйца в одну корзину, так и алгоритм распределения портфеля должен учитывать риски, корреляции активов и целевые доли каждого инструмента.

**Пример из реального трейдинга:**
- Портфель на $100,000
- 40% — BTC (Bitcoin)
- 30% — ETH (Ethereum)
- 20% — SOL (Solana)
- 10% — стейблкоины (USDT)

Когда цены меняются, доли "плывут". Если BTC вырос на 50%, его доля теперь больше 40%. Нужна **ребалансировка** — продать часть BTC и докупить другие активы, чтобы вернуться к целевым долям.

## Основные концепции Rust

В этой главе мы используем:
- **Структуры (struct)** — для представления активов и позиций
- **HashMap** — для хранения портфеля
- **Методы и impl блоки** — для операций над портфелем
- **Итераторы** — для расчёта метрик
- **Result и Option** — для обработки ошибок

## Базовая структура портфеля

```rust
use std::collections::HashMap;

/// Представляет отдельный актив в портфеле
#[derive(Debug, Clone)]
struct Asset {
    symbol: String,
    quantity: f64,
    current_price: f64,
}

impl Asset {
    fn new(symbol: &str, quantity: f64, current_price: f64) -> Self {
        Asset {
            symbol: symbol.to_string(),
            quantity,
            current_price,
        }
    }

    /// Рассчитывает стоимость позиции
    fn value(&self) -> f64 {
        self.quantity * self.current_price
    }
}

/// Портфель — коллекция активов
#[derive(Debug)]
struct Portfolio {
    assets: HashMap<String, Asset>,
    cash: f64,
}

impl Portfolio {
    fn new(initial_cash: f64) -> Self {
        Portfolio {
            assets: HashMap::new(),
            cash: initial_cash,
        }
    }

    /// Добавляет актив в портфель
    fn add_asset(&mut self, symbol: &str, quantity: f64, price: f64) {
        let cost = quantity * price;
        if cost <= self.cash {
            self.cash -= cost;
            self.assets
                .entry(symbol.to_string())
                .and_modify(|asset| {
                    asset.quantity += quantity;
                    asset.current_price = price;
                })
                .or_insert(Asset::new(symbol, quantity, price));
        }
    }

    /// Общая стоимость портфеля
    fn total_value(&self) -> f64 {
        let assets_value: f64 = self.assets.values().map(|a| a.value()).sum();
        assets_value + self.cash
    }

    /// Рассчитывает текущие доли каждого актива
    fn current_allocations(&self) -> HashMap<String, f64> {
        let total = self.total_value();
        let mut allocations = HashMap::new();

        for (symbol, asset) in &self.assets {
            let allocation = asset.value() / total * 100.0;
            allocations.insert(symbol.clone(), allocation);
        }

        // Добавляем долю кэша
        allocations.insert("CASH".to_string(), self.cash / total * 100.0);

        allocations
    }
}

fn main() {
    let mut portfolio = Portfolio::new(100_000.0);

    portfolio.add_asset("BTC", 1.0, 42_000.0);
    portfolio.add_asset("ETH", 10.0, 2_200.0);
    portfolio.add_asset("SOL", 100.0, 100.0);

    println!("Общая стоимость портфеля: ${:.2}", portfolio.total_value());
    println!("\nТекущие доли:");
    for (symbol, allocation) in portfolio.current_allocations() {
        println!("  {}: {:.2}%", symbol, allocation);
    }
}
```

## Целевое распределение и ребалансировка

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Asset {
    symbol: String,
    quantity: f64,
    current_price: f64,
}

impl Asset {
    fn new(symbol: &str, quantity: f64, current_price: f64) -> Self {
        Asset {
            symbol: symbol.to_string(),
            quantity,
            current_price,
        }
    }

    fn value(&self) -> f64 {
        self.quantity * self.current_price
    }
}

/// Ордер на ребалансировку
#[derive(Debug)]
struct RebalanceOrder {
    symbol: String,
    action: String, // "BUY" или "SELL"
    quantity: f64,
    value: f64,
}

#[derive(Debug)]
struct Portfolio {
    assets: HashMap<String, Asset>,
    cash: f64,
    target_allocations: HashMap<String, f64>, // Целевые доли в процентах
}

impl Portfolio {
    fn new(initial_cash: f64) -> Self {
        Portfolio {
            assets: HashMap::new(),
            cash: initial_cash,
            target_allocations: HashMap::new(),
        }
    }

    fn add_asset(&mut self, symbol: &str, quantity: f64, price: f64) {
        let cost = quantity * price;
        if cost <= self.cash {
            self.cash -= cost;
            self.assets
                .entry(symbol.to_string())
                .and_modify(|asset| {
                    asset.quantity += quantity;
                    asset.current_price = price;
                })
                .or_insert(Asset::new(symbol, quantity, price));
        }
    }

    /// Устанавливает целевое распределение
    fn set_target_allocation(&mut self, symbol: &str, percentage: f64) {
        self.target_allocations.insert(symbol.to_string(), percentage);
    }

    fn total_value(&self) -> f64 {
        let assets_value: f64 = self.assets.values().map(|a| a.value()).sum();
        assets_value + self.cash
    }

    fn current_allocations(&self) -> HashMap<String, f64> {
        let total = self.total_value();
        let mut allocations = HashMap::new();

        for (symbol, asset) in &self.assets {
            let allocation = asset.value() / total * 100.0;
            allocations.insert(symbol.clone(), allocation);
        }

        allocations
    }

    /// Обновляет цены активов
    fn update_prices(&mut self, prices: &HashMap<String, f64>) {
        for (symbol, price) in prices {
            if let Some(asset) = self.assets.get_mut(symbol) {
                asset.current_price = *price;
            }
        }
    }

    /// Рассчитывает необходимые сделки для ребалансировки
    fn calculate_rebalance(&self) -> Vec<RebalanceOrder> {
        let total = self.total_value();
        let current = self.current_allocations();
        let mut orders = Vec::new();

        for (symbol, target_pct) in &self.target_allocations {
            let current_pct = current.get(symbol).unwrap_or(&0.0);
            let diff_pct = target_pct - current_pct;

            if diff_pct.abs() > 0.5 {
                // Порог в 0.5% для избежания мелких сделок
                let diff_value = diff_pct / 100.0 * total;

                if let Some(asset) = self.assets.get(symbol) {
                    let quantity = diff_value.abs() / asset.current_price;

                    orders.push(RebalanceOrder {
                        symbol: symbol.clone(),
                        action: if diff_pct > 0.0 {
                            "BUY".to_string()
                        } else {
                            "SELL".to_string()
                        },
                        quantity,
                        value: diff_value.abs(),
                    });
                }
            }
        }

        // Сортируем: сначала продажи, потом покупки
        orders.sort_by(|a, b| b.action.cmp(&a.action));
        orders
    }
}

fn main() {
    let mut portfolio = Portfolio::new(100_000.0);

    // Начальное распределение
    portfolio.add_asset("BTC", 1.0, 42_000.0);
    portfolio.add_asset("ETH", 10.0, 2_200.0);
    portfolio.add_asset("SOL", 100.0, 100.0);

    // Устанавливаем целевые доли
    portfolio.set_target_allocation("BTC", 40.0);
    portfolio.set_target_allocation("ETH", 35.0);
    portfolio.set_target_allocation("SOL", 15.0);
    // 10% остаётся в кэше

    println!("=== Начальное состояние ===");
    println!("Общая стоимость: ${:.2}", portfolio.total_value());
    println!("Текущие доли: {:?}", portfolio.current_allocations());

    // Симулируем изменение цен
    let new_prices: HashMap<String, f64> = [
        ("BTC".to_string(), 55_000.0), // BTC вырос!
        ("ETH".to_string(), 2_000.0),  // ETH немного упал
        ("SOL".to_string(), 120.0),    // SOL вырос
    ]
    .into_iter()
    .collect();

    portfolio.update_prices(&new_prices);

    println!("\n=== После изменения цен ===");
    println!("Общая стоимость: ${:.2}", portfolio.total_value());
    println!("Текущие доли: {:?}", portfolio.current_allocations());

    // Рассчитываем ребалансировку
    println!("\n=== Ордера на ребалансировку ===");
    for order in portfolio.calculate_rebalance() {
        println!(
            "{} {} {:.4} (${:.2})",
            order.action, order.symbol, order.quantity, order.value
        );
    }
}
```

## Стратегии распределения портфеля

```rust
use std::collections::HashMap;

/// Стратегия распределения
trait AllocationStrategy {
    fn calculate(&self, symbols: &[String], total_value: f64) -> HashMap<String, f64>;
    fn name(&self) -> &str;
}

/// Равномерное распределение
struct EqualWeight;

impl AllocationStrategy for EqualWeight {
    fn calculate(&self, symbols: &[String], total_value: f64) -> HashMap<String, f64> {
        let weight = total_value / symbols.len() as f64;
        symbols.iter().map(|s| (s.clone(), weight)).collect()
    }

    fn name(&self) -> &str {
        "Equal Weight"
    }
}

/// Распределение по рыночной капитализации
struct MarketCapWeight {
    market_caps: HashMap<String, f64>,
}

impl MarketCapWeight {
    fn new(caps: HashMap<String, f64>) -> Self {
        MarketCapWeight { market_caps: caps }
    }
}

impl AllocationStrategy for MarketCapWeight {
    fn calculate(&self, symbols: &[String], total_value: f64) -> HashMap<String, f64> {
        let total_cap: f64 = symbols
            .iter()
            .filter_map(|s| self.market_caps.get(s))
            .sum();

        symbols
            .iter()
            .filter_map(|s| {
                self.market_caps
                    .get(s)
                    .map(|cap| (s.clone(), cap / total_cap * total_value))
            })
            .collect()
    }

    fn name(&self) -> &str {
        "Market Cap Weight"
    }
}

/// Распределение с учётом риска (обратно пропорционально волатильности)
struct RiskParity {
    volatilities: HashMap<String, f64>,
}

impl RiskParity {
    fn new(vols: HashMap<String, f64>) -> Self {
        RiskParity { volatilities: vols }
    }
}

impl AllocationStrategy for RiskParity {
    fn calculate(&self, symbols: &[String], total_value: f64) -> HashMap<String, f64> {
        // Инвертируем волатильности
        let inverse_vols: Vec<(String, f64)> = symbols
            .iter()
            .filter_map(|s| self.volatilities.get(s).map(|v| (s.clone(), 1.0 / v)))
            .collect();

        let total_inverse: f64 = inverse_vols.iter().map(|(_, v)| v).sum();

        inverse_vols
            .into_iter()
            .map(|(s, inv_vol)| (s, inv_vol / total_inverse * total_value))
            .collect()
    }

    fn name(&self) -> &str {
        "Risk Parity"
    }
}

/// Менеджер портфеля с настраиваемой стратегией
struct PortfolioManager {
    strategy: Box<dyn AllocationStrategy>,
    symbols: Vec<String>,
    total_value: f64,
}

impl PortfolioManager {
    fn new(strategy: Box<dyn AllocationStrategy>, symbols: Vec<String>, total_value: f64) -> Self {
        PortfolioManager {
            strategy,
            symbols,
            total_value,
        }
    }

    fn get_allocation(&self) -> HashMap<String, f64> {
        self.strategy.calculate(&self.symbols, self.total_value)
    }

    fn print_allocation(&self) {
        println!("\nСтратегия: {}", self.strategy.name());
        println!("Общий капитал: ${:.2}", self.total_value);
        println!("-".repeat(30));

        for (symbol, value) in self.get_allocation() {
            let pct = value / self.total_value * 100.0;
            println!("  {}: ${:.2} ({:.1}%)", symbol, value, pct);
        }
    }
}

fn main() {
    let symbols = vec![
        "BTC".to_string(),
        "ETH".to_string(),
        "SOL".to_string(),
        "AVAX".to_string(),
    ];

    let total_capital = 100_000.0;

    // 1. Равномерное распределение
    let equal = PortfolioManager::new(
        Box::new(EqualWeight),
        symbols.clone(),
        total_capital,
    );
    equal.print_allocation();

    // 2. По рыночной капитализации
    let market_caps: HashMap<String, f64> = [
        ("BTC".to_string(), 800_000_000_000.0),  // $800B
        ("ETH".to_string(), 250_000_000_000.0),  // $250B
        ("SOL".to_string(), 40_000_000_000.0),   // $40B
        ("AVAX".to_string(), 15_000_000_000.0),  // $15B
    ]
    .into_iter()
    .collect();

    let mcap = PortfolioManager::new(
        Box::new(MarketCapWeight::new(market_caps)),
        symbols.clone(),
        total_capital,
    );
    mcap.print_allocation();

    // 3. Risk Parity (меньше волатильных активов)
    let volatilities: HashMap<String, f64> = [
        ("BTC".to_string(), 0.60),   // 60% годовая волатильность
        ("ETH".to_string(), 0.75),   // 75%
        ("SOL".to_string(), 1.20),   // 120%
        ("AVAX".to_string(), 1.10),  // 110%
    ]
    .into_iter()
    .collect();

    let risk_parity = PortfolioManager::new(
        Box::new(RiskParity::new(volatilities)),
        symbols.clone(),
        total_capital,
    );
    risk_parity.print_allocation();
}
```

## Практический пример: Полная система управления портфелем

```rust
use std::collections::HashMap;
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    quantity: f64,
    avg_entry_price: f64,
    current_price: f64,
}

impl Position {
    fn value(&self) -> f64 {
        self.quantity * self.current_price
    }

    fn unrealized_pnl(&self) -> f64 {
        (self.current_price - self.avg_entry_price) * self.quantity
    }

    fn unrealized_pnl_pct(&self) -> f64 {
        (self.current_price - self.avg_entry_price) / self.avg_entry_price * 100.0
    }
}

#[derive(Debug)]
struct Trade {
    timestamp: u64,
    symbol: String,
    side: String,
    quantity: f64,
    price: f64,
}

#[derive(Debug)]
struct PortfolioMetrics {
    total_value: f64,
    cash_balance: f64,
    invested_value: f64,
    unrealized_pnl: f64,
    allocation_drift: f64,
}

struct TradingPortfolio {
    positions: HashMap<String, Position>,
    cash: f64,
    target_allocations: HashMap<String, f64>,
    trade_history: Vec<Trade>,
    rebalance_threshold: f64, // Порог отклонения для ребалансировки
}

impl TradingPortfolio {
    fn new(initial_cash: f64, rebalance_threshold: f64) -> Self {
        TradingPortfolio {
            positions: HashMap::new(),
            cash: initial_cash,
            target_allocations: HashMap::new(),
            trade_history: Vec::new(),
            rebalance_threshold,
        }
    }

    fn set_targets(&mut self, targets: HashMap<String, f64>) {
        self.target_allocations = targets;
    }

    fn execute_trade(&mut self, symbol: &str, side: &str, quantity: f64, price: f64) -> Result<(), String> {
        let cost = quantity * price;
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();

        match side {
            "BUY" => {
                if cost > self.cash {
                    return Err(format!("Недостаточно средств: нужно ${:.2}, доступно ${:.2}", cost, self.cash));
                }

                self.cash -= cost;

                self.positions
                    .entry(symbol.to_string())
                    .and_modify(|pos| {
                        let total_cost = pos.avg_entry_price * pos.quantity + price * quantity;
                        let total_qty = pos.quantity + quantity;
                        pos.avg_entry_price = total_cost / total_qty;
                        pos.quantity = total_qty;
                        pos.current_price = price;
                    })
                    .or_insert(Position {
                        symbol: symbol.to_string(),
                        quantity,
                        avg_entry_price: price,
                        current_price: price,
                    });
            }
            "SELL" => {
                let position = self.positions.get_mut(symbol)
                    .ok_or_else(|| format!("Нет позиции по {}", symbol))?;

                if quantity > position.quantity {
                    return Err(format!("Недостаточно {}: есть {:.4}, нужно {:.4}", symbol, position.quantity, quantity));
                }

                position.quantity -= quantity;
                self.cash += cost;

                if position.quantity < 0.0001 {
                    self.positions.remove(symbol);
                }
            }
            _ => return Err("Неверная сторона сделки".to_string()),
        }

        self.trade_history.push(Trade {
            timestamp,
            symbol: symbol.to_string(),
            side: side.to_string(),
            quantity,
            price,
        });

        Ok(())
    }

    fn update_prices(&mut self, prices: &HashMap<String, f64>) {
        for (symbol, price) in prices {
            if let Some(pos) = self.positions.get_mut(symbol) {
                pos.current_price = *price;
            }
        }
    }

    fn total_value(&self) -> f64 {
        let positions_value: f64 = self.positions.values().map(|p| p.value()).sum();
        positions_value + self.cash
    }

    fn current_allocations(&self) -> HashMap<String, f64> {
        let total = self.total_value();
        self.positions
            .iter()
            .map(|(symbol, pos)| (symbol.clone(), pos.value() / total * 100.0))
            .collect()
    }

    fn calculate_drift(&self) -> f64 {
        let current = self.current_allocations();
        let mut max_drift = 0.0f64;

        for (symbol, target) in &self.target_allocations {
            let actual = current.get(symbol).unwrap_or(&0.0);
            let drift = (target - actual).abs();
            max_drift = max_drift.max(drift);
        }

        max_drift
    }

    fn needs_rebalance(&self) -> bool {
        self.calculate_drift() > self.rebalance_threshold
    }

    fn metrics(&self) -> PortfolioMetrics {
        let total_value = self.total_value();
        let invested_value: f64 = self.positions.values().map(|p| p.value()).sum();
        let unrealized_pnl: f64 = self.positions.values().map(|p| p.unrealized_pnl()).sum();

        PortfolioMetrics {
            total_value,
            cash_balance: self.cash,
            invested_value,
            unrealized_pnl,
            allocation_drift: self.calculate_drift(),
        }
    }

    fn print_summary(&self) {
        let metrics = self.metrics();
        let current = self.current_allocations();

        println!("\n{'=':.^50}", " ПОРТФЕЛЬ ");
        println!("Общая стоимость:     ${:>12.2}", metrics.total_value);
        println!("Инвестировано:       ${:>12.2}", metrics.invested_value);
        println!("Свободные средства:  ${:>12.2}", metrics.cash_balance);
        println!("Нереализованный PnL: ${:>12.2}", metrics.unrealized_pnl);
        println!("Макс. отклонение:    {:>12.2}%", metrics.allocation_drift);

        println!("\n{:-^50}", " ПОЗИЦИИ ");
        println!("{:<8} {:>10} {:>12} {:>10} {:>8}", "Актив", "Кол-во", "Стоимость", "PnL", "Доля");
        println!("{}", "-".repeat(50));

        for (symbol, pos) in &self.positions {
            let allocation = current.get(symbol).unwrap_or(&0.0);
            let target = self.target_allocations.get(symbol).unwrap_or(&0.0);
            let drift_marker = if (allocation - target).abs() > self.rebalance_threshold { "*" } else { "" };

            println!(
                "{:<8} {:>10.4} ${:>11.2} {:>+9.2} {:>7.1}%{}",
                symbol,
                pos.quantity,
                pos.value(),
                pos.unrealized_pnl(),
                allocation,
                drift_marker
            );
        }

        if self.needs_rebalance() {
            println!("\n[!] Требуется ребалансировка (порог {}%)", self.rebalance_threshold);
        }
    }
}

fn main() {
    // Создаём портфель с порогом ребалансировки 5%
    let mut portfolio = TradingPortfolio::new(100_000.0, 5.0);

    // Устанавливаем целевое распределение
    let targets: HashMap<String, f64> = [
        ("BTC".to_string(), 40.0),
        ("ETH".to_string(), 30.0),
        ("SOL".to_string(), 20.0),
    ]
    .into_iter()
    .collect();
    portfolio.set_targets(targets);

    // Выполняем начальные покупки
    println!("=== НАЧАЛЬНЫЕ ПОКУПКИ ===");
    let _ = portfolio.execute_trade("BTC", "BUY", 0.95, 42_000.0);
    let _ = portfolio.execute_trade("ETH", "BUY", 13.6, 2_200.0);
    let _ = portfolio.execute_trade("SOL", "BUY", 200.0, 100.0);

    portfolio.print_summary();

    // Симулируем изменение цен
    println!("\n=== ПОСЛЕ ИЗМЕНЕНИЯ ЦЕН ===");
    let new_prices: HashMap<String, f64> = [
        ("BTC".to_string(), 52_000.0),
        ("ETH".to_string(), 2_100.0),
        ("SOL".to_string(), 85.0),
    ]
    .into_iter()
    .collect();
    portfolio.update_prices(&new_prices);

    portfolio.print_summary();

    // История сделок
    println!("\n{:-^50}", " ИСТОРИЯ СДЕЛОК ");
    for trade in &portfolio.trade_history {
        println!(
            "[{}] {} {} {:.4} @ ${:.2}",
            trade.timestamp, trade.side, trade.symbol, trade.quantity, trade.price
        );
    }
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Portfolio Allocation | Распределение капитала между различными активами |
| Ребалансировка | Приведение портфеля к целевым долям |
| HashMap | Эффективное хранение позиций по символу актива |
| Trait objects | Гибкие стратегии распределения через `dyn AllocationStrategy` |
| Метрики портфеля | Расчёт стоимости, PnL, отклонения от целей |
| Entry API | Удобное обновление или создание записей в HashMap |

## Домашнее задание

1. **Калькулятор ребалансировки**: Создай функцию, которая принимает текущий портфель и целевые доли, и возвращает список конкретных ордеров (symbol, side, quantity, estimated_cost) для ребалансировки.

2. **Стратегия Momentum**: Реализуй стратегию распределения, которая выделяет больше капитала активам с положительной доходностью за последний период (например, 30 дней).

3. **Лимиты позиций**: Добавь в структуру `TradingPortfolio`:
   - Максимальную долю на один актив (например, 50%)
   - Минимальный размер позиции
   - Проверку этих лимитов при исполнении сделок

4. **Отчёт о производительности**: Реализуй метод, который рассчитывает:
   - Общую доходность портфеля
   - Доходность каждой позиции
   - Sharpe ratio (если добавить историю цен)

## Навигация

[← Предыдущий день](../268-portfolio-metrics/ru.md) | [Следующий день →](../270-risk-adjusted-returns/ru.md)
