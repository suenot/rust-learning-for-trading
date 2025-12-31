# День 90: Проект — Order Book (Книга заявок)

## Введение

Это **проектная глава**, которая объединяет знания за месяц в практический мини-проект. Мы создадим полноценную реализацию **Order Book** — одной из ключевых структур данных в алготрейдинге.

## Что такое Order Book?

**Order Book (книга заявок)** — это структура данных, которая хранит все активные ордера на покупку и продажу для определённого торгового инструмента.

```
         ASKS (заявки на продажу)
         ┌─────────────────────────┐
         │ $42,150  │  2.5 BTC    │ ← Лучший ask (lowest)
         │ $42,200  │  1.8 BTC    │
         │ $42,300  │  3.2 BTC    │
         │ $42,500  │  5.0 BTC    │
         └─────────────────────────┘
                   SPREAD
         ┌─────────────────────────┐
         │ $42,100  │  1.5 BTC    │ ← Лучший bid (highest)
         │ $42,050  │  2.0 BTC    │
         │ $42,000  │  4.2 BTC    │
         │ $41,900  │  3.8 BTC    │
         └─────────────────────────┘
         BIDS (заявки на покупку)
```

### Ключевые концепции

- **Bid** — заявка на покупку (покупатель готов купить по этой цене)
- **Ask** — заявка на продажу (продавец готов продать по этой цене)
- **Spread** — разница между лучшим ask и лучшим bid
- **Depth** — количество уровней цен в стакане
- **Liquidity** — общий объём на каждой стороне стакана

## Часть 1: Базовая структура Order

Начнём с определения структуры для отдельного ордера:

```rust
use std::cmp::Ordering;

/// Сторона ордера
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side {
    Bid,  // Покупка
    Ask,  // Продажа
}

/// Отдельный ордер в книге заявок
#[derive(Debug, Clone)]
pub struct Order {
    pub id: u64,
    pub price: f64,
    pub quantity: f64,
    pub side: Side,
    pub timestamp: u64,
}

impl Order {
    pub fn new(id: u64, price: f64, quantity: f64, side: Side, timestamp: u64) -> Self {
        Order {
            id,
            price,
            quantity,
            side,
            timestamp,
        }
    }

    /// Общая стоимость ордера
    pub fn total_value(&self) -> f64 {
        self.price * self.quantity
    }
}

fn main() {
    let bid = Order::new(1, 42100.0, 1.5, Side::Bid, 1000);
    let ask = Order::new(2, 42150.0, 2.0, Side::Ask, 1001);

    println!("Bid: {:?}", bid);
    println!("Ask: {:?}", ask);
    println!("Bid value: ${:.2}", bid.total_value());
}
```

## Часть 2: Уровень цены (Price Level)

Группируем ордера по уровням цен:

```rust
use std::collections::VecDeque;

/// Уровень цены — все ордера по одной цене
#[derive(Debug, Clone)]
pub struct PriceLevel {
    pub price: f64,
    pub orders: VecDeque<Order>,
}

impl PriceLevel {
    pub fn new(price: f64) -> Self {
        PriceLevel {
            price,
            orders: VecDeque::new(),
        }
    }

    /// Добавить ордер на уровень
    pub fn add_order(&mut self, order: Order) {
        self.orders.push_back(order);
    }

    /// Общий объём на уровне
    pub fn total_quantity(&self) -> f64 {
        self.orders.iter().map(|o| o.quantity).sum()
    }

    /// Количество ордеров на уровне
    pub fn order_count(&self) -> usize {
        self.orders.len()
    }

    /// Убрать ордер по ID
    pub fn remove_order(&mut self, order_id: u64) -> Option<Order> {
        if let Some(pos) = self.orders.iter().position(|o| o.id == order_id) {
            self.orders.remove(pos)
        } else {
            None
        }
    }
}

fn main() {
    let mut level = PriceLevel::new(42100.0);

    level.add_order(Order::new(1, 42100.0, 1.5, Side::Bid, 1000));
    level.add_order(Order::new(2, 42100.0, 2.0, Side::Bid, 1001));
    level.add_order(Order::new(3, 42100.0, 0.8, Side::Bid, 1002));

    println!("Price level: ${}", level.price);
    println!("Total quantity: {} BTC", level.total_quantity());
    println!("Order count: {}", level.order_count());
}
```

