import pandas as pd
import pandas_ta as ta  # pip install pandas-ta
import ccxt
import numpy as np

def fetch_data(exchange, symbol, timeframe='1h', limit=500):
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    return df

def clean_signal_analysis(df, min_rr=2.0, max_risk_pct=1.0):
    # Add indicators
    df['ema_fast'] = ta.ema(df['close'], length=9)
    df['ema_slow'] = ta.ema(df['close'], length=21)
    df['rsi'] = ta.rsi(df['close'], length=14)
    df['atr'] = ta.atr(df['high'], df['low'], df['close'], length=14)
    df['volume_sma'] = df['volume'].rolling(20).mean()
    
    # Swing highs/lows for structure & Order Blocks
    df['swing_low'] = df['low'].rolling(5, center=True).min()
    df['swing_high'] = df['high'].rolling(5, center=True).max()
    
    signals = []
    for i in range(20, len(df)-1):  # Enough history
        current = df.iloc[i]
        prev = df.iloc[i-1]
        
        # === CLEAN SETUP FILTERS ===
        if current['volume'] < current['volume_sma'] * 1.2:  # Volume confirmation
            continue
        if abs(current['rsi'] - 50) < 10:  # Avoid chop (RSI neutral filter)
            continue
        
        # Bullish Clean Setup: Pullback to Order Block + Breakout
        bullish_ob = (current['close'] > prev['swing_low'] and 
                     current['low'] <= prev['swing_low'] * 1.02 and  # Retest
                     current['ema_fast'] > current['ema_slow'])
        
        if bullish_ob:
            entry = current['close']
            stop = current['swing_low'] - current['atr'] * 0.5  # Logical invalidation
            risk_dist = entry - stop
            if risk_dist <= 0: continue
            
            target1 = entry + risk_dist * 2.0   # Min 1:2 RR
            target2 = entry + risk_dist * 3.0
            
            signals.append({
                'direction': 'long',
                'entry': entry,
                'stop': stop,
                'target1': target1,
                'target2': target2,
                'rr': 2.5,  # Average
                'reason': 'OB_Pullback_Breakout',
                'confluence': 'EMA + Volume + Structure'
            })
        
        # Bearish symmetric logic here...
    
    return signals[-5:] if signals else []  # Latest clean signals
