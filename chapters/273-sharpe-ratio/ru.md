# День 273: Коэффициент Шарпа: Доходность с поправкой на риск

## Аналогия из трейдинга

Представь двух трейдеров: Трейдер А заработал 50% за год, а Трейдер Б — только 30%. На первый взгляд, Трейдер А лучше. Но подожди — портфель Трейдера А скакал от -20% до +80% ежемесячно, не давая спать по ночам. Доходность Трейдера Б была стабильной, от +2% до +4% в месяц. Кто из них лучший трейдер?

Здесь на помощь приходит **коэффициент Шарпа**. Названный в честь нобелевского лауреата Уильяма Шарпа, этот показатель измеряет **доходность с поправкой на риск** — сколько доходности вы получаете на единицу принятого риска. Это как сравнивать две машины не только по максимальной скорости, но и по плавности движения.

В реальном трейдинге коэффициент Шарпа помогает:
- Сравнивать стратегии с разными профилями риска
- Оценивать, стоит ли дополнительный риск дополнительной доходности
- Строить портфели, максимизирующие доходность на единицу риска

## Что такое коэффициент Шарпа?

Формула коэффициента Шарпа:

```
Коэффициент Шарпа = (Rp - Rf) / σp
```

Где:
- **Rp** = Доходность портфеля (средняя доходность вашей стратегии)
- **Rf** = Безрисковая ставка (например, казначейские облигации, обычно 2-5% годовых)
- **σp** = Стандартное отклонение доходности портфеля (волатильность/риск)

### Интерпретация значений коэффициента Шарпа

| Коэффициент Шарпа | Интерпретация |
|-------------------|---------------|
| < 0 | Стратегия убыточна или хуже безрисковой ставки |
| 0 - 1.0 | Неоптимальная доходность с поправкой на риск |
| 1.0 - 2.0 | Хорошо — приемлемо для большинства стратегий |
| 2.0 - 3.0 | Очень хорошо — отличная риск-доходность |
| > 3.0 | Исключительно — редко и требует проверки |

## Базовая реализация коэффициента Шарпа

```rust
/// Вычислить среднее значение для среза f64
fn mean(data: &[f64]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    data.iter().sum::<f64>() / data.len() as f64
}

/// Вычислить стандартное отклонение для среза f64
fn std_dev(data: &[f64]) -> f64 {
    if data.len() < 2 {
        return 0.0;
    }

    let avg = mean(data);
    let variance = data.iter()
        .map(|x| (x - avg).powi(2))
        .sum::<f64>() / (data.len() - 1) as f64;

    variance.sqrt()
}

/// Вычислить коэффициент Шарпа
///
/// # Аргументы
/// * `returns` - Срез периодических доходностей (например, дневных, месячных)
/// * `risk_free_rate` - Безрисковая ставка за тот же период
///
/// # Возвращает
/// Коэффициент Шарпа или 0.0, если расчёт невозможен
fn sharpe_ratio(returns: &[f64], risk_free_rate: f64) -> f64 {
    if returns.len() < 2 {
        return 0.0;
    }

    // Вычисляем избыточную доходность (доходность выше безрисковой ставки)
    let excess_returns: Vec<f64> = returns.iter()
        .map(|r| r - risk_free_rate)
        .collect();

    let avg_excess = mean(&excess_returns);
    let volatility = std_dev(&excess_returns);

    if volatility == 0.0 {
        return 0.0;
    }

    avg_excess / volatility
}

fn main() {
    // Пример: месячные доходности торговой стратегии
    let monthly_returns = vec![
        0.02, 0.03, -0.01, 0.04, 0.01, 0.02,
        0.03, -0.02, 0.05, 0.02, 0.01, 0.03
    ];

    // Месячная безрисковая ставка (годовые 3% / 12 месяцев)
    let monthly_rf = 0.03 / 12.0;

    let sharpe = sharpe_ratio(&monthly_returns, monthly_rf);

    println!("Анализ месячных доходностей:");
    println!("  Средняя доходность: {:.2}%", mean(&monthly_returns) * 100.0);
    println!("  Волатильность (ст. откл.): {:.2}%", std_dev(&monthly_returns) * 100.0);
    println!("  Коэффициент Шарпа: {:.2}", sharpe);

    // Аннуализируем коэффициент Шарпа (умножаем на корень из периодов в году)
    let annualized_sharpe = sharpe * (12.0_f64).sqrt();
    println!("  Годовой коэффициент Шарпа: {:.2}", annualized_sharpe);
}
```

