# Fixing ProfitTrailer Configuration Errors

I see two specific errors in your notification:

1. **"MACD_4H_signal is empty"**
2. **"Wicksniper_LONG - buy strategy CUSTOM is not valid, using INCORRECT instead"**

Let me fix both issues in your ProfitTrailer configuration:

## 1. Fix for "MACD_4H_signal is empty"

This error occurs when ProfitTrailer cannot locate or calculate the MACD signal properly. Let's update the MACD indicator definition in your INDICATORS.properties file:

```properties
# MACD - Fixed definition
macd_4h_candle_period = 14400
macd_4h_fast_length = 5
macd_4h_slow_length = 8
macd_4h_signal_length = 2
macd_4h_source = CLOSE
```

The key addition here is `macd_4h_source = CLOSE` which explicitly defines what price data to use for the calculation.

## 2. Fix for "CUSTOM is not valid"

This error occurs when a CUSTOM strategy type is defined incorrectly. The issue is likely that ProfitTrailer doesn't recognize the custom indicators we've created. There are two ways to fix this:

### Option 1: Use built-in indicators with specific conditions instead of CUSTOM

Replace in PAIRS.properties:

```properties
# Instead of CUSTOM indicators for crossovers, use standard indicators with conditions

# Entry Conditions - Modified to use standard indicators
default_a_buy_strategy = LOWBB
default_a_buy_strategy_label = lowbb_4h
default_a_buy_value = 55

default_b_buy_strategy = MACD
default_b_buy_strategy_label = macd_4h
default_b_buy_value = 0

default_c_buy_strategy = MACDSIGNAL  
default_c_buy_strategy_label = macd_4h
default_c_buy_value = 0

default_d_buy_strategy = STOCHRSIK
default_d_buy_strategy_label = stochrsi_4h
default_d_buy_value = 19

default_e_buy_strategy = STOCHRSID
default_e_buy_strategy_label = stochrsi_4h
default_e_buy_value = 19

# Buy Formula: Price below BB threshold AND MACD > Signal AND MACD < 0 AND StochRSI K and D below threshold
default_buy_strategy_formula = a && b > c && b < 0 && d < 20 && e < 20

# Sell Conditions - Modified
default_a_sell_strategy = HIGHBB
default_a_sell_strategy_label = lowbb_4h  # Uses the same BB indicator but looks at upper band
default_a_sell_value = 90

default_b_sell_strategy = MACD
default_b_sell_strategy_label = macd_4h
default_b_sell_value = 0

default_c_sell_strategy = MACDSIGNAL
default_c_sell_strategy_label = macd_4h
default_c_sell_value = 0

default_d_sell_strategy = STOCHRSIK
default_d_sell_strategy_label = stochrsi_4h
default_d_sell_value = 80

# Sell Formula: Price above upper BB threshold OR (MACD < Signal AND MACD > 0 AND StochRSI K > threshold)
default_sell_strategy_formula = a || (b < c && b > 0 && d > 80)
```

### Option 2: Define proper MACDCROSS indicator

If your version of ProfitTrailer supports it, you can define a proper MACDCROSS indicator:

```properties
# In INDICATORS.properties:
macdcross_4h_candle_period = 14400
macdcross_4h_fast_length = 5
macdcross_4h_slow_length = 8
macdcross_4h_signal_length = 2
macdcross_4h_trigger = CROSS_SIGNAL_UP
```

Then in PAIRS.properties:

```properties
default_a_buy_strategy = MACDCROSS
default_a_buy_strategy_label = macdcross_4h
default_a_buy_value = 0
```

## Updated PAIRS.properties (Complete Fix)

Here's a complete PAIRS.properties file with the fixes applied:

```properties
##########################################################
# PAIRS — BB + MACD + StochRSI STRATEGY - FIXED
##########################################################
MARKET = USDTM
ENABLED_PAIRS = BTC,ETH,BNB,LTC,SOL,AVAX,MATIC,DOT,LINK,ADA
keep_balance = 35%
max_trading_pairs = 6
max_trading_pairs_include_pending = false
trading_pairs_buy_priority = volumeasc

default_trading_enabled = true
default_buy_margin_type = ISOLATED
auto_leverage_calculation = true
default_buy_leverage = 3
default_initial_cost = 7%
default_dca_enabled = false

# Entry Conditions - Using standard indicators
default_a_buy_strategy = LOWBB
default_a_buy_strategy_label = lowbb_4h
default_a_buy_value = 55

default_b_buy_strategy = MACD
default_b_buy_strategy_label = macd_4h
default_b_buy_value = 0

default_c_buy_strategy = MACDSIGNAL  
default_c_buy_strategy_label = macd_4h
default_c_buy_value = 0

default_d_buy_strategy = STOCHRSIK
default_d_buy_strategy_label = stochrsi_4h
default_d_buy_value = 19

default_e_buy_strategy = STOCHRSID
default_e_buy_strategy_label = stochrsi_4h
default_e_buy_value = 19

# Buy Formula: BB position low AND MACD > Signal AND MACD < 0 AND StochRSI K/D low
default_buy_strategy_formula = a && b > c && b < 0 && d <= 19 && e <= 19

# Exit Conditions
default_a_sell_strategy = GAIN
default_a_sell_value = 3.0

default_b_sell_strategy = HIGHBB
default_b_sell_strategy_label = lowbb_4h
default_b_sell_value = 90

default_c_sell_strategy = MACD
default_c_sell_strategy_label = macd_4h
default_c_sell_value = 0

default_d_sell_strategy = MACDSIGNAL
default_d_sell_strategy_label = macd_4h
default_d_sell_value = 0

default_e_sell_strategy = STOCHRSIK
default_e_sell_strategy_label = stochrsi_4h
default_e_sell_value = 89

# Sell Formula: Take profit OR (Price high in BB AND MACD < Signal AND MACD > 0 AND StochRSI K high)
default_sell_strategy_formula = a || (b && c < d && c > 0 && e >= 89)

# Position management
default_take_profit = 3.0
default_trailing_stop_loss_trigger_arm = 1.5
default_trailing_stop_loss_trigger = -0.5
default_stop_loss_trigger = -3.0
default_stop_loss_timeout = 60
```

## Updated INDICATORS.properties (Complete Fix)

```properties
##########################################################
# INDICATORS — LONG STRATEGY (4h Timeframe) - FIXED
##########################################################

# Bollinger Bands (Lower Band logic, used in entry)
lowbb_4h_candle_period = 14400
lowbb_4h_length = 20
lowbb_4h_std = 1.55
lowbb_4h_source = CLOSE

# MACD - Fixed definition
macd_4h_candle_period = 14400
macd_4h_fast_length = 5
macd_4h_slow_length = 8
macd_4h_signal_length = 2
macd_4h_source = CLOSE

# Stochastic RSI
stochrsi_4h_candle_period = 14400
stochrsi_4h_rsi_length = 10
stochrsi_4h_stoch_length = 10
stochrsi_4h_k = 3
stochrsi_4h_d = 3
stochrsi_4h_source = CLOSE
```

## Recommendations

1. **Always specify source** for indicators (CLOSE, OPEN, HIGH, LOW)
2. **Avoid CUSTOM indicators** when possible - use standard indicators with conditional logic instead
3. **Check documentation** for your specific ProfitTrailer version - supported indicators may vary

If you continue experiencing issues, consider reaching out to ProfitTrailer support or checking their wiki for the exact syntax supported in your version of the software. Different versions may have different requirements for indicator configurations.
