# День 270: Корреляция активов

## Аналогия из трейдинга

Представь, что ты управляешь портфелем криптовалют. Ты заметил интересную закономерность: когда Bitcoin растёт, Ethereum обычно тоже растёт. Когда падает нефть, акции авиакомпаний часто идут вверх. Эта взаимосвязь между движением цен разных активов называется **корреляцией**.

Корреляция — это статистическая мера, показывающая, насколько синхронно движутся два актива:
- **+1** — идеальная положительная корреляция (активы движутся вместе)
- **0** — нет корреляции (движения независимы)
- **-1** — идеальная отрицательная корреляция (активы движутся противоположно)

В реальном трейдинге корреляция помогает:
- Диверсифицировать портфель (выбирать некоррелированные активы)
- Хеджировать риски (использовать отрицательно коррелированные активы)
- Находить арбитражные возможности (когда корреляция временно нарушается)
- Строить парный трейдинг (торговать спредом между коррелированными активами)

## Что такое корреляция Пирсона?

Коэффициент корреляции Пирсона — самый распространённый способ измерения линейной зависимости между двумя переменными:

```
r = Σ((x - x̄)(y - ȳ)) / √(Σ(x - x̄)² × Σ(y - ȳ)²)
```

Где:
- `x̄` и `ȳ` — средние значения рядов
- Числитель — ковариация
- Знаменатель — произведение стандартных отклонений

## Базовый расчёт корреляции в Rust

```rust
/// Вычисляет среднее значение вектора
fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

/// Вычисляет коэффициент корреляции Пирсона между двумя рядами
fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let n = x.len() as f64;
    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None; // Один из рядов константный
    }

    Some(covariance / denominator)
}

fn main() {
    // Дневные доходности BTC и ETH (в процентах)
    let btc_returns = vec![2.1, -1.5, 0.8, 3.2, -0.5, 1.2, -2.1, 0.3, 1.8, -1.0];
    let eth_returns = vec![2.8, -1.2, 1.1, 4.1, -0.3, 1.5, -1.8, 0.5, 2.2, -0.8];

    match pearson_correlation(&btc_returns, &eth_returns) {
        Some(corr) => {
            println!("Корреляция BTC/ETH: {:.4}", corr);

            if corr > 0.7 {
                println!("Высокая положительная корреляция — активы движутся вместе");
            } else if corr > 0.3 {
                println!("Умеренная положительная корреляция");
            } else if corr > -0.3 {
                println!("Слабая корреляция — активы относительно независимы");
            } else if corr > -0.7 {
                println!("Умеренная отрицательная корреляция");
            } else {
                println!("Высокая отрицательная корреляция — хороший инструмент для хеджирования");
            }
        }
        None => println!("Невозможно вычислить корреляцию"),
    }
}
```

## Структура для анализа корреляции портфеля

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Asset {
    symbol: String,
    returns: Vec<f64>,
}

#[derive(Debug)]
struct CorrelationMatrix {
    assets: Vec<String>,
    matrix: Vec<Vec<f64>>,
}

impl Asset {
    fn new(symbol: &str, prices: &[f64]) -> Self {
        // Конвертируем цены в доходности
        let returns = prices
            .windows(2)
            .map(|w| (w[1] - w[0]) / w[0] * 100.0)
            .collect();

        Asset {
            symbol: symbol.to_string(),
            returns,
        }
    }

    fn from_returns(symbol: &str, returns: Vec<f64>) -> Self {
        Asset {
            symbol: symbol.to_string(),
            returns,
        }
    }
}

impl CorrelationMatrix {
    fn calculate(assets: &[Asset]) -> Option<Self> {
        if assets.is_empty() {
            return None;
        }

        let n = assets.len();
        let mut matrix = vec![vec![0.0; n]; n];
        let symbols: Vec<String> = assets.iter().map(|a| a.symbol.clone()).collect();

        for i in 0..n {
            for j in 0..n {
                if i == j {
                    matrix[i][j] = 1.0; // Корреляция с самим собой = 1
                } else if i < j {
                    let corr = pearson_correlation(&assets[i].returns, &assets[j].returns)
                        .unwrap_or(0.0);
                    matrix[i][j] = corr;
                    matrix[j][i] = corr; // Матрица симметрична
                }
            }
        }

        Some(CorrelationMatrix {
            assets: symbols,
            matrix,
        })
    }

