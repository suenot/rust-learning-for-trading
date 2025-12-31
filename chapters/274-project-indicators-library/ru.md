# День 274: Проект: Библиотека индикаторов

## Аналогия из трейдинга

Представь, что ты строишь свой собственный торговый терминал. Каждый профессиональный трейдер имеет набор инструментов — индикаторов, которые помогают принимать решения. Одни смотрят на скользящие средние, другие на RSI, третьи комбинируют несколько индикаторов. Твоя задача — создать универсальную библиотеку, которая позволит легко использовать любой индикатор и комбинировать их для создания торговых стратегий.

Это как набор LEGO для трейдера: отдельные кирпичики (индикаторы) можно соединять в любые конструкции (стратегии). Хорошо спроектированная библиотека позволит:
- Легко добавлять новые индикаторы
- Комбинировать индикаторы для генерации сигналов
- Тестировать стратегии на исторических данных
- Измерять риск и эффективность

## Архитектура библиотеки

Наша библиотека будет состоять из нескольких модулей:

```
indicators_lib/
├── src/
│   ├── lib.rs           # Точка входа библиотеки
│   ├── candle.rs        # Структура OHLCV свечи
│   ├── indicator.rs     # Trait для индикаторов
│   ├── indicators/      # Реализации индикаторов
│   │   ├── mod.rs
│   │   ├── sma.rs
│   │   ├── ema.rs
│   │   ├── rsi.rs
│   │   ├── macd.rs
│   │   ├── bollinger.rs
│   │   ├── atr.rs
│   │   └── vwap.rs
│   ├── signal.rs        # Торговые сигналы
│   ├── strategy.rs      # Trait для стратегий
│   ├── risk.rs          # Риск-менеджмент
│   └── metrics.rs       # Метрики эффективности
└── Cargo.toml
```

## Базовые структуры данных

### Свеча OHLCV

```rust
use std::time::{SystemTime, UNIX_EPOCH};

/// Временная метка в миллисекундах
pub type Timestamp = u64;

/// OHLCV свеча — базовая структура ценовых данных
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Candle {
    pub timestamp: Timestamp,
    pub open: f64,
    pub high: f64,
    pub low: f64,
    pub close: f64,
    pub volume: f64,
}

impl Candle {
    pub fn new(timestamp: Timestamp, open: f64, high: f64, low: f64, close: f64, volume: f64) -> Self {
        Self { timestamp, open, high, low, close, volume }
    }

    /// Типичная цена (TP) = (High + Low + Close) / 3
    pub fn typical_price(&self) -> f64 {
        (self.high + self.low + self.close) / 3.0
    }

    /// Истинный диапазон (True Range)
    pub fn true_range(&self, prev_close: Option<f64>) -> f64 {
        match prev_close {
            Some(pc) => {
                let hl = self.high - self.low;
                let hc = (self.high - pc).abs();
                let lc = (self.low - pc).abs();
                hl.max(hc).max(lc)
            }
            None => self.high - self.low,
        }
    }

    /// Свеча бычья (закрытие выше открытия)?
    pub fn is_bullish(&self) -> bool {
        self.close > self.open
    }

    /// Свеча медвежья (закрытие ниже открытия)?
    pub fn is_bearish(&self) -> bool {
        self.close < self.open
    }

    /// Тело свечи (абсолютное значение)
    pub fn body(&self) -> f64 {
        (self.close - self.open).abs()
    }

    /// Верхняя тень
    pub fn upper_shadow(&self) -> f64 {
        self.high - self.close.max(self.open)
    }

    /// Нижняя тень
    pub fn lower_shadow(&self) -> f64 {
        self.close.min(self.open) - self.low
    }
}

/// Серия свечей для расчёта индикаторов
#[derive(Debug, Clone)]
pub struct CandleSeries {
    candles: Vec<Candle>,
    max_size: Option<usize>,
}

impl CandleSeries {
    pub fn new() -> Self {
        Self { candles: Vec::new(), max_size: None }
    }

    pub fn with_max_size(max_size: usize) -> Self {
        Self { candles: Vec::with_capacity(max_size), max_size: Some(max_size) }
    }

    pub fn push(&mut self, candle: Candle) {
        if let Some(max) = self.max_size {
            if self.candles.len() >= max {
                self.candles.remove(0);
            }
        }
        self.candles.push(candle);
    }

    pub fn len(&self) -> usize {
        self.candles.len()
    }

    pub fn is_empty(&self) -> bool {
        self.candles.is_empty()
    }

    pub fn get(&self, index: usize) -> Option<&Candle> {
        self.candles.get(index)
    }

    pub fn last(&self) -> Option<&Candle> {
        self.candles.last()
    }

    pub fn closes(&self) -> Vec<f64> {
        self.candles.iter().map(|c| c.close).collect()
    }

    pub fn highs(&self) -> Vec<f64> {
        self.candles.iter().map(|c| c.high).collect()
    }

    pub fn lows(&self) -> Vec<f64> {
        self.candles.iter().map(|c| c.low).collect()
    }

    pub fn volumes(&self) -> Vec<f64> {
        self.candles.iter().map(|c| c.volume).collect()
    }

    pub fn iter(&self) -> impl Iterator<Item = &Candle> {
        self.candles.iter()
    }
}

impl Default for CandleSeries {
    fn default() -> Self {
        Self::new()
    }
}
```

## Trait для индикаторов