## Сравнение торговых стратегий

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct TradingStrategy {
    name: String,
    returns: Vec<f64>,
}

impl TradingStrategy {
    fn new(name: &str, returns: Vec<f64>) -> Self {
        TradingStrategy {
            name: name.to_string(),
            returns,
        }
    }

    fn mean_return(&self) -> f64 {
        if self.returns.is_empty() {
            return 0.0;
        }
        self.returns.iter().sum::<f64>() / self.returns.len() as f64
    }

    fn volatility(&self) -> f64 {
        if self.returns.len() < 2 {
            return 0.0;
        }
        let avg = self.mean_return();
        let variance = self.returns.iter()
            .map(|x| (x - avg).powi(2))
            .sum::<f64>() / (self.returns.len() - 1) as f64;
        variance.sqrt()
    }

    fn sharpe_ratio(&self, risk_free_rate: f64) -> f64 {
        let volatility = self.volatility();
        if volatility == 0.0 {
            return 0.0;
        }
        (self.mean_return() - risk_free_rate) / volatility
    }

    fn total_return(&self) -> f64 {
        self.returns.iter()
            .fold(1.0, |acc, r| acc * (1.0 + r)) - 1.0
    }

    fn max_drawdown(&self) -> f64 {
        let mut peak = 1.0;
        let mut max_dd = 0.0;
        let mut value = 1.0;

        for r in &self.returns {
            value *= 1.0 + r;
            if value > peak {
                peak = value;
            }
            let drawdown = (peak - value) / peak;
            if drawdown > max_dd {
                max_dd = drawdown;
            }
        }
        max_dd
    }
}

fn main() {
    // Определяем три различные торговые стратегии
    let strategies = vec![
        TradingStrategy::new("Моментум", vec![
            0.05, 0.08, -0.03, 0.06, -0.02, 0.10,
            -0.04, 0.07, 0.03, -0.05, 0.09, 0.04
        ]),
        TradingStrategy::new("Возврат к среднему", vec![
            0.02, 0.03, 0.01, 0.02, 0.03, 0.01,
            0.02, 0.02, 0.03, 0.01, 0.02, 0.03
        ]),
        TradingStrategy::new("Следование тренду", vec![
            0.01, 0.02, 0.04, 0.06, 0.08, 0.03,
            -0.02, -0.03, 0.05, 0.07, 0.04, 0.02
        ]),
    ];

    let monthly_rf = 0.03 / 12.0; // 3% годовая безрисковая ставка

    println!("Сравнение стратегий:");
    println!("{:-<70}", "");
    println!("{:<20} {:>10} {:>12} {:>12} {:>12}",
             "Стратегия", "Доход", "Волат.", "Шарп", "Макс. DD");
    println!("{:-<70}", "");

    for strategy in &strategies {
        let sharpe = strategy.sharpe_ratio(monthly_rf);
        let annualized_sharpe = sharpe * (12.0_f64).sqrt();

        println!("{:<20} {:>9.1}% {:>11.1}% {:>12.2} {:>11.1}%",
                 strategy.name,
                 strategy.total_return() * 100.0,
                 strategy.volatility() * 100.0,
                 annualized_sharpe,
                 strategy.max_drawdown() * 100.0);
    }
    println!("{:-<70}", "");

    // Находим лучшую стратегию по коэффициенту Шарпа
    let best = strategies.iter()
        .max_by(|a, b| {
            a.sharpe_ratio(monthly_rf)
                .partial_cmp(&b.sharpe_ratio(monthly_rf))
                .unwrap()
        })
        .unwrap();

    println!("\nЛучшая стратегия по риск-доходности: {}", best.name);
}
```

## Скользящий коэффициент Шарпа для анализа в реальном времени

```rust
use std::collections::VecDeque;

struct RollingSharpe {
    window_size: usize,
    returns: VecDeque<f64>,
    risk_free_rate: f64,
}

