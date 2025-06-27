# StockMarketPredictiom
import yfinance as yf
import pandas_ta as ta
import pandas as pd
import datetime
import time
import matplotlib.pyplot as plt
import seaborn as sns
from dhanhq import dhanhq

# 🟢 Replace these with your Dhan credentials
client_id = ""
access_token = ""

# Connect to Dhan API
dhan = dhanhq("client_id","access_token")

# Setup plot style
sns.set_theme(style='darkgrid')

def get_live_data_and_trade(symbol="RELIANCE.NS", trading_symbol="RELIANCE", quantity=1):
    # Fetch 1-minute live data for the day
    df = yf.download(tickers=symbol, period="1d", interval="1m", progress=False)

    if df.empty or len(df) < 20:
        print(f"[{datetime.datetime.now().strftime('%H:%M:%S')}] Not enough data to calculate indicators.")
        return

    # Calculate EMA and RSI
    df['EMA'] = ta.ema(df['Close'], length=20)
    df['RSI'] = ta.rsi(df['Close'], length=14)

    latest = df.iloc[-1]
    close = latest['Close']
    ema = latest['EMA']
    rsi = latest['RSI']

    if pd.isna(ema.item()) or pd.isna(rsi.item()):

        print("Indicators not ready yet (waiting for more data)...")
        return

    # Signal logic
    signal = "No Trade"
    color = 'gray'

    if rsi > 40 and close > ema:
        signal = "Buy Call Option"
        color = 'green'
        try:
            dhan.place_order(
                exchangeSegment="NSE_EQ",
                transactionType="BUY",
                orderType="MARKET",
                tradingSymbol=trading_symbol,
                quantity=quantity,
                productType="INTRADAY"
            )
            print(f"✅ BUY CALL placed for {trading_symbol}")
        except Exception as e:
            print(f"❌ Failed to place BUY CALL: {e}")

    elif rsi < 40 and close < ema:
        signal = "Buy Put Option"
        color = 'red'
        try:
            dhan.place_order(
                exchangeSegment="NSE_EQ",
                transactionType="SELL",
                orderType="MARKET",
                tradingSymbol=trading_symbol,
                quantity=quantity,
                productType="INTRADAY"
            )
            print(f"✅ BUY PUT placed for {trading_symbol}")
        except Exception as e:
            print(f"❌ Failed to place BUY PUT: {e}")

    print(f"\n[{datetime.datetime.now().strftime('%H:%M:%S')}] Spot: ₹{close:.2f} | RSI: {rsi:.2f} | EMA: ₹{ema:.2f} → {signal}")

    # Optional plot
    fig, axs = plt.subplots(2, 1, figsize=(12, 8), sharex=True)
    axs[0].plot(df.index, df['Close'], label='Close Price', color='blue')
    axs[0].plot(df.index, df['EMA'], label='EMA (20)', color='orange')
    axs[0].axvline(df.index[-1], color=color, linestyle='--', alpha=0.5)
    axs[0].annotate(signal, xy=(df.index[-1], close), xytext=(df.index[-1], close + 5),
                    arrowprops=dict(facecolor=color, arrowstyle='->'), fontsize=10)
    axs[0].set_title(f"{symbol} Price & EMA")
    axs[0].legend()
    axs[0].set_ylabel("Price")

    axs[1].plot(df.index, df['RSI'], label='RSI (14)', color='purple')
    axs[1].axhline(70, color='red', linestyle='--', linewidth=1)
    axs[1].axhline(30, color='green', linestyle='--', linewidth=1)
    axs[1].set_title("Relative Strength Index (RSI)")
    axs[1].legend()
    axs[1].set_ylabel("RSI")

    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.show()

# 🔁 Run every minute during market hours
while True:
    now = datetime.datetime.now().time()
    if now > datetime.time(15, 30) or now < datetime.time(9, 15):
        print("⏹️ Market closed. Exiting.")
        break
    get_live_data_and_trade("RELIANCE.NS", "RELIANCE", quantity=1)
    time.sleep(60)
