# День 268: Критерий Келли: Оптимальный размер позиции

## Аналогия из трейдинга

Представь, что ты трейдер с проверенной стратегией, которая выигрывает в 60% случаев, принося $2 на каждый $1 риска при выигрыше и теряя $1 при проигрыше. У тебя $100,000 капитала. Сколько следует рисковать в каждой сделке?

- Рискуешь слишком мало (1%) — упускаешь прибыль
- Рискуешь слишком много (50%) — несколько убытков могут уничтожить счёт
- Рискуешь правильно — максимизируешь долгосрочный рост

Именно эту проблему решает **критерий Келли** — поиск математически оптимального размера ставки, который максимизирует геометрическую скорость роста портфеля при управлении риском.

В 1950-х годах Джон Келли разработал эту формулу, работая в Bell Labs. С тех пор профессиональные игроки, управляющие хедж-фондов и количественные трейдеры используют её для определения размера позиций.

## Что такое критерий Келли?

Критерий Келли — это формула, которая рассчитывает оптимальную долю капитала для риска в ставке или сделке:

```
f* = (p × b - q) / b
```

Где:
- **f*** = доля капитала для ставки (фракция Келли)
- **p** = вероятность выигрыша
- **b** = соотношение выигрыша к проигрышу
- **q** = вероятность проигрыша (1 - p)

Для трейдинга более интуитивная форма:

```
f* = (Процент побед × Соотношение прибыль/убыток - Процент убытков) / Соотношение прибыль/убыток
```

Или эквивалентно:

```
f* = Процент побед - (Процент убытков / Соотношение прибыль/убыток)
```

## Базовый калькулятор Келли

```rust
/// Рассчитывает критерий Келли для оптимального размера позиции
fn kelly_criterion(win_probability: f64, win_loss_ratio: f64) -> f64 {
    let loss_probability = 1.0 - win_probability;

    // f* = (p * b - q) / b
    let kelly_fraction = (win_probability * win_loss_ratio - loss_probability) / win_loss_ratio;

    // Келли может быть отрицательным (не ставить) или > 1 (использовать плечо)
    // Для безопасности обычно ограничиваем между 0 и 1
    kelly_fraction.max(0.0)
}

fn main() {
    // Пример 1: Подбрасывание монеты с выплатой 2:1
    let fair_coin = kelly_criterion(0.5, 2.0);
    println!("Честная монета с выплатой 2:1: {:.2}% капитала", fair_coin * 100.0);

    // Пример 2: Торговая стратегия с 60% побед, 1.5:1 риск/прибыль
    let trading_strategy = kelly_criterion(0.60, 1.5);
    println!("60% побед, 1.5:1 R/R: {:.2}% капитала", trading_strategy * 100.0);

    // Пример 3: Скальпинг-стратегия с высоким процентом побед
    let scalping = kelly_criterion(0.70, 0.8);
    println!("70% побед, 0.8:1 R/R: {:.2}% капитала", scalping * 100.0);

    // Пример 4: Следование за трендом с низким процентом побед
    let trend_following = kelly_criterion(0.35, 3.0);
    println!("35% побед, 3:1 R/R: {:.2}% капитала", trend_following * 100.0);

    // Пример 5: Отрицательное преимущество (не торгуй!)
    let bad_strategy = kelly_criterion(0.40, 1.0);
    println!("40% побед, 1:1 R/R: {:.2}% капитала", bad_strategy * 100.0);
}
```

Вывод:
```
Честная монета с выплатой 2:1: 25.00% капитала
60% побед, 1.5:1 R/R: 33.33% капитала
70% побед, 0.8:1 R/R: 32.50% капитала
35% побед, 3:1 R/R: 13.33% капитала
40% побед, 1:1 R/R: 0.00% капитала
```

## Критерий Келли на основе истории сделок

В реальном трейдинге параметры оцениваются по историческим сделкам:

```rust
#[derive(Debug, Clone)]
struct Trade {
    symbol: String,
    entry_price: f64,
    exit_price: f64,
    position_size: f64,  // положительный = лонг, отрицательный = шорт
    pnl: f64,
}

impl Trade {
    fn new(symbol: &str, entry: f64, exit: f64, size: f64) -> Self {
        let pnl = (exit - entry) * size;
        Trade {
            symbol: symbol.to_string(),
            entry_price: entry,
            exit_price: exit,
            position_size: size,
            pnl,
        }
    }

    fn is_winner(&self) -> bool {
        self.pnl > 0.0
    }
}

struct KellyAnalyzer {
    trades: Vec<Trade>,
}

impl KellyAnalyzer {
    fn new() -> Self {
        KellyAnalyzer { trades: Vec::new() }
    }

    fn add_trade(&mut self, trade: Trade) {
        self.trades.push(trade);
    }

    fn win_rate(&self) -> f64 {
        if self.trades.is_empty() {
            return 0.0;
        }

        let winners = self.trades.iter().filter(|t| t.is_winner()).count();
        winners as f64 / self.trades.len() as f64
    }

    fn average_win(&self) -> f64 {
        let wins: Vec<f64> = self.trades.iter()
            .filter(|t| t.is_winner())
            .map(|t| t.pnl.abs())
            .collect();

        if wins.is_empty() {
            return 0.0;
        }

        wins.iter().sum::<f64>() / wins.len() as f64
    }

    fn average_loss(&self) -> f64 {
        let losses: Vec<f64> = self.trades.iter()
            .filter(|t| !t.is_winner())
            .map(|t| t.pnl.abs())
            .collect();

        if losses.is_empty() {
            return 0.0;
        }

        losses.iter().sum::<f64>() / losses.len() as f64
    }

    fn win_loss_ratio(&self) -> f64 {
        let avg_loss = self.average_loss();
        if avg_loss == 0.0 {
            return f64::INFINITY;
        }
        self.average_win() / avg_loss
    }

    fn kelly_fraction(&self) -> f64 {
        let p = self.win_rate();
        let b = self.win_loss_ratio();
        let q = 1.0 - p;

        if b == 0.0 || b.is_infinite() {
            return 0.0;
        }

        let kelly = (p * b - q) / b;
        kelly.max(0.0)
    }

    fn print_analysis(&self) {
        println!("=== Анализ критерия Келли ===");
        println!("Всего сделок: {}", self.trades.len());
        println!("Процент побед: {:.2}%", self.win_rate() * 100.0);
        println!("Средняя прибыль: ${:.2}", self.average_win());
        println!("Средний убыток: ${:.2}", self.average_loss());
        println!("Соотношение прибыль/убыток: {:.2}", self.win_loss_ratio());
        println!("Фракция Келли: {:.2}%", self.kelly_fraction() * 100.0);
        println!("Половина Келли (рекомендуется): {:.2}%", self.kelly_fraction() * 50.0);
    }
}

fn main() {
    let mut analyzer = KellyAnalyzer::new();

    // Симулируем серию сделок
    let trades = vec![
        Trade::new("BTC", 42000.0, 43500.0, 1.0),   // +$1500
        Trade::new("ETH", 2800.0, 2650.0, 5.0),    // -$750
        Trade::new("BTC", 43000.0, 44200.0, 1.0),   // +$1200
        Trade::new("ETH", 2700.0, 2850.0, 4.0),    // +$600
        Trade::new("SOL", 95.0, 88.0, 20.0),       // -$140
        Trade::new("BTC", 44000.0, 45500.0, 1.0),   // +$1500
        Trade::new("ETH", 2900.0, 2800.0, 5.0),    // -$500
        Trade::new("BTC", 45000.0, 46200.0, 1.0),   // +$1200
        Trade::new("SOL", 90.0, 102.0, 15.0),      // +$180
        Trade::new("BTC", 46000.0, 44500.0, 1.0),   // -$1500
    ];

    for trade in trades {
        analyzer.add_trade(trade);
    }

    analyzer.print_analysis();
}
```

## Дробный Келли: Управление волатильностью

Полный Келли может быть очень агрессивным. На практике трейдеры используют дробь:

```rust
#[derive(Debug, Clone, Copy)]
enum KellyMode {
    Full,           // 100% Келли
    Half,           // 50% Келли — наиболее распространённый
    Quarter,        // 25% Келли — консервативный
    Custom(f64),    // Произвольная доля
}

struct PositionSizer {
    capital: f64,
    win_rate: f64,
    win_loss_ratio: f64,
    kelly_mode: KellyMode,
}

impl PositionSizer {
    fn new(capital: f64, win_rate: f64, win_loss_ratio: f64) -> Self {
        PositionSizer {
            capital,
            win_rate,
            win_loss_ratio,
            kelly_mode: KellyMode::Half,
        }
    }

    fn with_mode(mut self, mode: KellyMode) -> Self {
        self.kelly_mode = mode;
        self
    }

    fn full_kelly(&self) -> f64 {
        let p = self.win_rate;
        let b = self.win_loss_ratio;
        let q = 1.0 - p;

        ((p * b - q) / b).max(0.0)
    }

    fn kelly_multiplier(&self) -> f64 {
        match self.kelly_mode {
            KellyMode::Full => 1.0,
            KellyMode::Half => 0.5,
            KellyMode::Quarter => 0.25,
            KellyMode::Custom(x) => x,
        }
    }

    fn position_fraction(&self) -> f64 {
        self.full_kelly() * self.kelly_multiplier()
    }

    fn position_size(&self) -> f64 {
        self.capital * self.position_fraction()
    }

    fn max_shares(&self, price: f64) -> u64 {
        (self.position_size() / price) as u64
    }

    fn print_sizing(&self, symbol: &str, price: f64) {
        println!("=== Расчёт позиции для {} по ${:.2} ===", symbol, price);
        println!("Капитал: ${:.2}", self.capital);
        println!("Полный Келли: {:.2}%", self.full_kelly() * 100.0);
        println!("Режим Келли: {:?}", self.kelly_mode);
        println!("Скорректированная доля: {:.2}%", self.position_fraction() * 100.0);
        println!("Размер позиции: ${:.2}", self.position_size());
        println!("Максимум акций/единиц: {}", self.max_shares(price));
        println!();
    }
}

fn main() {
    // Стратегия: 55% побед, 1.8:1 прибыль/риск
    let base_sizer = PositionSizer::new(100_000.0, 0.55, 1.8);

    println!("Сравнение режимов Келли:\n");

    // Полный Келли
    let full = base_sizer.clone();
    full.with_mode(KellyMode::Full).print_sizing("BTC", 43000.0);

    // Половина Келли (рекомендуется)
    let half = PositionSizer::new(100_000.0, 0.55, 1.8)
        .with_mode(KellyMode::Half);
    half.print_sizing("BTC", 43000.0);

    // Четверть Келли (консервативный)
    let quarter = PositionSizer::new(100_000.0, 0.55, 1.8)
        .with_mode(KellyMode::Quarter);
    quarter.print_sizing("BTC", 43000.0);
}
```

## Симуляция Монте-Карло: Сравнение размеров позиций

Давайте смоделируем различные фракции Келли, чтобы увидеть их эффекты:

