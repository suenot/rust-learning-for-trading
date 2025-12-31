# День 271: Волатильность: измерение волатильности

## Аналогия из трейдинга

Представь, что ты моряк, планирующий плавание. Перед тем как отправиться в путь, ты хочешь знать, насколько бурным было море в последнее время. Спокойное море с небольшими предсказуемыми волнами сильно отличается от штормового океана с огромными непредсказуемыми волнами. **Волатильность** в трейдинге — это именно это: она измеряет, насколько «бурным» является рынок для конкретного актива.

Акции коммунальной компании могут быть спокойным прудом — цены движутся медленно и предсказуемо. Криптовалюта может быть штормовым океаном — цены могут колебаться на 10% за один час. Понимание и измерение волатильности помогает трейдерам:
- Правильно определять размер позиций (меньшие позиции в бурных рынках)
- Устанавливать стоп-лоссы на разумных расстояниях
- Оценивать опционы и другие деривативы
- Находить торговые возможности (высокая волатильность = больше потенциальной прибыли, но и больше риска)

## Что такое волатильность?

**Волатильность** — это статистическая мера разброса доходности для данного актива. Проще говоря, она показывает, насколько сильно цена обычно отклоняется от своего среднего значения.

### Типы волатильности

1. **Историческая волатильность** — рассчитывается на основе прошлых ценовых данных
2. **Подразумеваемая волатильность** — выводится из цен опционов, отражает ожидания рынка
3. **Реализованная волатильность** — фактическая волатильность за определённый период

В этой главе мы сосредоточимся на **исторической волатильности**, так как она является основой для понимания рыночного риска.

## Математика волатильности

Наиболее распространённый способ измерения волатильности — через **стандартное отклонение** доходностей:

1. Рассчитываем доходности: `r_t = (P_t - P_{t-1}) / P_{t-1}` или `r_t = ln(P_t / P_{t-1})`
2. Рассчитываем среднюю доходность: `μ = sum(r) / n`
3. Рассчитываем дисперсию: `σ² = sum((r - μ)²) / (n-1)`
4. Волатильность — это стандартное отклонение: `σ = sqrt(σ²)`

Для приведения дневной волатильности к годовой: `σ_годовая = σ_дневная * sqrt(252)` (252 торговых дня в году)

## Базовый калькулятор волатильности на Rust

Построим калькулятор волатильности с нуля:

```rust
/// Представляет одну ценовую точку с временной меткой
#[derive(Debug, Clone)]
struct PricePoint {
    timestamp: u64,
    price: f64,
}

/// Результаты расчёта волатильности
#[derive(Debug)]
struct VolatilityMetrics {
    daily_volatility: f64,
    annualized_volatility: f64,
    mean_return: f64,
    max_return: f64,
    min_return: f64,
}

/// Рассчитываем логарифмические доходности из ценовых данных
fn calculate_returns(prices: &[f64]) -> Vec<f64> {
    if prices.len() < 2 {
        return vec![];
    }

    prices
        .windows(2)
        .map(|window| (window[1] / window[0]).ln())
        .collect()
}

/// Рассчитываем среднее значение среза
fn mean(values: &[f64]) -> f64 {
    if values.is_empty() {
        return 0.0;
    }
    values.iter().sum::<f64>() / values.len() as f64
}

/// Рассчитываем стандартное отклонение (выборочное)
fn std_deviation(values: &[f64]) -> f64 {
    if values.len() < 2 {
        return 0.0;
    }

    let avg = mean(values);
    let variance = values
        .iter()
        .map(|x| (x - avg).powi(2))
        .sum::<f64>()
        / (values.len() - 1) as f64;

    variance.sqrt()
}

/// Рассчитываем комплексные метрики волатильности
fn calculate_volatility(prices: &[f64]) -> Option<VolatilityMetrics> {
    let returns = calculate_returns(prices);

    if returns.is_empty() {
        return None;
    }

    let daily_vol = std_deviation(&returns);
    let annualized_vol = daily_vol * (252.0_f64).sqrt();

    Some(VolatilityMetrics {
        daily_volatility: daily_vol,
        annualized_volatility: annualized_vol,
        mean_return: mean(&returns),
        max_return: returns.iter().cloned().fold(f64::NEG_INFINITY, f64::max),
        min_return: returns.iter().cloned().fold(f64::INFINITY, f64::min),
    })
}

fn main() {
    // Симулированные дневные цены закрытия BTC
    let btc_prices = vec![
        42000.0, 42500.0, 41800.0, 43200.0, 44100.0,
        43800.0, 45000.0, 44200.0, 46000.0, 45500.0,
        47000.0, 46200.0, 48000.0, 47500.0, 49000.0,
    ];

    // Симулированные цены стабильной акции
    let stable_stock_prices = vec![
        100.0, 100.2, 100.1, 100.3, 100.4,
        100.3, 100.5, 100.4, 100.6, 100.5,
        100.7, 100.6, 100.8, 100.7, 100.9,
    ];

    println!("=== Анализ волатильности ===\n");

    if let Some(btc_vol) = calculate_volatility(&btc_prices) {
        println!("Bitcoin (BTC):");
        println!("  Дневная волатильность:  {:.4} ({:.2}%)",
            btc_vol.daily_volatility, btc_vol.daily_volatility * 100.0);
        println!("  Годовая волатильность:  {:.4} ({:.2}%)",
            btc_vol.annualized_volatility, btc_vol.annualized_volatility * 100.0);
        println!("  Средняя дневная доходность: {:.4} ({:.2}%)",
            btc_vol.mean_return, btc_vol.mean_return * 100.0);
        println!("  Макс. однодневная доходность: {:.2}%", btc_vol.max_return * 100.0);
        println!("  Мин. однодневная доходность: {:.2}%", btc_vol.min_return * 100.0);
    }

    println!();

    if let Some(stock_vol) = calculate_volatility(&stable_stock_prices) {
        println!("Стабильная акция:");
        println!("  Дневная волатильность:  {:.4} ({:.2}%)",
            stock_vol.daily_volatility, stock_vol.daily_volatility * 100.0);
        println!("  Годовая волатильность:  {:.4} ({:.2}%)",
            stock_vol.annualized_volatility, stock_vol.annualized_volatility * 100.0);
        println!("  Средняя дневная доходность: {:.4} ({:.2}%)",
            stock_vol.mean_return, stock_vol.mean_return * 100.0);
    }
}
```