impl RollingSharpe {
    fn new(window_size: usize, risk_free_rate: f64) -> Self {
        RollingSharpe {
            window_size,
            returns: VecDeque::with_capacity(window_size),
            risk_free_rate,
        }
    }

    fn add_return(&mut self, ret: f64) {
        if self.returns.len() >= self.window_size {
            self.returns.pop_front();
        }
        self.returns.push_back(ret);
    }

    fn calculate(&self) -> Option<f64> {
        if self.returns.len() < 2 {
            return None;
        }

        let returns_vec: Vec<f64> = self.returns.iter().cloned().collect();

        // Вычисляем избыточную доходность
        let excess: Vec<f64> = returns_vec.iter()
            .map(|r| r - self.risk_free_rate)
            .collect();

        let mean = excess.iter().sum::<f64>() / excess.len() as f64;
        let variance = excess.iter()
            .map(|x| (x - mean).powi(2))
            .sum::<f64>() / (excess.len() - 1) as f64;
        let std_dev = variance.sqrt();

        if std_dev == 0.0 {
            return None;
        }

        Some(mean / std_dev)
    }

    fn is_ready(&self) -> bool {
        self.returns.len() >= self.window_size
    }
}

fn main() {
    // Симулируем дневные доходности от торгового бота
    let daily_returns = vec![
        0.005, 0.008, -0.003, 0.012, -0.002, 0.007, 0.003,
        -0.004, 0.009, 0.002, -0.006, 0.011, 0.004, -0.001,
        0.006, 0.003, -0.008, 0.010, 0.005, -0.003, 0.007,
        0.002, 0.008, -0.002, 0.006, 0.004, -0.005, 0.009,
    ];

    // 10-дневный скользящий Шарп с дневной безрисковой ставкой
    let daily_rf = 0.03 / 252.0; // 252 торговых дня
    let mut rolling = RollingSharpe::new(10, daily_rf);

    println!("Скользящий 10-дневный коэффициент Шарпа:");
    println!("{:-<40}", "");

    for (day, &ret) in daily_returns.iter().enumerate() {
        rolling.add_return(ret);

        if rolling.is_ready() {
            if let Some(sharpe) = rolling.calculate() {
                let status = if sharpe > 2.0 {
                    "Отлично"
                } else if sharpe > 1.0 {
                    "Хорошо"
                } else if sharpe > 0.0 {
                    "Неоптимально"
                } else {
                    "Плохо"
                };

                println!("День {:2}: Шарп = {:>6.2} ({})",
                         day + 1, sharpe, status);
            }
        }
    }
}
```

## Оптимизация портфеля с использованием коэффициента Шарпа

```rust
use std::f64::consts::PI;

#[derive(Debug, Clone)]
struct Asset {
    name: String,
    expected_return: f64,  // Ожидаемая годовая доходность
    volatility: f64,       // Годовая волатильность (ст. откл.)
}

#[derive(Debug)]
struct Portfolio {
    assets: Vec<Asset>,
    weights: Vec<f64>,
    correlation_matrix: Vec<Vec<f64>>,
}

impl Portfolio {
    fn new(assets: Vec<Asset>, correlation_matrix: Vec<Vec<f64>>) -> Self {
        let n = assets.len();
        let weights = vec![1.0 / n as f64; n]; // Равные веса изначально
        Portfolio {
            assets,
            weights,
            correlation_matrix,
        }
    }

    fn set_weights(&mut self, weights: Vec<f64>) {
        assert_eq!(weights.len(), self.assets.len());
        let sum: f64 = weights.iter().sum();
        self.weights = weights.iter().map(|w| w / sum).collect();
    }

    fn expected_return(&self) -> f64 {
        self.assets.iter()
            .zip(self.weights.iter())
            .map(|(asset, weight)| asset.expected_return * weight)
            .sum()
    }

    fn portfolio_volatility(&self) -> f64 {
        let n = self.assets.len();
        let mut variance = 0.0;

        for i in 0..n {
            for j in 0..n {
                variance += self.weights[i] * self.weights[j]
                    * self.assets[i].volatility * self.assets[j].volatility
                    * self.correlation_matrix[i][j];
            }
        }

        variance.sqrt()
    }

    fn sharpe_ratio(&self, risk_free_rate: f64) -> f64 {
        let vol = self.portfolio_volatility();
        if vol == 0.0 {
            return 0.0;
        }
        (self.expected_return() - risk_free_rate) / vol
    }
}