## Часть 3: Полная реализация Order Book

```rust
use std::collections::{BTreeMap, VecDeque};

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side {
    Bid,
    Ask,
}

#[derive(Debug, Clone)]
pub struct Order {
    pub id: u64,
    pub price: f64,
    pub quantity: f64,
    pub side: Side,
    pub timestamp: u64,
}

impl Order {
    pub fn new(id: u64, price: f64, quantity: f64, side: Side, timestamp: u64) -> Self {
        Order { id, price, quantity, side, timestamp }
    }
}

#[derive(Debug, Clone)]
pub struct PriceLevel {
    pub price: f64,
    pub orders: VecDeque<Order>,
}

impl PriceLevel {
    pub fn new(price: f64) -> Self {
        PriceLevel { price, orders: VecDeque::new() }
    }

    pub fn add_order(&mut self, order: Order) {
        self.orders.push_back(order);
    }

    pub fn total_quantity(&self) -> f64 {
        self.orders.iter().map(|o| o.quantity).sum()
    }

    pub fn is_empty(&self) -> bool {
        self.orders.is_empty()
    }
}

/// Полная книга заявок
#[derive(Debug)]
pub struct OrderBook {
    pub symbol: String,
    // Bids: ключ = price * -1 для обратной сортировки
    bids: BTreeMap<i64, PriceLevel>,
    // Asks: ключ = price (прямая сортировка)
    asks: BTreeMap<i64, PriceLevel>,
    next_order_id: u64,
    price_precision: u32,
}

impl OrderBook {
    pub fn new(symbol: &str, price_precision: u32) -> Self {
        OrderBook {
            symbol: symbol.to_string(),
            bids: BTreeMap::new(),
            asks: BTreeMap::new(),
            next_order_id: 1,
            price_precision,
        }
    }

    /// Конвертация цены в целочисленный ключ
    fn price_to_key(&self, price: f64) -> i64 {
        (price * 10_f64.powi(self.price_precision as i32)) as i64
    }

    /// Добавить ордер в книгу
    pub fn add_order(&mut self, price: f64, quantity: f64, side: Side) -> u64 {
        let order_id = self.next_order_id;
        self.next_order_id += 1;

        let timestamp = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_millis() as u64;

        let order = Order::new(order_id, price, quantity, side, timestamp);

        match side {
            Side::Bid => {
                let key = -self.price_to_key(price); // Отрицательный для обратной сортировки
                self.bids
                    .entry(key)
                    .or_insert_with(|| PriceLevel::new(price))
                    .add_order(order);
            }
            Side::Ask => {
                let key = self.price_to_key(price);
                self.asks
                    .entry(key)
                    .or_insert_with(|| PriceLevel::new(price))
                    .add_order(order);
            }
        }

        order_id
    }

    /// Лучший bid (самая высокая цена покупки)
    pub fn best_bid(&self) -> Option<(f64, f64)> {
        self.bids.values().next().map(|level| {
            (level.price, level.total_quantity())
        })
    }

    /// Лучший ask (самая низкая цена продажи)
    pub fn best_ask(&self) -> Option<(f64, f64)> {
        self.asks.values().next().map(|level| {
            (level.price, level.total_quantity())
        })
    }

    /// Спред (разница между лучшим ask и bid)
    pub fn spread(&self) -> Option<f64> {
        match (self.best_bid(), self.best_ask()) {
            (Some((bid, _)), Some((ask, _))) => Some(ask - bid),
            _ => None,
        }
    }

    /// Процентный спред
    pub fn spread_percent(&self) -> Option<f64> {
        match (self.best_bid(), self.best_ask()) {
            (Some((bid, _)), Some((ask, _))) => {
                let mid_price = (bid + ask) / 2.0;
                Some((ask - bid) / mid_price * 100.0)
            }
            _ => None,
        }
    }

    /// Средняя цена (mid price)
    pub fn mid_price(&self) -> Option<f64> {
        match (self.best_bid(), self.best_ask()) {
            (Some((bid, _)), Some((ask, _))) => Some((bid + ask) / 2.0),
            _ => None,
        }
    }

    /// Получить N лучших уровней bids
    pub fn top_bids(&self, n: usize) -> Vec<(f64, f64)> {
        self.bids
            .values()
            .take(n)
            .map(|level| (level.price, level.total_quantity()))
            .collect()
    }

    /// Получить N лучших уровней asks
    pub fn top_asks(&self, n: usize) -> Vec<(f64, f64)> {
        self.asks
            .values()
            .take(n)
            .map(|level| (level.price, level.total_quantity()))
            .collect()
    }

    /// Общий объём на стороне bid
    pub fn total_bid_volume(&self) -> f64 {
        self.bids.values().map(|l| l.total_quantity()).sum()
    }

    /// Общий объём на стороне ask
    pub fn total_ask_volume(&self) -> f64 {
        self.asks.values().map(|l| l.total_quantity()).sum()
    }

    /// Дисбаланс ордеров (отношение bid к ask)
    pub fn order_imbalance(&self) -> f64 {
        let bid_vol = self.total_bid_volume();
        let ask_vol = self.total_ask_volume();

        if ask_vol == 0.0 {
            return if bid_vol > 0.0 { 1.0 } else { 0.0 };
        }

        bid_vol / (bid_vol + ask_vol)
    }

    /// Красивый вывод книги заявок
    pub fn display(&self, depth: usize) {
        println!("\n╔══════════════════════════════════════════╗");
        println!("║         ORDER BOOK: {}               ║", self.symbol);
        println!("╠══════════════════════════════════════════╣");

        // Asks (в обратном порядке для отображения)
        let asks: Vec<_> = self.top_asks(depth);
        println!("║              ASKS (Sell)                 ║");
        println!("║──────────────────────────────────────────║");
        for (price, qty) in asks.iter().rev() {
            println!("║   {:>10.2}  │  {:>10.4}               ║", price, qty);
        }

        // Спред
        println!("║══════════════════════════════════════════║");
        if let Some(spread) = self.spread() {
            println!("║   SPREAD: ${:.2} ({:.4}%)                 ║",
                spread, self.spread_percent().unwrap_or(0.0));
        }
        println!("║══════════════════════════════════════════║");

        // Bids
        println!("║              BIDS (Buy)                  ║");
        println!("║──────────────────────────────────────────║");
        for (price, qty) in self.top_bids(depth) {
            println!("║   {:>10.2}  │  {:>10.4}               ║", price, qty);
        }

        println!("╚══════════════════════════════════════════╝");

        // Статистика
        println!("\n📊 Statistics:");
        println!("   Mid Price: ${:.2}", self.mid_price().unwrap_or(0.0));
        println!("   Bid Volume: {:.4}", self.total_bid_volume());
        println!("   Ask Volume: {:.4}", self.total_ask_volume());
        println!("   Imbalance: {:.2}% (bids)", self.order_imbalance() * 100.0);
    }
}

fn main() {
    let mut book = OrderBook::new("BTC/USDT", 2);

    // Добавляем ордера на покупку (bids)
    book.add_order(42100.0, 1.5, Side::Bid);
    book.add_order(42050.0, 2.0, Side::Bid);
    book.add_order(42000.0, 3.5, Side::Bid);
    book.add_order(41950.0, 1.8, Side::Bid);
    book.add_order(41900.0, 4.2, Side::Bid);

    // Добавляем ордера на продажу (asks)
    book.add_order(42150.0, 1.2, Side::Ask);
    book.add_order(42200.0, 2.5, Side::Ask);
    book.add_order(42250.0, 1.8, Side::Ask);
    book.add_order(42300.0, 3.0, Side::Ask);
    book.add_order(42400.0, 2.2, Side::Ask);

    // Отображаем книгу заявок
    book.display(5);
}
```

