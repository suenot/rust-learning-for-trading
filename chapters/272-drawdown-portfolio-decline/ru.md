# День 272: Drawdown: Просадка портфеля

## Аналогия из трейдинга

Представь, что ты управляешь торговым портфелем, который начинался с $100,000. За несколько месяцев твой портфель вырос до $150,000 — отличный рост на 50%! Но затем рынок пошёл против тебя. Портфель упал до $120,000. Хотя ты всё ещё в плюсе на 20% от начальной точки, ты испытал **просадку** (drawdown) в $30,000 (или 20%) от пикового значения.

Именно это и измеряет **просадка** — снижение от пика до минимума в стоимости портфеля. В трейдинге просадка является одной из важнейших метрик риска, потому что:
- Она показывает худший сценарий, который пережил инвестор
- Помогает установить реалистичные ожидания от эффективности стратегии
- Критически важна для определения размера позиций и управления рисками
- Большие просадки могут уничтожить годы прибыли

## Что такое Drawdown?

Просадка измеряет падение от пика до минимума за определённый период инвестирования. Существует несколько типов:

1. **Абсолютная просадка** — разница между начальным капиталом и минимальной точкой
2. **Максимальная просадка (MDD)** — наибольшее падение от пика до минимума в стоимости портфеля
3. **Относительная просадка** — просадка, выраженная в процентах от пикового значения
4. **Продолжительность просадки** — сколько времени требуется для восстановления после просадки

```
Стоимость портфеля во времени:

$150,000  ────────●─────────────────────────────────── Пик
                   \
                    \
$130,000             \                    ●────────── Восстановление
                      \                  /
                       \                /
$120,000                ●──────────────●               Минимум
                        |<-- Просадка -->|
                        |  Длительность  |
```

## Простой расчёт просадки

```rust
/// Рассчитывает текущую просадку от пикового значения
fn calculate_drawdown(peak: f64, current: f64) -> f64 {
    if peak <= 0.0 {
        return 0.0;
    }

    let drawdown = (peak - current) / peak * 100.0;
    drawdown.max(0.0) // Просадка не может быть отрицательной
}

/// Рассчитывает максимальную просадку из серии значений портфеля
fn calculate_max_drawdown(values: &[f64]) -> (f64, usize, usize) {
    if values.is_empty() {
        return (0.0, 0, 0);
    }

    let mut max_drawdown = 0.0;
    let mut peak = values[0];
    let mut peak_index = 0;
    let mut trough_index = 0;
    let mut max_peak_index = 0;
    let mut max_trough_index = 0;

    for (i, &value) in values.iter().enumerate() {
        if value > peak {
            peak = value;
            peak_index = i;
        }

        let drawdown = (peak - value) / peak * 100.0;

        if drawdown > max_drawdown {
            max_drawdown = drawdown;
            max_peak_index = peak_index;
            max_trough_index = i;
        }
    }

    (max_drawdown, max_peak_index, max_trough_index)
}

fn main() {
    // Имитация значений портфеля во времени
    let portfolio_values = vec![
        100_000.0, 105_000.0, 110_000.0, 108_000.0, 115_000.0,
        120_000.0, 118_000.0, 125_000.0, 130_000.0, 128_000.0,
        122_000.0, 115_000.0, 110_000.0, 112_000.0, 118_000.0,
        125_000.0, 132_000.0, 140_000.0, 138_000.0, 145_000.0,
    ];

    let (max_dd, peak_idx, trough_idx) = calculate_max_drawdown(&portfolio_values);

    println!("Анализ портфеля:");
    println!("  Начальная стоимость: ${:.2}", portfolio_values[0]);
    println!("  Конечная стоимость: ${:.2}", portfolio_values.last().unwrap());
    println!();
    println!("Максимальная просадка:");
    println!("  Просадка: {:.2}%", max_dd);
    println!("  Пиковое значение: ${:.2} (день {})", portfolio_values[peak_idx], peak_idx + 1);
    println!("  Минимум: ${:.2} (день {})", portfolio_values[trough_idx], trough_idx + 1);

    // Расчёт текущей просадки
    let current_peak = portfolio_values.iter().cloned().fold(0.0_f64, f64::max);
    let current_value = *portfolio_values.last().unwrap();
    let current_dd = calculate_drawdown(current_peak, current_value);

    println!();
    println!("Текущая просадка: {:.2}%", current_dd);
}
```

## Визуализация просадок