## Скользящая волатильность

В реальной торговле мы часто хотим отслеживать, как волатильность меняется со временем, используя **скользящее окно**:

```rust
/// Рассчитываем скользящую волатильность с заданным размером окна
fn rolling_volatility(prices: &[f64], window_size: usize) -> Vec<f64> {
    if prices.len() < window_size + 1 {
        return vec![];
    }

    let returns = calculate_returns(prices);

    returns
        .windows(window_size)
        .map(|window| std_deviation(window))
        .collect()
}

/// Классификация режима волатильности
#[derive(Debug, Clone, PartialEq)]
enum VolatilityRegime {
    Low,      // Низкая
    Normal,   // Нормальная
    High,     // Высокая
    Extreme,  // Экстремальная
}

impl VolatilityRegime {
    fn from_volatility(vol: f64, historical_avg: f64) -> Self {
        let ratio = vol / historical_avg;
        match ratio {
            r if r < 0.5 => VolatilityRegime::Low,
            r if r < 1.5 => VolatilityRegime::Normal,
            r if r < 2.5 => VolatilityRegime::High,
            _ => VolatilityRegime::Extreme,
        }
    }

    fn position_size_multiplier(&self) -> f64 {
        match self {
            VolatilityRegime::Low => 1.5,      // Можно брать большие позиции
            VolatilityRegime::Normal => 1.0,   // Стандартный размер позиции
            VolatilityRegime::High => 0.5,     // Уменьшаем размер позиции
            VolatilityRegime::Extreme => 0.25, // Минимальные позиции
        }
    }
}

fn main() {
    let prices = vec![
        100.0, 102.0, 101.0, 103.0, 105.0, 104.0, 106.0, 108.0,
        107.0, 110.0, 108.0, 112.0, 115.0, 113.0, 118.0, 120.0,
        118.0, 122.0, 125.0, 123.0, 128.0, 130.0, 127.0, 132.0,
    ];

    let window = 5;
    let rolling_vol = rolling_volatility(&prices, window);

    println!("Скользящая {}-дневная волатильность:", window);
    for (i, vol) in rolling_vol.iter().enumerate() {
        let annualized = vol * (252.0_f64).sqrt();
        println!("  День {}: {:.4} (годовая: {:.2}%)",
            i + window + 1, vol, annualized * 100.0);
    }

    // Рассчитываем среднюю волатильность для определения режима
    let avg_vol = mean(&rolling_vol);
    println!("\nСредняя скользящая волатильность: {:.4}", avg_vol);

    // Классифицируем текущий режим
    if let Some(&current_vol) = rolling_vol.last() {
        let regime = VolatilityRegime::from_volatility(current_vol, avg_vol);
        println!("Текущий режим: {:?}", regime);
        println!("Рекомендуемый множитель размера позиции: {:.2}x",
            regime.position_size_multiplier());
    }
}
```