```rust
/// Результат расчёта индикатора
#[derive(Debug, Clone)]
pub enum IndicatorValue {
    /// Одно значение (SMA, EMA, RSI)
    Single(f64),
    /// Два значения (MACD: линия и сигнал)
    Dual(f64, f64),
    /// Три значения (Bollinger Bands: верхняя, средняя, нижняя)
    Triple(f64, f64, f64),
    /// Множество значений
    Multiple(Vec<f64>),
    /// Значение ещё не готово (недостаточно данных)
    NotReady,
}

impl IndicatorValue {
    pub fn as_single(&self) -> Option<f64> {
        match self {
            IndicatorValue::Single(v) => Some(*v),
            _ => None,
        }
    }

    pub fn as_dual(&self) -> Option<(f64, f64)> {
        match self {
            IndicatorValue::Dual(a, b) => Some((*a, *b)),
            _ => None,
        }
    }

    pub fn as_triple(&self) -> Option<(f64, f64, f64)> {
        match self {
            IndicatorValue::Triple(a, b, c) => Some((*a, *b, *c)),
            _ => None,
        }
    }

    pub fn is_ready(&self) -> bool {
        !matches!(self, IndicatorValue::NotReady)
    }
}

/// Trait для всех индикаторов
pub trait Indicator: Send + Sync {
    /// Название индикатора
    fn name(&self) -> &str;

    /// Обновить индикатор новой свечой
    fn update(&mut self, candle: &Candle);

    /// Получить текущее значение
    fn value(&self) -> IndicatorValue;

    /// Сбросить индикатор в начальное состояние
    fn reset(&mut self);

    /// Минимальное количество свечей для расчёта
    fn min_periods(&self) -> usize;

    /// Готов ли индикатор к выдаче значений
    fn is_ready(&self) -> bool {
        self.value().is_ready()
    }
}

/// Trait для индикаторов, которые можно рассчитать на массиве данных
pub trait BatchIndicator: Indicator {
    fn calculate_batch(&mut self, candles: &[Candle]) -> Vec<IndicatorValue> {
        self.reset();
        candles.iter().map(|c| {
            self.update(c);
            self.value()
        }).collect()
    }
}

// Автоматически реализуем BatchIndicator для всех Indicator
impl<T: Indicator> BatchIndicator for T {}
```

## Реализация индикаторов

### SMA (Simple Moving Average)

```rust
/// Простая скользящая средняя
#[derive(Debug, Clone)]
pub struct SMA {
    period: usize,
    values: Vec<f64>,
    sum: f64,
}

impl SMA {
    pub fn new(period: usize) -> Self {
        assert!(period > 0, "Period must be greater than 0");
        Self {
            period,
            values: Vec::with_capacity(period),
            sum: 0.0,
        }
    }

    pub fn period(&self) -> usize {
        self.period
    }
}

impl Indicator for SMA {
    fn name(&self) -> &str {
        "SMA"
    }

    fn update(&mut self, candle: &Candle) {
        let price = candle.close;

        if self.values.len() >= self.period {
            self.sum -= self.values.remove(0);
        }

        self.values.push(price);
        self.sum += price;
    }

    fn value(&self) -> IndicatorValue {
        if self.values.len() >= self.period {
            IndicatorValue::Single(self.sum / self.period as f64)
        } else {
            IndicatorValue::NotReady
        }
    }

    fn reset(&mut self) {
        self.values.clear();
        self.sum = 0.0;
    }

    fn min_periods(&self) -> usize {
        self.period
    }
}
```

### EMA (Exponential Moving Average)

```rust
/// Экспоненциальная скользящая средняя
#[derive(Debug, Clone)]
pub struct EMA {
    period: usize,
    multiplier: f64,
    current_ema: Option<f64>,
    count: usize,
    initial_sum: f64,
}

impl EMA {
    pub fn new(period: usize) -> Self {
        assert!(period > 0, "Period must be greater than 0");
        let multiplier = 2.0 / (period as f64 + 1.0);
        Self {
            period,
            multiplier,
            current_ema: None,
            count: 0,
            initial_sum: 0.0,
        }
    }

    pub fn period(&self) -> usize {
        self.period
    }
}

impl Indicator for EMA {
    fn name(&self) -> &str {
        "EMA"
    }

    fn update(&mut self, candle: &Candle) {
        let price = candle.close;
        self.count += 1;

        match self.current_ema {
            Some(ema) => {
                // EMA = Price * multiplier + EMA_prev * (1 - multiplier)
                self.current_ema = Some(price * self.multiplier + ema * (1.0 - self.multiplier));
            }
            None => {
                self.initial_sum += price;
                if self.count >= self.period {
                    // Первое значение EMA = SMA
                    self.current_ema = Some(self.initial_sum / self.period as f64);
                }
            }
        }
    }

    fn value(&self) -> IndicatorValue {
        match self.current_ema {
            Some(v) => IndicatorValue::Single(v),
            None => IndicatorValue::NotReady,
        }
    }

    fn reset(&mut self) {
        self.current_ema = None;
        self.count = 0;
        self.initial_sum = 0.0;
    }

    fn min_periods(&self) -> usize {
        self.period
    }
}
```

### RSI (Relative Strength Index)