fn optimize_portfolio(assets: Vec<Asset>, correlation: Vec<Vec<f64>>,
                      risk_free_rate: f64) -> (Vec<f64>, f64) {
    let mut portfolio = Portfolio::new(assets, correlation);
    let n = portfolio.assets.len();

    let mut best_weights = portfolio.weights.clone();
    let mut best_sharpe = portfolio.sharpe_ratio(risk_free_rate);

    // Простой поиск по сетке
    // На практике используйте квадратичное программирование или градиентный спуск
    let steps = 20;

    if n == 2 {
        for i in 0..=steps {
            let w1 = i as f64 / steps as f64;
            let w2 = 1.0 - w1;

            portfolio.set_weights(vec![w1, w2]);
            let sharpe = portfolio.sharpe_ratio(risk_free_rate);

            if sharpe > best_sharpe {
                best_sharpe = sharpe;
                best_weights = portfolio.weights.clone();
            }
        }
    } else if n == 3 {
        for i in 0..=steps {
            for j in 0..=(steps - i) {
                let w1 = i as f64 / steps as f64;
                let w2 = j as f64 / steps as f64;
                let w3 = 1.0 - w1 - w2;

                if w3 >= 0.0 {
                    portfolio.set_weights(vec![w1, w2, w3]);
                    let sharpe = portfolio.sharpe_ratio(risk_free_rate);

                    if sharpe > best_sharpe {
                        best_sharpe = sharpe;
                        best_weights = portfolio.weights.clone();
                    }
                }
            }
        }
    }

    (best_weights, best_sharpe)
}

fn main() {
    let assets = vec![
        Asset {
            name: "BTC".to_string(),
            expected_return: 0.50,  // 50% ожидаемая годовая доходность
            volatility: 0.80,       // 80% годовая волатильность
        },
        Asset {
            name: "ETH".to_string(),
            expected_return: 0.60,  // 60% ожидаемая годовая доходность
            volatility: 0.90,       // 90% годовая волатильность
        },
        Asset {
            name: "Стабильная стратегия".to_string(),
            expected_return: 0.15,  // 15% ожидаемая годовая доходность
            volatility: 0.20,       // 20% годовая волатильность
        },
    ];

    // Корреляционная матрица
    let correlation = vec![
        vec![1.0, 0.7, 0.2],   // Корреляции BTC
        vec![0.7, 1.0, 0.3],   // Корреляции ETH
        vec![0.2, 0.3, 1.0],   // Корреляции стабильной стратегии
    ];

    let risk_free_rate = 0.05; // 5% годовых

    println!("Оптимизация портфеля для максимального коэффициента Шарпа");
    println!("{:-<60}", "");

    // Показываем коэффициенты Шарпа отдельных активов
    println!("\nКоэффициенты Шарпа отдельных активов:");
    for asset in &assets {
        let sharpe = (asset.expected_return - risk_free_rate) / asset.volatility;
        println!("  {}: {:.2}", asset.name, sharpe);
    }

    // Находим оптимальные веса
    let (optimal_weights, optimal_sharpe) =
        optimize_portfolio(assets.clone(), correlation.clone(), risk_free_rate);

    println!("\nОптимальное распределение портфеля:");
    for (asset, weight) in assets.iter().zip(optimal_weights.iter()) {
        println!("  {}: {:.1}%", asset.name, weight * 100.0);
    }

    println!("\nОптимальный коэффициент Шарпа портфеля: {:.2}", optimal_sharpe);

    // Создаём и показываем статистику оптимального портфеля
    let mut optimal_portfolio = Portfolio::new(assets, correlation);
    optimal_portfolio.set_weights(optimal_weights);

    println!("\nМетрики оптимального портфеля:");
    println!("  Ожидаемая доходность: {:.1}%",
             optimal_portfolio.expected_return() * 100.0);
    println!("  Волатильность: {:.1}%",
             optimal_portfolio.portfolio_volatility() * 100.0);
}
```

## Практическое применение: Мониторинг стратегии

```rust
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
struct Trade {
    timestamp: u64,
    symbol: String,
    pnl_percent: f64,
}

