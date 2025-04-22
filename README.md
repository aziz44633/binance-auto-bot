import os
import time
from binance.client import Client
from dotenv import load_dotenv
import pandas as pd
import ta

# Load API keys
load_dotenv()
API_KEY = os.getenv("BINANCE_API_KEY")
API_SECRET = os.getenv("BINANCE_SECRET_KEY")

client = Client(API_KEY, API_SECRET)

symbol = "BTCUSDT"
interval = Client.KLINE_INTERVAL_1MINUTE
lookback = "1 day ago UTC"

def get_ohlcv_data():
    df = pd.DataFrame(client.get_historical_klines(symbol, interval, lookback))
    df = df.iloc[:, :6]
    df.columns = ["timestamp", "open", "high", "low", "close", "volume"]
    df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms")
    for col in ["open", "high", "low", "close", "volume"]:
        df[col] = df[col].astype(float)
    return df

def apply_indicators(df):
    df["rsi"] = ta.momentum.rsi(df["close"], window=14)
    macd = ta.trend.MACD(df["close"])
    df["macd"] = macd.macd()
    df["macd_signal"] = macd.macd_signal()
    return df

def check_signals(df):
    latest = df.iloc[-1]
    previous = df.iloc[-2]
    if (latest["rsi"] < 30) and (previous["macd"] < previous["macd_signal"]) and (latest["macd"] > latest["macd_signal"]):
        return "BUY"
    if (latest["rsi"] > 70) and (previous["macd"] > previous["macd_signal"]) and (latest["macd"] < latest["macd_signal"]):
        return "SELL"
    return None

def execute_trade(signal):
    balance = float(client.get_asset_balance(asset="USDT")["free"])
    price = float(client.get_symbol_ticker(symbol=symbol)["price"])
    qty = round((balance * 0.1) / price, 6)
    if signal == "BUY":
        order = client.order_market_buy(symbol=symbol, quantity=qty)
        print("Bought:", order)
    elif signal == "SELL":
        asset_qty = float(client.get_asset_balance(asset="BTC")["free"])
        order = client.order_market_sell(symbol=symbol, quantity=asset_qty)
        print("Sold:", order)

def main():
    while True:
        df = get_ohlcv_data()
        df = apply_indicators(df)
        signal = check_signals(df)
        if signal:
            execute_trade(signal)
        time.sleep(60)

if __name__ == "__main__":
    main()
