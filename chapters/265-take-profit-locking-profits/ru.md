# День 265: Take-Profit: фиксация прибыли

## Аналогия из трейдинга

Представь, что ты купил Bitcoin по $40,000 и он начал расти. Цена достигла $44,000 — у тебя прибыль 10%! Но что если не зафиксировать прибыль, а цена развернётся и упадёт обратно к $40,000? Вся бумажная прибыль испарится.

**Take-Profit (TP)** — это автоматический ордер на закрытие позиции при достижении целевой цены. Это как страховка от жадности: ты заранее решаешь, при какой прибыли выходишь из сделки, и система делает это автоматически.

В реальном трейдинге take-profit помогает:
- Фиксировать прибыль без эмоций
- Защищать от разворота цены
- Дисциплинировать торговую стратегию
- Освобождать капитал для новых сделок

## Базовая структура Take-Profit ордера

```rust
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum OrderSide {
    Buy,
    Sell,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum OrderStatus {
    Pending,
    Triggered,
    Filled,
    Cancelled,
}

#[derive(Debug, Clone)]
pub struct TakeProfitOrder {
    pub id: u64,
    pub symbol: String,
    pub side: OrderSide,           // Направление закрытия (противоположное позиции)
    pub quantity: f64,             // Количество для закрытия
    pub trigger_price: f64,        // Цена активации take-profit
    pub status: OrderStatus,
    pub created_at: u64,           // Timestamp создания
    pub triggered_at: Option<u64>, // Timestamp срабатывания
}

impl TakeProfitOrder {
    pub fn new(
        id: u64,
        symbol: &str,
        side: OrderSide,
        quantity: f64,
        trigger_price: f64,
        created_at: u64,
    ) -> Self {
        TakeProfitOrder {
            id,
            symbol: symbol.to_string(),
            side,
            quantity,
            trigger_price,
            status: OrderStatus::Pending,
            created_at,
            triggered_at: None,
        }
    }

    /// Проверяем, должен ли take-profit сработать при текущей цене
    pub fn should_trigger(&self, current_price: f64) -> bool {
        if self.status != OrderStatus::Pending {
            return false;
        }

        match self.side {
            // Для длинной позиции: продаём когда цена >= trigger_price
            OrderSide::Sell => current_price >= self.trigger_price,
            // Для короткой позиции: покупаем когда цена <= trigger_price
            OrderSide::Buy => current_price <= self.trigger_price,
        }
    }

    /// Активируем take-profit
    pub fn trigger(&mut self, timestamp: u64) {
        self.status = OrderStatus::Triggered;
        self.triggered_at = Some(timestamp);
    }

    /// Отмечаем как исполненный
    pub fn fill(&mut self) {
        self.status = OrderStatus::Filled;
    }
}

impl fmt::Display for TakeProfitOrder {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "TP#{} {} {} {} @ {} [{:?}]",
            self.id,
            self.symbol,
            match self.side {
                OrderSide::Buy => "BUY",
                OrderSide::Sell => "SELL",
            },
            self.quantity,
            self.trigger_price,
            self.status
        )
    }
}

fn main() {
    // Создаём take-profit для длинной позиции BTC
    // Купили по $40,000, хотим зафиксировать прибыль на $44,000
    let mut tp_order = TakeProfitOrder::new(
        1,
        "BTC/USDT",
        OrderSide::Sell,
        0.5,
        44000.0,
        1000,
    );

    println!("Создан ордер: {}", tp_order);

    // Симуляция движения цены
    let prices = [41000.0, 42500.0, 43800.0, 44200.0, 44500.0];

    for (i, &price) in prices.iter().enumerate() {
        println!("\nТик {}: Цена = ${}", i + 1, price);

        if tp_order.should_trigger(price) {
            tp_order.trigger(1000 + i as u64);
            println!(">>> Take-Profit сработал! {}", tp_order);
            tp_order.fill();
            println!(">>> Ордер исполнен: {}", tp_order);
            break;
        } else {
            println!("   Take-Profit ожидает (нужно >= ${})", tp_order.trigger_price);
        }
    }
}
```

## Расчёт Take-Profit цены