```rust
/// Индекс относительной силы
#[derive(Debug, Clone)]
pub struct RSI {
    period: usize,
    prev_close: Option<f64>,
    avg_gain: Option<f64>,
    avg_loss: Option<f64>,
    gains: Vec<f64>,
    losses: Vec<f64>,
}

impl RSI {
    pub fn new(period: usize) -> Self {
        assert!(period > 0, "Period must be greater than 0");
        Self {
            period,
            prev_close: None,
            avg_gain: None,
            avg_loss: None,
            gains: Vec::with_capacity(period),
            losses: Vec::with_capacity(period),
        }
    }
}

impl Indicator for RSI {
    fn name(&self) -> &str {
        "RSI"
    }

    fn update(&mut self, candle: &Candle) {
        if let Some(prev) = self.prev_close {
            let change = candle.close - prev;
            let gain = if change > 0.0 { change } else { 0.0 };
            let loss = if change < 0.0 { -change } else { 0.0 };

            match (&mut self.avg_gain, &mut self.avg_loss) {
                (Some(ag), Some(al)) => {
                    // Сглаженное среднее: (prev_avg * (period-1) + current) / period
                    *ag = (*ag * (self.period - 1) as f64 + gain) / self.period as f64;
                    *al = (*al * (self.period - 1) as f64 + loss) / self.period as f64;
                }
                _ => {
                    self.gains.push(gain);
                    self.losses.push(loss);

                    if self.gains.len() >= self.period {
                        self.avg_gain = Some(self.gains.iter().sum::<f64>() / self.period as f64);
                        self.avg_loss = Some(self.losses.iter().sum::<f64>() / self.period as f64);
                    }
                }
            }
        }
        self.prev_close = Some(candle.close);
    }

    fn value(&self) -> IndicatorValue {
        match (self.avg_gain, self.avg_loss) {
            (Some(ag), Some(al)) => {
                if al == 0.0 {
                    IndicatorValue::Single(100.0)
                } else {
                    let rs = ag / al;
                    let rsi = 100.0 - (100.0 / (1.0 + rs));
                    IndicatorValue::Single(rsi)
                }
            }
            _ => IndicatorValue::NotReady,
        }
    }

    fn reset(&mut self) {
        self.prev_close = None;
        self.avg_gain = None;
        self.avg_loss = None;
        self.gains.clear();
        self.losses.clear();
    }

    fn min_periods(&self) -> usize {
        self.period + 1
    }
}
```

### MACD (Moving Average Convergence Divergence)

```rust
/// MACD — схождение/расхождение скользящих средних
#[derive(Debug, Clone)]
pub struct MACD {
    fast_ema: EMA,
    slow_ema: EMA,
    signal_ema: EMA,
    macd_values: Vec<f64>,
    signal_period: usize,
}

impl MACD {
    pub fn new(fast_period: usize, slow_period: usize, signal_period: usize) -> Self {
        assert!(fast_period < slow_period, "Fast period must be less than slow period");
        Self {
            fast_ema: EMA::new(fast_period),
            slow_ema: EMA::new(slow_period),
            signal_ema: EMA::new(signal_period),
            macd_values: Vec::new(),
            signal_period,
        }
    }

    /// Стандартные параметры MACD (12, 26, 9)
    pub fn standard() -> Self {
        Self::new(12, 26, 9)
    }

    /// Получить гистограмму (MACD - Signal)
    pub fn histogram(&self) -> Option<f64> {
        if let IndicatorValue::Dual(macd, signal) = self.value() {
            Some(macd - signal)
        } else {
            None
        }
    }
}

impl Indicator for MACD {
    fn name(&self) -> &str {
        "MACD"
    }

    fn update(&mut self, candle: &Candle) {
        self.fast_ema.update(candle);
        self.slow_ema.update(candle);

        if let (IndicatorValue::Single(fast), IndicatorValue::Single(slow)) =
            (self.fast_ema.value(), self.slow_ema.value())
        {
            let macd_value = fast - slow;
            self.macd_values.push(macd_value);

            // Обновляем сигнальную линию
            let signal_candle = Candle::new(candle.timestamp, macd_value, macd_value, macd_value, macd_value, 0.0);
            self.signal_ema.update(&signal_candle);
        }
    }

    fn value(&self) -> IndicatorValue {
        if let (IndicatorValue::Single(fast), IndicatorValue::Single(slow), IndicatorValue::Single(signal)) =
            (self.fast_ema.value(), self.slow_ema.value(), self.signal_ema.value())
        {
            let macd = fast - slow;
            IndicatorValue::Dual(macd, signal)
        } else {
            IndicatorValue::NotReady
        }
    }

    fn reset(&mut self) {
        self.fast_ema.reset();
        self.slow_ema.reset();
        self.signal_ema.reset();
        self.macd_values.clear();
    }

    fn min_periods(&self) -> usize {
        self.slow_ema.period() + self.signal_period
    }
}
```

### Bollinger Bands