## Экспоненциально взвешенная скользящая средняя (EWMA) волатильность

EWMA придаёт больший вес недавним наблюдениям, делая её более чувствительной к текущим рыночным условиям:

```rust
/// Рассчитываем EWMA волатильность
/// lambda: коэффициент затухания (обычно 0.94 для дневных данных по RiskMetrics)
fn ewma_volatility(returns: &[f64], lambda: f64) -> Vec<f64> {
    if returns.is_empty() {
        return vec![];
    }

    let mut ewma_var = vec![0.0; returns.len()];

    // Инициализируем первым квадратом доходности
    ewma_var[0] = returns[0].powi(2);

    // Рассчитываем EWMA дисперсию рекурсивно
    for i in 1..returns.len() {
        ewma_var[i] = lambda * ewma_var[i - 1] + (1.0 - lambda) * returns[i].powi(2);
    }

    // Возвращаем волатильность (корень из дисперсии)
    ewma_var.iter().map(|v| v.sqrt()).collect()
}

/// Сравниваем различные методы оценки волатильности
fn compare_volatility_methods(prices: &[f64]) {
    let returns = calculate_returns(prices);

    if returns.len() < 5 {
        println!("Недостаточно данных для сравнения");
        return;
    }

    // Простое стандартное отклонение
    let simple_vol = std_deviation(&returns);

    // EWMA с разными коэффициентами затухания
    let ewma_094 = ewma_volatility(&returns, 0.94);
    let ewma_097 = ewma_volatility(&returns, 0.97);

    println!("Сравнение методов расчёта волатильности:");
    println!("  Простое стд. откл.:    {:.4}", simple_vol);
    println!("  EWMA (λ=0.94):         {:.4}", ewma_094.last().unwrap_or(&0.0));
    println!("  EWMA (λ=0.97):         {:.4}", ewma_097.last().unwrap_or(&0.0));
}

fn main() {
    // Симулированные цены со всплеском волатильности в середине
    let prices: Vec<f64> = vec![
        100.0, 101.0, 100.5, 101.5, 102.0,  // Спокойный период
        102.5, 108.0, 103.0, 110.0, 105.0,  // Волатильный период
        106.0, 106.5, 107.0, 107.5, 108.0,  // Снова спокойный
    ];

    compare_volatility_methods(&prices);

    let returns = calculate_returns(&prices);
    let ewma_vol = ewma_volatility(&returns, 0.94);

    println!("\nЭволюция EWMA волатильности:");
    for (i, vol) in ewma_vol.iter().enumerate() {
        let bar = "█".repeat((vol * 200.0) as usize);
        println!("  День {:2}: {:.4} {}", i + 2, vol, bar);
    }
}
```

## Практическое применение: размер позиции на основе волатильности

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct Asset {
    symbol: String,
    prices: Vec<f64>,
    current_price: f64,
}

#[derive(Debug)]
struct PositionSizer {
    total_capital: f64,
    risk_per_trade: f64,  // Как дробь (например, 0.02 для 2%)
    volatility_window: usize,
}

impl PositionSizer {
    fn new(capital: f64, risk_fraction: f64, window: usize) -> Self {
        PositionSizer {
            total_capital: capital,
            risk_per_trade: risk_fraction,
            volatility_window: window,
        }
    }

    /// Рассчитываем размер позиции на основе волатильности
    fn calculate_position(&self, asset: &Asset) -> Option<PositionResult> {
        if asset.prices.len() < self.volatility_window + 1 {
            return None;
        }

        let returns = calculate_returns(&asset.prices);
        let recent_returns: Vec<f64> = returns
            .iter()
            .rev()
            .take(self.volatility_window)
            .copied()
            .collect();

        let volatility = std_deviation(&recent_returns);
        let atr_estimate = volatility * asset.current_price;

        // Сумма риска в долларах
        let risk_amount = self.total_capital * self.risk_per_trade;

        // Размер позиции: risk_amount / (волатильность * цена)
        // Это означает, что мы рискуем одинаковой суммой независимо от волатильности
        let stop_distance = 2.0 * atr_estimate; // 2x волатильности для стопа
        let shares = risk_amount / stop_distance;
        let position_value = shares * asset.current_price;

        Some(PositionResult {
            symbol: asset.symbol.clone(),
            volatility,
            annualized_volatility: volatility * (252.0_f64).sqrt(),
            shares: shares.floor(),
            position_value,
            stop_loss_price: asset.current_price - stop_distance,
            risk_amount,
        })
    }
}

