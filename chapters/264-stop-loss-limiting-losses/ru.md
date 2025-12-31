# День 264: Stop-Loss: ограничение убытков

## Аналогия из трейдинга

Представь, что ты купил акции компании по $100. Ты веришь в рост, но понимаешь: рынок непредсказуем. Что если цена упадёт до $50? $30? $10? Без защитного механизма ты можешь потерять все вложения.

**Stop-Loss** — это как страховой полис для твоей позиции. Ты устанавливаешь границу: "Если цена упадёт до $90, автоматически продай". Это ограничивает максимальный убыток 10%.

В реальном трейдинге stop-loss критически важен для:
- Защиты капитала от катастрофических потерь
- Удаления эмоций из процесса принятия решений
- Автоматического управления рисками
- Поддержания торговой дисциплины

## Что такое Stop-Loss?

Stop-Loss — это ордер на продажу актива при достижении определённого уровня цены. Основные типы:

1. **Фиксированный Stop-Loss** — продажа при достижении конкретной цены
2. **Процентный Stop-Loss** — продажа при падении на заданный процент
3. **Trailing Stop-Loss** — динамический стоп, следующий за ценой
4. **Стоп на основе волатильности** — адаптируется к рыночным условиям

## Базовая структура Stop-Loss

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum StopLossType {
    Fixed(f64),           // Фиксированная цена
    Percentage(f64),      // Процент от цены входа
    Trailing(f64),        // Следует за ценой с отступом
    VolatilityBased(f64), // На основе ATR или другого индикатора
}

#[derive(Debug, Clone)]
pub struct StopLoss {
    stop_type: StopLossType,
    trigger_price: f64,
    is_active: bool,
}

impl StopLoss {
    pub fn new_fixed(stop_price: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Fixed(stop_price),
            trigger_price: stop_price,
            is_active: true,
        }
    }

    pub fn new_percentage(entry_price: f64, percentage: f64) -> Self {
        let trigger = entry_price * (1.0 - percentage / 100.0);
        StopLoss {
            stop_type: StopLossType::Percentage(percentage),
            trigger_price: trigger,
            is_active: true,
        }
    }

    pub fn new_trailing(current_price: f64, trail_amount: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Trailing(trail_amount),
            trigger_price: current_price - trail_amount,
            is_active: true,
        }
    }

    pub fn should_trigger(&self, current_price: f64) -> bool {
        self.is_active && current_price <= self.trigger_price
    }

    pub fn update_trailing(&mut self, current_price: f64) {
        if let StopLossType::Trailing(trail_amount) = self.stop_type {
            let new_trigger = current_price - trail_amount;
            // Trailing stop только поднимается, никогда не опускается
            if new_trigger > self.trigger_price {
                self.trigger_price = new_trigger;
            }
        }
    }
}

fn main() {
    // Пример 1: Фиксированный стоп-лосс
    let fixed_stop = StopLoss::new_fixed(95.0);
    println!("Фиксированный стоп на: ${:.2}", fixed_stop.trigger_price);

    // Пример 2: Процентный стоп-лосс (5% от цены входа $100)
    let percentage_stop = StopLoss::new_percentage(100.0, 5.0);
    println!("5% стоп на: ${:.2}", percentage_stop.trigger_price);

    // Пример 3: Trailing стоп ($3 от текущей цены)
    let mut trailing_stop = StopLoss::new_trailing(100.0, 3.0);
    println!("Trailing стоп на: ${:.2}", trailing_stop.trigger_price);

    // Цена растёт — стоп следует за ней
    trailing_stop.update_trailing(105.0);
    println!("После роста до $105, стоп на: ${:.2}", trailing_stop.trigger_price);
}
```

## Позиция с защитой Stop-Loss

```rust
#[derive(Debug, Clone)]
pub struct Position {
    symbol: String,
    quantity: f64,
    entry_price: f64,
    current_price: f64,
    stop_loss: Option<StopLoss>,
    is_open: bool,
}

#[derive(Debug, Clone, PartialEq)]
pub enum StopLossType {
    Fixed(f64),
    Percentage(f64),
    Trailing(f64),
}

#[derive(Debug, Clone)]
pub struct StopLoss {
    stop_type: StopLossType,
    trigger_price: f64,
    is_active: bool,
}

