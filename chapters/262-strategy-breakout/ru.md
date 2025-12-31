# День 262: Стратегия: Пробой уровня

## Торговая аналогия

Представьте, что вы наблюдаете за торговлей Bitcoin в узком диапазоне между $40,000 и $42,000 в течение нескольких дней. Трейдеры в неопределённости — покупатели толкают цену вверх, но продавцы сопротивляются на верхней границе. Внезапно крупный ордер на покупку толкает цену выше $42,000 с высоким объёмом. Это **пробой (breakout)** — момент, когда цена решительно выходит за ключевой уровень, часто сигнализируя о начале нового тренда.

Торговля на пробоях — одна из самых популярных стратегий в алгоритмическом трейдинге. Идея проста: определить ключевые ценовые уровни (поддержку и сопротивление), дождаться пробоя цены через эти уровни и войти в позицию в направлении пробоя. Сложность заключается в различении настоящих пробоев от ложных.

## Понимание стратегии пробоя

Пробой происходит, когда:
1. Цена консолидировалась в диапазоне
2. Диапазон ограничен уровнями поддержки (снизу) и сопротивления (сверху)
3. Цена решительно выходит за одну из этих границ
4. Объём торгов обычно увеличивается при настоящем пробое

Давайте смоделируем это на Rust:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum BreakoutDirection {
    Bullish,  // Цена пробивает сопротивление вверх
    Bearish,  // Цена пробивает поддержку вниз
}

#[derive(Debug, Clone)]
struct PriceRange {
    support: f64,      // Нижняя граница
    resistance: f64,   // Верхняя граница
    symbol: String,
}

impl PriceRange {
    fn new(symbol: &str, support: f64, resistance: f64) -> Self {
        PriceRange {
            support,
            resistance,
            symbol: symbol.to_string(),
        }
    }

    fn range_width(&self) -> f64 {
        self.resistance - self.support
    }

    fn is_within_range(&self, price: f64) -> bool {
        price >= self.support && price <= self.resistance
    }
}
```

## Обнаружение пробоев с помощью сопоставления с образцом

Сопоставление с образцом в Rust идеально подходит для логики обнаружения пробоев:

```rust
#[derive(Debug)]
struct BreakoutSignal {
    direction: BreakoutDirection,
    entry_price: f64,
    target_price: f64,
    stop_loss: f64,
}

fn detect_breakout(
    range: &PriceRange,
    current_price: f64,
    previous_price: f64,
) -> Option<BreakoutSignal> {
    let range_width = range.range_width();

    match (current_price, previous_price) {
        // Бычий пробой: цена пересекает сопротивление вверх
        (curr, prev) if curr > range.resistance && prev <= range.resistance => {
            Some(BreakoutSignal {
                direction: BreakoutDirection::Bullish,
                entry_price: curr,
                target_price: curr + range_width,  // Цель = вход + ширина диапазона
                stop_loss: range.resistance - (range_width * 0.25),  // Стоп ниже сопротивления
            })
        }
        // Медвежий пробой: цена пересекает поддержку вниз
        (curr, prev) if curr < range.support && prev >= range.support => {
            Some(BreakoutSignal {
                direction: BreakoutDirection::Bearish,
                entry_price: curr,
                target_price: curr - range_width,  // Цель = вход - ширина диапазона
                stop_loss: range.support + (range_width * 0.25),  // Стоп выше поддержки
            })
        }
        // Нет пробоя
        _ => None,
    }
}
```

## Подтверждение объёмом

Настоящие пробои обычно сопровождаются увеличением торгового объёма. Добавим анализ объёма:

```rust
#[derive(Debug, Clone)]
struct Candle {
    open: f64,
    high: f64,
    low: f64,
    close: f64,
    volume: f64,
    timestamp: u64,
}

struct BreakoutDetector {
    range: PriceRange,
    volume_threshold_multiplier: f64,
    average_volume: f64,
}

impl BreakoutDetector {
    fn new(range: PriceRange, average_volume: f64) -> Self {
        BreakoutDetector {
            range,
            volume_threshold_multiplier: 1.5,  // Требуем 150% от среднего объёма
            average_volume,
        }
    }

    fn is_volume_confirmed(&self, current_volume: f64) -> bool {
        current_volume >= self.average_volume * self.volume_threshold_multiplier
    }