```
День: 1    5    10   15   20   25   30
      |    |    |    |    |    |    |
$150k ─────────────────●─────────────── Пик
                      /│\
$140k ───────────────/ │ \────────────
                    /  │  \
$130k ─────────────●   │   \────●─────
                  /    │    \  /
$120k ────────●──/     │     \/
             /         │
$110k ──────/          │      ●──────── Минимум
           /           │      │
$100k ────●            │      │
          Старт        │      │
                       |------|
                       Просадка: 26.7%
                       ($150k → $110k)
```

## Отслеживание портфеля с мониторингом просадки

```rust
use std::collections::VecDeque;

#[derive(Debug, Clone)]
struct DrawdownInfo {
    current_drawdown: f64,
    max_drawdown: f64,
    peak_value: f64,
    trough_value: f64,
    days_in_drawdown: u32,
    recovery_needed: f64,
}

#[derive(Debug)]
struct PortfolioTracker {
    values: VecDeque<f64>,
    peak_value: f64,
    max_drawdown: f64,
    trough_value: f64,
    days_since_peak: u32,
    max_history: usize,
}

impl PortfolioTracker {
    fn new(initial_value: f64, max_history: usize) -> Self {
        let mut values = VecDeque::with_capacity(max_history);
        values.push_back(initial_value);

        PortfolioTracker {
            values,
            peak_value: initial_value,
            max_drawdown: 0.0,
            trough_value: initial_value,
            days_since_peak: 0,
            max_history,
        }
    }

    fn update(&mut self, new_value: f64) {
        // Добавляем новое значение в историю
        self.values.push_back(new_value);
        if self.values.len() > self.max_history {
            self.values.pop_front();
        }

        // Обновляем отслеживание пика
        if new_value > self.peak_value {
            self.peak_value = new_value;
            self.days_since_peak = 0;
        } else {
            self.days_since_peak += 1;
        }

        // Рассчитываем текущую просадку
        let current_dd = (self.peak_value - new_value) / self.peak_value * 100.0;

        // Обновляем макс. просадку, если текущая хуже
        if current_dd > self.max_drawdown {
            self.max_drawdown = current_dd;
            self.trough_value = new_value;
        }
    }

    fn get_drawdown_info(&self) -> DrawdownInfo {
        let current_value = *self.values.back().unwrap_or(&0.0);
        let current_drawdown = (self.peak_value - current_value) / self.peak_value * 100.0;

        // Расчёт необходимого восстановления: если портфель упал на 20%, нужен рост 25%
        // Формула: (пик / текущее) - 1
        let recovery_needed = if current_value > 0.0 {
            (self.peak_value / current_value - 1.0) * 100.0
        } else {
            100.0
        };

        DrawdownInfo {
            current_drawdown: current_drawdown.max(0.0),
            max_drawdown: self.max_drawdown,
            peak_value: self.peak_value,
            trough_value: self.trough_value,
            days_in_drawdown: self.days_since_peak,
            recovery_needed: recovery_needed.max(0.0),
        }
    }

    fn get_underwater_chart(&self) -> Vec<f64> {
        let mut peak = self.values[0];
        self.values.iter().map(|&value| {
            if value > peak {
                peak = value;
            }
            -((peak - value) / peak * 100.0)
        }).collect()
    }
}

fn main() {
    let mut tracker = PortfolioTracker::new(100_000.0, 100);

    // Имитация дневных изменений портфеля
    let daily_changes = [
        1.02, 1.01, 0.98, 1.03, 1.02, 0.97, 0.95, 1.01, 1.04, 1.02,
        0.96, 0.94, 0.98, 1.03, 1.05, 1.02, 0.99, 1.01, 1.03, 1.02,
    ];

    let mut current_value = 100_000.0;

    println!("Ежедневные обновления портфеля:");
    println!("{:>4} {:>12} {:>10} {:>10} {:>12}",
        "День", "Стоимость", "Дневн.%", "Просадка%", "Макс.DD%");
    println!("{}", "-".repeat(52));

    for (day, &change) in daily_changes.iter().enumerate() {
        let prev_value = current_value;
        current_value *= change;
        tracker.update(current_value);

        let info = tracker.get_drawdown_info();
        let daily_pct = (current_value / prev_value - 1.0) * 100.0;

        println!("{:>4} {:>12.2} {:>+10.2} {:>10.2} {:>12.2}",
            day + 1, current_value, daily_pct, info.current_drawdown, info.max_drawdown);
    }

    let final_info = tracker.get_drawdown_info();
    println!();
    println!("Итоговый анализ:");
    println!("  Пиковое значение: ${:.2}", final_info.peak_value);
    println!("  Текущая просадка: {:.2}%", final_info.current_drawdown);
    println!("  Максимальная просадка: {:.2}%", final_info.max_drawdown);
    println!("  Дней в просадке: {}", final_info.days_in_drawdown);
    println!("  Требуемое восстановление: {:.2}%", final_info.recovery_needed);

    // Показать underwater chart
    let underwater = tracker.get_underwater_chart();
    println!();
    println!("Underwater Chart (просадка во времени):");
    for (i, &dd) in underwater.iter().enumerate() {
        let bar_len = (-dd * 2.0) as usize;
        let bar: String = "█".repeat(bar_len.min(40));
        println!("День {:>2}: {:>6.2}% {}", i + 1, dd, bar);
    }
}
```