## Часть 4: Обработка рыночных ордеров

Добавим функционал для исполнения рыночных ордеров:

```rust
/// Результат исполнения ордера
#[derive(Debug)]
pub struct ExecutionResult {
    pub filled_quantity: f64,
    pub average_price: f64,
    pub fills: Vec<Fill>,
}

#[derive(Debug)]
pub struct Fill {
    pub price: f64,
    pub quantity: f64,
    pub order_id: u64,
}

impl OrderBook {
    /// Исполнить рыночный ордер на покупку
    pub fn execute_market_buy(&mut self, quantity: f64) -> ExecutionResult {
        let mut remaining = quantity;
        let mut fills = Vec::new();
        let mut total_cost = 0.0;
        let mut keys_to_remove = Vec::new();

        for (&key, level) in self.asks.iter_mut() {
            if remaining <= 0.0 {
                break;
            }

            while let Some(mut order) = level.orders.pop_front() {
                if remaining <= 0.0 {
                    level.orders.push_front(order);
                    break;
                }

                let fill_qty = remaining.min(order.quantity);
                fills.push(Fill {
                    price: order.price,
                    quantity: fill_qty,
                    order_id: order.id,
                });

                total_cost += fill_qty * order.price;
                remaining -= fill_qty;
                order.quantity -= fill_qty;

                if order.quantity > 0.0 {
                    level.orders.push_front(order);
                    break;
                }
            }

            if level.is_empty() {
                keys_to_remove.push(key);
            }
        }

        // Удаляем пустые уровни
        for key in keys_to_remove {
            self.asks.remove(&key);
        }

        let filled_quantity = quantity - remaining;
        let average_price = if filled_quantity > 0.0 {
            total_cost / filled_quantity
        } else {
            0.0
        };

        ExecutionResult {
            filled_quantity,
            average_price,
            fills,
        }
    }

    /// Исполнить рыночный ордер на продажу
    pub fn execute_market_sell(&mut self, quantity: f64) -> ExecutionResult {
        let mut remaining = quantity;
        let mut fills = Vec::new();
        let mut total_revenue = 0.0;
        let mut keys_to_remove = Vec::new();

        for (&key, level) in self.bids.iter_mut() {
            if remaining <= 0.0 {
                break;
            }

            while let Some(mut order) = level.orders.pop_front() {
                if remaining <= 0.0 {
                    level.orders.push_front(order);
                    break;
                }

                let fill_qty = remaining.min(order.quantity);
                fills.push(Fill {
                    price: order.price,
                    quantity: fill_qty,
                    order_id: order.id,
                });

                total_revenue += fill_qty * order.price;
                remaining -= fill_qty;
                order.quantity -= fill_qty;

                if order.quantity > 0.0 {
                    level.orders.push_front(order);
                    break;
                }
            }

            if level.is_empty() {
                keys_to_remove.push(key);
            }
        }

        for key in keys_to_remove {
            self.bids.remove(&key);
        }

        let filled_quantity = quantity - remaining;
        let average_price = if filled_quantity > 0.0 {
            total_revenue / filled_quantity
        } else {
            0.0
        };

        ExecutionResult {
            filled_quantity,
            average_price,
            fills,
        }
    }
}
```