```rust
#[derive(Debug, Clone, Copy)]
pub enum TakeProfitStrategy {
    /// Фиксированный процент прибыли
    PercentageGain(f64),
    /// Фиксированная сумма в пунктах
    FixedPoints(f64),
    /// Соотношение риск/прибыль (относительно stop-loss)
    RiskRewardRatio { stop_loss: f64, ratio: f64 },
    /// Уровень сопротивления/поддержки
    PriceLevel(f64),
}

pub struct TakeProfitCalculator;

impl TakeProfitCalculator {
    /// Рассчитать цену take-profit для длинной позиции
    pub fn calculate_long(
        entry_price: f64,
        strategy: TakeProfitStrategy,
    ) -> f64 {
        match strategy {
            TakeProfitStrategy::PercentageGain(percent) => {
                entry_price * (1.0 + percent / 100.0)
            }
            TakeProfitStrategy::FixedPoints(points) => {
                entry_price + points
            }
            TakeProfitStrategy::RiskRewardRatio { stop_loss, ratio } => {
                let risk = entry_price - stop_loss;
                entry_price + (risk * ratio)
            }
            TakeProfitStrategy::PriceLevel(level) => level,
        }
    }

    /// Рассчитать цену take-profit для короткой позиции
    pub fn calculate_short(
        entry_price: f64,
        strategy: TakeProfitStrategy,
    ) -> f64 {
        match strategy {
            TakeProfitStrategy::PercentageGain(percent) => {
                entry_price * (1.0 - percent / 100.0)
            }
            TakeProfitStrategy::FixedPoints(points) => {
                entry_price - points
            }
            TakeProfitStrategy::RiskRewardRatio { stop_loss, ratio } => {
                let risk = stop_loss - entry_price;
                entry_price - (risk * ratio)
            }
            TakeProfitStrategy::PriceLevel(level) => level,
        }
    }
}

fn main() {
    let entry_price = 40000.0;
    let stop_loss = 38000.0;

    println!("=== Расчёт Take-Profit для длинной позиции ===");
    println!("Цена входа: ${}", entry_price);
    println!("Stop-Loss: ${}", stop_loss);
    println!();

    // Разные стратегии
    let strategies = [
        ("5% прибыли", TakeProfitStrategy::PercentageGain(5.0)),
        ("10% прибыли", TakeProfitStrategy::PercentageGain(10.0)),
        ("+$3000", TakeProfitStrategy::FixedPoints(3000.0)),
        ("R:R 1:2", TakeProfitStrategy::RiskRewardRatio {
            stop_loss: 38000.0,
            ratio: 2.0,
        }),
        ("R:R 1:3", TakeProfitStrategy::RiskRewardRatio {
            stop_loss: 38000.0,
            ratio: 3.0,
        }),
        ("Уровень $45000", TakeProfitStrategy::PriceLevel(45000.0)),
    ];

    for (name, strategy) in strategies {
        let tp_price = TakeProfitCalculator::calculate_long(entry_price, strategy);
        let profit_percent = ((tp_price - entry_price) / entry_price) * 100.0;
        println!(
            "{}: TP = ${:.2} (+{:.2}%)",
            name, tp_price, profit_percent
        );
    }
}
```

## Множественные Take-Profit уровни

Профессиональные трейдеры часто используют несколько уровней take-profit для постепенной фиксации прибыли:

```rust
use std::collections::BTreeMap;

#[derive(Debug, Clone)]
pub struct MultiLevelTakeProfit {
    pub symbol: String,
    pub total_quantity: f64,
    /// Уровни: цена -> процент от позиции
    pub levels: BTreeMap<u64, TakeProfitLevel>,
    pub filled_quantity: f64,
}

#[derive(Debug, Clone)]
pub struct TakeProfitLevel {
    pub price: f64,
    pub percentage: f64,  // Процент от общей позиции
    pub quantity: f64,    // Рассчитанное количество
    pub is_filled: bool,
}

impl MultiLevelTakeProfit {
    pub fn new(symbol: &str, total_quantity: f64) -> Self {
        MultiLevelTakeProfit {
            symbol: symbol.to_string(),
            total_quantity,
            levels: BTreeMap::new(),
            filled_quantity: 0.0,
        }
    }

    /// Добавить уровень take-profit
    pub fn add_level(&mut self, price: f64, percentage: f64) -> Result<(), String> {
        // Проверяем, что общий процент не превышает 100%
        let current_total: f64 = self.levels.values()
            .map(|l| l.percentage)
            .sum();

        if current_total + percentage > 100.0 {
            return Err(format!(
                "Превышение 100%: текущий {:.1}% + новый {:.1}% = {:.1}%",
                current_total, percentage, current_total + percentage
            ));
        }

        let quantity = self.total_quantity * (percentage / 100.0);
        let price_key = (price * 100.0) as u64; // Для сортировки

        self.levels.insert(price_key, TakeProfitLevel {
            price,
            percentage,
            quantity,
            is_filled: false,
        });

        Ok(())
    }

    /// Проверить и исполнить уровни при текущей цене
    pub fn check_and_fill(&mut self, current_price: f64) -> Vec<(f64, f64)> {
        let mut filled = Vec::new();

        for level in self.levels.values_mut() {
            if !level.is_filled && current_price >= level.price {
                level.is_filled = true;
                self.filled_quantity += level.quantity;
                filled.push((level.price, level.quantity));
            }
        }

        filled
    }

    /// Получить оставшееся количество
    pub fn remaining_quantity(&self) -> f64 {
        self.total_quantity - self.filled_quantity
    }

    /// Вывести статус
    pub fn print_status(&self) {
        println!("\n=== Статус Take-Profit для {} ===", self.symbol);
        println!("Общее количество: {}", self.total_quantity);
        println!("Исполнено: {}", self.filled_quantity);
        println!("Осталось: {}", self.remaining_quantity());
        println!("\nУровни:");

        for level in self.levels.values() {
            let status = if level.is_filled { "[ИСПОЛНЕН]" } else { "[ОЖИДАЕТ]" };
            println!(
                "  ${:.2}: {:.1}% ({} шт) {}",
                level.price, level.percentage, level.quantity, status
            );
        }
    }
}

fn main() {
    // Создаём многоуровневый take-profit
    let mut tp = MultiLevelTakeProfit::new("BTC/USDT", 1.0);

    // Добавляем уровни: постепенная фиксация прибыли
    tp.add_level(42000.0, 25.0).unwrap();  // 25% на $42,000
    tp.add_level(44000.0, 35.0).unwrap();  // 35% на $44,000
    tp.add_level(48000.0, 40.0).unwrap();  // 40% на $48,000

    tp.print_status();

    // Симуляция движения цены
    let prices = [41000.0, 42500.0, 43000.0, 44100.0, 46000.0, 48500.0];

    for price in prices {
        println!("\n--- Цена: ${} ---", price);
        let filled = tp.check_and_fill(price);

        for (tp_price, qty) in filled {
            println!(
                ">>> Сработал TP на ${}: продано {} BTC",
                tp_price, qty
            );
        }
    }

    tp.print_status();
}
```

## Take-Profit с Trailing (трейлинг)

Trailing take-profit позволяет "подтягивать" уровень фиксации прибыли за ценой:

```rust
#[derive(Debug, Clone)]
pub struct TrailingTakeProfit {
    pub symbol: String,
    pub quantity: f64,
    pub activation_price: f64,    // Цена активации трейлинга
    pub trailing_percent: f64,    // Отступ в процентах
    pub highest_price: f64,       // Максимальная достигнутая цена
    pub current_tp_price: f64,    // Текущий уровень TP
    pub is_activated: bool,       // Активирован ли трейлинг
    pub is_filled: bool,
}

impl TrailingTakeProfit {
    pub fn new(
        symbol: &str,
        quantity: f64,
        activation_price: f64,
        trailing_percent: f64,
    ) -> Self {
        TrailingTakeProfit {
            symbol: symbol.to_string(),
            quantity,
            activation_price,
            trailing_percent,
            highest_price: 0.0,
            current_tp_price: 0.0,
            is_activated: false,
            is_filled: false,
        }
    }

    /// Обновить при новой цене
    pub fn update(&mut self, current_price: f64) -> Option<f64> {
        if self.is_filled {
            return None;
        }

        // Проверяем активацию
        if !self.is_activated && current_price >= self.activation_price {
            self.is_activated = true;
            self.highest_price = current_price;
            self.current_tp_price = current_price * (1.0 - self.trailing_percent / 100.0);
            println!(
                "[Trailing TP] Активирован при ${:.2}, начальный TP: ${:.2}",
                current_price, self.current_tp_price
            );
        }

        if !self.is_activated {
            return None;
        }

        // Обновляем максимальную цену
        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.current_tp_price = current_price * (1.0 - self.trailing_percent / 100.0);
            println!(
                "[Trailing TP] Новый максимум ${:.2}, TP подтянут до ${:.2}",
                current_price, self.current_tp_price
            );
        }

        // Проверяем срабатывание
        if current_price <= self.current_tp_price {
            self.is_filled = true;
            println!(
                "[Trailing TP] ИСПОЛНЕН при ${:.2}! Прибыль зафиксирована",
                current_price
            );
            return Some(self.quantity);
        }

        None
    }

    pub fn status(&self) -> String {
        if self.is_filled {
            format!("ИСПОЛНЕН при ${:.2}", self.current_tp_price)
        } else if self.is_activated {
            format!(
                "Активен: макс ${:.2}, TP ${:.2}",
                self.highest_price, self.current_tp_price
            )
        } else {
            format!("Ожидает активации при ${:.2}", self.activation_price)
        }
    }
}

fn main() {
    // Создаём trailing take-profit
    // Активируется при $42,000, отступ 3%
    let mut trailing_tp = TrailingTakeProfit::new(
        "BTC/USDT",
        0.5,
        42000.0,
        3.0,
    );

    println!("=== Trailing Take-Profit ===");
    println!("Активация: $42,000");
    println!("Trailing: 3%");
    println!();

    // Симуляция: цена растёт, потом падает
    let prices = [
        40000.0, 41000.0, 42000.0, 43000.0, 44500.0,
        46000.0, 47000.0, 46500.0, 45000.0, 44000.0,
    ];

    for (i, &price) in prices.iter().enumerate() {
        println!("\nТик {}: Цена = ${}", i + 1, price);
        trailing_tp.update(price);
        println!("Статус: {}", trailing_tp.status());

        if trailing_tp.is_filled {
            break;
        }
    }

    // Расчёт итоговой прибыли
    if trailing_tp.is_filled {
        let entry = 40000.0;
        let exit = trailing_tp.current_tp_price;
        let profit = exit - entry;
        let profit_percent = (profit / entry) * 100.0;
        println!("\n=== Итог ===");
        println!("Вход: ${}", entry);
        println!("Выход: ${:.2}", exit);
        println!("Прибыль: ${:.2} ({:.2}%)", profit, profit_percent);
    }
}
```

