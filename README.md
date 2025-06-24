# algo-trading-bot
How the Bot Works: The Overall Flow
The bot operates through several interconnected components, working in a supervisor-worker architecture:

Initialization and Configuration (Cells 0-2):

First, it ensures all necessary Python libraries are installed.
It loads all operational parameters from a config.yaml file and environment variables, including API keys, trading symbols, technical indicator periods, model parameters, and risk management rules.
It initializes a custom logger to track all activities in Indian Standard Time (IST).
It establishes a global asyncio.Lock (bot_state_lock) to manage concurrent access to shared global state variables, crucial for preventing race conditions.
Data Management (Cell 4):

Historical Data Update: For each selected trading symbol, the bot maintains a local SQLite database of historical market data. It intelligently updates this data by fetching recent candles from the Upstox API. This process handles pagination, retries, and merges new data while pruning old records to maintain a specified lookback period.
Feature Engineering: It calculates a comprehensive set of technical indicators (like SMA, EMA, RSI, MACD, ATR, Bollinger Bands) and advanced price action/Smart Money Concepts (SMC) patterns (like Order Blocks, Engulfing patterns, Liquidity Sweeps, Institutional trading patterns, Market Character, and Market Sentiment) on the historical data. These form the features for the deep learning model.
Data Augmentation: A unique adaptive learning mechanism. It processes its own trade_logs (records of past live trades) to extract performance insights (win rate, PnL, SL/TP hit frequencies). For profitable trades, it duplicates the historical data point. For significant losing trades, it might generate a "contrary" augmented sample to teach the model from its mistakes, giving these augmented samples a higher weight during training.
Model Training and Tuning (Cells 5-6):

Model Architecture: The bot employs an advanced deep learning model, a hybrid of Temporal Convolutional Networks (TCN), Bidirectional LSTMs (BiLSTM), and Multi-Head Attention layers. This architecture is designed to capture complex temporal dependencies and patterns in time-series data.
Hyperparameter Tuning: Using KerasTuner, the bot can automatically search for the optimal hyperparameters for its model (e.g., number of layers, units, learning rate). Crucially, this tuning process uses a TradingMetricsCallback that simulates trades on validation data during training, allowing the tuner to optimize directly for risk-adjusted returns (Sharpe Ratio) rather than just abstract accuracy.
Model Training: The model is trained using processed and augmented historical data. It supports Walk-Forward Validation (WFV) or standard train/test splits, and can train an ensemble of models for more robust predictions. Early Stopping and ReduceLROnPlateau callbacks are used for efficient training.
Strategy Adaptation: After analyzing live trade performance, the adapt_strategy_parameters_for_symbol function dynamically adjusts the Stop-Loss (SL) and Take-Profit (TP) ATR multipliers for each symbol. For example, if a symbol has a low win rate and high SL hit frequency, its SL multiplier might be increased (loosening the SL).
Live Trading (Cell 8 - The "Brain"):

Supervisor-Worker Architecture: The live trading system runs under a run_adv_live_trading_supervisor function that orchestrates multiple asyncio worker tasks concurrently. This provides robust fault tolerance with exponential backoff and a "circuit breaker" that can shut down the bot after too many consecutive failures.
State Persistence: Before initiating live trading, the bot loads its previous state (open positions, PnL, daily limits) from a live_bot_state.json file. It also performs a reconciliation with the broker's actual open positions to ensure internal state is accurate after any potential bot restarts or crashes.
API Rate Limiting: A RateLimiter (token bucket algorithm) proactively manages API calls to prevent exceeding broker rate limits.
Real-time Data Ingestion (data_ingestion_task): This worker connects to the Upstox WebSocket to receive live tick data. It then aggregates these ticks into 10-second "micro-candles" and then into full 1-minute candles. It pushes these completed candles onto an asyncio.Queue for the signal generation task, and constantly updates the latest_ltp_by_symbol.
Signal Generation (signal_generation_task): This worker processes signals at a controlled LIVE_PROCESSING_INTERVAL_SECONDS (e.g., every 5 seconds). It prioritizes newly completed 1-minute candles from the queue; if none are available, it uses the latest partial micro-candle. It then calculates features for this latest candle and feeds it to the trained deep learning model to get a BUY/HOLD/SELL prediction and confidence score.
Order Management (order_polling_task): This worker continuously polls the Upstox API for the status of all pending entry and exit orders. It confirms order fills, processes rejections/cancellations, updates the bot's internal PnL and position state, and can even force-exit "stuck" orders that remain pending for too long.
Position Monitoring (position_monitor_task): This worker frequently checks the latest live price against the calculated Stop-Loss (SL) and Take-Profit (TP) levels for all open positions. If a level is hit, it triggers an exit order.
Command Line Interface (CLI) (Cell 9):