## Часть 5: Анализ ликвидности

```rust
impl OrderBook {
    /// Рассчитать цену для покупки определённого объёма
    pub fn price_for_buy(&self, quantity: f64) -> Option<f64> {
        let mut remaining = quantity;
        let mut total_cost = 0.0;

        for level in self.asks.values() {
            if remaining <= 0.0 {
                break;
            }

            let available = level.total_quantity();
            let fill_qty = remaining.min(available);
            total_cost += fill_qty * level.price;
            remaining -= fill_qty;
        }

        if remaining > 0.0 {
            None // Недостаточно ликвидности
        } else {
            Some(total_cost / quantity)
        }
    }

    /// Рассчитать цену для продажи определённого объёма
    pub fn price_for_sell(&self, quantity: f64) -> Option<f64> {
        let mut remaining = quantity;
        let mut total_revenue = 0.0;

        for level in self.bids.values() {
            if remaining <= 0.0 {
                break;
            }

            let available = level.total_quantity();
            let fill_qty = remaining.min(available);
            total_revenue += fill_qty * level.price;
            remaining -= fill_qty;
        }

        if remaining > 0.0 {
            None
        } else {
            Some(total_revenue / quantity)
        }
    }

    /// Рассчитать slippage (проскальзывание) для покупки
    pub fn buy_slippage(&self, quantity: f64) -> Option<f64> {
        let exec_price = self.price_for_buy(quantity)?;
        let best_ask = self.best_ask()?.0;
        Some((exec_price - best_ask) / best_ask * 100.0)
    }

    /// Рассчитать slippage для продажи
    pub fn sell_slippage(&self, quantity: f64) -> Option<f64> {
        let exec_price = self.price_for_sell(quantity)?;
        let best_bid = self.best_bid()?.0;
        Some((best_bid - exec_price) / best_bid * 100.0)
    }

    /// Глубина рынка до определённой цены
    pub fn depth_to_price(&self, side: Side, target_price: f64) -> f64 {
        match side {
            Side::Bid => {
                self.bids
                    .values()
                    .filter(|l| l.price >= target_price)
                    .map(|l| l.total_quantity())
                    .sum()
            }
            Side::Ask => {
                self.asks
                    .values()
                    .filter(|l| l.price <= target_price)
                    .map(|l| l.total_quantity())
                    .sum()
            }
        }
    }
}

fn main() {
    let mut book = OrderBook::new("BTC/USDT", 2);

    // Наполняем книгу
    for i in 0..10 {
        book.add_order(42000.0 - (i as f64 * 50.0), 1.0 + (i as f64 * 0.5), Side::Bid);
        book.add_order(42100.0 + (i as f64 * 50.0), 1.0 + (i as f64 * 0.5), Side::Ask);
    }

    book.display(5);

    // Анализ ликвидности
    println!("\n📈 Liquidity Analysis:");

    for qty in [1.0, 5.0, 10.0, 20.0] {
        if let Some(price) = book.price_for_buy(qty) {
            let slippage = book.buy_slippage(qty).unwrap_or(0.0);
            println!("   Buy {} BTC @ ${:.2} (slippage: {:.4}%)", qty, price, slippage);
        } else {
            println!("   Buy {} BTC: Insufficient liquidity!", qty);
        }
    }
}
```

