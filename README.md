# Crypto-Day-Trading-Algo

This Pine Script indicator (with the current parameters) was designed primarily for trading Bitcoin on TradingView with strategy automation via 3Commas and your cryptocurrency exchange account API (note that 3Commas only supports centralized exchanges). For this project, I created a custom signal bot within the 3Commas platform. The script sends JSON webhook signals from TradingView's `alert()` function calls to 3Commas. These signals include a dynamically calculated buy quantity (in the trading pair's base currency) and the current market price, enabling automated trading for cryptocurrencies.

The indicator combines multiple technical indicators—Stochastic RSI, MACD, Money Flow Index (MFI), and Bollinger Bands—to identify trends, generate entry and exit signals, and manage multiple sub-positions. The script dynamically adjusts position sizes based on market conditions and remaining equity, making it a versatile tool for day/swing trading.

## Overview

The "Bitcoin Swing Trading Indicator" is a sophisticated swing trading tool that leverages a combination of technical indicators to generate buy and sell signals for Bitcoin (or any cryptocurrency supported by your exchange and 3Commas). These signals are transmitted to 3Commas via JSON webhooks for automated trade execution. The strategy uses:

- **Stochastic RSI** for overbought/oversold conditions and entry/exit triggers.
- **MACD** to determine bullish or bearish trends.
- **Money Flow Index (MFI)** to confirm momentum and trend strength.
- **Bollinger Bands** (two sets with different periods) for additional entry and exit signals.

The script manages multiple sub-positions, tracks remaining equity, and adjusts position sizes dynamically based on trend conditions and predefined parameters. It’s designed to operate in real-time bars only, ensuring signals align with live market data.

## Step-by-Step Breakdown of Primary Functions

### 1. General Variables and 3Commas Credentials

#### Purpose
Sets up foundational variables and authentication for 3Commas integration.

#### Details
- **3Commas Credentials**: `SECRET` and `BOT_UUID` are defined for secure communication with the 3Commas bot.
- **Remaining Equity**: Initialized at `1000` (assumed to be in the quote currency, e.g., USD), tracking available funds for trading.
- **Entry Percentages**:
  - `BASE_PERCENTAGE_BULLISH = 35.0`: Base position size percentage in bullish trends.
  - `BASE_PERCENTAGE_BEARISH = 15.0`: Base position size percentage in bearish trends.
  - `BOLLINGERPOSSIZE = 50.0`: Fixed percentage for Bollinger Band-based positions.
  - `INCREMENT = 5`: Adjusts position sizes dynamically as more positions are opened.

### 2. Internal Variables for Position Tracking

#### Purpose
Manages sub-positions and prevents redundant alerts.

#### Details
- **Arrays**:
  - `bearishEntryPrices` and `bearishEntrySizes`: Track entry prices and sizes for bearish trend positions.
  - `bullishEntryPrices` and `bullishEntrySizes`: Track entry prices and sizes for bullish trend positions.
- **Bollinger Variables**: `bollingerPrice` and `bollingerSize` store the entry price and size for Bollinger-based trades.
- **Counters**: `x` (bullish) and `y` (bearish) track the number of open positions.
- **Alert Flags**: `bollingerBuyAlertFired` and `bollingerSellAlertFired` prevent repeated buy/sell alerts for Bollinger trades.

### 3. Technical Indicators

#### Purpose
Calculates and plots indicators for trend identification and signal generation.

#### Details
- **Stochastic RSI**:
  - Parameters: `SMOOTH_K = 6`, `SMOOTH_D = 6`, `LENGTH_RSI = 28`, `LENGTH_STOCH = 28`.
  - Plots `K` (blue) and `D` (orange) lines with custom overbought (`25`) and oversold (`10`) levels.
- **MACD**:
  - Parameters: `FAST_LEN = 216`, `SLOW_LEN = 432`, `SIGNAL_LEN = 144`.
  - Plots `macdLine` (blue) and `macdSignal` (orange) to determine trend direction.
- **Money Flow Index (MFI)**:
  - Parameters: `MFI_LENGTH = 864`, source `hlc3`.
  - Plots MFI (purple) with custom overbought (`56`) and oversold (`45`) levels.