Provides an interactive menu for the user to initiate various operations: loading data, tuning hyperparameters, training models, running backtests, and starting live trading (alerts-only or auto-order execution).
Displays real-time status of the portfolio, symbols, PnL, and daily limits.
Includes a "PANIC!" button for emergency liquidation of all positions.
Features automated retraining for symbols that have been halted due to poor consecutive performance.
How the Bot Orders: Conditions and Logic
Orders are placed via the place_upstox_order_live function.

Entry Conditions (When to Buy/Sell to Open a Position):

Signal Confidence: The model's prediction (BUY or SELL) must have a prediction_confidence score greater than or equal to CONFIDENCE_THRESHOLD_TRADE (e.g., 90% or 98% confident).
No Existing Position: The bot will only attempt a new entry if there is no current open position for that symbol (current_position == 'None') and no pending_entry_order_id.
Daily Trade Limits: The symbol must not have exceeded its MAX_TRADES_PER_SYMBOL_PER_DAY, and the overall portfolio must not have exceeded MAX_TRADES_PER_DAY_GLOBAL.
Trading Halts: Neither global trading nor the specific symbol should be is_trading_halted_for_day_global or is_halted_for_performance.
Time-Based Rules: It must be within the allowed trading hours (is_market_open_now_live) and after MIN_ENTRY_TIME_AFTER_OPEN_STR (e.g., don't trade immediately at market open) and before NO_NEW_ENTRY_AFTER_TIME_STR (e.g., no new entries near market close).
Capital Allocation: Sufficient capital must be allocated to the symbol for the trade based on capital_per_symbol_allowance.
Order Quantity: A valid order_qty greater than 0 must be calculated based on calculate_dynamic_order_quantity_live (considering current price, allocated capital, margin utilization, and leverage).
Order Type: Entry orders are defaulted to ENTRY_ORDER_TYPE_DEFAULT (e.g., "LIMIT").
For LIMIT entry orders, the bot dynamically calculates the price by adding a small ENTRY_LIMIT_PRICE_BUFFER_PCT (e.g., 0.05%) to the current Last Traded Price (LTP) for BUY orders, or subtracting it for SELL orders. This aims to improve fill probability for limit orders.
Exit Conditions (When to Sell/Buy to Close a Position):

Stop-Loss (SL) Hit: If the current Live Traded Price (LTP) crosses the calculated current_sl_price.
Take-Profit (TP) Hit: If the current LTP crosses the calculated current_tp_price.
End-of-Day (EOD) Square-Off: If the market enters the SQUARE_OFF_ALL_START_TIME_STR to SQUARE_OFF_ALL_END_TIME_STR window, all open positions will be closed.
Forced Exit (Stuck Orders): If a pending_exit_order_id remains unfulfilled for a prolonged period (EXIT_TIMEOUT_SECONDS), the bot attempts to cancel it and place a new MARKET order to force the exit.
Global Daily Loss Limit: If the portfolio_daily_pnl_achieved reaches the calculated_max_daily_loss_global, all trading (including new entries and potentially forcing exits) will halt for the day.
Panic Button: Manual user intervention via the CLI's 'X' option can trigger an immediate market exit for all positions and cancel all pending orders.
Order Type: Exit orders typically use EXIT_ORDER_TYPE (e.g., "MARKET") to ensure quick execution.
Stop-Loss (SL) and Take-Profit (TP) Calculation:

ATR-Based: SL and TP levels are dynamically calculated based on the Average True Range (ATR) at the time of entry. ATR is a measure of market volatility.
Multipliers:
current_sl_price = entry_price - (atr_at_entry * current_sl_atr_multiplier) for Long positions.
current_sl_price = entry_price + (atr_at_entry * current_sl_atr_multiplier) for Short positions.
current_tp_price = entry_price + (atr_at_entry * current_tp_atr_multiplier) for Long positions.
current_tp_price = entry_price - (atr_at_entry * current_tp_atr_multiplier) for Short positions.
Adaptation: The current_sl_atr_multiplier and current_tp_atr_multiplier are not fixed. They are dynamically adapted by the adapt_strategy_parameters_for_symbol function (Cell 6) based on the bot's observed performance (win rate, SL hit frequency, TP hit frequency) from its past live trades. This ensures the bot learns and adjusts its risk parameters over time.
In essence, the bot is a self-improving, fault-tolerant trading system that makes decisions by processing vast amounts of market data through deep learning, adheres to strict risk management rules, and is capable of autonomous live order execution and position management.