```rust
/// Полосы Боллинджера
#[derive(Debug, Clone)]
pub struct BollingerBands {
    period: usize,
    std_dev_multiplier: f64,
    values: Vec<f64>,
}

impl BollingerBands {
    pub fn new(period: usize, std_dev_multiplier: f64) -> Self {
        Self {
            period,
            std_dev_multiplier,
            values: Vec::with_capacity(period),
        }
    }

    /// Стандартные параметры (20, 2.0)
    pub fn standard() -> Self {
        Self::new(20, 2.0)
    }

    fn calculate_std_dev(&self, mean: f64) -> f64 {
        let variance: f64 = self.values.iter()
            .map(|v| (v - mean).powi(2))
            .sum::<f64>() / self.values.len() as f64;
        variance.sqrt()
    }

    /// Ширина полос (верхняя - нижняя)
    pub fn bandwidth(&self) -> Option<f64> {
        if let IndicatorValue::Triple(upper, _, lower) = self.value() {
            Some(upper - lower)
        } else {
            None
        }
    }

    /// Процентное положение цены в полосах (0 = нижняя, 1 = верхняя)
    pub fn percent_b(&self, price: f64) -> Option<f64> {
        if let IndicatorValue::Triple(upper, _, lower) = self.value() {
            if upper != lower {
                Some((price - lower) / (upper - lower))
            } else {
                None
            }
        } else {
            None
        }
    }
}

impl Indicator for BollingerBands {
    fn name(&self) -> &str {
        "Bollinger Bands"
    }

    fn update(&mut self, candle: &Candle) {
        if self.values.len() >= self.period {
            self.values.remove(0);
        }
        self.values.push(candle.close);
    }

    fn value(&self) -> IndicatorValue {
        if self.values.len() >= self.period {
            let mean: f64 = self.values.iter().sum::<f64>() / self.values.len() as f64;
            let std_dev = self.calculate_std_dev(mean);

            let upper = mean + std_dev * self.std_dev_multiplier;
            let lower = mean - std_dev * self.std_dev_multiplier;

            IndicatorValue::Triple(upper, mean, lower)
        } else {
            IndicatorValue::NotReady
        }
    }

    fn reset(&mut self) {
        self.values.clear();
    }

    fn min_periods(&self) -> usize {
        self.period
    }
}
```

### ATR (Average True Range)

```rust
/// Средний истинный диапазон — мера волатильности
#[derive(Debug, Clone)]
pub struct ATR {
    period: usize,
    prev_close: Option<f64>,
    tr_values: Vec<f64>,
    current_atr: Option<f64>,
}

impl ATR {
    pub fn new(period: usize) -> Self {
        Self {
            period,
            prev_close: None,
            tr_values: Vec::with_capacity(period),
            current_atr: None,
        }
    }
}

impl Indicator for ATR {
    fn name(&self) -> &str {
        "ATR"
    }

    fn update(&mut self, candle: &Candle) {
        let tr = candle.true_range(self.prev_close);
        self.prev_close = Some(candle.close);

        match self.current_atr {
            Some(atr) => {
                // Сглаженное ATR: ((prev_atr * (period-1)) + current_tr) / period
                self.current_atr = Some((atr * (self.period - 1) as f64 + tr) / self.period as f64);
            }
            None => {
                self.tr_values.push(tr);
                if self.tr_values.len() >= self.period {
                    self.current_atr = Some(self.tr_values.iter().sum::<f64>() / self.period as f64);
                }
            }
        }
    }

    fn value(&self) -> IndicatorValue {
        match self.current_atr {
            Some(v) => IndicatorValue::Single(v),
            None => IndicatorValue::NotReady,
        }
    }

    fn reset(&mut self) {
        self.prev_close = None;
        self.tr_values.clear();
        self.current_atr = None;
    }

    fn min_periods(&self) -> usize {
        self.period
    }
}
```

### VWAP (Volume Weighted Average Price)

```rust
/// Средневзвешенная по объёму цена
#[derive(Debug, Clone)]
pub struct VWAP {
    cumulative_tp_volume: f64,
    cumulative_volume: f64,
    count: usize,
}

impl VWAP {
    pub fn new() -> Self {
        Self {
            cumulative_tp_volume: 0.0,
            cumulative_volume: 0.0,
            count: 0,
        }
    }
}

impl Default for VWAP {
    fn default() -> Self {
        Self::new()
    }
}

impl Indicator for VWAP {
    fn name(&self) -> &str {
        "VWAP"
    }

    fn update(&mut self, candle: &Candle) {
        let tp = candle.typical_price();
        self.cumulative_tp_volume += tp * candle.volume;
        self.cumulative_volume += candle.volume;
        self.count += 1;
    }

    fn value(&self) -> IndicatorValue {
        if self.cumulative_volume > 0.0 {
            IndicatorValue::Single(self.cumulative_tp_volume / self.cumulative_volume)
        } else {
            IndicatorValue::NotReady
        }
    }

    fn reset(&mut self) {
        self.cumulative_tp_volume = 0.0;
        self.cumulative_volume = 0.0;
        self.count = 0;
    }

    fn min_periods(&self) -> usize {
        1
    }
}
```

## Торговые сигналы

```rust
/// Направление торгового сигнала
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SignalDirection {
    Buy,
    Sell,
    Hold,
}

/// Сила сигнала (от 0 до 1)
#[derive(Debug, Clone, Copy)]
pub struct SignalStrength(f64);

impl SignalStrength {
    pub fn new(value: f64) -> Self {
        Self(value.clamp(0.0, 1.0))
    }

    pub fn value(&self) -> f64 {
        self.0
    }

    pub fn is_strong(&self) -> bool {
        self.0 >= 0.7
    }

    pub fn is_weak(&self) -> bool {
        self.0 <= 0.3
    }
}

/// Торговый сигнал
#[derive(Debug, Clone)]
pub struct Signal {
    pub direction: SignalDirection,
    pub strength: SignalStrength,
    pub source: String,
    pub timestamp: Timestamp,
    pub price: f64,
}

impl Signal {
    pub fn buy(source: &str, strength: f64, timestamp: Timestamp, price: f64) -> Self {
        Self {
            direction: SignalDirection::Buy,
            strength: SignalStrength::new(strength),
            source: source.to_string(),
            timestamp,
            price,
        }
    }

    pub fn sell(source: &str, strength: f64, timestamp: Timestamp, price: f64) -> Self {
        Self {
            direction: SignalDirection::Sell,
            strength: SignalStrength::new(strength),
            source: source.to_string(),
            timestamp,
            price,
        }
    }

    pub fn hold(source: &str, timestamp: Timestamp, price: f64) -> Self {
        Self {
            direction: SignalDirection::Hold,
            strength: SignalStrength::new(0.0),
            source: source.to_string(),
            timestamp,
            price,
        }
    }
}

/// Trait для генераторов сигналов
pub trait SignalGenerator: Send + Sync {
    fn name(&self) -> &str;
    fn generate(&mut self, candle: &Candle) -> Option<Signal>;
    fn reset(&mut self);
}
```