## Упражнения

### Упражнение 1: Отмена ордеров

Реализуйте метод `cancel_order` который удаляет ордер по его ID:

```rust
impl OrderBook {
    pub fn cancel_order(&mut self, order_id: u64) -> Option<Order> {
        // Ваш код здесь
        // Подсказка: нужно искать в обеих сторонах (bids и asks)
        todo!()
    }
}
```

### Упражнение 2: Модификация ордера

Реализуйте метод для изменения количества в ордере:

```rust
impl OrderBook {
    pub fn modify_order(&mut self, order_id: u64, new_quantity: f64) -> bool {
        // Ваш код здесь
        // Подсказка: найти ордер и изменить его quantity
        todo!()
    }
}
```

### Упражнение 3: Агрегированный стакан

Создайте метод который возвращает агрегированные данные стакана:

```rust
#[derive(Debug)]
pub struct AggregatedLevel {
    pub price: f64,
    pub quantity: f64,
    pub order_count: usize,
    pub cumulative_quantity: f64,  // Накопленный объём
}

impl OrderBook {
    pub fn aggregated_book(&self, depth: usize) -> (Vec<AggregatedLevel>, Vec<AggregatedLevel>) {
        // Возвращает (bids, asks) с накопленным объёмом
        todo!()
    }
}
```