#[derive(Debug)]
struct StrategyMonitor {
    name: String,
    trades: Vec<Trade>,
    daily_returns: Vec<f64>,
    risk_free_rate: f64,
    min_sharpe_threshold: f64,
}

impl StrategyMonitor {
    fn new(name: &str, annual_rf: f64, min_sharpe: f64) -> Self {
        StrategyMonitor {
            name: name.to_string(),
            trades: Vec::new(),
            daily_returns: Vec::new(),
            risk_free_rate: annual_rf / 252.0, // Конвертируем в дневную
            min_sharpe_threshold: min_sharpe,
        }
    }

    fn record_trade(&mut self, symbol: &str, pnl_percent: f64) {
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();

        self.trades.push(Trade {
            timestamp,
            symbol: symbol.to_string(),
            pnl_percent,
        });
    }

    fn close_day(&mut self, daily_return: f64) {
        self.daily_returns.push(daily_return);
    }

    fn calculate_sharpe(&self) -> Option<f64> {
        if self.daily_returns.len() < 5 {
            return None; // Нужно минимум 5 дней данных
        }

        let excess: Vec<f64> = self.daily_returns.iter()
            .map(|r| r - self.risk_free_rate)
            .collect();

        let mean = excess.iter().sum::<f64>() / excess.len() as f64;
        let variance = excess.iter()
            .map(|x| (x - mean).powi(2))
            .sum::<f64>() / (excess.len() - 1) as f64;
        let std_dev = variance.sqrt();

        if std_dev == 0.0 {
            return None;
        }

        // Аннуализируем: умножаем на sqrt(252)
        Some((mean / std_dev) * (252.0_f64).sqrt())
    }

    fn get_status(&self) -> StrategyStatus {
        match self.calculate_sharpe() {
            None => StrategyStatus::InsufficientData,
            Some(sharpe) if sharpe >= self.min_sharpe_threshold => {
                StrategyStatus::Healthy(sharpe)
            }
            Some(sharpe) if sharpe > 0.0 => {
                StrategyStatus::Warning(sharpe)
            }
            Some(sharpe) => {
                StrategyStatus::Critical(sharpe)
            }
        }
    }

    fn generate_report(&self) -> String {
        let mut report = format!("Отчёт по стратегии: {}\n", self.name);
        report.push_str(&format!("{:-<50}\n", ""));

        report.push_str(&format!("Всего сделок: {}\n", self.trades.len()));
        report.push_str(&format!("Дней отслежено: {}\n", self.daily_returns.len()));

        if !self.daily_returns.is_empty() {
            let total_return: f64 = self.daily_returns.iter()
                .fold(1.0, |acc, r| acc * (1.0 + r)) - 1.0;
            report.push_str(&format!("Общая доходность: {:.2}%\n", total_return * 100.0));
        }

        match self.get_status() {
            StrategyStatus::InsufficientData => {
                report.push_str("Статус: Недостаточно данных для расчёта Шарпа\n");
            }
            StrategyStatus::Healthy(sharpe) => {
                report.push_str(&format!("Статус: ЗДОРОВЫЙ\n"));
                report.push_str(&format!("Годовой Шарп: {:.2}\n", sharpe));
            }
            StrategyStatus::Warning(sharpe) => {
                report.push_str(&format!("Статус: ПРЕДУПРЕЖДЕНИЕ - Ниже порога\n"));
                report.push_str(&format!("Годовой Шарп: {:.2}\n", sharpe));
            }
            StrategyStatus::Critical(sharpe) => {
                report.push_str(&format!("Статус: КРИТИЧЕСКИЙ - Отрицательный Шарп!\n"));
                report.push_str(&format!("Годовой Шарп: {:.2}\n", sharpe));
            }
        }

        report
    }
}

#[derive(Debug)]
enum StrategyStatus {
    InsufficientData,
    Healthy(f64),
    Warning(f64),
    Critical(f64),
}

