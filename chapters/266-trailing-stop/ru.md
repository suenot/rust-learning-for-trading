# День 266: Trailing Stop — скользящий стоп-лосс

## Аналогия из трейдинга

Представь, что ты альпинист, поднимающийся на гору. У тебя есть страховочная верёвка, которая автоматически подтягивается вверх по мере твоего подъёма, но никогда не опускается вниз. Если ты оступишься, верёвка удержит тебя на последней достигнутой высоте.

**Trailing Stop (скользящий стоп)** работает точно так же в трейдинге:
- Когда цена растёт в твою пользу, стоп-лосс автоматически поднимается вслед за ней
- Когда цена падает, стоп остаётся на месте
- Если цена достигает стопа — позиция автоматически закрывается

Это позволяет **защитить прибыль** и **ограничить убытки** одновременно, не требуя постоянного мониторинга рынка.

## Что такое Trailing Stop?

Trailing Stop — это динамический стоп-лосс, который следует за ценой на заданном расстоянии. Бывает два типа:

1. **Процентный (Percentage)** — стоп на X% ниже максимальной цены
2. **Фиксированный (Fixed)** — стоп на X пунктов ниже максимальной цены

### Пример работы

```
Покупка BTC по $40,000 с trailing stop 5%

Цена движется:
$40,000 → стоп = $38,000 (5% ниже)
$42,000 → стоп = $39,900 (стоп поднялся!)
$45,000 → стоп = $42,750 (стоп снова поднялся!)
$43,000 → стоп = $42,750 (цена упала, стоп остался)
$42,750 → СРАБОТАЛ! Продажа по $42,750
```

## Базовая структура Trailing Stop

```rust
#[derive(Debug, Clone)]
pub struct TrailingStop {
    /// Начальная цена входа
    entry_price: f64,
    /// Процент отступа от максимума (например, 0.05 = 5%)
    trail_percent: f64,
    /// Максимальная достигнутая цена
    highest_price: f64,
    /// Текущий уровень стопа
    stop_price: f64,
    /// Направление позиции (Long/Short)
    direction: PositionDirection,
    /// Сработал ли стоп
    triggered: bool,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum PositionDirection {
    Long,  // Покупка — защищаемся от падения
    Short, // Продажа — защищаемся от роста
}

impl TrailingStop {
    /// Создаёт новый trailing stop для длинной позиции
    pub fn new_long(entry_price: f64, trail_percent: f64) -> Self {
        let stop_price = entry_price * (1.0 - trail_percent);
        TrailingStop {
            entry_price,
            trail_percent,
            highest_price: entry_price,
            stop_price,
            direction: PositionDirection::Long,
            triggered: false,
        }
    }

    /// Создаёт новый trailing stop для короткой позиции
    pub fn new_short(entry_price: f64, trail_percent: f64) -> Self {
        let stop_price = entry_price * (1.0 + trail_percent);
        TrailingStop {
            entry_price,
            trail_percent,
            highest_price: entry_price, // Для шорта это lowest_price
            stop_price,
            direction: PositionDirection::Short,
            triggered: false,
        }
    }

    /// Обновляет стоп на основе новой цены
    pub fn update(&mut self, current_price: f64) -> bool {
        if self.triggered {
            return true; // Уже сработал
        }

        match self.direction {
            PositionDirection::Long => {
                // Для длинной позиции следим за максимумом
                if current_price > self.highest_price {
                    self.highest_price = current_price;
                    self.stop_price = current_price * (1.0 - self.trail_percent);
                }

                // Проверяем, сработал ли стоп
                if current_price <= self.stop_price {
                    self.triggered = true;
                }
            }
            PositionDirection::Short => {
                // Для короткой позиции следим за минимумом
                if current_price < self.highest_price {
                    self.highest_price = current_price; // lowest_price для шорта
                    self.stop_price = current_price * (1.0 + self.trail_percent);
                }

                // Проверяем, сработал ли стоп
                if current_price >= self.stop_price {
                    self.triggered = true;
                }
            }
        }

        self.triggered
    }

    /// Возвращает текущий уровень стопа
    pub fn get_stop_price(&self) -> f64 {
        self.stop_price
    }

    /// Проверяет, сработал ли стоп
    pub fn is_triggered(&self) -> bool {
        self.triggered
    }

    /// Возвращает максимальную достигнутую цену
    pub fn get_highest_price(&self) -> f64 {
        self.highest_price
    }
}

fn main() {
    // Пример: покупаем BTC по $40,000 с trailing stop 5%
    let mut stop = TrailingStop::new_long(40_000.0, 0.05);

    let prices = vec![
        40_000.0, 41_000.0, 42_000.0, 43_000.0, 45_000.0,
        44_000.0, 43_500.0, 43_000.0, 42_800.0, 42_750.0,
    ];

    println!("=== Симуляция Trailing Stop ===");
    println!("Вход: ${:.2}, Trail: 5%\n", 40_000.0);

    for price in prices {
        let was_triggered = stop.is_triggered();
        stop.update(price);

        println!(
            "Цена: ${:.2} | Стоп: ${:.2} | Максимум: ${:.2} | {}",
            price,
            stop.get_stop_price(),
            stop.get_highest_price(),
            if stop.is_triggered() && !was_triggered {
                "СРАБОТАЛ!"
            } else if stop.is_triggered() {
                "уже закрыт"
            } else {
                "активен"
            }
        );
    }
}
```