## Стратегии

### SMA Crossover Strategy

```rust
/// Стратегия пересечения скользящих средних
pub struct SMACrossover {
    fast_sma: SMA,
    slow_sma: SMA,
    prev_fast: Option<f64>,
    prev_slow: Option<f64>,
}

impl SMACrossover {
    pub fn new(fast_period: usize, slow_period: usize) -> Self {
        assert!(fast_period < slow_period, "Fast period must be less than slow period");
        Self {
            fast_sma: SMA::new(fast_period),
            slow_sma: SMA::new(slow_period),
            prev_fast: None,
            prev_slow: None,
        }
    }
}

impl SignalGenerator for SMACrossover {
    fn name(&self) -> &str {
        "SMA Crossover"
    }

    fn generate(&mut self, candle: &Candle) -> Option<Signal> {
        self.fast_sma.update(candle);
        self.slow_sma.update(candle);

        let result = match (self.fast_sma.value().as_single(), self.slow_sma.value().as_single()) {
            (Some(fast), Some(slow)) => {
                let signal = match (self.prev_fast, self.prev_slow) {
                    (Some(pf), Some(ps)) => {
                        // Бычье пересечение: быстрая пересекает медленную снизу вверх
                        if pf <= ps && fast > slow {
                            Some(Signal::buy(self.name(), 0.8, candle.timestamp, candle.close))
                        }
                        // Медвежье пересечение: быстрая пересекает медленную сверху вниз
                        else if pf >= ps && fast < slow {
                            Some(Signal::sell(self.name(), 0.8, candle.timestamp, candle.close))
                        } else {
                            None
                        }
                    }
                    _ => None,
                };

                self.prev_fast = Some(fast);
                self.prev_slow = Some(slow);
                signal
            }
            _ => None,
        };

        result
    }

    fn reset(&mut self) {
        self.fast_sma.reset();
        self.slow_sma.reset();
        self.prev_fast = None;
        self.prev_slow = None;
    }
}
```

### RSI Strategy

```rust
/// Стратегия на основе RSI (перекупленность/перепроданность)
pub struct RSIStrategy {
    rsi: RSI,
    overbought: f64,
    oversold: f64,
    prev_rsi: Option<f64>,
}

impl RSIStrategy {
    pub fn new(period: usize, overbought: f64, oversold: f64) -> Self {
        Self {
            rsi: RSI::new(period),
            overbought,
            oversold,
            prev_rsi: None,
        }
    }

    /// Стандартные параметры (14, 70, 30)
    pub fn standard() -> Self {
        Self::new(14, 70.0, 30.0)
    }
}

impl SignalGenerator for RSIStrategy {
    fn name(&self) -> &str {
        "RSI Strategy"
    }

    fn generate(&mut self, candle: &Candle) -> Option<Signal> {
        self.rsi.update(candle);

        let signal = if let Some(rsi) = self.rsi.value().as_single() {
            let result = match self.prev_rsi {
                Some(prev) => {
                    // Выход из зоны перепроданности — сигнал на покупку
                    if prev <= self.oversold && rsi > self.oversold {
                        let strength = (self.oversold - prev.min(self.oversold)) / self.oversold;
                        Some(Signal::buy(self.name(), 0.5 + strength * 0.5, candle.timestamp, candle.close))
                    }
                    // Выход из зоны перекупленности — сигнал на продажу
                    else if prev >= self.overbought && rsi < self.overbought {
                        let strength = (prev.max(self.overbought) - self.overbought) / (100.0 - self.overbought);
                        Some(Signal::sell(self.name(), 0.5 + strength * 0.5, candle.timestamp, candle.close))
                    } else {
                        None
                    }
                }
                None => None,
            };

            self.prev_rsi = Some(rsi);
            result
        } else {
            None
        };

        signal
    }

    fn reset(&mut self) {
        self.rsi.reset();
        self.prev_rsi = None;
    }
}
```

### Комбинированная стратегия

```rust
/// Комбинация нескольких генераторов сигналов
pub struct CompositeStrategy {
    strategies: Vec<Box<dyn SignalGenerator>>,
    min_agreement: usize,
}

impl CompositeStrategy {
    pub fn new(min_agreement: usize) -> Self {
        Self {
            strategies: Vec::new(),
            min_agreement,
        }
    }

    pub fn add_strategy(&mut self, strategy: Box<dyn SignalGenerator>) {
        self.strategies.push(strategy);
    }
}

impl SignalGenerator for CompositeStrategy {
    fn name(&self) -> &str {
        "Composite Strategy"
    }

    fn generate(&mut self, candle: &Candle) -> Option<Signal> {
        let signals: Vec<Signal> = self.strategies
            .iter_mut()
            .filter_map(|s| s.generate(candle))
            .collect();

        let buy_count = signals.iter().filter(|s| s.direction == SignalDirection::Buy).count();
        let sell_count = signals.iter().filter(|s| s.direction == SignalDirection::Sell).count();

        if buy_count >= self.min_agreement {
            let avg_strength = signals.iter()
                .filter(|s| s.direction == SignalDirection::Buy)
                .map(|s| s.strength.value())
                .sum::<f64>() / buy_count as f64;
            Some(Signal::buy(self.name(), avg_strength, candle.timestamp, candle.close))
        } else if sell_count >= self.min_agreement {
            let avg_strength = signals.iter()
                .filter(|s| s.direction == SignalDirection::Sell)
                .map(|s| s.strength.value())
                .sum::<f64>() / sell_count as f64;
            Some(Signal::sell(self.name(), avg_strength, candle.timestamp, candle.close))
        } else {
            None
        }
    }

    fn reset(&mut self) {
        for strategy in &mut self.strategies {
            strategy.reset();
        }
    }
}
```

