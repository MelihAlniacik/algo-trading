# Statistical Trading with Python: Build an Advanced Mean Reversion Bot

> **DISCLAIMER:** This repository and the accompanying code are provided strictly for educational, research, and informational purposes only. This is **not** financial advice. Algorithmic trading involves extreme risk. Do not use this code to trade with real capital.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [The Quantitative Edge: Mean Reversion](#2-the-quantitative-edge-mean-reversion)
3. [Prerequisites & Setup](#3-prerequisites--setup)
4. [System Architecture: The Code](#4-system-architecture-the-code)
5. [The Art of Survival: Risk Management](#5-the-art-of-survival-risk-management)
6. [Next Steps & Improvements](#6-next-steps--improvements)
7. [Advanced Position Sizing: The Kelly Criterion](#7-advanced-position-sizing-the-kelly-criterion)
8. [High-Frequency Upgrades: WebSocket Integration](#8-high-frequency-upgrades-websocket-integration)
9. [Legal & Risk Disclaimer](#9-legal--risk-disclaimer)

---

## 1. Introduction

The internet is flooded with "make money while you sleep" promises regarding algorithmic trading. However, the reality of quantitative finance is a ruthless battlefield. When you deploy a trading bot, you aren't just trading against the market — you are competing against institutional funds, high-frequency algorithms, and hyper-optimized infrastructure.

Algorithmic trading is not about finding a magical money-printing machine. It's about discovering a mathematical edge (alpha) within the chaos of the market, managing risk strictly, and executing trades with emotionless discipline.

In this comprehensive guide, we will build a robust, Object-Oriented (OOP) trading bot in Python. Using `ccxt` and `pandas`, we will construct a system that detects statistical anomalies in the market, calculates dynamic risk, and executes trades on a cryptocurrency Testnet environment.

---

## 2. The Quantitative Edge: Mean Reversion

Financial markets are heavily driven by human psychology — primarily fear and greed. This causes asset prices to frequently overreact, stretching far beyond their fundamental or historical baseline. The **Mean Reversion** strategy hypothesizes that these extreme price movements are temporary. Like a stretched rubber band, prices will eventually snap back to their historical average.

### The Math: Z-Score and Volatility Normalization

To objectively measure statistical deviation rather than relying on visual chart patterns, quantitative analysts use the **Z-Score**.

Why not just use a fixed percentage drop? Because a 5% drop in Bitcoin is ordinary, but a 5% drop in the S&P 500 is a market crash. The Z-Score dynamically normalizes the price deviation based on the asset's current volatility (Standard Deviation).

The formula is:

$$Z = \frac{x - \mu}{\sigma}$$

Where:

- $x$ = The current closing price of the asset
- $\mu$ = The Simple Moving Average (SMA) over a specific rolling window
- $\sigma$ = The Standard Deviation over that same window

### Interpreting the Signal

| Z-Score | Interpretation | Action |
|---------|---------------|--------|
| $Z \leq -2.0$ | Oversold Anomaly — price is more than 2 standard deviations below the mean | **Buy Signal** |
| $Z \geq +2.0$ | Overbought Anomaly — price is more than 2 standard deviations above the mean | **Sell / Short Signal** |
| $Z \approx 0$ | Equilibrium — price has reverted to its historical mean | **Close Position** |

> **Important:** The Z-Score assumes a roughly normal distribution. In practice, financial price returns exhibit **fat tails** (leptokurtosis), meaning extreme moves occur more frequently than a normal distribution would predict. The Z-Score is a useful signal, not a guarantee of reversion.

---

## 3. Prerequisites & Setup

### Install Dependencies

```bash
pip install ccxt pandas
pip install "ccxt[pro]"   # Required for WebSocket streams in Section 8
```

### Secure Your API Keys

Never hardcode credentials in your source files. Use environment variables:

```bash
# .env file (add this to .gitignore!)
OKX_API_KEY=your_api_key_here
OKX_SECRET=your_secret_here
OKX_PASSWORD=your_passphrase_here
```

```python
import os
from dotenv import load_dotenv

load_dotenv()

api_key  = os.getenv("OKX_API_KEY")
secret   = os.getenv("OKX_SECRET")
password = os.getenv("OKX_PASSWORD")
```

### Create a Testnet Account

Never test new algorithms with real capital. Open a Testnet (Sandbox) account on an exchange like **OKX** or **Binance**, generate your API Keys, and pass them to the bot.

---

## 4. System Architecture: The Code

We use Object-Oriented Programming (OOP) to manage state, handle errors gracefully, and keep the system extensible.

```python
import ccxt
import pandas as pd
import time
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

class QuantitativeBot:
    def __init__(self, api_key, secret, password, symbol='BTC/USDT', timeframe='15m', period=20):
        self.symbol    = symbol
        self.timeframe = timeframe
        self.period    = period
        self.risk_pct  = 0.02  # 2% risk per trade (see Section 5)

        # Initialize Exchange in Sandbox/Testnet Mode
        self.exchange = ccxt.okx({
            'apiKey':          api_key,
            'secret':          secret,
            'password':        password,
            'enableRateLimit': True,
        })
        self.exchange.set_sandbox_mode(True)

        # State Management
        self.in_position   = False
        self.entry_price   = 0.0
        self.position_size = 0.0

    def fetch_data_and_analyze(self):
        """Fetches OHLCV data and calculates the Z-Score."""
        try:
            ohlcv = self.exchange.fetch_ohlcv(self.symbol, self.timeframe, limit=self.period + 10)
            df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])

            df['sma']     = df['close'].rolling(window=self.period).mean()
            df['std']     = df['close'].rolling(window=self.period).std()
            # Replace std=0 to avoid ZeroDivisionError (e.g. during flat markets)
            df['z_score'] = (df['close'] - df['sma']) / df['std'].replace(0, float('nan'))

            last = df.iloc[-1]

            # Guard against NaN z_score if not enough data yet
            if pd.isna(last['z_score']):
                logging.warning("Z-Score is NaN — insufficient data for current period.")
                return None

            return last

        except Exception as e:
            logging.error(f"Data fetching error: {e}")
            return None

    def execute_trade(self, side, price, balance):
        """
        Mock execution with dynamic position sizing.
        In production, replace with self.exchange.create_order(...).
        """
        if side == 'buy':
            usdt_to_spend      = balance * self.risk_pct
            self.position_size = usdt_to_spend / price

            logging.info(f"EXECUTING BUY:  {self.position_size:.5f} BTC @ {price:.2f} USDT")
            self.entry_price = price
            self.in_position = True

        elif side == 'sell':
            pnl = (price - self.entry_price) * self.position_size
            logging.info(f"EXECUTING SELL: @ {price:.2f} USDT | PnL ≈ {pnl:.2f} USDT")
            self.in_position   = False
            self.entry_price   = 0.0
            self.position_size = 0.0

    def run(self):
        """Main event loop."""
        logging.info(f"Starting QuantitativeBot on {self.symbol} [{self.timeframe}]...")

        # Mock balance — replace with self.exchange.fetch_balance() in production
        mock_balance = 10_000

        while True:
            data = self.fetch_data_and_analyze()

            if data is not None:
                price   = data['close']
                z_score = data['z_score']

                logging.info(f"Price: {price:.2f} USDT | Z-Score: {z_score:.4f}")

                if z_score < -2.0 and not self.in_position:
                    logging.info(f"Oversold anomaly detected (Z={z_score:.2f}). Initiating BUY.")
                    self.execute_trade('buy', price, mock_balance)

                elif z_score >= 0.0 and self.in_position:
                    logging.info(f"Mean reversion confirmed (Z={z_score:.2f}). Initiating SELL.")
                    self.execute_trade('sell', price, mock_balance)

            time.sleep(60)  # Poll once per minute; replace with WebSocket for lower latency


if __name__ == "__main__":
    import os
    from dotenv import load_dotenv
    load_dotenv()

    bot = QuantitativeBot(
        api_key  = os.getenv("OKX_API_KEY"),
        secret   = os.getenv("OKX_SECRET"),
        password = os.getenv("OKX_PASSWORD"),
    )
    bot.run()
```

---

## 5. The Art of Survival: Risk Management

The most elegantly coded model will still blow up your account if you ignore risk management.

### The 2% Rule

The line `self.risk_pct = 0.02` enforces the golden rule: **never risk more than 1–2% of your total equity on a single trade**.

Drawdown mathematics is asymmetric:

| Account Loss | Gain Required to Recover |
|-------------|--------------------------|
| 10%         | 11%                      |
| 25%         | 33%                      |
| 50%         | 100%                     |
| 75%         | 300%                     |

By risking only 2% per trade, you can endure a 10-trade losing streak and still retain over 80% of your capital — giving your algorithm time to recover.

### Stop-Loss Mechanisms

Mean Reversion assumes prices will bounce back, but sometimes a breakdown is **structural** (e.g., a catastrophic news event or exchange insolvency). A production bot must include a hard Stop-Loss that exits the trade if the price moves a certain percentage against the position, rather than waiting indefinitely for a reversion that may never come.

```python
STOP_LOSS_PCT = 0.05  # Exit if price drops 5% below entry

if self.in_position:
    drawdown = (price - self.entry_price) / self.entry_price
    if drawdown <= -STOP_LOSS_PCT:
        logging.warning(f"Stop-loss triggered at {drawdown:.1%}. Exiting position.")
        self.execute_trade('sell', price, mock_balance)
```

---

## 6. Next Steps & Improvements

This architecture provides a solid foundation. A production-ready algorithm requires further optimization:

1. **WebSocket Integration:** Replace `fetch_ohlcv` polling with real-time WebSocket streams (see Section 8).
2. **Backtesting:** Before deploying on a live testnet, validate the logic against historical data. Recommended frameworks: [`vectorbt`](https://github.com/polakowo/vectorbt) or [`backtesting.py`](https://github.com/kernc/backtesting.py).
3. **Kelly Criterion:** Upgrade the static 2% risk rule to a dynamic position sizer based on the system's historical win rate and risk/reward ratio (see Section 7).
4. **Multi-Symbol Support:** Run independent instances per trading pair, each with isolated state.
5. **Alerting:** Integrate Telegram or Discord notifications for trade events and errors.

---

## 7. Advanced Position Sizing: The Kelly Criterion

A static 2% risk rule is simple and safe, but mathematically suboptimal for long-term compound growth. If your algorithm is outperforming, risking only 2% leaves money on the table. If conditions deteriorate and the win rate drops, continuing to risk 2% drains your account unnecessarily.

Quantitative funds solve this with the **Kelly Criterion** — developed by John L. Kelly Jr. at Bell Labs — which dictates the mathematically optimal fraction of your bankroll to risk on each trade.

### The Kelly Formula

The standard Kelly formula for a trading system is:

$$K = \frac{W \cdot R - (1 - W)}{R}$$

Where:

- $K$ = Optimal fraction of capital to risk (Kelly Fraction)
- $W$ = Win Rate of your algorithm (e.g., `0.55` for 55%)
- $R$ = Risk/Reward Ratio = Average profit per winning trade ÷ Average loss per losing trade

### Python Implementation

```python
def calculate_kelly_fraction(self, historical_win_rate: float, average_profit: float, average_loss: float) -> float:
    """
    Calculates the optimal risk percentage using the Kelly Criterion.

    Args:
        historical_win_rate: Fraction of trades that were profitable (e.g., 0.55).
        average_profit:      Mean profit of winning trades (positive number).
        average_loss:        Mean loss of losing trades (positive number).

    Returns:
        Recommended fraction of capital to risk (capped at 5%).
    """
    if average_loss == 0:
        return 0.0  # Prevent division by zero

    reward_risk_ratio = average_profit / average_loss

    # Standard Kelly formula
    kelly_pct = (historical_win_rate * reward_risk_ratio - (1 - historical_win_rate)) / reward_risk_ratio

    if kelly_pct <= 0:
        logging.warning("Kelly Criterion suggests 0% risk — system edge lost. Halting trades.")
        return 0.0

    # Half-Kelly: industry standard to reduce variance while preserving most of the growth benefit
    half_kelly = kelly_pct / 2.0

    # Hard cap at 5% maximum risk per trade
    return min(half_kelly, 0.05)
```

Substitute `self.risk_pct` with the output of this method before each trade, feeding it updated statistics from your trade log.

> **Note:** The Kelly Criterion requires a reliable historical sample (≥ 30–50 trades) to produce meaningful estimates. Running it on too few trades will overfit to noise.

---

## 8. High-Frequency Upgrades: WebSocket Integration

Polling a REST API every 60 seconds is sufficient for educational purposes or slow swing trading. In a live market, however, structural breakdowns and anomalies can develop in milliseconds. To react instantly, your bot must transition from REST polling to **WebSockets** — a persistent, server-push connection.

### Installation

```bash
pip install "ccxt[pro]"
```

### Asynchronous Implementation with CCXT Pro

```python
import ccxt.pro as ccxtpro
import asyncio
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')

class AsyncQuantitativeBot:
    def __init__(self, symbol='BTC/USDT', timeframe='1m'):
        self.symbol    = symbol
        self.timeframe = timeframe

        self.exchange = ccxtpro.okx({'enableRateLimit': True})
        self.exchange.set_sandbox_mode(True)

    async def watch_market_data(self):
        """Streams real-time candlestick data via WebSocket without polling."""
        logging.info(f"Connecting to WebSocket stream for {self.symbol}...")

        while True:
            try:
                # Execution pauses here until the exchange pushes a live candle update
                ohlcv = await self.exchange.watch_ohlcv(self.symbol, self.timeframe)

                # watch_ohlcv returns a list of candles; the most recent is the last element
                latest_candle = ohlcv[-1]  # [timestamp, open, high, low, close, volume]
                current_price = latest_candle[4]

                logging.info(f"Real-time update: {current_price:.2f} USDT")

                # Inject Z-Score calculation and Kelly-based trade execution logic here

            except Exception as e:
                logging.error(f"WebSocket error: {e}. Reconnecting in 5 seconds...")
                await asyncio.sleep(5)

    async def run(self):
        try:
            await self.watch_market_data()
        finally:
            await self.exchange.close()
            logging.info("WebSocket connection closed cleanly.")


if __name__ == "__main__":
    bot = AsyncQuantitativeBot()
    asyncio.run(bot.run())
```

---

## 9. Legal & Risk Disclaimer

**1. Not Financial Advice:**
The information, code, algorithms, and strategies in this repository do not constitute financial, investment, or trading advice of any kind. Conduct your own due diligence and consult a licensed financial advisor before making any investment decisions.

**2. High Risk of Capital Loss:**
Algorithmic trading — especially in cryptocurrency markets — carries a high level of risk and may not be suitable for all investors. There is a real possibility of losing some or all of your initial investment. Never trade with funds you cannot afford to lose.

**3. "As-Is" Software Clause:**
This code is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and non-infringement. The authors, contributors, and copyright holders bear no liability for any claim, damages, or other liability arising from the use of this software.

**4. API Key Security:**
You are solely responsible for the security of your exchange API keys. **Never commit API keys, secrets, or passwords to a public repository.** Always use environment variables (`.env` files, never committed to version control) to manage sensitive credentials.