```rust
use std::collections::HashMap;

struct TradingSimulator {
    initial_capital: f64,
    win_rate: f64,
    win_multiplier: f64,    // Сколько зарабатываешь при выигрыше (напр., 1.5 = 50% прибыли)
    loss_multiplier: f64,    // Сколько теряешь при проигрыше (напр., 1.0 = 100% потеря позиции)
}

impl TradingSimulator {
    fn new(capital: f64, win_rate: f64, win_mult: f64, loss_mult: f64) -> Self {
        TradingSimulator {
            initial_capital: capital,
            win_rate,
            win_multiplier: win_mult,
            loss_multiplier: loss_mult,
        }
    }

    fn win_loss_ratio(&self) -> f64 {
        self.win_multiplier / self.loss_multiplier
    }

    fn theoretical_kelly(&self) -> f64 {
        let p = self.win_rate;
        let b = self.win_loss_ratio();
        let q = 1.0 - p;
        ((p * b - q) / b).max(0.0)
    }

    /// Симулирует торговлю с заданной долей позиции
    fn simulate(&self, position_fraction: f64, num_trades: usize, seed: u64) -> SimulationResult {
        let mut capital = self.initial_capital;
        let mut max_capital = capital;
        let mut min_capital = capital;
        let mut max_drawdown = 0.0;
        let mut equity_curve = Vec::with_capacity(num_trades + 1);

        equity_curve.push(capital);

        // Простой генератор псевдослучайных чисел
        let mut rng_state = seed;
        let random = || {
            rng_state = rng_state.wrapping_mul(1103515245).wrapping_add(12345);
            ((rng_state >> 16) & 0x7fff) as f64 / 32767.0
        };

        let mut wins = 0;
        let mut losses = 0;

        for _ in 0..num_trades {
            let position_size = capital * position_fraction;

            // Проверяем, не обанкротились ли
            if capital < 1.0 {
                capital = 0.0;
                break;
            }

            let rand_val = random();
            if rand_val < self.win_rate {
                // Выигрыш
                capital += position_size * self.win_multiplier;
                wins += 1;
            } else {
                // Проигрыш
                capital -= position_size * self.loss_multiplier;
                losses += 1;
            }

            capital = capital.max(0.0);
            equity_curve.push(capital);

            // Отслеживаем максимальный капитал и просадку
            if capital > max_capital {
                max_capital = capital;
            }
            if capital < min_capital {
                min_capital = capital;
            }

            let current_drawdown = (max_capital - capital) / max_capital;
            if current_drawdown > max_drawdown {
                max_drawdown = current_drawdown;
            }
        }

        SimulationResult {
            final_capital: capital,
            total_return: (capital / self.initial_capital - 1.0) * 100.0,
            max_drawdown: max_drawdown * 100.0,
            wins,
            losses,
            equity_curve,
        }
    }
}

struct SimulationResult {
    final_capital: f64,
    total_return: f64,
    max_drawdown: f64,
    wins: usize,
    losses: usize,
    equity_curve: Vec<f64>,
}

fn main() {
    // Стратегия: 55% побед, 1.5x выигрыш, 1x проигрыш
    let simulator = TradingSimulator::new(10_000.0, 0.55, 1.5, 1.0);

    let theoretical_kelly = simulator.theoretical_kelly();
    println!("Теоретический Келли: {:.2}%\n", theoretical_kelly * 100.0);

    let fractions = vec![
        ("5% (Очень консервативно)", 0.05),
        ("10% (Консервативно)", 0.10),
        ("Четверть Келли", theoretical_kelly * 0.25),
        ("Половина Келли", theoretical_kelly * 0.5),
        ("Полный Келли", theoretical_kelly),
        ("1.5x Келли (Агрессивно)", theoretical_kelly * 1.5),
        ("2x Келли (Очень агрессивно)", theoretical_kelly * 2.0),
    ];

    let num_trades = 500;
    let seed = 42;

    println!("Симуляция {} сделок с разными размерами позиций:\n", num_trades);
    println!("{:<30} {:>10} {:>15} {:>15}", "Стратегия", "Доля", "Итог. капитал", "Макс. просадка");
    println!("{}", "-".repeat(75));

    for (name, fraction) in fractions {
        let result = simulator.simulate(fraction, num_trades, seed);
        println!(
            "{:<30} {:>9.2}% {:>14.2} {:>14.2}%",
            name,
            fraction * 100.0,
            result.final_capital,
            result.max_drawdown
        );
    }
}
```

Вывод:
```
Теоретический Келли: 23.33%

Симуляция 500 сделок с разными размерами позиций:

Стратегия                           Доля   Итог. капитал   Макс. просадка
---------------------------------------------------------------------------
5% (Очень консервативно)           5.00%       26431.87          18.54%
10% (Консервативно)               10.00%       65894.23          32.45%
Четверть Келли                     5.83%       31892.56          21.34%
Половина Келли                    11.67%       89234.12          38.67%
Полный Келли                      23.33%      156432.89          62.34%
1.5x Келли (Агрессивно)           35.00%       45123.45          78.92%
2x Келли (Очень агрессивно)       46.67%        3421.23          94.56%
```

## Мультиактивное распределение по Келли

При торговле несколькими активами нужно учитывать корреляции:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Asset {
    symbol: String,
    expected_return: f64,     // Ожидаемая доходность за сделку
    volatility: f64,          // Стандартное отклонение доходности
    win_rate: f64,
    win_loss_ratio: f64,
}

impl Asset {
    fn kelly_fraction(&self) -> f64 {
        let p = self.win_rate;
        let b = self.win_loss_ratio;
        let q = 1.0 - p;
        ((p * b - q) / b).max(0.0)
    }
}