## Риск-менеджмент

```rust
/// Параметры риск-менеджмента
#[derive(Debug, Clone)]
pub struct RiskParams {
    /// Максимальный риск на сделку (% от капитала)
    pub max_risk_per_trade: f64,
    /// Максимальная просадка портфеля
    pub max_drawdown: f64,
    /// Соотношение риск/прибыль
    pub risk_reward_ratio: f64,
}

impl Default for RiskParams {
    fn default() -> Self {
        Self {
            max_risk_per_trade: 0.02, // 2%
            max_drawdown: 0.20,       // 20%
            risk_reward_ratio: 2.0,   // 1:2
        }
    }
}

/// Калькулятор размера позиции
#[derive(Debug)]
pub struct PositionSizer {
    params: RiskParams,
}

impl PositionSizer {
    pub fn new(params: RiskParams) -> Self {
        Self { params }
    }

    /// Расчёт размера позиции на основе фиксированного риска
    pub fn calculate_position_size(
        &self,
        capital: f64,
        entry_price: f64,
        stop_loss_price: f64,
    ) -> f64 {
        let risk_amount = capital * self.params.max_risk_per_trade;
        let price_risk = (entry_price - stop_loss_price).abs();

        if price_risk > 0.0 {
            risk_amount / price_risk
        } else {
            0.0
        }
    }

    /// Расчёт уровня тейк-профита на основе risk/reward
    pub fn calculate_take_profit(&self, entry_price: f64, stop_loss_price: f64, is_long: bool) -> f64 {
        let risk = (entry_price - stop_loss_price).abs();
        let reward = risk * self.params.risk_reward_ratio;

        if is_long {
            entry_price + reward
        } else {
            entry_price - reward
        }
    }

    /// Критерий Келли для оптимального размера ставки
    pub fn kelly_criterion(win_rate: f64, avg_win: f64, avg_loss: f64) -> f64 {
        if avg_loss == 0.0 {
            return 0.0;
        }

        let win_loss_ratio = avg_win / avg_loss;
        let kelly = win_rate - (1.0 - win_rate) / win_loss_ratio;

        kelly.max(0.0).min(1.0)
    }
}

/// Стоп-лосс менеджер
#[derive(Debug)]
pub struct StopLossManager {
    atr: ATR,
    atr_multiplier: f64,
}

impl StopLossManager {
    pub fn new(atr_period: usize, atr_multiplier: f64) -> Self {
        Self {
            atr: ATR::new(atr_period),
            atr_multiplier,
        }
    }

    pub fn update(&mut self, candle: &Candle) {
        self.atr.update(candle);
    }

    /// Динамический стоп-лосс на основе ATR
    pub fn calculate_stop_loss(&self, entry_price: f64, is_long: bool) -> Option<f64> {
        self.atr.value().as_single().map(|atr| {
            let distance = atr * self.atr_multiplier;
            if is_long {
                entry_price - distance
            } else {
                entry_price + distance
            }
        })
    }

    /// Трейлинг стоп на основе ATR
    pub fn trailing_stop(&self, current_price: f64, current_stop: f64, is_long: bool) -> Option<f64> {
        self.atr.value().as_single().map(|atr| {
            let distance = atr * self.atr_multiplier;
            if is_long {
                let new_stop = current_price - distance;
                new_stop.max(current_stop)
            } else {
                let new_stop = current_price + distance;
                new_stop.min(current_stop)
            }
        })
    }
}
```

## Метрики эффективности