#[derive(Debug)]
struct PositionResult {
    symbol: String,
    volatility: f64,
    annualized_volatility: f64,
    shares: f64,
    position_value: f64,
    stop_loss_price: f64,
    risk_amount: f64,
}

fn main() {
    let sizer = PositionSizer::new(100_000.0, 0.02, 20);

    // Высоковолатильный актив (крипто)
    let btc = Asset {
        symbol: "BTC".to_string(),
        prices: vec![
            40000.0, 41000.0, 39500.0, 42000.0, 43500.0,
            42000.0, 44000.0, 43000.0, 45000.0, 44500.0,
            46000.0, 45000.0, 47000.0, 46500.0, 48000.0,
            47000.0, 49000.0, 48000.0, 50000.0, 49500.0,
            51000.0,
        ],
        current_price: 51000.0,
    };

    // Низковолатильный актив (стабильная акция)
    let stable = Asset {
        symbol: "STABLE".to_string(),
        prices: vec![
            100.0, 100.2, 100.1, 100.3, 100.4,
            100.3, 100.5, 100.4, 100.6, 100.5,
            100.7, 100.6, 100.8, 100.7, 100.9,
            100.8, 101.0, 100.9, 101.1, 101.0,
            101.2,
        ],
        current_price: 101.2,
    };

    println!("=== Размер позиции на основе волатильности ===");
    println!("Капитал: ${:.0}", sizer.total_capital);
    println!("Риск на сделку: {:.1}%\n", sizer.risk_per_trade * 100.0);

    for asset in [&btc, &stable] {
        if let Some(result) = sizer.calculate_position(asset) {
            println!("{}:", result.symbol);
            println!("  Текущая цена:          ${:.2}", asset.current_price);
            println!("  Дневная волатильность: {:.2}%", result.volatility * 100.0);
            println!("  Годовая волатильность: {:.2}%", result.annualized_volatility * 100.0);
            println!("  Рекомендуемые акции:   {:.0}", result.shares);
            println!("  Стоимость позиции:     ${:.2}", result.position_value);
            println!("  Цена стоп-лосса:       ${:.2}", result.stop_loss_price);
            println!("  Сумма риска:           ${:.2}", result.risk_amount);
            println!();
        }
    }
}
```

## Индикаторы волатильности для торговых сигналов

```rust
/// Расчёт полос Боллинджера
#[derive(Debug)]
struct BollingerBands {
    upper: f64,
    middle: f64,
    lower: f64,
    bandwidth: f64,
}

fn calculate_bollinger_bands(prices: &[f64], period: usize, num_std: f64) -> Option<BollingerBands> {
    if prices.len() < period {
        return None;
    }

    let recent: Vec<f64> = prices.iter().rev().take(period).copied().collect();
    let sma = mean(&recent);
    let std = std_deviation(&recent);

    let upper = sma + num_std * std;
    let lower = sma - num_std * std;
    let bandwidth = (upper - lower) / sma;

    Some(BollingerBands {
        upper,
        middle: sma,
        lower,
        bandwidth,
    })
}

/// Торговый сигнал на основе волатильности
#[derive(Debug, Clone)]
enum VolatilitySignal {
    Squeeze,         // Низкая волатильность, ожидается пробой
    Expansion,       // Высокая волатильность, тренд в процессе
    MeanReversion,   // Цена на экстремумах полос
    Neutral,         // Нейтрально
}

fn analyze_volatility_signal(
    current_price: f64,
    bands: &BollingerBands,
    historical_bandwidth: f64,
) -> VolatilitySignal {
    // Сжатие: ширина полос значительно ниже средней
    if bands.bandwidth < historical_bandwidth * 0.5 {
        return VolatilitySignal::Squeeze;
    }

    // Расширение: ширина полос значительно выше средней
    if bands.bandwidth > historical_bandwidth * 1.5 {
        return VolatilitySignal::Expansion;
    }

    // Возврат к среднему: цена на экстремумах полос
    let position = (current_price - bands.lower) / (bands.upper - bands.lower);
    if position > 0.95 || position < 0.05 {
        return VolatilitySignal::MeanReversion;
    }

    VolatilitySignal::Neutral
}