    fn print(&self) {
        // Заголовок
        print!("{:>10}", "");
        for symbol in &self.assets {
            print!("{:>10}", symbol);
        }
        println!();

        // Данные
        for (i, symbol) in self.assets.iter().enumerate() {
            print!("{:>10}", symbol);
            for j in 0..self.assets.len() {
                print!("{:>10.4}", self.matrix[i][j]);
            }
            println!();
        }
    }

    fn get_correlation(&self, asset1: &str, asset2: &str) -> Option<f64> {
        let i = self.assets.iter().position(|a| a == asset1)?;
        let j = self.assets.iter().position(|a| a == asset2)?;
        Some(self.matrix[i][j])
    }

    /// Находит пары с наименьшей корреляцией для диверсификации
    fn find_diversification_pairs(&self, threshold: f64) -> Vec<(String, String, f64)> {
        let mut pairs = Vec::new();
        let n = self.assets.len();

        for i in 0..n {
            for j in (i + 1)..n {
                if self.matrix[i][j].abs() < threshold {
                    pairs.push((
                        self.assets[i].clone(),
                        self.assets[j].clone(),
                        self.matrix[i][j],
                    ));
                }
            }
        }

        pairs.sort_by(|a, b| a.2.abs().partial_cmp(&b.2.abs()).unwrap());
        pairs
    }

    /// Находит пары для хеджирования (отрицательная корреляция)
    fn find_hedging_pairs(&self, threshold: f64) -> Vec<(String, String, f64)> {
        let mut pairs = Vec::new();
        let n = self.assets.len();

        for i in 0..n {
            for j in (i + 1)..n {
                if self.matrix[i][j] < -threshold {
                    pairs.push((
                        self.assets[i].clone(),
                        self.assets[j].clone(),
                        self.matrix[i][j],
                    ));
                }
            }
        }

        pairs.sort_by(|a, b| a.2.partial_cmp(&b.2).unwrap());
        pairs
    }
}

fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None;
    }

    Some(covariance / denominator)
}

fn main() {
    // Исторические цены активов (условные данные)
    let btc_prices = vec![
        42000.0, 42500.0, 41800.0, 43200.0, 44000.0,
        43500.0, 44200.0, 43800.0, 45000.0, 44500.0,
    ];

    let eth_prices = vec![
        2800.0, 2850.0, 2780.0, 2900.0, 2980.0,
        2920.0, 3000.0, 2950.0, 3100.0, 3050.0,
    ];

    let sol_prices = vec![
        100.0, 105.0, 98.0, 110.0, 115.0,
        108.0, 120.0, 112.0, 125.0, 118.0,
    ];

    let gold_prices = vec![
        1950.0, 1945.0, 1960.0, 1940.0, 1935.0,
        1955.0, 1930.0, 1965.0, 1920.0, 1970.0,
    ];

    // Создаём активы
    let assets = vec![
        Asset::new("BTC", &btc_prices),
        Asset::new("ETH", &eth_prices),
        Asset::new("SOL", &sol_prices),
        Asset::new("GOLD", &gold_prices),
    ];

    // Вычисляем матрицу корреляций
    if let Some(corr_matrix) = CorrelationMatrix::calculate(&assets) {
        println!("=== Матрица корреляций ===\n");
        corr_matrix.print();

        println!("\n=== Пары для диверсификации (|r| < 0.3) ===");
        for (a1, a2, corr) in corr_matrix.find_diversification_pairs(0.3) {
            println!("{}/{}: {:.4}", a1, a2, corr);
        }

        println!("\n=== Пары для хеджирования (r < -0.5) ===");
        for (a1, a2, corr) in corr_matrix.find_hedging_pairs(0.5) {
            println!("{}/{}: {:.4}", a1, a2, corr);
        }

        // Проверка конкретной пары
        if let Some(corr) = corr_matrix.get_correlation("BTC", "ETH") {
            println!("\nКорреляция BTC/ETH: {:.4}", corr);
        }
    }
}
```

## Скользящая корреляция

В реальном трейдинге корреляция между активами меняется со временем. Скользящая корреляция помогает отслеживать эти изменения:

```rust
#[derive(Debug)]
struct RollingCorrelation {
    window_size: usize,
    x_buffer: Vec<f64>,
    y_buffer: Vec<f64>,
}