impl StopLoss {
    pub fn new_fixed(stop_price: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Fixed(stop_price),
            trigger_price: stop_price,
            is_active: true,
        }
    }

    pub fn new_percentage(entry_price: f64, percentage: f64) -> Self {
        let trigger = entry_price * (1.0 - percentage / 100.0);
        StopLoss {
            stop_type: StopLossType::Percentage(percentage),
            trigger_price: trigger,
            is_active: true,
        }
    }

    pub fn new_trailing(current_price: f64, trail_amount: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Trailing(trail_amount),
            trigger_price: current_price - trail_amount,
            is_active: true,
        }
    }

    pub fn should_trigger(&self, current_price: f64) -> bool {
        self.is_active && current_price <= self.trigger_price
    }

    pub fn update_trailing(&mut self, current_price: f64) {
        if let StopLossType::Trailing(trail_amount) = self.stop_type {
            let new_trigger = current_price - trail_amount;
            if new_trigger > self.trigger_price {
                self.trigger_price = new_trigger;
            }
        }
    }
}

impl Position {
    pub fn new(symbol: &str, quantity: f64, entry_price: f64) -> Self {
        Position {
            symbol: symbol.to_string(),
            quantity,
            entry_price,
            current_price: entry_price,
            stop_loss: None,
            is_open: true,
        }
    }

    pub fn set_stop_loss(&mut self, stop_loss: StopLoss) {
        self.stop_loss = Some(stop_loss);
    }

    pub fn update_price(&mut self, new_price: f64) -> Option<String> {
        self.current_price = new_price;

        // Обновляем trailing stop если есть
        if let Some(ref mut stop) = self.stop_loss {
            stop.update_trailing(new_price);

            // Проверяем срабатывание стоп-лосса
            if stop.should_trigger(new_price) {
                self.is_open = false;
                let loss = (self.entry_price - new_price) * self.quantity;
                return Some(format!(
                    "STOP-LOSS сработал для {}: продано {} по ${:.2}, убыток: ${:.2}",
                    self.symbol, self.quantity, new_price, loss
                ));
            }
        }
        None
    }

    pub fn unrealized_pnl(&self) -> f64 {
        (self.current_price - self.entry_price) * self.quantity
    }

    pub fn pnl_percentage(&self) -> f64 {
        ((self.current_price - self.entry_price) / self.entry_price) * 100.0
    }
}

fn main() {
    // Открываем позицию
    let mut position = Position::new("AAPL", 100.0, 150.0);

    // Устанавливаем trailing stop-loss на $5
    position.set_stop_loss(StopLoss::new_trailing(150.0, 5.0));

    println!("Открыта позиция: {} акций по ${}", position.quantity, position.entry_price);
    println!("Начальный стоп: ${:.2}", position.stop_loss.as_ref().unwrap().trigger_price);

    // Симулируем движение цены
    let prices = vec![152.0, 155.0, 158.0, 156.0, 153.0, 151.0, 148.0];

    for price in prices {
        if let Some(message) = position.update_price(price) {
            println!("{}", message);
            break;
        } else {
            println!(
                "Цена: ${:.2}, Стоп: ${:.2}, P&L: ${:.2} ({:.2}%)",
                price,
                position.stop_loss.as_ref().unwrap().trigger_price,
                position.unrealized_pnl(),
                position.pnl_percentage()
            );
        }
    }
}
```

## Визуализация Trailing Stop-Loss

```
Цена                    Trailing Stop
  |                         |
158 ─────●                  |
  |       \                 |
156 ───────●                |──── 153 (стоп поднялся)
  |         \               |
154 ─────────●              |
  |           \             |
152 ───────────●            |──── 147 (начальный)
  |             \           |
150 ─────────────● (вход)   |──── 145
  |               \         |
148 ────────────────X       | СТОП СРАБОТАЛ!
  |                         |