## Trailing Stop с фиксированным отступом

```rust
#[derive(Debug, Clone)]
pub struct FixedTrailingStop {
    /// Фиксированный отступ в единицах цены
    trail_amount: f64,
    /// Максимальная достигнутая цена
    highest_price: f64,
    /// Текущий уровень стопа
    stop_price: f64,
    /// Сработал ли стоп
    triggered: bool,
}

impl FixedTrailingStop {
    pub fn new(entry_price: f64, trail_amount: f64) -> Self {
        FixedTrailingStop {
            trail_amount,
            highest_price: entry_price,
            stop_price: entry_price - trail_amount,
            triggered: false,
        }
    }

    pub fn update(&mut self, current_price: f64) -> bool {
        if self.triggered {
            return true;
        }

        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.stop_price = current_price - self.trail_amount;
        }

        if current_price <= self.stop_price {
            self.triggered = true;
        }

        self.triggered
    }

    pub fn get_stop_price(&self) -> f64 {
        self.stop_price
    }
}

fn main() {
    // Trailing stop с отступом $100
    let mut stop = FixedTrailingStop::new(1000.0, 100.0);

    let prices = [1000.0, 1050.0, 1100.0, 1080.0, 1000.0, 990.0];

    println!("=== Fixed Trailing Stop ($100) ===\n");

    for price in prices {
        let triggered = stop.update(price);
        println!(
            "Цена: ${:.2} | Стоп: ${:.2} | {}",
            price,
            stop.get_stop_price(),
            if triggered { "TRIGGERED!" } else { "active" }
        );
    }
}
```

## Продвинутый Trailing Stop с историей