impl RollingCorrelation {
    fn new(window_size: usize) -> Self {
        RollingCorrelation {
            window_size,
            x_buffer: Vec::with_capacity(window_size),
            y_buffer: Vec::with_capacity(window_size),
        }
    }

    fn update(&mut self, x: f64, y: f64) -> Option<f64> {
        self.x_buffer.push(x);
        self.y_buffer.push(y);

        // Удаляем старые данные, если буфер переполнен
        if self.x_buffer.len() > self.window_size {
            self.x_buffer.remove(0);
            self.y_buffer.remove(0);
        }

        // Вычисляем корреляцию только если буфер заполнен
        if self.x_buffer.len() == self.window_size {
            pearson_correlation(&self.x_buffer, &self.y_buffer)
        } else {
            None
        }
    }

    fn current_correlation(&self) -> Option<f64> {
        if self.x_buffer.len() == self.window_size {
            pearson_correlation(&self.x_buffer, &self.y_buffer)
        } else {
            None
        }
    }

    fn reset(&mut self) {
        self.x_buffer.clear();
        self.y_buffer.clear();
    }
}

fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None;
    }

    Some(covariance / denominator)
}

fn main() {
    // Симуляция потока доходностей BTC и ETH
    let btc_returns = vec![
        1.2, -0.8, 0.5, 2.1, -1.5, 0.3, 1.8, -0.4, 0.9, -1.2,
        2.5, -0.9, 0.7, 1.4, -2.0, 0.6, 1.1, -0.3, 0.8, -1.0,
    ];

    let eth_returns = vec![
        1.5, -0.6, 0.8, 2.8, -1.2, 0.5, 2.2, -0.2, 1.2, -0.9,
        3.0, -0.7, 1.0, 1.8, -1.5, 0.9, 1.4, -0.1, 1.1, -0.7,
    ];

    let mut rolling = RollingCorrelation::new(10); // Окно 10 периодов

    println!("=== Скользящая корреляция BTC/ETH (окно = 10) ===\n");
    println!("{:>6} {:>10} {:>10} {:>12}", "День", "BTC", "ETH", "Корреляция");
    println!("{}", "-".repeat(42));

    for (i, (btc, eth)) in btc_returns.iter().zip(eth_returns.iter()).enumerate() {
        let corr = rolling.update(*btc, *eth);

        match corr {
            Some(c) => println!(
                "{:>6} {:>10.2} {:>10.2} {:>12.4}",
                i + 1, btc, eth, c
            ),
            None => println!(
                "{:>6} {:>10.2} {:>10.2} {:>12}",
                i + 1, btc, eth, "недост. данных"
            ),
        }
    }
}
```

## Торговая стратегия на основе корреляции

```rust
use std::collections::VecDeque;

#[derive(Debug, Clone, Copy, PartialEq)]
enum Signal {
    Long,        // Открыть длинную позицию
    Short,       // Открыть короткую позицию
    Close,       // Закрыть позицию
    Hold,        // Держать текущую позицию
}

#[derive(Debug)]
struct PairsTradingStrategy {
    window_size: usize,
    entry_threshold: f64,      // Порог для входа (отклонение от нормы)
    exit_threshold: f64,       // Порог для выхода
    normal_correlation: f64,   // "Нормальная" корреляция пары
    x_buffer: VecDeque<f64>,
    y_buffer: VecDeque<f64>,
    in_position: bool,
    position_type: Option<Signal>,
}

impl PairsTradingStrategy {
    fn new(
        window_size: usize,
        entry_threshold: f64,
        exit_threshold: f64,
        normal_correlation: f64,
    ) -> Self {
        PairsTradingStrategy {
            window_size,
            entry_threshold,
            exit_threshold,
            normal_correlation,
            x_buffer: VecDeque::with_capacity(window_size),
            y_buffer: VecDeque::with_capacity(window_size),
            in_position: false,
            position_type: None,
        }
    }