## Интеграция Take-Profit в торговую систему

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct Position {
    pub id: u64,
    pub symbol: String,
    pub side: OrderSide,
    pub quantity: f64,
    pub entry_price: f64,
    pub take_profit: Option<f64>,
    pub stop_loss: Option<f64>,
}

#[derive(Debug)]
pub struct TradingSystem {
    positions: HashMap<u64, Position>,
    tp_orders: HashMap<u64, TakeProfitOrder>,
    next_position_id: u64,
    next_order_id: u64,
    current_timestamp: u64,
}

impl TradingSystem {
    pub fn new() -> Self {
        TradingSystem {
            positions: HashMap::new(),
            tp_orders: HashMap::new(),
            next_position_id: 1,
            next_order_id: 1,
            current_timestamp: 0,
        }
    }

    /// Открыть позицию с take-profit
    pub fn open_position(
        &mut self,
        symbol: &str,
        side: OrderSide,
        quantity: f64,
        entry_price: f64,
        take_profit_percent: Option<f64>,
    ) -> u64 {
        let position_id = self.next_position_id;
        self.next_position_id += 1;

        // Рассчитываем take-profit
        let take_profit = take_profit_percent.map(|percent| {
            match side {
                OrderSide::Buy => entry_price * (1.0 + percent / 100.0),
                OrderSide::Sell => entry_price * (1.0 - percent / 100.0),
            }
        });

        let position = Position {
            id: position_id,
            symbol: symbol.to_string(),
            side,
            quantity,
            entry_price,
            take_profit,
            stop_loss: None,
        };

        println!(
            "[Система] Открыта позиция #{}: {} {} {} @ ${}",
            position_id,
            match side { OrderSide::Buy => "LONG", OrderSide::Sell => "SHORT" },
            quantity,
            symbol,
            entry_price
        );

        // Создаём take-profit ордер
        if let Some(tp_price) = take_profit {
            let tp_side = match side {
                OrderSide::Buy => OrderSide::Sell,
                OrderSide::Sell => OrderSide::Buy,
            };

            let tp_order = TakeProfitOrder::new(
                self.next_order_id,
                symbol,
                tp_side,
                quantity,
                tp_price,
                self.current_timestamp,
            );

            println!(
                "[Система] Создан Take-Profit #{} на ${}",
                self.next_order_id, tp_price
            );

            self.tp_orders.insert(self.next_order_id, tp_order);
            self.next_order_id += 1;
        }

        self.positions.insert(position_id, position);
        position_id
    }

    /// Обработать тик цены
    pub fn on_price_update(&mut self, symbol: &str, price: f64) {
        self.current_timestamp += 1;

        // Проверяем take-profit ордера
        let mut triggered_orders = Vec::new();

        for (order_id, order) in &mut self.tp_orders {
            if order.symbol == symbol && order.should_trigger(price) {
                order.trigger(self.current_timestamp);
                order.fill();
                triggered_orders.push(*order_id);
                println!(
                    "[Система] Take-Profit #{} исполнен при ${:.2}!",
                    order_id, price
                );
            }
        }

        // Закрываем соответствующие позиции
        for order_id in triggered_orders {
            // В реальной системе здесь была бы связь order -> position
            println!("[Система] Прибыль зафиксирована");
        }
    }