```rust
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
pub struct StopUpdate {
    timestamp: u64,
    price: f64,
    stop_level: f64,
    highest: f64,
}

#[derive(Debug)]
pub struct AdvancedTrailingStop {
    trail_percent: f64,
    highest_price: f64,
    stop_price: f64,
    triggered: bool,
    trigger_price: Option<f64>,
    history: Vec<StopUpdate>,
}

impl AdvancedTrailingStop {
    pub fn new(entry_price: f64, trail_percent: f64) -> Self {
        let stop_price = entry_price * (1.0 - trail_percent);
        let mut stop = AdvancedTrailingStop {
            trail_percent,
            highest_price: entry_price,
            stop_price,
            triggered: false,
            trigger_price: None,
            history: Vec::new(),
        };

        stop.record_update(entry_price);
        stop
    }

    fn get_timestamp() -> u64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_millis() as u64
    }

    fn record_update(&mut self, price: f64) {
        self.history.push(StopUpdate {
            timestamp: Self::get_timestamp(),
            price,
            stop_level: self.stop_price,
            highest: self.highest_price,
        });
    }

    pub fn update(&mut self, current_price: f64) -> bool {
        if self.triggered {
            return true;
        }

        let old_stop = self.stop_price;

        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.stop_price = current_price * (1.0 - self.trail_percent);
        }

        if current_price <= self.stop_price {
            self.triggered = true;
            self.trigger_price = Some(current_price);
        }

        // Записываем только если стоп изменился или сработал
        if self.stop_price != old_stop || self.triggered {
            self.record_update(current_price);
        }

        self.triggered
    }

    /// Возвращает прибыль/убыток в процентах
    pub fn get_pnl_percent(&self, entry_price: f64) -> Option<f64> {
        self.trigger_price.map(|exit| {
            ((exit - entry_price) / entry_price) * 100.0
        })
    }

    /// Возвращает историю обновлений стопа
    pub fn get_history(&self) -> &[StopUpdate] {
        &self.history
    }

    /// Возвращает количество раз, когда стоп поднимался
    pub fn get_adjustment_count(&self) -> usize {
        self.history.len().saturating_sub(1)
    }
}

fn main() {
    let entry = 50_000.0;
    let mut stop = AdvancedTrailingStop::new(entry, 0.03); // 3% trailing

    let prices = vec![
        50_000.0, 51_000.0, 52_000.0, 53_000.0, 54_000.0,
        53_500.0, 53_000.0, 52_500.0, 52_380.0,
    ];

    println!("=== Advanced Trailing Stop ===");
    println!("Вход: ${:.2}, Trail: 3%\n", entry);

    for price in prices {
        stop.update(price);
    }

    println!("История изменений стопа:");
    for (i, update) in stop.get_history().iter().enumerate() {
        println!(
            "  {}. Цена: ${:.2} | Стоп: ${:.2} | Макс: ${:.2}",
            i + 1,
            update.price,
            update.stop_level,
            update.highest
        );
    }

    println!("\nСтатистика:");
    println!("  Корректировок стопа: {}", stop.get_adjustment_count());

    if let Some(pnl) = stop.get_pnl_percent(entry) {
        println!("  P&L: {:.2}%", pnl);
    }
}
```

## Trailing Stop Manager для нескольких позиций

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct Position {
    pub symbol: String,
    pub entry_price: f64,
    pub quantity: f64,
    pub trailing_stop: TrailingStopState,
}

#[derive(Debug, Clone)]
pub struct TrailingStopState {
    pub trail_percent: f64,
    pub highest_price: f64,
    pub stop_price: f64,
    pub triggered: bool,
}

impl TrailingStopState {
    pub fn new(entry_price: f64, trail_percent: f64) -> Self {
        TrailingStopState {
            trail_percent,
            highest_price: entry_price,
            stop_price: entry_price * (1.0 - trail_percent),
            triggered: false,
        }
    }

    pub fn update(&mut self, price: f64) -> bool {
        if self.triggered {
            return true;
        }

        if price > self.highest_price {
            self.highest_price = price;
            self.stop_price = price * (1.0 - self.trail_percent);
        }

        if price <= self.stop_price {
            self.triggered = true;
        }

        self.triggered
    }
}

#[derive(Debug)]
pub struct TrailingStopManager {
    positions: HashMap<String, Position>,
    closed_positions: Vec<Position>,
}

impl TrailingStopManager {
    pub fn new() -> Self {
        TrailingStopManager {
            positions: HashMap::new(),
            closed_positions: Vec::new(),
        }
    }

    /// Открывает новую позицию с trailing stop
    pub fn open_position(
        &mut self,
        symbol: &str,
        entry_price: f64,
        quantity: f64,
        trail_percent: f64,
    ) {
        let position = Position {
            symbol: symbol.to_string(),
            entry_price,
            quantity,
            trailing_stop: TrailingStopState::new(entry_price, trail_percent),
        };

        self.positions.insert(symbol.to_string(), position);
        println!(
            "Открыта позиция: {} | Цена: ${:.2} | Количество: {} | Trail: {:.1}%",
            symbol, entry_price, quantity, trail_percent * 100.0
        );
    }