Trailing stop следует за ценой вверх, но никогда не опускается!
```

## Торговый движок с риск-менеджментом

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq)]
pub enum StopLossType {
    Fixed(f64),
    Percentage(f64),
    Trailing(f64),
}

#[derive(Debug, Clone)]
pub struct StopLoss {
    stop_type: StopLossType,
    trigger_price: f64,
    is_active: bool,
}

impl StopLoss {
    pub fn new_fixed(stop_price: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Fixed(stop_price),
            trigger_price: stop_price,
            is_active: true,
        }
    }

    pub fn new_percentage(entry_price: f64, percentage: f64) -> Self {
        let trigger = entry_price * (1.0 - percentage / 100.0);
        StopLoss {
            stop_type: StopLossType::Percentage(percentage),
            trigger_price: trigger,
            is_active: true,
        }
    }

    pub fn new_trailing(current_price: f64, trail_amount: f64) -> Self {
        StopLoss {
            stop_type: StopLossType::Trailing(trail_amount),
            trigger_price: current_price - trail_amount,
            is_active: true,
        }
    }

    pub fn should_trigger(&self, current_price: f64) -> bool {
        self.is_active && current_price <= self.trigger_price
    }

    pub fn update_trailing(&mut self, current_price: f64) {
        if let StopLossType::Trailing(trail_amount) = self.stop_type {
            let new_trigger = current_price - trail_amount;
            if new_trigger > self.trigger_price {
                self.trigger_price = new_trigger;
            }
        }
    }
}

#[derive(Debug, Clone)]
pub struct Position {
    symbol: String,
    quantity: f64,
    entry_price: f64,
    current_price: f64,
    stop_loss: Option<StopLoss>,
    take_profit: Option<f64>,
    is_open: bool,
}

impl Position {
    pub fn new(symbol: &str, quantity: f64, entry_price: f64) -> Self {
        Position {
            symbol: symbol.to_string(),
            quantity,
            entry_price,
            current_price: entry_price,
            stop_loss: None,
            take_profit: None,
            is_open: true,
        }
    }

    pub fn unrealized_pnl(&self) -> f64 {
        (self.current_price - self.entry_price) * self.quantity
    }
}

#[derive(Debug)]
struct RiskManager {
    max_position_risk: f64,     // Максимальный риск на позицию (%)
    max_portfolio_risk: f64,    // Максимальный риск портфеля (%)
    max_daily_loss: f64,        // Максимальный дневной убыток ($)
    daily_loss: f64,            // Текущий дневной убыток
}

impl RiskManager {
    fn new(max_position_risk: f64, max_portfolio_risk: f64, max_daily_loss: f64) -> Self {
        RiskManager {
            max_position_risk,
            max_portfolio_risk,
            max_daily_loss,
            daily_loss: 0.0,
        }
    }

    fn calculate_position_size(
        &self,
        capital: f64,
        entry_price: f64,
        stop_price: f64,
    ) -> f64 {
        // Риск на сделку = max_position_risk % от капитала
        let risk_amount = capital * (self.max_position_risk / 100.0);
        let risk_per_share = entry_price - stop_price;

        if risk_per_share <= 0.0 {
            return 0.0;
        }

        risk_amount / risk_per_share
    }

    fn can_take_new_position(&self, potential_loss: f64) -> bool {
        self.daily_loss + potential_loss <= self.max_daily_loss
    }

    fn record_loss(&mut self, loss: f64) {
        self.daily_loss += loss;
    }

    fn reset_daily_loss(&mut self) {
        self.daily_loss = 0.0;
    }
}

#[derive(Debug)]
struct TradingEngine {
    capital: f64,
    positions: HashMap<String, Position>,
    risk_manager: RiskManager,
    closed_trades: Vec<(String, f64)>, // (symbol, pnl)
}

impl TradingEngine {
    fn new(capital: f64) -> Self {
        TradingEngine {
            capital,
            positions: HashMap::new(),
            risk_manager: RiskManager::new(2.0, 6.0, capital * 0.05),
            closed_trades: Vec::new(),
        }
    }

    fn open_position(
        &mut self,
        symbol: &str,
        entry_price: f64,
        stop_loss_price: f64,
    ) -> Result<(), String> {
        // Рассчитываем размер позиции на основе риска
        let position_size = self.risk_manager.calculate_position_size(
            self.capital,
            entry_price,
            stop_loss_price,
        );

        if position_size <= 0.0 {
            return Err("Некорректные параметры stop-loss".to_string());
        }

        let potential_loss = (entry_price - stop_loss_price) * position_size;

        if !self.risk_manager.can_take_new_position(potential_loss) {
            return Err(format!(
                "Превышен дневной лимит убытков. Текущий: ${:.2}, Лимит: ${:.2}",
                self.risk_manager.daily_loss, self.risk_manager.max_daily_loss
            ));
        }

        let cost = position_size * entry_price;
        if cost > self.capital {
            return Err(format!(
                "Недостаточно капитала. Нужно: ${:.2}, Есть: ${:.2}",
                cost, self.capital
            ));
        }

        let mut position = Position::new(symbol, position_size, entry_price);
        position.stop_loss = Some(StopLoss::new_fixed(stop_loss_price));

        self.capital -= cost;
        self.positions.insert(symbol.to_string(), position);

        println!(
            "Открыта позиция: {} шт {} по ${:.2}, стоп: ${:.2}",
            position_size.round(),
            symbol,
            entry_price,
            stop_loss_price
        );

        Ok(())
    }

    fn update_prices(&mut self, price_updates: &HashMap<String, f64>) {
        let mut to_close = Vec::new();

        for (symbol, new_price) in price_updates {
            if let Some(position) = self.positions.get_mut(symbol) {
                position.current_price = *new_price;

                // Обновляем trailing stop
                if let Some(ref mut stop) = position.stop_loss {
                    stop.update_trailing(*new_price);

                    // Проверяем срабатывание
                    if stop.should_trigger(*new_price) {
                        let pnl = position.unrealized_pnl();
                        println!(
                            "STOP-LOSS: {} закрыта по ${:.2}, P&L: ${:.2}",
                            symbol, new_price, pnl
                        );
                        to_close.push((symbol.clone(), pnl));
                    }
                }

                // Проверяем take-profit
                if let Some(tp) = position.take_profit {
                    if *new_price >= tp {
                        let pnl = position.unrealized_pnl();
                        println!(
                            "TAKE-PROFIT: {} закрыта по ${:.2}, P&L: ${:.2}",
                            symbol, new_price, pnl
                        );
                        to_close.push((symbol.clone(), pnl));
                    }
                }
            }
        }

        // Закрываем позиции
        for (symbol, pnl) in to_close {
            if let Some(position) = self.positions.remove(&symbol) {
                self.capital += position.quantity * position.current_price;
                if pnl < 0.0 {
                    self.risk_manager.record_loss(-pnl);
                }
                self.closed_trades.push((symbol, pnl));
            }
        }
    }

    fn get_portfolio_status(&self) -> String {
        let total_exposure: f64 = self.positions.values()
            .map(|p| p.current_price * p.quantity)
            .sum();
        let total_pnl: f64 = self.positions.values()
            .map(|p| p.unrealized_pnl())
            .sum();

        format!(
            "Капитал: ${:.2}, Позиции: {}, Экспозиция: ${:.2}, Нереализованный P&L: ${:.2}",
            self.capital,
            self.positions.len(),
            total_exposure,
            total_pnl
        )
    }
}

fn main() {
    let mut engine = TradingEngine::new(100_000.0);

    // Открываем позиции с риск-менеджментом
    let _ = engine.open_position("AAPL", 150.0, 145.0); // Стоп на $5 ниже
    let _ = engine.open_position("GOOGL", 140.0, 133.0); // Стоп на $7 ниже
    let _ = engine.open_position("MSFT", 380.0, 370.0);  // Стоп на $10 ниже

    println!("\n{}\n", engine.get_portfolio_status());

    // Симулируем движение цен
    let scenarios = vec![
        ("День 1", vec![("AAPL", 152.0), ("GOOGL", 142.0), ("MSFT", 385.0)]),
        ("День 2", vec![("AAPL", 148.0), ("GOOGL", 138.0), ("MSFT", 375.0)]),
        ("День 3", vec![("AAPL", 144.0), ("GOOGL", 135.0), ("MSFT", 368.0)]), // AAPL и MSFT стоп
    ];

    for (day, prices) in scenarios {
        println!("=== {} ===", day);
        let updates: HashMap<String, f64> = prices.into_iter()
            .map(|(s, p)| (s.to_string(), p))
            .collect();
        engine.update_prices(&updates);
        println!("{}\n", engine.get_portfolio_status());
    }
}
```