```rust
use std::collections::VecDeque;

/// Результат сделки
#[derive(Debug, Clone)]
pub struct TradeResult {
    pub entry_price: f64,
    pub exit_price: f64,
    pub quantity: f64,
    pub is_long: bool,
    pub entry_time: Timestamp,
    pub exit_time: Timestamp,
}

impl TradeResult {
    pub fn pnl(&self) -> f64 {
        let diff = self.exit_price - self.entry_price;
        if self.is_long {
            diff * self.quantity
        } else {
            -diff * self.quantity
        }
    }

    pub fn pnl_percent(&self) -> f64 {
        let diff = self.exit_price - self.entry_price;
        if self.is_long {
            diff / self.entry_price * 100.0
        } else {
            -diff / self.entry_price * 100.0
        }
    }

    pub fn is_winner(&self) -> bool {
        self.pnl() > 0.0
    }
}

/// Калькулятор метрик стратегии
#[derive(Debug)]
pub struct PerformanceMetrics {
    trades: Vec<TradeResult>,
    equity_curve: Vec<f64>,
    initial_capital: f64,
}

impl PerformanceMetrics {
    pub fn new(initial_capital: f64) -> Self {
        Self {
            trades: Vec::new(),
            equity_curve: vec![initial_capital],
            initial_capital,
        }
    }

    pub fn add_trade(&mut self, trade: TradeResult) {
        let current_equity = *self.equity_curve.last().unwrap_or(&self.initial_capital);
        self.equity_curve.push(current_equity + trade.pnl());
        self.trades.push(trade);
    }

    /// Общее количество сделок
    pub fn total_trades(&self) -> usize {
        self.trades.len()
    }

    /// Количество прибыльных сделок
    pub fn winning_trades(&self) -> usize {
        self.trades.iter().filter(|t| t.is_winner()).count()
    }

    /// Количество убыточных сделок
    pub fn losing_trades(&self) -> usize {
        self.trades.iter().filter(|t| !t.is_winner()).count()
    }

    /// Процент прибыльных сделок
    pub fn win_rate(&self) -> f64 {
        if self.trades.is_empty() {
            return 0.0;
        }
        self.winning_trades() as f64 / self.trades.len() as f64 * 100.0
    }

    /// Общая прибыль
    pub fn total_pnl(&self) -> f64 {
        self.trades.iter().map(|t| t.pnl()).sum()
    }

    /// Средняя прибыль на сделку
    pub fn average_pnl(&self) -> f64 {
        if self.trades.is_empty() {
            return 0.0;
        }
        self.total_pnl() / self.trades.len() as f64
    }

    /// Средняя прибыль в прибыльных сделках
    pub fn average_win(&self) -> f64 {
        let wins: Vec<f64> = self.trades.iter()
            .filter(|t| t.is_winner())
            .map(|t| t.pnl())
            .collect();

        if wins.is_empty() {
            0.0
        } else {
            wins.iter().sum::<f64>() / wins.len() as f64
        }
    }

    /// Средний убыток в убыточных сделках
    pub fn average_loss(&self) -> f64 {
        let losses: Vec<f64> = self.trades.iter()
            .filter(|t| !t.is_winner())
            .map(|t| t.pnl().abs())
            .collect();

        if losses.is_empty() {
            0.0
        } else {
            losses.iter().sum::<f64>() / losses.len() as f64
        }
    }

    /// Profit Factor = Gross Profit / Gross Loss
    pub fn profit_factor(&self) -> f64 {
        let gross_profit: f64 = self.trades.iter()
            .filter(|t| t.is_winner())
            .map(|t| t.pnl())
            .sum();

        let gross_loss: f64 = self.trades.iter()
            .filter(|t| !t.is_winner())
            .map(|t| t.pnl().abs())
            .sum();

        if gross_loss == 0.0 {
            f64::INFINITY
        } else {
            gross_profit / gross_loss
        }
    }

    /// Максимальная просадка
    pub fn max_drawdown(&self) -> f64 {
        let mut max_equity = self.initial_capital;
        let mut max_drawdown = 0.0;

        for &equity in &self.equity_curve {
            if equity > max_equity {
                max_equity = equity;
            }
            let drawdown = (max_equity - equity) / max_equity;
            if drawdown > max_drawdown {
                max_drawdown = drawdown;
            }
        }

        max_drawdown * 100.0
    }

    /// Коэффициент Шарпа (упрощённый, без безрисковой ставки)
    pub fn sharpe_ratio(&self, periods_per_year: f64) -> f64 {
        if self.trades.len() < 2 {
            return 0.0;
        }

        let returns: Vec<f64> = self.trades.iter().map(|t| t.pnl_percent()).collect();
        let mean_return = returns.iter().sum::<f64>() / returns.len() as f64;

        let variance: f64 = returns.iter()
            .map(|r| (r - mean_return).powi(2))
            .sum::<f64>() / returns.len() as f64;

        let std_dev = variance.sqrt();

        if std_dev == 0.0 {
            0.0
        } else {
            (mean_return / std_dev) * (periods_per_year).sqrt()
        }
    }

    /// Текущий капитал
    pub fn current_equity(&self) -> f64 {
        *self.equity_curve.last().unwrap_or(&self.initial_capital)
    }

    /// Общая доходность в процентах
    pub fn total_return(&self) -> f64 {
        (self.current_equity() - self.initial_capital) / self.initial_capital * 100.0
    }

    /// Вывод отчёта
    pub fn print_report(&self) {
        println!("═══════════════════════════════════════");
        println!("        ОТЧЁТ О ПРОИЗВОДИТЕЛЬНОСТИ      ");
        println!("═══════════════════════════════════════");
        println!("Начальный капитал:    ${:.2}", self.initial_capital);
        println!("Текущий капитал:      ${:.2}", self.current_equity());
        println!("Общая доходность:     {:.2}%", self.total_return());
        println!("───────────────────────────────────────");
        println!("Всего сделок:         {}", self.total_trades());
        println!("Прибыльных:           {}", self.winning_trades());
        println!("Убыточных:            {}", self.losing_trades());
        println!("Win Rate:             {:.2}%", self.win_rate());
        println!("───────────────────────────────────────");
        println!("Общий PnL:            ${:.2}", self.total_pnl());
        println!("Средний PnL:          ${:.2}", self.average_pnl());
        println!("Средняя прибыль:      ${:.2}", self.average_win());
        println!("Средний убыток:       ${:.2}", self.average_loss());
        println!("───────────────────────────────────────");
        println!("Profit Factor:        {:.2}", self.profit_factor());
        println!("Max Drawdown:         {:.2}%", self.max_drawdown());
        println!("Sharpe Ratio (годовой): {:.2}", self.sharpe_ratio(252.0));
        println!("═══════════════════════════════════════");
    }
}
```

## Полный пример использования