    fn analyze_candle(&self, candle: &Candle, prev_close: f64) -> Option<BreakoutSignal> {
        // Сначала проверяем базовый пробой
        let signal = detect_breakout(&self.range, candle.close, prev_close)?;

        // Подтверждаем объёмом
        if self.is_volume_confirmed(candle.volume) {
            Some(signal)
        } else {
            println!(
                "Потенциальный пробой отклонён: объём {} ниже порога {}",
                candle.volume,
                self.average_volume * self.volume_threshold_multiplier
            );
            None
        }
    }
}
```

## Практический пример: Полная система торговли на пробоях

Построим полную систему торговли на пробоях, которая отслеживает позиции и рассчитывает прибыль/убыток:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    direction: BreakoutDirection,
    entry_price: f64,
    quantity: f64,
    stop_loss: f64,
    target_price: f64,
}

impl Position {
    fn unrealized_pnl(&self, current_price: f64) -> f64 {
        let price_diff = current_price - self.entry_price;
        match self.direction {
            BreakoutDirection::Bullish => price_diff * self.quantity,
            BreakoutDirection::Bearish => -price_diff * self.quantity,
        }
    }

    fn should_close(&self, current_price: f64) -> bool {
        match self.direction {
            BreakoutDirection::Bullish => {
                current_price <= self.stop_loss || current_price >= self.target_price
            }
            BreakoutDirection::Bearish => {
                current_price >= self.stop_loss || current_price <= self.target_price
            }
        }
    }
}

struct BreakoutTradingSystem {
    positions: HashMap<String, Position>,
    detectors: HashMap<String, BreakoutDetector>,
    capital: f64,
    risk_per_trade: f64,  // Процент капитала под риском на сделку
    realized_pnl: f64,
}

impl BreakoutTradingSystem {
    fn new(capital: f64, risk_per_trade: f64) -> Self {
        BreakoutTradingSystem {
            positions: HashMap::new(),
            detectors: HashMap::new(),
            capital,
            risk_per_trade,
            realized_pnl: 0.0,
        }
    }

    fn add_detector(&mut self, symbol: &str, range: PriceRange, avg_volume: f64) {
        let detector = BreakoutDetector::new(range, avg_volume);
        self.detectors.insert(symbol.to_string(), detector);
    }

    fn calculate_position_size(&self, entry: f64, stop_loss: f64) -> f64 {
        let risk_amount = self.capital * self.risk_per_trade;
        let risk_per_unit = (entry - stop_loss).abs();
        risk_amount / risk_per_unit
    }

    fn process_candle(&mut self, symbol: &str, candle: &Candle, prev_close: f64) {
        // Проверяем существующую позицию
        if let Some(position) = self.positions.get(symbol) {
            if position.should_close(candle.close) {
                let pnl = position.unrealized_pnl(candle.close);
                println!(
                    "[{}] Закрытие позиции: {:?} по {} | P&L: {:.2}",
                    symbol, position.direction, candle.close, pnl
                );
                self.realized_pnl += pnl;
                self.capital += pnl;
                self.positions.remove(symbol);
            }
            return;
        }

        // Ищем новый пробой
        if let Some(detector) = self.detectors.get(symbol) {
            if let Some(signal) = detector.analyze_candle(candle, prev_close) {
                let quantity = self.calculate_position_size(
                    signal.entry_price,
                    signal.stop_loss,
                );

                let position = Position {
                    symbol: symbol.to_string(),
                    direction: signal.direction,
                    entry_price: signal.entry_price,
                    quantity,
                    stop_loss: signal.stop_loss,
                    target_price: signal.target_price,
                };

                println!(
                    "[{}] Открытие {:?} позиции: вход={:.2}, кол-во={:.4}, стоп={:.2}, цель={:.2}",
                    symbol,
                    position.direction,
                    position.entry_price,
                    position.quantity,
                    position.stop_loss,
                    position.target_price
                );

                self.positions.insert(symbol.to_string(), position);
            }
        }
    }

    fn portfolio_status(&self) {
        println!("\n=== Статус портфеля ===");
        println!("Капитал: ${:.2}", self.capital);
        println!("Реализованный P&L: ${:.2}", self.realized_pnl);
        println!("Открытых позиций: {}", self.positions.len());
        for (symbol, pos) in &self.positions {
            println!("  {} - {:?} @ {:.2}", symbol, pos.direction, pos.entry_price);
        }
    }
}

fn main() {
    // Инициализируем торговую систему с капиталом $100,000, риск 1% на сделку
    let mut system = BreakoutTradingSystem::new(100_000.0, 0.01);

    // Настраиваем обнаружение пробоев для BTC
    let btc_range = PriceRange::new("BTC", 40_000.0, 42_000.0);
    system.add_detector("BTC", btc_range, 1000.0);

    // Настраиваем обнаружение пробоев для ETH
    let eth_range = PriceRange::new("ETH", 2_200.0, 2_400.0);
    system.add_detector("ETH", eth_range, 5000.0);

    // Симулируем ценовые данные - бычий пробой BTC
    let btc_candles = vec![
        Candle { open: 41_500.0, high: 41_800.0, low: 41_200.0, close: 41_700.0, volume: 900.0, timestamp: 1 },
        Candle { open: 41_700.0, high: 42_500.0, low: 41_600.0, close: 42_300.0, volume: 1800.0, timestamp: 2 },
        Candle { open: 42_300.0, high: 43_500.0, low: 42_100.0, close: 43_200.0, volume: 2200.0, timestamp: 3 },
        Candle { open: 43_200.0, high: 44_500.0, low: 43_000.0, close: 44_200.0, volume: 1500.0, timestamp: 4 },
    ];

    // Обрабатываем свечи BTC
    println!("Обработка ценового движения BTC...\n");
    let mut prev_close = 41_500.0;
    for candle in &btc_candles {
        system.process_candle("BTC", candle, prev_close);
        prev_close = candle.close;
    }

    system.portfolio_status();
}
```