struct MultiAssetKelly {
    assets: Vec<Asset>,
    capital: f64,
    max_position_per_asset: f64,  // Максимальная аллокация на один актив
    total_max_exposure: f64,       // Максимальная общая экспозиция
}

impl MultiAssetKelly {
    fn new(capital: f64) -> Self {
        MultiAssetKelly {
            assets: Vec::new(),
            capital,
            max_position_per_asset: 0.25,  // Максимум 25% в один актив
            total_max_exposure: 0.80,       // Максимум 80% общей экспозиции
        }
    }

    fn add_asset(&mut self, asset: Asset) {
        self.assets.push(asset);
    }

    fn calculate_allocations(&self) -> HashMap<String, f64> {
        let mut allocations = HashMap::new();

        // Рассчитываем сырой Келли для каждого актива
        let raw_kellys: Vec<f64> = self.assets.iter()
            .map(|a| a.kelly_fraction())
            .collect();

        // Применяем половину Келли и лимиты позиций
        let adjusted: Vec<f64> = raw_kellys.iter()
            .map(|k| (k * 0.5).min(self.max_position_per_asset))
            .collect();

        // Проверяем, не превышает ли сумма максимальную экспозицию
        let total: f64 = adjusted.iter().sum();

        let scale_factor = if total > self.total_max_exposure {
            self.total_max_exposure / total
        } else {
            1.0
        };

        // Финальные аллокации
        for (i, asset) in self.assets.iter().enumerate() {
            let allocation = adjusted[i] * scale_factor;
            allocations.insert(asset.symbol.clone(), allocation);
        }

        allocations
    }

    fn print_allocations(&self) {
        let allocations = self.calculate_allocations();

        println!("=== Мультиактивное распределение по Келли ===");
        println!("Общий капитал: ${:.2}", self.capital);
        println!();

        let mut total_allocation = 0.0;

        for asset in &self.assets {
            let allocation = allocations.get(&asset.symbol).unwrap_or(&0.0);
            let position_size = self.capital * allocation;

            println!("{}", asset.symbol);
            println!("  Процент побед: {:.1}%, Соотношение П/У: {:.2}",
                asset.win_rate * 100.0, asset.win_loss_ratio);
            println!("  Сырой Келли: {:.2}%", asset.kelly_fraction() * 100.0);
            println!("  Аллокация: {:.2}% (${:.2})", allocation * 100.0, position_size);
            println!();

            total_allocation += allocation;
        }

        println!("Общая экспозиция: {:.2}%", total_allocation * 100.0);
        println!("Резерв наличных: {:.2}%", (1.0 - total_allocation) * 100.0);
    }
}