## Адаптивный Stop-Loss на основе волатильности

```rust
/// Рассчитывает Average True Range (ATR) — меру волатильности
fn calculate_atr(high: &[f64], low: &[f64], close: &[f64], period: usize) -> f64 {
    if high.len() < period + 1 || low.len() < period + 1 || close.len() < period + 1 {
        return 0.0;
    }

    let mut true_ranges = Vec::with_capacity(period);

    for i in 1..=period {
        let idx = high.len() - period - 1 + i;
        let prev_close = close[idx - 1];

        let tr = (high[idx] - low[idx])
            .max((high[idx] - prev_close).abs())
            .max((low[idx] - prev_close).abs());

        true_ranges.push(tr);
    }

    true_ranges.iter().sum::<f64>() / period as f64
}

#[derive(Debug)]
struct VolatilityBasedStop {
    atr_multiplier: f64,
    current_atr: f64,
    stop_price: f64,
}

impl VolatilityBasedStop {
    fn new(entry_price: f64, atr: f64, multiplier: f64) -> Self {
        VolatilityBasedStop {
            atr_multiplier: multiplier,
            current_atr: atr,
            stop_price: entry_price - (atr * multiplier),
        }
    }

    fn update(&mut self, current_price: f64, new_atr: f64) {
        self.current_atr = new_atr;
        let new_stop = current_price - (new_atr * self.atr_multiplier);

        // Стоп только поднимается
        if new_stop > self.stop_price {
            self.stop_price = new_stop;
        }
    }

    fn should_trigger(&self, current_price: f64) -> bool {
        current_price <= self.stop_price
    }
}

fn main() {
    // Исторические данные для расчёта ATR
    let high = vec![102.0, 104.0, 103.0, 105.0, 108.0, 107.0, 110.0, 109.0, 112.0, 111.0, 113.0];
    let low = vec![99.0, 101.0, 100.0, 102.0, 104.0, 103.0, 106.0, 105.0, 108.0, 107.0, 109.0];
    let close = vec![101.0, 103.0, 101.0, 104.0, 106.0, 105.0, 108.0, 107.0, 110.0, 109.0, 111.0];

    let atr = calculate_atr(&high, &low, &close, 5);
    println!("ATR (5 периодов): {:.2}", atr);

    let entry_price = 111.0;
    let mut stop = VolatilityBasedStop::new(entry_price, atr, 2.0);

    println!("Цена входа: ${:.2}", entry_price);
    println!("Stop-Loss (2 ATR): ${:.2}", stop.stop_price);
    println!("Расстояние до стопа: ${:.2} ({:.2}%)",
        entry_price - stop.stop_price,
        ((entry_price - stop.stop_price) / entry_price) * 100.0
    );

    // Симулируем движение цены и обновление ATR
    let new_prices = vec![(115.0, 3.2), (118.0, 3.5), (116.0, 3.8), (112.0, 4.0)];

    for (price, new_atr) in new_prices {
        stop.update(price, new_atr);
        println!(
            "Цена: ${:.2}, ATR: {:.2}, Стоп: ${:.2}{}",
            price,
            new_atr,
            stop.stop_price,
            if stop.should_trigger(price) { " [СРАБОТАЛ!]" } else { "" }
        );

        if stop.should_trigger(price) {
            break;
        }
    }
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Stop-Loss | Ордер на продажу при достижении заданной цены |
| Фиксированный стоп | Конкретная ценовая отметка |
| Процентный стоп | Процент от цены входа |
| Trailing Stop | Динамический стоп, следующий за ценой |
| ATR-стоп | Стоп на основе волатильности рынка |
| Риск-менеджмент | Ограничение убытков на позицию и портфель |
| Размер позиции | Расчёт на основе допустимого риска |

## Домашнее задание

1. **Множественные стопы**: Реализуй систему с несколькими уровнями stop-loss:
   - Первый стоп закрывает 50% позиции
   - Второй стоп закрывает оставшиеся 50%
   - Добавь логирование каждого срабатывания

2. **Стоп по времени**: Создай time-based stop-loss, который:
   - Закрывает позицию если за N свечей не достигнут take-profit
   - Ужесточает стоп каждые M свечей (подтягивает ближе к текущей цене)

3. **Break-even стоп**: Реализуй логику перевода стопа в безубыток:
   - При достижении прибыли в X% — стоп переносится на уровень входа
   - При достижении прибыли в 2X% — стоп переносится на уровень +X%
   - Протестируй на исторических данных

4. **Анализатор стопов**: Создай программу, которая:
   - Загружает исторические данные цены
   - Тестирует разные типы stop-loss (фиксированный, %, trailing, ATR)
   - Выводит статистику: количество срабатываний, средний убыток, максимальный убыток
   - Определяет оптимальный тип стопа для данного актива

## Навигация

[← Предыдущий день](../263-take-profit-securing-gains/ru.md) | [Следующий день →](../265-position-sizing-risk-per-trade/ru.md)