fn main() {
    let prices = vec![
        100.0, 102.0, 101.0, 103.0, 104.0, 103.5, 105.0, 104.5,
        106.0, 105.5, 107.0, 106.5, 108.0, 107.5, 109.0, 108.5,
        110.0, 109.5, 111.0, 110.5,
    ];

    let period = 10;
    let num_std = 2.0;

    // Рассчитываем полосы Боллинджера для каждой точки
    println!("Анализ полос Боллинджера (период={}, стд.откл.={}):\n", period, num_std);

    let mut bandwidths = Vec::new();

    for i in period..=prices.len() {
        let slice = &prices[..i];
        if let Some(bands) = calculate_bollinger_bands(slice, period, num_std) {
            bandwidths.push(bands.bandwidth);
            let current_price = prices[i - 1];

            println!("День {}:", i);
            println!("  Цена:    ${:.2}", current_price);
            println!("  Верхняя: ${:.2}", bands.upper);
            println!("  Средняя: ${:.2}", bands.middle);
            println!("  Нижняя:  ${:.2}", bands.lower);
            println!("  Ширина:  {:.4}", bands.bandwidth);

            if bandwidths.len() > 1 {
                let avg_bandwidth = mean(&bandwidths);
                let signal = analyze_volatility_signal(current_price, &bands, avg_bandwidth);
                println!("  Сигнал:  {:?}", signal);
            }
            println!();
        }
    }
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Волатильность | Статистическая мера разброса цен |
| Стандартное отклонение | Наиболее распространённая мера волатильности |
| Логарифмические доходности | `ln(P_t / P_{t-1})` — предпочтительны для финансовых расчётов |
| Аннуализация | Умножаем дневную волатильность на √252 для годовой оценки |
| Скользящая волатильность | Отслеживаем изменения волатильности во времени |
| EWMA | Придаёт больший вес недавним наблюдениям |
| Размер позиции | Корректируем размер позиции обратно пропорционально волатильности |
| Полосы Боллинджера | Ценовые каналы на основе волатильности |

## Упражнения

1. **Базовый калькулятор волатильности**: Реализуй функцию, которая принимает вектор цен и возвращает волатильность как простых, так и логарифмических доходностей. Протестируй её на минимум 3 разных активах.

2. **Сравнение волатильности**: Создай программу, которая сравнивает волатильность двух активов и определяет, какой из них более рискованный. Добавь визуализацию с помощью ASCII-графиков.

3. **Реализация ATR**: Реализуй индикатор Average True Range (ATR), который использует цены high, low и close:
   ```
   TR = max(high - low, |high - prev_close|, |low - prev_close|)
   ATR = rolling_mean(TR, period)
   ```

4. **Система оповещения о волатильности**: Построй систему, которая мониторит волатильность и генерирует оповещения, когда:
   - Волатильность превышает 2x от 30-дневного среднего
   - Волатильность падает ниже 0.5x от 30-дневного среднего
   - Волатильность изменяется более чем на 50% за один день

## Домашнее задание

1. **Волатильность Паркинсона**: Реализуй оценщик волатильности Паркинсона, который использует цены high и low вместо только close:
   ```
   σ² = (1/4ln(2)) * mean((ln(high/low))²)
   ```
   Сравни его результаты со стандартным отклонением на том же наборе данных.

2. **Стратегия таргетирования волатильности**: Создай торговую систему, которая:
   - Нацелена на определённую волатильность портфеля (например, 15% годовых)
   - Ежедневно корректирует размеры позиций на основе скользящей волатильности
   - Отслеживает реализованную волатильность, чтобы увидеть, насколько близко ты к цели

3. **Кластеризация волатильности**: Реализуй детектор кластеризации волатильности (эффект GARCH) — явление, при котором дни с высокой волатильностью обычно следуют за днями с высокой волатильностью. Рассчитай автокорреляцию квадратов доходностей.

4. **Мультиактивный дашборд волатильности**: Построй дашборд, который отслеживает волатильность для нескольких активов и показывает:
   - Текущую волатильность vs 30-дневное среднее
   - Персентиль волатильности (где текущая волатильность находится исторически)
   - Корреляцию между волатильностями активов
   - Предлагаемое распределение портфеля на основе обратного взвешивания по волатильности

## Навигация

[← Предыдущий день](../270-risk-metrics-foundations/ru.md) | [Следующий день →](../272-volatility-models/ru.md)
