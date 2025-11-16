# Crypto-Graph-Test
# Import the required tools (Tkinter used for GUI)
    import tkinter as tk
    from tkinter import ttk
    import requests
    import pandas as pd
    import matplotlib.pyplot as plt
    import mplfinance as mpf
    import threading
    import time
    
# Identify the Cryptocurrency to view
    SUPPORTED_COINS = {
    "Ethereum (ETH)": "ethereum",
    "Bitcoin (BTC)": "bitcoin"}
    
# Set Chart Types
    CHART_TYPES = ["Candlestick", "Line"]
    # URL of where to fetch the data of ongoing markets
    def fetch_data(coin_id):
    url = f"https://api.coingecko.com/api/v3/coins/{coin_id}/market_chart?vs_currency=sgd&days=30"
    response = requests.get(url)
    data = response.json()
    prices = data['prices']
    df = pd.DataFrame(prices, columns=['Timestamp', 'Close'])
    df['Date'] = pd.to_datetime(df['Timestamp'], unit='ms')
    df.set_index('Date', inplace=True)
    df['Open'] = df['Close'].shift(1)
    df['High'] = df['Close'].rolling(2).max()
    df['Low'] = df['Close'].rolling(2).min()
    df['Volume'] = 1000  # Placeholder, as CoinGecko's basic API lacks volume
    df = df[['Open', 'High', 'Low', 'Close', 'Volume']]
    df.dropna(inplace=True)

# Indicators
    df['SMA_7'] = df['Close'].rolling(window=7).mean()
    df['SMA_21'] = df['Close'].rolling(window=21).mean()
    df['BB_upper'] = df['Close'].rolling(20).mean() + 2 * df['Close'].rolling(20).std()
    df['BB_lower'] = df['Close'].rolling(20).mean() - 2 * df['Close'].rolling(20).std()
    delta = df['Close'].diff()
    gain = delta.clip(lower=0)
    loss = -delta.clip(upper=0)
    avg_gain = gain.rolling(14).mean()
    avg_loss = loss.rolling(14).mean()
    rs = avg_gain / avg_loss
    df['RSI'] = 100 - (100 / (1 + rs))
    return df

# Signals
    def plot_chart(df, coin_id, chart_type, save=False):
    last_price = df['Close'].iloc[-1]
    sma7 = df['SMA_7'].iloc[-1]
    sma21 = df['SMA_21'].iloc[-1]
    if sma7 > sma21:
        signal = "BUY 🔼"
    elif sma7 < sma21:
        signal = "SELL 🔽"
    else:
        signal = "HOLD ⚖️"
    label_result.config(text=f"{coin_id.upper()} Signal: {signal}\nPrice: S${last_price:,.2f}")
    if chart_type == "Candlestick":
        addplots = [
            mpf.make_addplot(df["SMA_7"], color='blue'),
            mpf.make_addplot(df["SMA_21"], color='red'),
            mpf.make_addplot(df["BB_upper"], color='grey'),
            mpf.make_addplot(df["BB_lower"], color='grey')]
        style = mpf.make_mpf_style(base_mpf_style='yahoo', rc={'figure.facecolor': 'white'})
        if save:
            mpf.plot(df, type='candle', style=style, addplot=addplots,
                     title=f"{coin_id.upper()} Candlestick Chart",
                     volume=True, savefig=f"{coin_id}_chart.pdf")
        else:
            mpf.plot(df, type='candle', style=style, addplot=addplots,
                     title=f"{coin_id.upper()} Candlestick Chart",
                     volume=True)
    else:  # Line chart
        plt.figure(figsize=(10, 5))
        plt.plot(df.index, df['Close'], label='Price')
        plt.plot(df.index, df['SMA_7'], label='SMA 7', linestyle='--')
        plt.plot(df.index, df['SMA_21'], label='SMA 21', linestyle='--')
        plt.fill_between(df.index, df['BB_lower'], df['BB_upper'], color='gray', alpha=0.2, label='Bollinger Bands')
        plt.title(f"{coin_id.upper()} Line Chart")
        plt.legend()
        plt.tight_layout()
        if save:
            plt.savefig(f"{coin_id}_chart.pdf")
        else:
            plt.show()
    def refresh_chart(save=False):
        selected_coin = coin_var.get()
        coin_id = SUPPORTED_COINS[selected_coin]
        chart_type = chart_var.get()
        df = fetch_data(coin_id)
        plot_chart(df, coin_id, chart_type, save)

    def save_to_pdf():
        refresh_chart(save=True)
        label_result.config(text=label_result.cget("text") + "\n📄 Chart saved to PDF.")

    def auto_refresh_loop():
        while auto_refresh.get():
            time.sleep(300)
            refresh_chart()

    def toggle_auto_refresh():
        if auto_refresh.get():
            threading.Thread(target=auto_refresh_loop, daemon=True).start()

# GUI Setup
    window = tk.Tk()
    window.title("Crypto Dashboard")
    window.geometry("420x350")
    label_title = tk.Label(window, text="Crypto Dashboard (SGD)", font=("Arial", 14))
    label_title.pack(pady=10)

# Coin Selector
    coin_var = tk.StringVar(value=list(SUPPORTED_COINS.keys())[0])
    coin_menu = ttk.Combobox(window, textvariable=coin_var, values=list(SUPPORTED_COINS.keys()), state="readonly")
    coin_menu.pack(pady=5)

# Chart Type Selector
    chart_var = tk.StringVar(value=CHART_TYPES[0])
    chart_menu = ttk.Combobox(window, textvariable=chart_var, values=CHART_TYPES, state="readonly")
    chart_menu.pack(pady=5)

    label_result = tk.Label(window, text="Click to fetch data...", font=("Arial", 11))
    label_result.pack(pady=10)

    btn_refresh = tk.Button(window, text="Show Chart", command=refresh_chart)
    btn_refresh.pack(pady=5)

    btn_save = tk.Button(window, text="Save Chart to PDF", command=save_to_pdf)
    btn_save.pack(pady=5)

    auto_refresh = tk.BooleanVar()
    chk_auto = tk.Checkbutton(window, text="Auto-refresh every 5 min", variable=auto_refresh, command=toggle_auto_refresh)
    chk_auto.pack(pady=5)

    window.mainloop()