## Управление рисками с лимитами просадки

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum RiskLevel {
    Normal,    // Нормальный
    Elevated,  // Повышенный
    High,      // Высокий
    Critical,  // Критический
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum TradingAction {
    FullPosition,              // Полная позиция
    ReducedPosition(f64),      // Уменьшенная позиция (процент от нормальной)
    StopTrading,               // Остановка торговли
}

#[derive(Debug)]
struct DrawdownRiskManager {
    max_allowed_drawdown: f64,
    warning_threshold: f64,
    critical_threshold: f64,
    current_drawdown: f64,
    peak_value: f64,
}

impl DrawdownRiskManager {
    fn new(max_allowed_drawdown: f64) -> Self {
        DrawdownRiskManager {
            max_allowed_drawdown,
            warning_threshold: max_allowed_drawdown * 0.5,
            critical_threshold: max_allowed_drawdown * 0.8,
            current_drawdown: 0.0,
            peak_value: 0.0,
        }
    }

    fn update(&mut self, portfolio_value: f64) {
        if portfolio_value > self.peak_value {
            self.peak_value = portfolio_value;
        }

        self.current_drawdown = if self.peak_value > 0.0 {
            (self.peak_value - portfolio_value) / self.peak_value * 100.0
        } else {
            0.0
        };
    }

    fn get_risk_level(&self) -> RiskLevel {
        if self.current_drawdown >= self.max_allowed_drawdown {
            RiskLevel::Critical
        } else if self.current_drawdown >= self.critical_threshold {
            RiskLevel::High
        } else if self.current_drawdown >= self.warning_threshold {
            RiskLevel::Elevated
        } else {
            RiskLevel::Normal
        }
    }

    fn get_trading_action(&self) -> TradingAction {
        match self.get_risk_level() {
            RiskLevel::Normal => TradingAction::FullPosition,
            RiskLevel::Elevated => TradingAction::ReducedPosition(0.75),
            RiskLevel::High => TradingAction::ReducedPosition(0.50),
            RiskLevel::Critical => TradingAction::StopTrading,
        }
    }

    fn calculate_position_size(&self, base_position: f64) -> f64 {
        match self.get_trading_action() {
            TradingAction::FullPosition => base_position,
            TradingAction::ReducedPosition(pct) => base_position * pct,
            TradingAction::StopTrading => 0.0,
        }
    }

    fn get_status(&self) -> String {
        let risk_level = self.get_risk_level();
        let action = self.get_trading_action();

        format!(
            "Просадка: {:.2}% | Риск: {:?} | Действие: {:?}",
            self.current_drawdown, risk_level, action
        )
    }
}

fn main() {
    // Максимально допустимая просадка 20%
    let mut risk_manager = DrawdownRiskManager::new(20.0);

    let portfolio_values = [
        100_000.0, 105_000.0, 110_000.0, 108_000.0, 103_000.0,
        98_000.0, 95_000.0, 92_000.0, 89_000.0, 88_000.0,
        90_000.0, 94_000.0, 98_000.0, 102_000.0, 107_000.0,
    ];

    let base_position_size = 10_000.0;

    println!("Симуляция управления рисками по просадке");
    println!("Макс. допустимая просадка: {:.2}%", 20.0);
    println!();
    println!("{:>4} {:>12} {:>10} {:>10} {:>12}",
        "День", "Портфель", "DD%", "Риск", "Позиция");
    println!("{}", "-".repeat(54));

    for (day, &value) in portfolio_values.iter().enumerate() {
        risk_manager.update(value);

        let risk = risk_manager.get_risk_level();
        let position = risk_manager.calculate_position_size(base_position_size);

        println!("{:>4} {:>12.2} {:>10.2} {:>10?} {:>12.2}",
            day + 1, value, risk_manager.current_drawdown, risk, position);
    }
}
```

## Анализ эффективности стратегии

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct StrategyMetrics {
    total_return: f64,
    max_drawdown: f64,
    calmar_ratio: f64,     // Годовая доходность / Макс. просадка
    recovery_factor: f64,   // Чистая прибыль / Макс. просадка
    longest_drawdown_days: u32,
    number_of_drawdowns: u32,
    average_drawdown: f64,
}

#[derive(Debug)]
struct StrategyAnalyzer {
    equity_curve: Vec<f64>,
    drawdowns: Vec<f64>,
    drawdown_periods: Vec<(usize, usize, f64)>, // (начало, конец, величина)
}

impl StrategyAnalyzer {
    fn new() -> Self {
        StrategyAnalyzer {
            equity_curve: Vec::new(),
            drawdowns: Vec::new(),
            drawdown_periods: Vec::new(),
        }
    }

    fn analyze(&mut self, equity_values: &[f64]) -> StrategyMetrics {
        self.equity_curve = equity_values.to_vec();
        self.calculate_drawdowns();
        self.identify_drawdown_periods();

        let total_return = if !equity_values.is_empty() {
            (equity_values.last().unwrap() / equity_values[0] - 1.0) * 100.0
        } else {
            0.0
        };

        let max_drawdown = self.drawdowns.iter().cloned().fold(0.0_f64, f64::max);

        // Предполагаем 252 торговых дня в году для аннуализации
        let trading_days = equity_values.len() as f64;
        let annual_return = total_return * (252.0 / trading_days);

        let calmar_ratio = if max_drawdown > 0.0 {
            annual_return / max_drawdown
        } else {
            0.0
        };

        let net_profit = equity_values.last().unwrap_or(&0.0) - equity_values.first().unwrap_or(&0.0);
        let max_dd_absolute = max_drawdown / 100.0 * equity_values.iter().cloned().fold(0.0_f64, f64::max);

        let recovery_factor = if max_dd_absolute > 0.0 {
            net_profit / max_dd_absolute
        } else {
            0.0
        };

        let longest_drawdown = self.drawdown_periods.iter()
            .map(|(start, end, _)| (end - start) as u32)
            .max()
            .unwrap_or(0);

        let avg_drawdown = if !self.drawdown_periods.is_empty() {
            self.drawdown_periods.iter().map(|(_, _, mag)| mag).sum::<f64>()
                / self.drawdown_periods.len() as f64
        } else {
            0.0
        };

        StrategyMetrics {
            total_return,
            max_drawdown,
            calmar_ratio,
            recovery_factor,
            longest_drawdown_days: longest_drawdown,
            number_of_drawdowns: self.drawdown_periods.len() as u32,
            average_drawdown: avg_drawdown,
        }
    }

    fn calculate_drawdowns(&mut self) {
        self.drawdowns.clear();
        let mut peak = self.equity_curve[0];

        for &value in &self.equity_curve {
            if value > peak {
                peak = value;
            }
            let dd = (peak - value) / peak * 100.0;
            self.drawdowns.push(dd);
        }
    }

    fn identify_drawdown_periods(&mut self) {
        self.drawdown_periods.clear();

        let mut in_drawdown = false;
        let mut start_idx = 0;
        let mut max_dd_in_period = 0.0;

        for (i, &dd) in self.drawdowns.iter().enumerate() {
            if dd > 0.0 && !in_drawdown {
                // Начало новой просадки
                in_drawdown = true;
                start_idx = i;
                max_dd_in_period = dd;
            } else if dd > 0.0 && in_drawdown {
                // Продолжение просадки
                max_dd_in_period = max_dd_in_period.max(dd);
            } else if dd == 0.0 && in_drawdown {
                // Восстановление после просадки
                self.drawdown_periods.push((start_idx, i, max_dd_in_period));
                in_drawdown = false;
            }
        }

        // Обработка случая, когда всё ещё в просадке в конце
        if in_drawdown {
            self.drawdown_periods.push((start_idx, self.drawdowns.len(), max_dd_in_period));
        }
    }
}

fn main() {
    // Имитация кривых капитала для разных стратегий
    let mut strategies: HashMap<&str, Vec<f64>> = HashMap::new();

    // Консервативная стратегия: стабильный рост, малые просадки
    strategies.insert("Консервативная", vec![
        100000.0, 100500.0, 101000.0, 100800.0, 101200.0,
        101800.0, 102300.0, 102100.0, 102500.0, 103000.0,
        103500.0, 103300.0, 103800.0, 104200.0, 104800.0,
    ]);

    // Агрессивная стратегия: высокая доходность, большие просадки
    strategies.insert("Агрессивная", vec![
        100000.0, 103000.0, 106000.0, 102000.0, 108000.0,
        112000.0, 105000.0, 110000.0, 118000.0, 115000.0,
        108000.0, 115000.0, 122000.0, 118000.0, 125000.0,
    ]);

    // Волатильная стратегия: большие колебания
    strategies.insert("Волатильная", vec![
        100000.0, 110000.0, 95000.0, 105000.0, 90000.0,
        100000.0, 115000.0, 100000.0, 120000.0, 105000.0,
        95000.0, 110000.0, 100000.0, 115000.0, 120000.0,
    ]);

    println!("Сравнение эффективности стратегий");
    println!("{}", "=".repeat(70));

    let mut analyzer = StrategyAnalyzer::new();

    for (name, equity) in &strategies {
        let metrics = analyzer.analyze(equity);

        println!();
        println!("Стратегия: {}", name);
        println!("{}", "-".repeat(40));
        println!("  Общая доходность:    {:>8.2}%", metrics.total_return);
        println!("  Макс. просадка:      {:>8.2}%", metrics.max_drawdown);
        println!("  Коэфф. Кальмара:     {:>8.2}", metrics.calmar_ratio);
        println!("  Фактор восстановления: {:>6.2}", metrics.recovery_factor);
        println!("  Макс. длит. DD (дн): {:>8}", metrics.longest_drawdown_days);
        println!("  Кол-во просадок:     {:>8}", metrics.number_of_drawdowns);
        println!("  Средняя просадка:    {:>8.2}%", metrics.average_drawdown);
    }
}
```

## Практический пример: Мониторинг портфеля в реальном времени

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

#[derive(Debug, Clone)]
struct Alert {
    timestamp: u64,
    message: String,
    level: AlertLevel,
}

#[derive(Debug, Clone, Copy)]
enum AlertLevel {
    Info,      // Информация
    Warning,   // Предупреждение
    Critical,  // Критический
}

#[derive(Debug)]
struct RealTimePortfolioMonitor {
    portfolio_value: Arc<Mutex<f64>>,
    peak_value: Arc<Mutex<f64>>,
    max_drawdown: Arc<Mutex<f64>>,
    alerts: Arc<Mutex<Vec<Alert>>>,
    drawdown_threshold: f64,
    critical_threshold: f64,
    tick_counter: Arc<Mutex<u64>>,
}

impl RealTimePortfolioMonitor {
    fn new(initial_value: f64, drawdown_threshold: f64, critical_threshold: f64) -> Self {
        RealTimePortfolioMonitor {
            portfolio_value: Arc::new(Mutex::new(initial_value)),
            peak_value: Arc::new(Mutex::new(initial_value)),
            max_drawdown: Arc::new(Mutex::new(0.0)),
            alerts: Arc::new(Mutex::new(Vec::new())),
            drawdown_threshold,
            critical_threshold,
            tick_counter: Arc::new(Mutex::new(0)),
        }
    }

    fn update_price(&self, new_value: f64) {
        let mut value = self.portfolio_value.lock().unwrap();
        let mut peak = self.peak_value.lock().unwrap();
        let mut max_dd = self.max_drawdown.lock().unwrap();
        let mut alerts = self.alerts.lock().unwrap();
        let mut tick = self.tick_counter.lock().unwrap();

        *tick += 1;
        *value = new_value;

        // Обновляем пик при новом максимуме
        if new_value > *peak {
            *peak = new_value;
        }

        // Рассчитываем текущую просадку
        let current_dd = (*peak - new_value) / *peak * 100.0;

        // Обновляем макс. просадку
        if current_dd > *max_dd {
            *max_dd = current_dd;
        }

        // Генерируем алерты на основе порогов
        if current_dd >= self.critical_threshold {
            alerts.push(Alert {
                timestamp: *tick,
                message: format!(
                    "КРИТИЧНО: Просадка {:.2}% превышает критический порог {:.2}%!",
                    current_dd, self.critical_threshold
                ),
                level: AlertLevel::Critical,
            });
        } else if current_dd >= self.drawdown_threshold {
            alerts.push(Alert {
                timestamp: *tick,
                message: format!(
                    "ВНИМАНИЕ: Просадка {:.2}% превышает порог {:.2}%",
                    current_dd, self.drawdown_threshold
                ),
                level: AlertLevel::Warning,
            });
        }
    }

    fn get_status(&self) -> String {
        let value = *self.portfolio_value.lock().unwrap();
        let peak = *self.peak_value.lock().unwrap();
        let max_dd = *self.max_drawdown.lock().unwrap();

        let current_dd = (peak - value) / peak * 100.0;

        format!(
            "Портфель: ${:.2} | Пик: ${:.2} | Текущ. DD: {:.2}% | Макс. DD: {:.2}%",
            value, peak, current_dd, max_dd
        )
    }

    fn get_alerts(&self) -> Vec<Alert> {
        self.alerts.lock().unwrap().clone()
    }
}

fn main() {
    let monitor = Arc::new(RealTimePortfolioMonitor::new(100_000.0, 5.0, 10.0));

    // Имитация обновлений рыночных цен
    let price_updates = [
        100_000.0, 102_000.0, 104_000.0, 103_000.0, 101_000.0,
        99_000.0, 97_000.0, 95_000.0, 93_000.0, 91_000.0,
        93_000.0, 96_000.0, 98_000.0, 101_000.0, 104_000.0,
    ];

    println!("Мониторинг портфеля в реальном времени");
    println!("Порог просадки: 5.0% | Критический порог: 10.0%");
    println!("{}", "=".repeat(70));
    println!();

    for (i, &price) in price_updates.iter().enumerate() {
        monitor.update_price(price);

        println!("Тик {}: {}", i + 1, monitor.get_status());

        // Проверяем новые алерты
        let alerts = monitor.get_alerts();
        for alert in alerts.iter().filter(|a| a.timestamp == (i + 1) as u64) {
            match alert.level {
                AlertLevel::Critical => println!("  [!!!] {}", alert.message),
                AlertLevel::Warning => println!("  [!] {}", alert.message),
                AlertLevel::Info => println!("  [i] {}", alert.message),
            }
        }

        thread::sleep(Duration::from_millis(100));
    }

    println!();
    println!("Итоговый статус: {}", monitor.get_status());
    println!();
    println!("Все сгенерированные алерты:");
    for alert in monitor.get_alerts() {
        println!("  Тик {}: {:?} - {}", alert.timestamp, alert.level, alert.message);
    }
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Просадка (Drawdown) | Падение от пика до минимума стоимости портфеля |
| Максимальная просадка | Наибольшее падение от пика за весь период |
| Фактор восстановления | Чистая прибыль, делённая на макс. просадку |
| Коэффициент Кальмара | Годовая доходность, делённая на макс. просадку |
| Underwater Chart | Визуальное представление просадки во времени |
| Длительность просадки | Время, необходимое для восстановления после просадки |

## Домашнее задание

1. **Калькулятор просадки**: Создай структуру `DrawdownCalculator`, которая:
   - Принимает вектор дневных значений портфеля
   - Рассчитывает все типы просадок (абсолютная, относительная, максимальная)
   - Находит все периоды просадок с датами начала/конца
   - Возвращает среднюю продолжительность просадки

2. **Доходность с поправкой на риск**: Реализуй функцию, которая сравнивает несколько торговых стратегий, используя:
   - Коэффициент Шарпа (доходность на единицу риска)
   - Коэффициент Кальмара (доходность на единицу просадки)
   - Коэффициент Сортино (доходность на единицу нисходящего риска)
   Выведи сравнительную таблицу с ранжированием стратегий по каждой метрике.

3. **Система оповещения о просадке**: Построй многопоточную систему мониторинга, которая:
   - Получает обновления цен в реальном времени через канал
   - Отслеживает просадку в реальном времени
   - Отправляет оповещения разных уровней серьёзности (предупреждение, критический, экстренный)
   - Автоматически уменьшает размер позиции при превышении порогов просадки

4. **Монте-Карло анализ просадки**: Создай симуляцию, которая:
   - Принимает исторические дневные доходности
   - Запускает 1000 случайных симуляций возможных будущих путей
   - Рассчитывает распределение вероятностей максимальных просадок
   - Сообщает о худшем сценарии на уровне 95-го процентиля

## Навигация

[← Предыдущий день](../271-portfolio-variance/ru.md) | [Следующий день →](../273-sharpe-ratio/ru.md)