    /// Показать статус портфеля
    pub fn print_portfolio(&self, current_prices: &HashMap<String, f64>) {
        println!("\n=== Портфель ===");

        for position in self.positions.values() {
            let current_price = current_prices
                .get(&position.symbol)
                .copied()
                .unwrap_or(position.entry_price);

            let pnl = match position.side {
                OrderSide::Buy => (current_price - position.entry_price) * position.quantity,
                OrderSide::Sell => (position.entry_price - current_price) * position.quantity,
            };

            let pnl_percent = (pnl / (position.entry_price * position.quantity)) * 100.0;

            println!(
                "#{} {} {} {} @ ${} | Текущая: ${} | PnL: ${:.2} ({:.2}%)",
                position.id,
                match position.side { OrderSide::Buy => "LONG", OrderSide::Sell => "SHORT" },
                position.quantity,
                position.symbol,
                position.entry_price,
                current_price,
                pnl,
                pnl_percent
            );

            if let Some(tp) = position.take_profit {
                println!("   Take-Profit: ${}", tp);
            }
        }
    }
}

fn main() {
    let mut system = TradingSystem::new();

    // Открываем позиции с take-profit
    system.open_position("BTC/USDT", OrderSide::Buy, 0.5, 40000.0, Some(10.0));
    system.open_position("ETH/USDT", OrderSide::Buy, 2.0, 2500.0, Some(15.0));

    println!("\n--- Симуляция рынка ---");

    // Симуляция ценового движения
    system.on_price_update("BTC/USDT", 41000.0);
    system.on_price_update("BTC/USDT", 42500.0);
    system.on_price_update("BTC/USDT", 44500.0); // Take-Profit сработает

    system.on_price_update("ETH/USDT", 2600.0);
    system.on_price_update("ETH/USDT", 2800.0);
    system.on_price_update("ETH/USDT", 2900.0); // Ещё не достигнут TP

    // Текущие цены
    let mut prices = HashMap::new();
    prices.insert("BTC/USDT".to_string(), 44500.0);
    prices.insert("ETH/USDT".to_string(), 2900.0);

    system.print_portfolio(&prices);
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Take-Profit (TP) | Автоматический ордер на фиксацию прибыли при достижении целевой цены |
| Trigger Price | Цена, при которой активируется take-profit |
| Risk/Reward Ratio | Соотношение потенциальной прибыли к риску (stop-loss) |
| Multi-level TP | Постепенная фиксация прибыли на нескольких уровнях |
| Trailing TP | Динамический take-profit, следующий за ценой |

## Практические упражнения

1. **Базовый Take-Profit**: Модифицируй структуру `TakeProfitOrder`, добавив поле `limit_price` — минимальную цену исполнения после срабатывания (защита от проскальзывания).

2. **Динамический расчёт**: Создай функцию, которая рассчитывает оптимальный take-profit на основе:
   - ATR (Average True Range) — волатильности актива
   - Исторических уровней сопротивления
   - Заданного соотношения риск/прибыль

3. **Время-зависимый TP**: Реализуй take-profit, который автоматически понижает целевую цену, если позиция открыта слишком долго (decay TP).

## Домашнее задание

1. **Частичное закрытие**: Расширь `MultiLevelTakeProfit`, добавив возможность изменять проценты на уровнях после частичного исполнения.

2. **Комбинированный ордер**: Создай структуру `BracketOrder`, которая содержит:
   - Entry order (вход в позицию)
   - Take-profit order
   - Stop-loss order

   Реализуй логику, где срабатывание TP автоматически отменяет SL и наоборот.

3. **Бэктестинг TP-стратегий**: Напиши программу, которая:
   - Загружает исторические цены (можно захардкодить массив)
   - Тестирует разные стратегии take-profit
   - Выводит статистику: процент выигрышных сделок, средняя прибыль, максимальная просадка

4. **Trailing с уровнями**: Реализуй trailing take-profit, который подтягивается не плавно, а уровнями. Например, после каждых +5% роста цены take-profit подтягивается на 4%.

## Навигация

[← Предыдущий день](../264-stop-loss-protecting-capital/ru.md) | [Следующий день →](../266-bracket-orders-entry-with-exits/ru.md)