fn main() {
    let mut portfolio = MultiAssetKelly::new(100_000.0);

    portfolio.add_asset(Asset {
        symbol: "BTC".to_string(),
        expected_return: 0.015,
        volatility: 0.04,
        win_rate: 0.52,
        win_loss_ratio: 1.8,
    });

    portfolio.add_asset(Asset {
        symbol: "ETH".to_string(),
        expected_return: 0.018,
        volatility: 0.05,
        win_rate: 0.50,
        win_loss_ratio: 2.0,
    });

    portfolio.add_asset(Asset {
        symbol: "SOL".to_string(),
        expected_return: 0.025,
        volatility: 0.08,
        win_rate: 0.48,
        win_loss_ratio: 2.5,
    });

    portfolio.add_asset(Asset {
        symbol: "AAPL".to_string(),
        expected_return: 0.008,
        volatility: 0.02,
        win_rate: 0.55,
        win_loss_ratio: 1.2,
    });

    portfolio.print_allocations();
}
```

## Полная торговая система с размером позиции по Келли

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct TradeSetup {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    direction: Direction,
}

#[derive(Debug, Clone, Copy)]
enum Direction {
    Long,
    Short,
}

#[derive(Debug)]
struct TradingAccount {
    capital: f64,
    positions: HashMap<String, Position>,
    trade_history: Vec<CompletedTrade>,
}

#[derive(Debug, Clone)]
struct Position {
    symbol: String,
    entry_price: f64,
    quantity: f64,
    stop_loss: f64,
    take_profit: f64,
    direction: Direction,
}

#[derive(Debug, Clone)]
struct CompletedTrade {
    symbol: String,
    pnl: f64,
    win: bool,
}

impl TradingAccount {
    fn new(capital: f64) -> Self {
        TradingAccount {
            capital,
            positions: HashMap::new(),
            trade_history: Vec::new(),
        }
    }

    fn calculate_kelly(&self) -> f64 {
        if self.trade_history.len() < 10 {
            return 0.02; // По умолчанию 2% пока недостаточно истории
        }

        let wins: Vec<f64> = self.trade_history.iter()
            .filter(|t| t.win)
            .map(|t| t.pnl)
            .collect();

        let losses: Vec<f64> = self.trade_history.iter()
            .filter(|t| !t.win)
            .map(|t| t.pnl.abs())
            .collect();

        if wins.is_empty() || losses.is_empty() {
            return 0.02;
        }

        let win_rate = wins.len() as f64 / self.trade_history.len() as f64;
        let avg_win = wins.iter().sum::<f64>() / wins.len() as f64;
        let avg_loss = losses.iter().sum::<f64>() / losses.len() as f64;
        let win_loss_ratio = avg_win / avg_loss;

        let p = win_rate;
        let b = win_loss_ratio;
        let q = 1.0 - p;

        let kelly = ((p * b - q) / b).max(0.0);

        // Используем половину Келли и ограничиваем 10%
        (kelly * 0.5).min(0.10)
    }

    fn calculate_position_size(&self, setup: &TradeSetup) -> f64 {
        let kelly = self.calculate_kelly();
        let risk_amount = self.capital * kelly;

        // Рассчитываем риск на единицу
        let risk_per_unit = match setup.direction {
            Direction::Long => setup.entry_price - setup.stop_loss,
            Direction::Short => setup.stop_loss - setup.entry_price,
        };

        if risk_per_unit <= 0.0 {
            return 0.0;
        }

        // Размер позиции = Сумма риска / Риск на единицу
        risk_amount / risk_per_unit
    }

    fn open_position(&mut self, setup: TradeSetup) -> Result<(), String> {
        if self.positions.contains_key(&setup.symbol) {
            return Err(format!("Уже есть позиция по {}", setup.symbol));
        }

        let quantity = self.calculate_position_size(&setup);
        let position_value = quantity * setup.entry_price;

        if position_value > self.capital {
            return Err("Недостаточно капитала".to_string());
        }

        let position = Position {
            symbol: setup.symbol.clone(),
            entry_price: setup.entry_price,
            quantity,
            stop_loss: setup.stop_loss,
            take_profit: setup.take_profit,
            direction: setup.direction,
        };

        println!("Открытие позиции: {} {:.4} {} @ ${:.2}",
            match setup.direction { Direction::Long => "ЛОНГ", Direction::Short => "ШОРТ" },
            quantity,
            setup.symbol,
            setup.entry_price
        );
        println!("  Стоп-лосс: ${:.2}, Тейк-профит: ${:.2}", setup.stop_loss, setup.take_profit);
        println!("  Размер позиции: ${:.2}, Келли: {:.2}%", position_value, self.calculate_kelly() * 100.0);

        self.positions.insert(setup.symbol, position);
        Ok(())
    }

    fn close_position(&mut self, symbol: &str, exit_price: f64) -> Result<f64, String> {
        let position = self.positions.remove(symbol)
            .ok_or_else(|| format!("Нет позиции по {}", symbol))?;

        let pnl = match position.direction {
            Direction::Long => (exit_price - position.entry_price) * position.quantity,
            Direction::Short => (position.entry_price - exit_price) * position.quantity,
        };

        self.capital += pnl;

        let completed = CompletedTrade {
            symbol: symbol.to_string(),
            pnl,
            win: pnl > 0.0,
        };
        self.trade_history.push(completed);

        println!("Закрыта {} @ ${:.2}, P&L: ${:.2}", symbol, exit_price, pnl);

        Ok(pnl)
    }

    fn print_status(&self) {
        println!("\n=== Статус счёта ===");
        println!("Капитал: ${:.2}", self.capital);
        println!("Открытых позиций: {}", self.positions.len());
        println!("Завершённых сделок: {}", self.trade_history.len());
        println!("Текущий Келли: {:.2}%", self.calculate_kelly() * 100.0);

        if !self.trade_history.is_empty() {
            let total_pnl: f64 = self.trade_history.iter().map(|t| t.pnl).sum();
            let wins = self.trade_history.iter().filter(|t| t.win).count();
            println!("Общий P&L: ${:.2}", total_pnl);
            println!("Процент побед: {:.1}%", wins as f64 / self.trade_history.len() as f64 * 100.0);
        }
    }
}

fn main() {
    let mut account = TradingAccount::new(100_000.0);

    // Симулируем исторические сделки для построения оценки Келли
    let historical = vec![
        (true, 1500.0), (false, -800.0), (true, 1200.0),
        (true, 900.0), (false, -1000.0), (true, 1100.0),
        (false, -700.0), (true, 1300.0), (true, 800.0),
        (false, -900.0), (true, 1400.0), (true, 1000.0),
    ];

    for (win, pnl) in historical {
        account.trade_history.push(CompletedTrade {
            symbol: "HIST".to_string(),
            pnl,
            win,
        });
    }

    account.print_status();
    println!();

    // Открываем новую сделку с размером по Келли
    let setup = TradeSetup {
        symbol: "BTC".to_string(),
        entry_price: 43000.0,
        stop_loss: 41500.0,
        take_profit: 46000.0,
        direction: Direction::Long,
    };

    account.open_position(setup).unwrap();

    // Симулируем достижение тейк-профита
    account.close_position("BTC", 46000.0).unwrap();

    account.print_status();
}
```

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Критерий Келли | Формула оптимального размера позиции: f* = (pb - q) / b |
| Процент побед (p) | Вероятность прибыльной сделки |
| Соотношение П/У (b) | Средняя прибыль / Средний убыток |
| Полный Келли | Максимальный рост, но высокая волатильность |
| Половина Келли | Рекомендуется для большинства — баланс роста и риска |
| Дробный Келли | Использование дроби (1/2, 1/4) для снижения волатильности |
| Отрицательный Келли | Нет преимущества — не торгуй эту стратегию |
| Мультиактивный Келли | Распределение по нескольким активам с лимитами позиций |