    fn update(&mut self, x_return: f64, y_return: f64) -> Signal {
        self.x_buffer.push_back(x_return);
        self.y_buffer.push_back(y_return);

        if self.x_buffer.len() > self.window_size {
            self.x_buffer.pop_front();
            self.y_buffer.pop_front();
        }

        if self.x_buffer.len() < self.window_size {
            return Signal::Hold;
        }

        let x_vec: Vec<f64> = self.x_buffer.iter().cloned().collect();
        let y_vec: Vec<f64> = self.y_buffer.iter().cloned().collect();

        let current_corr = match pearson_correlation(&x_vec, &y_vec) {
            Some(c) => c,
            None => return Signal::Hold,
        };

        let deviation = current_corr - self.normal_correlation;

        if self.in_position {
            // Проверяем условие выхода
            if deviation.abs() < self.exit_threshold {
                self.in_position = false;
                self.position_type = None;
                return Signal::Close;
            }
            Signal::Hold
        } else {
            // Проверяем условие входа
            if deviation > self.entry_threshold {
                // Корреляция выше нормы — ожидаем возврат к норме
                self.in_position = true;
                self.position_type = Some(Signal::Short);
                Signal::Short
            } else if deviation < -self.entry_threshold {
                // Корреляция ниже нормы — ожидаем возврат к норме
                self.in_position = true;
                self.position_type = Some(Signal::Long);
                Signal::Long
            } else {
                Signal::Hold
            }
        }
    }

    fn get_current_correlation(&self) -> Option<f64> {
        if self.x_buffer.len() < self.window_size {
            return None;
        }
        let x_vec: Vec<f64> = self.x_buffer.iter().cloned().collect();
        let y_vec: Vec<f64> = self.y_buffer.iter().cloned().collect();
        pearson_correlation(&x_vec, &y_vec)
    }
}

fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

fn pearson_correlation(x: &[f64], y: &[f64]) -> Option<f64> {
    if x.len() != y.len() || x.len() < 2 {
        return None;
    }

    let mean_x = mean(x);
    let mean_y = mean(y);

    let mut covariance = 0.0;
    let mut var_x = 0.0;
    let mut var_y = 0.0;

    for i in 0..x.len() {
        let dx = x[i] - mean_x;
        let dy = y[i] - mean_y;
        covariance += dx * dy;
        var_x += dx * dx;
        var_y += dy * dy;
    }

    let denominator = (var_x * var_y).sqrt();
    if denominator == 0.0 {
        return None;
    }

    Some(covariance / denominator)
}

fn main() {
    // Стратегия парного трейдинга BTC/ETH
    let mut strategy = PairsTradingStrategy::new(
        10,    // Окно корреляции
        0.15,  // Порог входа (отклонение на 0.15 от нормы)
        0.05,  // Порог выхода
        0.85,  // Нормальная корреляция BTC/ETH
    );

    // Симуляция доходностей
    let btc_returns = vec![
        1.2, -0.8, 0.5, 2.1, -1.5, 0.3, 1.8, -0.4, 0.9, -1.2,
        2.5, -0.9, 0.7, 1.4, -2.0, 0.6, 1.1, -0.3, 0.8, -1.0,
        3.0, -1.5, 0.2, 1.0, -0.5,
    ];

    let eth_returns = vec![
        1.5, -0.6, 0.8, 2.8, -1.2, 0.5, 2.2, -0.2, 1.2, -0.9,
        1.0, 0.5, -0.3, 0.2, 0.8,  // Временное нарушение корреляции
        0.9, 1.4, -0.1, 1.1, -0.7,
        2.8, -1.2, 0.5, 1.3, -0.3,
    ];

    println!("=== Парный трейдинг BTC/ETH ===\n");
    println!("{:>4} {:>8} {:>8} {:>10} {:>10}",
             "День", "BTC", "ETH", "Корр.", "Сигнал");
    println!("{}", "-".repeat(46));

    for (i, (btc, eth)) in btc_returns.iter().zip(eth_returns.iter()).enumerate() {
        let signal = strategy.update(*btc, *eth);
        let corr = strategy.get_current_correlation();

        let corr_str = match corr {
            Some(c) => format!("{:.4}", c),
            None => "N/A".to_string(),
        };

        let signal_str = match signal {
            Signal::Long => "LONG",
            Signal::Short => "SHORT",
            Signal::Close => "CLOSE",
            Signal::Hold => "HOLD",
        };

        println!(
            "{:>4} {:>8.2} {:>8.2} {:>10} {:>10}",
            i + 1, btc, eth, corr_str, signal_str
        );
    }
}
```

## Визуализация корреляций

```rust
/// Выводит тепловую карту корреляций в текстовом виде
fn print_correlation_heatmap(matrix: &[Vec<f64>], symbols: &[String]) {
    println!("\n=== Тепловая карта корреляций ===\n");

    // Символы для визуализации уровней корреляции
    fn corr_to_symbol(corr: f64) -> &'static str {
        if corr > 0.8 { "██" }
        else if corr > 0.6 { "▓▓" }
        else if corr > 0.3 { "▒▒" }
        else if corr > -0.3 { "░░" }
        else if corr > -0.6 { "▒▒" }
        else if corr > -0.8 { "▓▓" }
        else { "██" }
    }

    fn corr_to_sign(corr: f64) -> &'static str {
        if corr > 0.3 { "+" }
        else if corr < -0.3 { "-" }
        else { " " }
    }

    // Заголовок
    print!("{:>8}", "");
    for symbol in symbols {
        print!("{:>6}", symbol);
    }
    println!();

    // Данные
    for (i, symbol) in symbols.iter().enumerate() {
        print!("{:>8}", symbol);
        for j in 0..symbols.len() {
            let corr = matrix[i][j];
            print!(" {}{}", corr_to_sign(corr), corr_to_symbol(corr));
        }
        println!();
    }

    // Легенда
    println!("\nЛегенда:");
    println!("  +██ : сильная положительная (> 0.8)");
    println!("  +▓▓ : умеренная положительная (0.6 - 0.8)");
    println!("  +▒▒ : слабая положительная (0.3 - 0.6)");
    println!("   ░░ : нет корреляции (-0.3 - 0.3)");
    println!("  -▒▒ : слабая отрицательная (-0.6 - -0.3)");
    println!("  -▓▓ : умеренная отрицательная (-0.8 - -0.6)");
    println!("  -██ : сильная отрицательная (< -0.8)");
}