```rust
fn main() {
    // Создаём индикаторы
    let mut sma_fast = SMA::new(10);
    let mut sma_slow = SMA::new(20);
    let mut rsi = RSI::new(14);
    let mut macd = MACD::standard();
    let mut bb = BollingerBands::standard();
    let mut atr = ATR::new(14);

    // Генерируем тестовые данные
    let candles = generate_test_candles(100);

    // Создаём стратегию и метрики
    let mut strategy = SMACrossover::new(10, 20);
    let mut metrics = PerformanceMetrics::new(10000.0);
    let position_sizer = PositionSizer::new(RiskParams::default());

    let mut position: Option<(f64, f64, bool)> = None; // (entry_price, quantity, is_long)

    // Симуляция торговли
    for candle in &candles {
        // Обновляем индикаторы
        sma_fast.update(candle);
        sma_slow.update(candle);
        rsi.update(candle);
        macd.update(candle);
        bb.update(candle);
        atr.update(candle);

        // Получаем сигнал
        if let Some(signal) = strategy.generate(candle) {
            match signal.direction {
                SignalDirection::Buy if position.is_none() => {
                    // Открываем длинную позицию
                    let stop_loss = candle.close * 0.98; // 2% стоп-лосс
                    let size = position_sizer.calculate_position_size(
                        metrics.current_equity(),
                        candle.close,
                        stop_loss,
                    );
                    position = Some((candle.close, size, true));
                    println!("[{}] BUY @ {:.2}, size: {:.4}", candle.timestamp, candle.close, size);
                }
                SignalDirection::Sell if position.is_some() => {
                    // Закрываем позицию
                    if let Some((entry, qty, is_long)) = position.take() {
                        let trade = TradeResult {
                            entry_price: entry,
                            exit_price: candle.close,
                            quantity: qty,
                            is_long,
                            entry_time: candle.timestamp - 1000,
                            exit_time: candle.timestamp,
                        };
                        println!("[{}] SELL @ {:.2}, PnL: {:.2}", candle.timestamp, candle.close, trade.pnl());
                        metrics.add_trade(trade);
                    }
                }
                _ => {}
            }
        }

        // Выводим состояние индикаторов
        if let (
            IndicatorValue::Single(fast),
            IndicatorValue::Single(slow),
            IndicatorValue::Single(rsi_val),
        ) = (sma_fast.value(), sma_slow.value(), rsi.value()) {
            if candle.timestamp % 10 == 0 {
                println!(
                    "[{}] Price: {:.2} | SMA10: {:.2} | SMA20: {:.2} | RSI: {:.1}",
                    candle.timestamp, candle.close, fast, slow, rsi_val
                );
            }
        }
    }

    // Закрываем открытую позицию
    if let Some((entry, qty, is_long)) = position {
        let last_candle = candles.last().unwrap();
        let trade = TradeResult {
            entry_price: entry,
            exit_price: last_candle.close,
            quantity: qty,
            is_long,
            entry_time: last_candle.timestamp - 1000,
            exit_time: last_candle.timestamp,
        };
        metrics.add_trade(trade);
    }

    // Выводим отчёт
    metrics.print_report();
}

/// Генерация тестовых свечей
fn generate_test_candles(count: usize) -> Vec<Candle> {
    let mut candles = Vec::with_capacity(count);
    let mut price = 100.0;

    for i in 0..count {
        // Добавляем тренд и шум
        let trend = (i as f64 * 0.01).sin() * 5.0;
        let noise = (i as f64 * 0.1).cos() * 2.0;
        price += trend + noise;
        price = price.max(50.0);

        let volatility = 2.0;
        let open = price;
        let close = price + (i as f64 * 0.05).sin() * volatility;
        let high = open.max(close) + volatility.abs();
        let low = open.min(close) - volatility.abs();
        let volume = 1000.0 + (i as f64 * 0.2).cos() * 500.0;

        candles.push(Candle::new(i as u64, open, high, low, close, volume));
    }

    candles
}
```

## Что мы изучили

| Концепция | Описание |
|-----------|----------|
| Trait Indicator | Единый интерфейс для всех индикаторов |
| IndicatorValue | Универсальное представление значений индикаторов |
| BatchIndicator | Расчёт индикаторов на массиве данных |
| SignalGenerator | Генерация торговых сигналов |
| CompositeStrategy | Комбинирование нескольких стратегий |
| PositionSizer | Расчёт размера позиции на основе риска |
| PerformanceMetrics | Измерение эффективности стратегии |

## Домашнее задание

1. **Добавь новый индикатор**: Реализуй Stochastic Oscillator, следуя паттерну trait Indicator. Stochastic рассчитывается как:
   ```
   %K = (Close - Lowest Low) / (Highest High - Lowest Low) * 100
   %D = SMA(%K, 3)
   ```

2. **Создай стратегию Bollinger Breakout**: Генерируй сигнал на покупку, когда цена пробивает верхнюю полосу Боллинджера с увеличенным объёмом, и на продажу — когда пробивает нижнюю.

3. **Реализуй трейлинг-стоп**: Добавь в симуляцию торговли логику трейлинг-стопа на основе ATR. Стоп должен следовать за ценой, но никогда не двигаться против позиции.

4. **Добавь метрику Sortino Ratio**: Sortino Ratio похож на Sharpe, но учитывает только отрицательную волатильность (downside deviation). Реализуй его в PerformanceMetrics.

5. **Создай мульти-таймфрейм анализ**: Используй один индикатор (например, RSI) на разных таймфреймах (свечи разной длины) и генерируй сигнал только когда все таймфреймы согласуются.

## Навигация

[← День 273: Sharpe Ratio](../273-sharpe-ratio/ru.md) | [День 275: Что такое бэктестинг →](../275-what-is-backtesting/ru.md)