fn main() {
    let mut monitor = StrategyMonitor::new(
        "Крипто Моментум Бот",
        0.05,  // 5% годовая безрисковая ставка
        1.5,   // Минимально приемлемый Шарп
    );

    // Симулируем 20 торговых дней
    let simulated_daily_returns = vec![
        0.02, 0.01, -0.005, 0.015, 0.008,
        0.012, -0.01, 0.02, 0.005, 0.018,
        -0.008, 0.025, 0.01, -0.003, 0.015,
        0.007, 0.02, -0.012, 0.018, 0.01,
    ];

    println!("Симуляция 20 торговых дней...\n");

    for (day, &daily_return) in simulated_daily_returns.iter().enumerate() {
        // Записываем сделки
        monitor.record_trade("BTC/USDT", daily_return * 0.6);
        monitor.record_trade("ETH/USDT", daily_return * 0.4);

        // Закрываем день
        monitor.close_day(daily_return);

        // Проверяем статус каждые 5 дней
        if (day + 1) % 5 == 0 {
            println!("Контрольная точка, день {}:", day + 1);
            match monitor.get_status() {
                StrategyStatus::InsufficientData => {
                    println!("  Собираем данные...\n");
                }
                StrategyStatus::Healthy(sharpe) => {
                    println!("  Шарп: {:.2} - Стратегия работает хорошо\n", sharpe);
                }
                StrategyStatus::Warning(sharpe) => {
                    println!("  Шарп: {:.2} - Рассмотрите корректировку параметров\n", sharpe);
                }
                StrategyStatus::Critical(sharpe) => {
                    println!("  Шарп: {:.2} - ОСТАНОВИТЕ ТОРГОВЛЮ!\n", sharpe);
                }
            }
        }
    }

    println!("{}", monitor.generate_report());
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Коэффициент Шарпа | Измеряет избыточную доходность на единицу риска |
| Формула | (Доходность портфеля - Безрисковая ставка) / Волатильность |
| Аннуализация | Умножаем на sqrt(периодов в году) |
| Скользящий Шарп | Отслеживает производительность стратегии во времени |
| Оптимизация портфеля | Поиск весов, максимизирующих коэффициент Шарпа |
| Мониторинг стратегии | Использование порогов Шарпа для управления рисками |

## Упражнения

1. **Калькулятор Шарпа**: Реализуй функцию, которая принимает CSV-файл с дневными ценами и вычисляет годовой коэффициент Шарпа. Обработай крайние случаи: отсутствующие данные и нулевую волатильность.

2. **Панель сравнения стратегий**: Создай структуру, которая хранит несколько стратегий и ранжирует их по коэффициенту Шарпа. Добавь методы для фильтрации стратегий по минимальному порогу Шарпа.

3. **Динамическая безрисковая ставка**: Модифицируй `StrategyMonitor`, чтобы принимать временной ряд безрисковых ставок вместо одного значения. Это отражает реальные сценарии, где ставки меняются со временем.

4. **Обнаружение падения Шарпа**: Реализуй функцию, которая определяет, когда скользящий коэффициент Шарпа стратегии упал более чем на 20% от пикового значения. Это полезно для выявления стратегий, требующих переоптимизации.

## Домашнее задание

1. **Модифицированный коэффициент Шарпа**: Изучи и реализуй **коэффициент Сортино**, который штрафует только нисходящую волатильность. Сравни результаты между Шарпом и Сортино для стратегии с асимметричными доходностями.

2. **Симуляция Монте-Карло**: Создай программу, которая:
   - Генерирует 1000 случайных рядов доходностей с заданным средним и волатильностью
   - Вычисляет коэффициент Шарпа для каждого ряда
   - Строит гистограмму распределения коэффициента Шарпа
   - Определяет доверительные интервалы для «истинного» Шарпа

3. **Мультиактивный оптимизатор**: Расширь пример оптимизации портфеля:
   - Прими любое количество активов
   - Используй градиентный спуск вместо поиска по сетке
   - Добавь ограничения (например, максимум 30% в одном активе)
   - Выведи эффективную границу (график риск-доходность)

4. **Система оповещений в реальном времени**: Построй монитор торгового бота, который:
   - Отслеживает несколько стратегий одновременно
   - Вычисляет 30-дневные скользящие коэффициенты Шарпа
   - Отправляет оповещения, когда Шарп падает ниже порога
   - Автоматически уменьшает размеры позиций для отстающих стратегий

## Навигация

[← Предыдущий день](../272-risk-parity-balancing/ru.md) | [Следующий день →](../274-sortino-ratio-downside-risk/ru.md)