- **Bollinger Bands**:
  - Short-term (sell signals): `LENGTH = 2016`, `MULT = 2.5`, plots upper band.
  - Long-term (buy signals): `LONG_LENGTH = 4032`, `LONG_MULT = 2.5`, plots lower band.

### 4. Trend Identification and Trading Conditions

#### Purpose
Defines market trends and conditions for trading.

#### Details
- **Trends**:
  - Bullish: `macdLine > macdSignal` and `mfi < 56`.
  - Bearish: `macdLine <= macdSignal` or `mfi > 56`.
- **Trading Rules**:
  - `allowTrading`: True only in real-time bars (`barstate.isrealtime`).
  - `haltTrading`: Stops trading if `macdLine <= macdSignal` and `mfi > 45`.
- **Stochastic Conditions**:
  - Entry: Bullish (`k` crosses over `d`, `k < 20`); Bearish (`k` crosses over `d`, `k < 10`).
  - Exit: Bullish (`k` crosses under `d`, `k > 50`); Bearish (`k` crosses under `d`, `k > 25`).
- **Bollinger Conditions**:
  - Entry: Price crosses above long lower band (`ta.crossover(src, longLower)`).
  - Exit: Price crosses below short upper band (`ta.crossunder(src, upper)`).

### 5. Entry and Exit Execution

#### Purpose
Executes trades based on conditions, manages equity, and sends signals to 3Commas.

#### Details
- **Stochastic Entries**:
  - Trigger: `allowTrading`, `entryConditionStoch`, less than 6 positions (`x + y < 6`), and `not haltTrading`.
  - Position Size:
    - Bullish: `BASE_PERCENTAGE_BULLISH - (INCREMENT * x)`.
    - Bearish: `BASE_PERCENTAGE_BEARISH - (INCREMENT * y)`.
  - Calculates number of contracts based on `remainingEquity` and current price, updates arrays, and reduces `remainingEquity`.
  - Sends a JSON `enter_long` alert to 3Commas with the calculated amount.
- **Bollinger Entry**:
  - Trigger: `allowTrading`, `entryConditionBollinger`, and `not bollingerBuyAlertFired`.
  - Uses `BOLLINGERPOSSIZE` (50%) of `remainingEquity`, stores price/size, and sends a JSON `enter_long` alert.
  - Sets `bollingerBuyAlertFired = true` and resets `bollingerSellAlertFired`.
- **Stochastic Exits**:
  - Trigger: `allowTrading`, `exitConditionStoch`, and existing positions.
  - Checks bullish and bearish arrays for profitable positions (current price > entry price).
  - Updates `remainingEquity`, removes positions from arrays, adjusts counters, and sends a JSON `exit_long` alert for total profitable contracts.
- **Bollinger Exit**:
  - Trigger: `allowTrading`, `exitConditionBollinger`, `not bollingerSellAlertFired`, and price > entry price.
  - Updates `remainingEquity`, sends a JSON `exit_long` alert, and resets alert flags.

## Usage Instructions

- **TradingView Setup**:
  - Copy the script into TradingView’s Pine Editor and save it as an indicator.
  - Add it to your chart (appears in a separate pane).
- **3Commas Configuration**:
  - Create a bot in 3Commas and note its `BOT_UUID`.
  - Obtain the `SECRET` key from 3Commas for webhook authentication.
  - Configure the bot to accept TradingView webhook signals (provide the webhook URL in TradingView alerts).
- **Alert Setup**:
  - In TradingView, set up alerts using the indicator with the `alert()` function calls embedded in the script.
  - Ensure the webhook URL points to your 3Commas bot.
- **API Integration**:
  - Link your centralized exchange account to 3Commas via API keys with trading permissions.

## Customization

Users can tweak the following parameters to suit their trading style:
- Indicator lengths (e.g., `LENGTH_RSI`, `FAST_LEN`, `MFI_LENGTH`).
- Overbought/oversold thresholds for Stochastic RSI and MFI.
- Bollinger Band periods and multipliers.
- Entry percentages (`BASE_PERCENTAGE_BULLISH`, `BASE_PERCENTAGE_BEARISH`, `BOLLINGERPOSSIZE`) and `INCREMENT`.

## Disclaimer

This script is provided for educational purposes only and does not constitute financial advice. Cryptocurrency trading involves significant risks, including the potential loss of capital. Test the script thoroughly in a demo environment before using it with real funds.