    /// Обновляет цены и проверяет стопы
    pub fn update_prices(&mut self, prices: &HashMap<String, f64>) {
        let mut to_close = Vec::new();

        for (symbol, position) in &mut self.positions {
            if let Some(&price) = prices.get(symbol) {
                if position.trailing_stop.update(price) {
                    to_close.push(symbol.clone());
                }
            }
        }

        // Закрываем сработавшие позиции
        for symbol in to_close {
            if let Some(position) = self.positions.remove(&symbol) {
                let pnl = (position.trailing_stop.stop_price - position.entry_price)
                    * position.quantity;
                let pnl_percent = ((position.trailing_stop.stop_price - position.entry_price)
                    / position.entry_price)
                    * 100.0;

                println!(
                    "ЗАКРЫТА позиция: {} | Выход: ${:.2} | P&L: ${:.2} ({:.2}%)",
                    symbol, position.trailing_stop.stop_price, pnl, pnl_percent
                );

                self.closed_positions.push(position);
            }
        }
    }

    /// Выводит статус всех позиций
    pub fn print_status(&self) {
        println!("\n=== Статус портфеля ===");

        if self.positions.is_empty() {
            println!("Нет открытых позиций");
        } else {
            for (symbol, pos) in &self.positions {
                println!(
                    "{}: Вход ${:.2} | Макс ${:.2} | Стоп ${:.2}",
                    symbol,
                    pos.entry_price,
                    pos.trailing_stop.highest_price,
                    pos.trailing_stop.stop_price
                );
            }
        }

        if !self.closed_positions.is_empty() {
            println!("\nЗакрытые позиции: {}", self.closed_positions.len());
        }
    }

    /// Возвращает общий P&L закрытых позиций
    pub fn get_total_pnl(&self) -> f64 {
        self.closed_positions
            .iter()
            .map(|p| {
                (p.trailing_stop.stop_price - p.entry_price) * p.quantity
            })
            .sum()
    }
}

fn main() {
    let mut manager = TrailingStopManager::new();

    // Открываем несколько позиций
    manager.open_position("BTC", 40_000.0, 0.5, 0.05);
    manager.open_position("ETH", 2_500.0, 4.0, 0.04);
    manager.open_position("SOL", 100.0, 50.0, 0.06);

    // Симулируем движение цен
    let price_updates = vec![
        HashMap::from([
            ("BTC".to_string(), 41_000.0),
            ("ETH".to_string(), 2_600.0),
            ("SOL".to_string(), 105.0),
        ]),
        HashMap::from([
            ("BTC".to_string(), 43_000.0),
            ("ETH".to_string(), 2_700.0),
            ("SOL".to_string(), 110.0),
        ]),
        HashMap::from([
            ("BTC".to_string(), 42_000.0),
            ("ETH".to_string(), 2_650.0),
            ("SOL".to_string(), 103.0), // Близко к стопу
        ]),
        HashMap::from([
            ("BTC".to_string(), 41_500.0),
            ("ETH".to_string(), 2_600.0),
            ("SOL".to_string(), 100.0), // Стоп сработает
        ]),
    ];

    println!("\n=== Симуляция торговли ===\n");

    for (i, prices) in price_updates.iter().enumerate() {
        println!("--- Тик {} ---", i + 1);
        manager.update_prices(prices);
        manager.print_status();
        println!();
    }

    println!("=== Итого P&L: ${:.2} ===", manager.get_total_pnl());
}
```

## Trailing Stop с активацией

Иногда trailing stop активируется только после достижения определённой прибыли:

```rust
#[derive(Debug, Clone)]
pub struct ActivatedTrailingStop {
    entry_price: f64,
    /// Процент прибыли для активации (например, 0.02 = 2%)
    activation_percent: f64,
    /// Процент отступа после активации
    trail_percent: f64,
    /// Активирован ли trailing stop
    activated: bool,
    highest_price: f64,
    stop_price: f64,
    triggered: bool,
}