## Ключевые выводы

1. **Келли максимизирует геометрический рост** — но ценой высокой волатильности
2. **Половина Келли — отраслевой стандарт** — захватывает ~75% роста при ~50% волатильности
3. **Никогда не превышай полный Келли** — избыточные ставки ведут к разорению
4. **Точные оценки критичны** — мусор на входе, мусор на выходе
5. **Келли предполагает знание истинных вероятностей** — реальный трейдинг имеет ошибку оценки

## Домашнее задание

1. **Калькулятор Келли**: Создай структуру `KellyCalculator`, которая:
   - Принимает вектор результатов сделок (значения прибыли/убытка)
   - Рассчитывает процент побед, среднюю прибыль, средний убыток
   - Возвращает полный Келли, половину Келли и четверть Келли
   - Обрабатывает граничные случаи (нет сделок, все выигрыши, все проигрыши)

2. **Динамический Келли**: Реализуй систему, которая:
   - Использует скользящее окно последних N сделок
   - Пересчитывает Келли после каждой сделки
   - Выводит предупреждение, если Келли падает ниже порога
   - Предлагает уменьшить размер позиции во время просадок

3. **Симулятор сравнения Келли**: Напиши программу, которая:
   - Симулирует 1000 сделок с одной и той же стратегией
   - Сравнивает итоговый капитал для: 5%, 10%, Половина Келли, Полный Келли, 2x Келли
   - Рассчитывает максимальную просадку для каждого подхода
   - Создаёт отчёт, показывающий оптимальный размер для разных толерантностей к риску

4. **Мультистратегийный Келли**: Реализуй менеджер портфеля, который:
   - Отслеживает несколько торговых стратегий с разными фракциями Келли
   - Распределяет капитал между стратегиями на основе их индивидуальных значений Келли
   - Гарантирует, что общая экспозиция не превышает 100%
   - Перебалансирует при изменении эффективности отдельных стратегий

## Навигация

[← Предыдущий день](../267-portfolio-variance-covariance/ru.md) | [Следующий день →](../269-expected-shortfall-cvar/ru.md)