## Защита от ложных пробоев

Ложные пробои — распространённая проблема. Вот как добавить защиту:

```rust
struct BreakoutFilter {
    min_close_beyond_level: f64,  // Минимальный % закрытия за уровнем
    confirmation_candles: usize,   // Количество свечей для подтверждения
    breakout_history: Vec<(u64, f64)>,  // (timestamp, close_price)
}

impl BreakoutFilter {
    fn new(min_close_pct: f64, confirmation_candles: usize) -> Self {
        BreakoutFilter {
            min_close_beyond_level: min_close_pct,
            confirmation_candles,
            breakout_history: Vec::new(),
        }
    }

    fn validate_bullish_breakout(
        &mut self,
        resistance: f64,
        candle: &Candle,
    ) -> bool {
        // Проверяем, достаточно ли далеко закрытие за сопротивлением
        let breakout_pct = (candle.close - resistance) / resistance * 100.0;
        if breakout_pct < self.min_close_beyond_level {
            return false;
        }

        // Отслеживаем для подтверждения
        self.breakout_history.push((candle.timestamp, candle.close));

        // Очищаем старые записи
        self.breakout_history.retain(|(ts, _)| {
            candle.timestamp - ts <= self.confirmation_candles as u64
        });

        // Проверяем, достаточно ли подтверждающих свечей выше сопротивления
        let confirmed_candles = self.breakout_history
            .iter()
            .filter(|(_, close)| *close > resistance)
            .count();

        confirmed_candles >= self.confirmation_candles
    }
}
```

## Определение уровней поддержки и сопротивления

Чтобы сделать систему более динамичной, можно автоматически определять ключевые уровни:

```rust
fn find_support_resistance(prices: &[f64], lookback: usize) -> Option<PriceRange> {
    if prices.len() < lookback {
        return None;
    }

    let recent_prices = &prices[prices.len() - lookback..];

    let high = recent_prices
        .iter()
        .cloned()
        .fold(f64::NEG_INFINITY, f64::max);

    let low = recent_prices
        .iter()
        .cloned()
        .fold(f64::INFINITY, f64::min);

    // Вычисляем ширину диапазона в процентах
    let range_pct = (high - low) / low * 100.0;

    // Возвращаем диапазон только если это консолидация (узкий диапазон)
    if range_pct < 10.0 {  // Менее 10% диапазон = консолидация
        Some(PriceRange::new("AUTO", low, high))
    } else {
        None
    }
}

fn calculate_average_volume(candles: &[Candle], periods: usize) -> f64 {
    let slice = if candles.len() > periods {
        &candles[candles.len() - periods..]
    } else {
        candles
    };

    let total: f64 = slice.iter().map(|c| c.volume).sum();
    total / slice.len() as f64
}
```

## Что мы изучили

| Концепция | Применение в трейдинге |
|-----------|----------------------|
| Перечисления (Enums) | Моделирование направления пробоя (Bullish/Bearish) |
| Сопоставление с образцом | Обнаружение и классификация условий пробоя |
| Option<T> | Обработка случаев отсутствия пробоя |
| Структуры (Structs) | Представление ценовых диапазонов, позиций и сигналов |
| HashMap | Отслеживание множества позиций и детекторов |
| Итераторы | Анализ данных о цене и объёме |
| Методы | Инкапсуляция торговой логики в типах |
| Операции с f64 | Расчёт целей, стопов и прибыли/убытка |

## Домашнее задание

1. **Улучшенный фильтр пробоя**: Реализуйте фильтр "ретеста", который ждёт пробоя цены, отката к уровню пробоя, а затем продолжения движения в направлении пробоя перед входом в позицию.

2. **Подтверждение на нескольких таймфреймах**: Модифицируйте `BreakoutDetector`, чтобы требовать подтверждения пробоя на нескольких таймфреймах (например, часовой и 4-часовой графики должны показывать пробой).

3. **Реализация трейлинг-стопа**: Добавьте механизм трейлинг-стопа в структуру `Position`, который перемещает стоп-лосс в направлении прибыли по мере развития сделки.

4. **Трекер статистики пробоев**: Создайте структуру `BreakoutStats`, которая отслеживает историческую эффективность пробоев, включая процент выигрышных сделок, среднюю прибыль/убыток и максимальную просадку стратегии.

## Навигация

[← Предыдущий день](../261-strategy-mean-reversion/ru.md) | [Следующий день →](../263-strategy-momentum/ru.md)