fn main() {
    // Пример матрицы корреляций
    let symbols = vec![
        "BTC".to_string(),
        "ETH".to_string(),
        "SOL".to_string(),
        "GOLD".to_string(),
        "USD".to_string(),
    ];

    let matrix = vec![
        vec![1.00,  0.85,  0.75,  0.10, -0.20],  // BTC
        vec![0.85,  1.00,  0.80,  0.05, -0.15],  // ETH
        vec![0.75,  0.80,  1.00, -0.10, -0.25],  // SOL
        vec![0.10,  0.05, -0.10,  1.00,  0.30],  // GOLD
        vec![-0.20, -0.15, -0.25, 0.30,  1.00],  // USD
    ];

    print_correlation_heatmap(&matrix, &symbols);
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Корреляция | Статистическая мера взаимосвязи между активами |
| Коэффициент Пирсона | Значение от -1 до +1, измеряющее линейную зависимость |
| Матрица корреляций | Таблица корреляций между всеми парами активов |
| Скользящая корреляция | Корреляция, вычисляемая на скользящем окне данных |
| Диверсификация | Использование некоррелированных активов для снижения риска |
| Хеджирование | Использование отрицательно коррелированных активов |
| Парный трейдинг | Стратегия торговли на отклонениях корреляции от нормы |

## Домашнее задание

1. **Корреляция Спирмена**: Реализуй функцию расчёта корреляции Спирмена (ранговая корреляция). Она более устойчива к выбросам. Сравни результаты с корреляцией Пирсона на данных с экстремальными значениями.

2. **Детектор смены режима**: Создай структуру `CorrelationRegimeDetector`, которая:
   - Отслеживает скользящую корреляцию между двумя активами
   - Определяет "нормальный" диапазон корреляции
   - Генерирует сигнал при выходе корреляции за пределы нормы
   - Поддерживает несколько режимов: "нормальный", "высокая корреляция", "низкая корреляция"

3. **Оптимизатор портфеля**: Реализуй функцию `optimize_portfolio`, которая:
   - Принимает список активов и их исторические доходности
   - Вычисляет матрицу корреляций
   - Находит комбинацию активов с минимальной средней корреляцией
   - Возвращает веса активов для диверсифицированного портфеля

4. **Мониторинг корреляции в реальном времени**: Создай структуру `CorrelationMonitor` с методами:
   - `add_asset(symbol, prices)` — добавление актива
   - `update_price(symbol, price)` — обновление цены
   - `get_correlation_matrix()` — текущая матрица корреляций
   - `get_alerts()` — оповещения о значительных изменениях корреляций

## Навигация

[← Предыдущий день](../269-portfolio-variance/ru.md) | [Следующий день →](../271-covariance-matrix/ru.md)