impl ActivatedTrailingStop {
    pub fn new(entry_price: f64, activation_percent: f64, trail_percent: f64) -> Self {
        ActivatedTrailingStop {
            entry_price,
            activation_percent,
            trail_percent,
            activated: false,
            highest_price: entry_price,
            stop_price: 0.0, // Стоп не установлен до активации
            triggered: false,
        }
    }

    pub fn update(&mut self, current_price: f64) -> TrailStatus {
        if self.triggered {
            return TrailStatus::Triggered;
        }

        // Проверяем активацию
        if !self.activated {
            let profit_percent = (current_price - self.entry_price) / self.entry_price;

            if profit_percent >= self.activation_percent {
                self.activated = true;
                self.highest_price = current_price;
                self.stop_price = current_price * (1.0 - self.trail_percent);
                return TrailStatus::JustActivated;
            }

            return TrailStatus::WaitingActivation;
        }

        // Trailing stop активирован
        if current_price > self.highest_price {
            self.highest_price = current_price;
            self.stop_price = current_price * (1.0 - self.trail_percent);
        }

        if current_price <= self.stop_price {
            self.triggered = true;
            return TrailStatus::Triggered;
        }

        TrailStatus::Active
    }

    pub fn is_activated(&self) -> bool {
        self.activated
    }

    pub fn get_stop_price(&self) -> Option<f64> {
        if self.activated {
            Some(self.stop_price)
        } else {
            None
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum TrailStatus {
    WaitingActivation,
    JustActivated,
    Active,
    Triggered,
}

fn main() {
    // Активируется при +3% прибыли, затем trailing 2%
    let mut stop = ActivatedTrailingStop::new(1000.0, 0.03, 0.02);

    let prices = [
        1000.0, 1010.0, 1020.0, 1030.0, // Активация при 1030
        1050.0, 1060.0, 1055.0, 1040.0, 1038.8, // Сработает около 1038.8
    ];

    println!("=== Activated Trailing Stop ===");
    println!("Вход: $1000 | Активация: +3% | Trail: 2%\n");

    for price in prices {
        let status = stop.update(price);

        let stop_str = match stop.get_stop_price() {
            Some(s) => format!("${:.2}", s),
            None => "---".to_string(),
        };

        println!(
            "Цена: ${:.2} | Стоп: {} | Статус: {:?}",
            price, stop_str, status
        );
    }
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Trailing Stop | Динамический стоп-лосс, следующий за ценой |
| Процентный стоп | Стоп на X% ниже максимальной цены |
| Фиксированный стоп | Стоп на X пунктов ниже максимума |
| Активируемый стоп | Включается после достижения целевой прибыли |
| Long/Short | Разная логика для длинных и коротких позиций |
| Stop Manager | Управление стопами для нескольких позиций |

## Упражнения

1. **Базовый trailing stop**: Реализуй trailing stop, который выводит сообщение каждый раз, когда стоп поднимается на новый уровень.

2. **Двунаправленный стоп**: Добавь поддержку Short позиций в `TrailingStopManager`, где стоп опускается при падении цены.

3. **Стоп с минимальным шагом**: Модифицируй trailing stop так, чтобы он поднимался только если новый уровень выше предыдущего минимум на 0.5%.

4. **Статистика по стопам**: Добавь в `TrailingStopManager` метод, который возвращает статистику: средний P&L, процент прибыльных сделок, максимальную просадку.

## Домашнее задание

1. **ATR-based Trailing Stop**: Реализуй trailing stop, который использует Average True Range (ATR) для определения расстояния до стопа. ATR — это средний диапазон движения цены за N периодов.

2. **Chandelier Exit**: Создай trailing stop на основе Chandelier Exit — стоп устанавливается на расстоянии N * ATR от максимума.

3. **Параболический SAR**: Реализуй упрощённую версию Parabolic SAR — trailing stop, который ускоряется по мере роста цены.

4. **Backtesting**: Напиши функцию, которая тестирует разные параметры trailing stop (1%, 2%, 3%, ..., 10%) на исторических данных и выводит сравнительную таблицу результатов.

## Навигация

[← Предыдущий день](../265-stop-loss-take-profit/ru.md) | [Следующий день →](../267-position-sizing/ru.md)