### Упражнение 4: VWAP калькулятор

Рассчитайте VWAP (Volume Weighted Average Price) для заданного объёма:

```rust
impl OrderBook {
    pub fn vwap_buy(&self, quantity: f64) -> Option<f64> {
        // Взвешенная по объёму средняя цена для покупки quantity
        todo!()
    }

    pub fn vwap_sell(&self, quantity: f64) -> Option<f64> {
        // Взвешенная по объёму средняя цена для продажи quantity
        todo!()
    }
}
```

## Домашнее задание

### 1. Добавьте поддержку лимитных ордеров с matching

Реализуйте логику, когда новый ордер автоматически матчится с противоположной стороной, если цены пересекаются:

```rust
pub fn add_limit_order(&mut self, price: f64, quantity: f64, side: Side)
    -> (u64, Vec<Fill>) {
    // Если buy order с ценой >= best ask — частично или полностью исполнить
    // Если sell order с ценой <= best bid — частично или полностью исполнить
    // Остаток добавить в книгу
}
```

### 2. Order Book снэпшоты и дельта-обновления

Реализуйте систему для создания снэпшотов и применения дельта-обновлений:

```rust
#[derive(Debug, Clone)]
pub struct OrderBookSnapshot {
    pub symbol: String,
    pub bids: Vec<(f64, f64)>,
    pub asks: Vec<(f64, f64)>,
    pub timestamp: u64,
}

#[derive(Debug)]
pub enum DeltaUpdate {
    Add { side: Side, price: f64, quantity: f64 },
    Remove { side: Side, price: f64 },
    Update { side: Side, price: f64, quantity: f64 },
}

impl OrderBook {
    pub fn snapshot(&self, depth: usize) -> OrderBookSnapshot { todo!() }
    pub fn apply_delta(&mut self, delta: DeltaUpdate) { todo!() }
}
```

### 3. Визуализация глубины рынка

Создайте ASCII-визуализацию глубины рынка:

```
DEPTH CHART: BTC/USDT
    ASK ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░  42400: 15.5 BTC
        ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░  42300: 10.2 BTC
        ▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░  42200:  6.5 BTC
    ────────────────────────────  SPREAD: $100
    BID ▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░  42100:  8.0 BTC
        ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░  42000: 12.3 BTC
        ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░  41900: 18.7 BTC
```

### 4. Анализ микроструктуры

Реализуйте расчёт показателей микроструктуры рынка:

```rust
pub struct MicrostructureMetrics {
    pub bid_ask_spread: f64,
    pub spread_bps: f64,        // Spread в базисных пунктах
    pub order_imbalance: f64,
    pub depth_imbalance: f64,   // Дисбаланс на первых N уровнях
    pub weighted_mid_price: f64, // Взвешенный mid price
}

impl OrderBook {
    pub fn microstructure_metrics(&self, depth: usize) -> MicrostructureMetrics {
        todo!()
    }
}
```

## Что мы изучили

| Концепция | Применение |
|-----------|------------|
| `struct` | Определение Order, PriceLevel, OrderBook |
| `enum` | Side (Bid/Ask), результаты операций |
| `BTreeMap` | Отсортированное хранение уровней цен |
| `VecDeque` | FIFO-очередь ордеров |
| `impl` | Методы для структур |
| Итераторы | Обработка коллекций |
| `Option` | Безопасная обработка отсутствующих данных |

## Ключевые концепции трейдинга

- **Order Book** — центральная структура данных биржи
- **Spread** — индикатор ликвидности рынка
- **Slippage** — реальная стоимость исполнения крупных ордеров
- **Order Imbalance** — предиктор краткосрочного движения цены
- **Depth** — устойчивость цены к крупным ордерам

## Навигация

[← Предыдущий день](../089-btreemap-sorted-prices/ru.md) | [Следующий день →](../091-vectors-dynamic-orders/ru.md)
