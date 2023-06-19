# Adaptive Market Making Trading System 13 for Kucoin Spot WS API
import sys
import traceback
import time
from datetime import datetime
import warnings
import os
import math
import asyncio
from asyncio import run
import threading
import ta.volume as volume
import ccxt.pro as ccxtpro
import numpy as np
import pandas as pd
import ta.volatility as volatility
import ta.trend as trend
import csv
from apscheduler.schedulers.background import BackgroundScheduler

class AdaptiveMarketMaker: 
    def __init__(self, exchange, exchange_trade, symbol, timeframe, limit, window_length):
        
        # Data exchange account
        self.exchange = exchange
        
        # Trade exchange account
        self.exchange_trade = exchange_trade
        
        # Market symbol
        self.symbol = symbol
        
        # Timeframe
        self.timeframe = timeframe
        
        # Limit
        self.limit = limit
        
        # Current time (t): The current time since the start of the trading day, in seconds.
        self.t = t = time.time()

        # Current price (St): The current price of the asset, typically calculated as the mid-price (average of best bid and ask prices).
        self.St = None

        # Initial inventory (IT): The initial inventory level of the asset held by the market maker. This should ideally be updated dynamically with the actual balance.
        self.IT = 0.0 

        # Time horizon (T): The total time horizon for the market making strategy, in seconds. This is used to determine the remaining time in the trading period.
        self.T = 1

        # Risk aversion (gamma): A parameter that represents the market maker's risk aversion. Higher values indicate greater risk aversion, which will affect the optimal bid and ask prices.
        self.gamma = 75

        # Volatility (sigma): The current volatility of the asset. This is typically calculated based on recent trade prices and is used in the calculation of optimal bid and ask prices.
        self.sigma = 0.0 # Initialize t as a float with an initial value of 0.0

        # Cost of liquidity provision (k): A parameter representing the cost of providing liquidity to the market. This is typically incorporated into the calculation of optimal bid and ask prices.
        self.k = None

        # Position risk (c): A parameter representing the risk associated with holding a position in the asset. This can be used to adjust the optimal bid and ask prices based on the market maker's position risk tolerance.
        self.c = 0.0 # Initialize t as a float with an initial value of 0.0
        
        # ohlcv
        self.ohlcv = []
        
        # Get the last available OHLCV data when no new price data is available
        self.last_ohlcv = None
        
        # Indicator period
        self.indicator_period = 2
        
        # Store trade entry values for sell order
        self.entry_prices = []

        # Store recent market prices
        self.recent_prices = []

        # Historical trade prices, integer represents nunber of days
        self.window_length = 1

        self.position_open = False
        self.breached_tsl_levels = []
        self.break_even_price = None
        self.bes_price = None
        self.recent_trade_prices = []
        self.ema_window = 1  # You can change the window size as needed
        self.ema_lambda = 2 / (self.ema_window + 1)
        self.variance = None
        self.max_trade_history = 100
        self.buy_price = 0.0 # Initialize t as a float with an initial value of 0.0
        self.start_time = time.time()
        self.surpassed_tsls = []
        self.active_sell_orders = []

        
    # Write data to csv file for analysis
    def write_data_to_csv(self, file_name, data, header=None):
        directory = os.path.dirname(file_name)  # Get the directory from the file_name
        if not os.path.exists(directory):  # Check if the directory exists
            os.makedirs(directory)  # Create the directory if it doesn't exist
        
        mode = "a" if os.path.exists(file_name) else "w"
        with open(file_name, mode, newline='') as file:
            writer = csv.writer(file)
            if mode == "w" and header:
                writer.writerow(header)
            writer.writerow(data)    

    # Fetch historical data for optimal market making strategy
    async def fetch_historical_data(self, exchange, symbol, timeframe, since):
        historical_data = []
        while since < exchange.milliseconds():
            try:
                ohlcv = await exchange.fetch_ohlcv(symbol, timeframe, since)
                if not ohlcv:
                    break
                since = ohlcv[-1][0] + 1
                historical_data += ohlcv
            except Exception as e:
                print(f'Error fetching historical data: {e}')
                await asyncio.sleep(exchange.rateLimit / 1000)
        return historical_data
    
    # Load historical data function for optimal market making strategy
    async def load_historical_data(self, csv_file_name, exchange, symbol, timeframe, since):
        if os.path.exists(csv_file_name) and os.path.getsize(csv_file_name) > 0:
            # Load historical data from the existing CSV file
            historical_data = pd.read_csv(csv_file_name).values.tolist()
        else:
            # Fetch historical data from the exchange API
            historical_data = await self.fetch_historical_data(exchange, symbol, timeframe, since)
            # Save the fetched data to the CSV file
            pd.DataFrame(historical_data).to_csv(csv_file_name, index=False)

        return historical_data

    # Update historical data function for optimal market making strategy
    async def update_historical_data(self, csv_file_name, exchange, symbol, timeframe, since):
        # Fetch historical data from the exchange API
        historical_data = await self.fetch_historical_data(exchange, symbol, timeframe, since)
        # Save the fetched data to the CSV file
        pd.DataFrame(historical_data).to_csv(csv_file_name, index=False)
    
    # Update OHLCV data if there's new data, otherwise use the last available data
    def update_OHLCV(self, data):
        if data:
            self.ohlcv.append(data[0])
        elif self.ohlcv:
            self.ohlcv.append(self.ohlcv[-1])
    
    # Watch real-time trades via websocket API for trade account
    async def watch_trades(self):
        trades = await self.exchange_trade.watch_trades(self.symbol)
        return trades
    
    # Watch real-time order book via websocket API for trade account
    async def watch_order_book(self):
        orderbook = await self.exchange_trade.watch_order_book(self.symbol)
        return orderbook    
    
    # Build OHLCV data from real time trades
    async def watch_trades_and_build_ohlcvc_and_watch_order_book(self): 
        trades = await self.exchange_trade.watch_trades(self.symbol)
        ohlcv = None  # Initialize ohlcv as None instead of an empty list
        try:
            ohlcv = self.exchange_trade.build_ohlcvc(trades, self.timeframe)
        except Exception as e:
            print(f"Error: {e}")
        orderbook = await self.exchange_trade.watch_order_book(self.symbol)

        # If there's no new price data, use the last available data
        if not ohlcv and hasattr(self, 'last_ohlcv'):
            ohlcv = self.last_ohlcv
        elif ohlcv and len(ohlcv) >= 2 and ohlcv[-1][4] == ohlcv[-2][4]:
            ohlcv[-1] = ohlcv[-2]

        # Save the last available OHLCV data for future use
        if ohlcv:
            self.last_ohlcv = ohlcv

        if ohlcv:
            opens = pd.Series([x[1] for x in ohlcv], dtype=float) 
            highs = pd.Series([x[2] for x in ohlcv], dtype=float)
            lows = pd.Series([x[3] for x in ohlcv], dtype=float)
            closes = pd.Series([x[4] for x in ohlcv], dtype=float)
            
        else:
            opens  = None
            highs = None
            lows = None
            closes = None

        return {
            'ohlcv': ohlcv,
            'orderbook': orderbook,
            'opens': opens,
            'highs': highs,
            'lows': lows,
            'closes': closes            
}    
    def update_recent_trade_prices(self, new_trade_price):
        self.recent_trade_prices.append(new_trade_price)
        if len(self.recent_trade_prices) > self.max_trade_history:
            #print('Recent Trade Prices:', self.recent_trade_prices)
            self.recent_trade_prices.pop(0)
    
    # Calculate technical indicators
    #def calculate_technical_indicators(self, ohlcv):
        #if len(ohlcv) < self.indicator_period:
            #return None
        #opens = pd.Series([x[1] for x in ohlcv], dtype=float)
        #highs = pd.Series([x[2] for x in ohlcv], dtype=float)
        #lows = pd.Series([x[3] for x in ohlcv], dtype=float)
        #closes = pd.Series([x[4] for x in ohlcv], dtype=float)
        
        # Indicator and rolling window period
        #adx_period = 2
        #atrp_period = 2
        #rolling_window = 2

        #if len(closes) >= adx_period:
            # Calculate ADX, DI+, and DI-
            #adx = trend.ADXIndicator(opens, highs, lows, closes, adx_period)
            #adx_value = adx.adx().iloc[-1]
            #plus_di = adx.adx_pos().iloc[-1]
            #minus_di = adx.adx_neg().iloc[-1]
            
            # Calculate ATRP
            #atr = volatility.AverageTrueRange(opens, highs, lows, closes, atrp_period).average_true_range()
            #atrp = (atr / closes.iloc[-1]) * 100

            # Calculate rolling means for ATRP, ADX, DI+, and DI-
            #atrp_rolling_mean_series = atrp.rolling(window=rolling_window).mean()
            #adx_rolling_mean_series = adx.adx().rolling(window=rolling_window).mean()
            #plus_di_rolling_mean_series = adx.adx_pos().rolling(window=rolling_window).mean()
            #minus_di_rolling_mean_series = adx.adx_neg().rolling(window=rolling_window).mean()

            #atrp_rolling_mean = atrp_rolling_mean_series.iloc[-1]
            #adx_rolling_mean = adx_rolling_mean_series.iloc[-1]
            #plus_di_rolling_mean = plus_di_rolling_mean_series.iloc[-1]
            #minus_di_rolling_mean = minus_di_rolling_mean_series.iloc[-1]

            #return {
                #'atrp_rolling_mean': atrp_rolling_mean,
                #'adx_rolling_mean': adx_rolling_mean,
                #'plus_di_rolling_mean': plus_di_rolling_mean,
                #'minus_di_rolling_mean': minus_di_rolling_mean,
                #'adx': adx_value,
                #'plus_di': plus_di,
                #'minus_di': minus_di,
                #'atrp': atrp,
                #'plus_di_rolling_mean_series': adx.adx_pos(), 
                #'minus_di_rolling_mean_series': adx.adx_neg()
            #}
        
        #return None
            
    # Update recent prices
    def update_recent_prices(self, new_price):

        self.recent_prices.append(new_price)
        if len(self.recent_prices) > self.window_length:
            self.recent_prices.pop(0)
            #print('Recent Prices:', self.recent_prices)

    # Calculate position risk
    def calculate_position_risk(self, historical_ohlcv, window_length=30):
        # Combine historical and real-time data to calculate position risk
        historical_closing_prices = np.array(historical_ohlcv)[:, 3]  # Extract closing prices from historical_ohlcv
        recent_closing_prices = np.array(self.recent_trade_prices)  # Extract closing prices from self.recent_trade_prices
        combined_prices = np.concatenate((historical_closing_prices, recent_closing_prices))

        if len(combined_prices) < window_length:
            raise ValueError("Insufficient data to calculate position risk.")

        # Calculate 1-minute returns
        minute_returns = np.diff(combined_prices) / combined_prices[:-1]
        absolute_returns = np.abs(minute_returns)

        # Calculate EMA of absolute returns
        ema_volatility = pd.Series(absolute_returns).ewm(span=window_length, adjust=False).mean().values[-1]

        #print("Position risk: ", ema_volatility)
        return ema_volatility

    def calculate_trend(self, historical_ohlcv, window_length=30):
        # Combine historical and real-time data to calculate trend
        historical_closing_prices = np.array(historical_ohlcv)[:, 3]  # Extract closing prices from historical_ohlcv
        recent_closing_prices = np.array(self.recent_trade_prices)  # Extract closing prices from self.recent_trade_prices
        combined_prices = np.concatenate((historical_closing_prices, recent_closing_prices))

        if len(combined_prices) < window_length:
            raise ValueError("Insufficient data to calculate trend.")

        # Calculate 1-minute returns
        minute_returns = np.diff(combined_prices) / combined_prices[:-1]

        # Calculate EMA of returns
        ema_trend = pd.Series(minute_returns).ewm(span=window_length, adjust=False).mean().values[-1]

        return ema_trend
    
    # Calculate order book imbalance
    def calculate_order_book_imbalance(self, orderbook):
        bids = orderbook['bids']
        asks = orderbook['asks']
        bid_volume = sum([bid[1] for bid in bids])
        ask_volume = sum([ask[1] for ask in asks])
        imbalance = (bid_volume - ask_volume) / (bid_volume + ask_volume)
        #print(imbalance)
        return imbalance
    
    # Calculate significant order book levels and their sizes
    def find_significant_bid_ask_levels(self, orderbook):
        significant_ask_levels = []

        # Calculate the total value of the order book
        total_order_book_value = sum([size for price, size in orderbook['asks']])

        # Set a percentage threshold for filtering significant ask levels
        percentage_threshold = 0.005  # Consider levels that have at least % of the total order book value

        for price, size in orderbook['asks']:
            order_percentage = size / total_order_book_value
            if order_percentage >= percentage_threshold:
                significant_ask_levels.append(price)

        ask_level_sizes = [size for price, size in orderbook['asks'] if price in significant_ask_levels]

        return significant_ask_levels, ask_level_sizes
    
    # Calculate order book imbalance and dynamic threshold
    #def calculate_order_imbalance_with_dynamic_threshold(self, orderbook, historical_imbalances, lookback_window):

        # Calculate order book imbalance
        #imbalance = self.calculate_order_book_imbalance(orderbook)
        #historical_imbalances.append(imbalance)

        # Truncate historical imbalances to the lookback window
        #if len(historical_imbalances) > lookback_window:
            #historical_imbalances.pop(0)

        # Calculate dynamic threshold based on historical imbalances
        #mean_imbalance = np.mean(historical_imbalances)
        #std_imbalance = np.std(historical_imbalances)
        #dynamic_threshold = mean_imbalance + std_imbalance

        #return imbalance, dynamic_threshold, historical_imbalances
    
    # Define function to determine whether to place a buy order
    #def should_place_buy_order(self, atrp_rolling_mean, adx_rolling_mean, di_difference_series, imbalance, adx_threshold, plus_di, #minus_di, dynamic_threshold):
        
        # Determine whether to place a buy order based on the given technical indicators and threshold
        #atrp_dynamic_threshold = atrp_rolling_mean
        #adx_dynamic_threshold = adx_rolling_mean
        #di_diff_rolling_window = 2
        
        # Calculate the most recent value in the di_difference_series
        #di_dynamic_threshold = di_difference_series.iloc[-1]

        # Calculate the mean of the di_difference_series values over a rolling window of size di_diff_rolling_window
        #di_diff_threshold = di_difference_series.rolling(window=di_diff_rolling_window).mean().iloc[-1]

        #significant_di_diff = di_diff_threshold > plus_di - minus_di
        #significant_di_diff = np.all(di_dynamic_threshold > di_diff_threshold)
        #strong_trend = adx_dynamic_threshold > adx_threshold
        #strong_bullish_trend = strong_trend and significant_di_diff

        #return strong_bullish_trend
    
    # Get best bid prices
    def get_best_bid(self, orderbook):
        bids = orderbook['bids']
        best_bid_price = bids[0][0] if bids else None
        #print('Best bid price: ', best_bid_price)
        return best_bid_price
    
    # Get best ask prices
    def get_best_ask(self, orderbook):
        asks = orderbook['asks']
        best_ask_price = asks[0][0] if asks else None
        #print('Best ask price: ', best_ask_price)
        return best_ask_price
        
    # Define wait for order to fill function
    async def wait_for_order_fill(self, order_id, timeout=1):
        start_time = self.exchange.milliseconds()
        while True:
            order = await self.exchange_trade.fetch_order(order_id, self.symbol)
            if order['status'] == 'closed':
                return order
            elif self.exchange.milliseconds() - start_time > timeout * 1000:
                return None
            await asyncio.sleep(1.0)  # Sleep for 1 second before checking the order status again

    # Define cancel unfilled orders function
    async def cancel_unfilled_orders(self, order_id, timeout=0.01):
        await asyncio.sleep(timeout)
        order = await self.exchange_trade.fetch_order(order_id, self.symbol)
        if order['status'] == 'open':
            await self.exchange_trade.cancel_order(order_id, self.symbol)
        
    
    # Calculate real time volatility
    def update_real_time_volatility(self):
        if len(self.recent_trade_prices) > 1:
            price_change = self.recent_trade_prices[-1] / self.recent_trade_prices[-2] - 1
            if self.variance is None:
                self.variance = price_change ** 2
            else:
                self.variance = self.ema_lambda * (price_change ** 2) + (1 - self.ema_lambda) * self.variance
            self.sigma = np.sqrt(self.variance)
            
            return self.sigma
        
    # Define cost of liquidity provision 
    def update_cost_of_liquidity_provision(self, commission_rate, best_bid_price, best_ask_price, order_amount, St):
        self.St = St

        # Calculate the cost of liquidity provision
        commission_cost = order_amount * commission_rate
        spread = abs(best_ask_price - best_bid_price)
        self.k = commission_cost + spread
        #print('Commission_rate:', commission_rate)
        #print('Commission_cost:', commission_cost)
        #print('Spread:', spread)
        #print('k', self.k)
    
    # Calculate volatility risk adjusted mid price
    def calculate_inventory_risk_adjusted_mid_price(self, St, It, t, T, gamma, sigma, k, c):
        self.sigma = sigma
        bid_adjustment = (k + c * math.exp(sigma))
        ask_adjustment = (k - c * math.exp(sigma))
        bid_price = St - (gamma * sigma**2 * (T - t)) / 2 - bid_adjustment
        ask_price = St + (gamma * sigma**2 * (T - t)) / 2 + ask_adjustment
        Q_mt = (bid_price + ask_price) / 2
        #print('Gamma:', gamma)
        #print('Sigma:', sigma)
        #print('Cost of liquidity provision:', k)
        #print('Position Risk:', c)
        return Q_mt
        
    # Calculate optimal bid price
    def calculate_optimal_bid_price(self, sigma, best_bid_price):
        scaled_sigma = sigma / (1 / self.St)
        optimal_bid_price = best_bid_price #- scaled_sigma
        #print('Scaled Sigma:', scaled_sigma)
        #print('Optimal bid price:', optimal_bid_price)
        return optimal_bid_price

    # Calculate optimal ask price
    def calculate_optimal_ask_price(self, sigma, best_ask_price):
        scaled_sigma = sigma / (1 / self.St)
        optimal_ask_price = best_ask_price + scaled_sigma
        #print('Scaled Sigma:', scaled_sigma)
        #print('Optimal ask price:', optimal_ask_price)
        return optimal_ask_price

    # Define function to place a buy order
    async def execute_buy_order(self, exchange_trade, symbol, optimal_bid_price, available_quote_balance, commission_rate):
        
        # Initialize the variable to indicate if the position was opened
        position_opened = False

        # Initialize variables for analysis
        #actual_order_amount = None
        #buy_price = None
        #buy_filled_order = None
        #bes_price = None
    
        # Define the USDT amount to spend on the buy order
        usdt_to_spend = 30
        
        # Check if the available balance is sufficient
        if available_quote_balance >= usdt_to_spend and not self.position_open:

                # Calculate the actual order amount based on the USDT to spend and the optimal bid price
                actual_order_amount = usdt_to_spend / optimal_bid_price

                # Allow real time ema to calculate before placing buy order
                current_time = time.time()
                elapsed_time = current_time - self.start_time

                # If time elapsed is greater than the ema window, place the buy order
                if elapsed_time >= self.ema_window * 60:
                    # Create a limit buy order using the optimal_bid_price
                    buy_order = await exchange_trade.create_limit_buy_order(symbol, actual_order_amount, optimal_bid_price)
                    print('Buy Order', buy_order)
                    
                    # Wait for the order to be filled
                    filled_order = await self.wait_for_order_fill(buy_order['id'])
                    #print("Filled order:", filled_order)

                    # Store the buy price when the order is filled
                    if filled_order:
                        buy_price = filled_order['price']
                        self.buy_price = buy_price  # Store the buy price
                        self.entry_prices.append(buy_price)
                        self.position_open = True
                        #position_opened = True

                        # Calculate the break-even price with commission costs
                        bes_price = buy_price * (1 + commission_rate)
                        #print('Break-even price:', bes_price)
                        
                        # Store the break-even price
                        self.break_even_price = bes_price

                    # Cancel unfilled orders after a timeout
                    timeout = 0.01
                    cancel_task = asyncio.ensure_future(self.cancel_unfilled_orders(buy_order['id'], timeout))
                    await cancel_task
                else:
                    print("Trading system is functioning, calculating eponential moving average and scaled sigma values before placing orders")

            #else:
                #print("No strong trend or orderbook imbalance detected. No buy orders placed.")
        else:
            print("Position filled, sell orders placed")

        return position_opened 
        
    # Define bes_price function
    #def calculate_bes_price(self, buy_price, commission_rate):
        #bes_price = buy_price * (1 + commission_rate)
        #return bes_price

    # Define sell order function
    async def execute_sell_order(self, exchange_trade, symbol, optimal_ask_price, available_base_balance, available_quote_balance, significant_ask_levels, ask_equity_allocations, amount_precision, trailing_stop_levels, buffers, retraced_breached_tsl):
        
        # Store the sell order details
        #sell_order_price = None
        #sell_order_amount = None
        #sell_order_id = None
        
        # Check if the available balance is sufficient to place a sell order
        if available_base_balance > 0.0002 and self.position_open:
            if len(self.entry_prices) > 0:
                last_entry_price = self.entry_prices[-1]
                
                #if not retraced_breached_tsl:
                for tsl_price, buffer, equity_allocation in zip(trailing_stop_levels, buffers, ask_equity_allocations):
                    #print("TSL price:", tsl_price)
                    #print("Buffer:", buffer)
                    #print("Equity allocation:", equity_allocation)
                    #print("Trailing stop levels:", trailing_stop_levels)
                    #print("Buffers:", buffers)
                    #print("Ask equity allocations:", ask_equity_allocations)    
                    
                    # Ensure not to place a sell order below buy price value
                    #if tsl_price > last_entry_price:
                        #print("last_entry_price:", last_entry_price)
                        
                    #print("Equity allocation:", equity_allocation)
                    #print("TSL price:", tsl_price)
                    #print("Equity allocation / TSL price:", equity_allocation / tsl_price)

                    # Calculate the minimum order size precision
                    market = await exchange_trade.load_markets()
                    amount_precision = market[symbol]['precision']['amount']
                    #print(f"Amount precision: {amount_precision}")
                        
                    # Calculate the amount to sell
                    amount_to_sell = equity_allocation * available_base_balance
                        
                    # Round the amount to sell to the nearest amount precision
                    decimal_places = abs(int(math.log10(amount_precision)))
                    amount_to_sell = round(amount_to_sell, decimal_places)
                    #print("Amount to sell:", amount_to_sell)
                       
                    # Ensure that the amount to sell is greater than the minimum precision
                    #print("Amount to sell before rounding:", max(equity_allocation, market[symbol]['limits']['amount']['min']))
                        
                    # Only place the sell order if the amount to sell is greater than 0
                    if amount_to_sell > 0.01: 
                        print(f"Placing sell order at {tsl_price} with amount {amount_to_sell}")

                    # Round the available base balance to the nearest amount precision
                    available_base_balance = round(available_base_balance, int(amount_precision))

                    #print(f"Placing sell order at {tsl_price} with amount {amount_to_sell}")
                        
                    # Create the sell order with the updated target price and amount to sell
                    #print("Amount to sell:", amount_to_sell)
                    sell_order = await exchange_trade.create_limit_sell_order(symbol, amount_to_sell, tsl_price)
                    #print('Sell Order', sell_order)
                    #print("Sell order status:", sell_order['status'])

                    # Store the sell order details for analysis tool
                    #sell_order_price = tsl_price
                    #sell_order_amount = amount_to_sell
                    #sell_order_id = sell_order['id']

                    # Wait for the order to be filled
                    filled_order = await self.wait_for_order_fill(sell_order['id'])
                    print("Filled order:", filled_order)

                    if filled_order:
                        self.active_sell_orders.remove(sell_order)  # Remove the filled order from the active sell orders list
                        self.surpassed_tsls.append(tsl_price)
                        print("Surpassed TSLs:", self.surpassed_tsls)
                        if not self.active_sell_orders:  # If there are no active sell orders, the position is closed
                            self.position_open = False
                                            
                    else:
                        print(f"Amount to sell {amount_to_sell} is less than the minimum allowed amount. Skipping this sell order.")

        return retraced_breached_tsl

    # Define function to calculate dynamic equity allocations
    def calculate_dynamic_equity_allocations(self, available_balance, tsl_prices):
        # Calculate the total number of possible allocation levels
        total_levels = len(tsl_prices)

        # Create a new allocation vector with descending values
        new_allocation_vector = np.linspace(start=1, stop=0, num=total_levels)
    
        # Normalize the new allocation vector to ensure the sum is 1
        new_allocation_vector /= new_allocation_vector.sum()

        # Multiply the available balance by the new allocation vector to obtain the equity allocations for each level
        equity_allocations = available_balance * new_allocation_vector

        return equity_allocations
            
    # Modify cancel_all_sell_orders to return the prices of the cancelled orders
    #print('Calling cancel_all_sell_orders')
    async def cancel_all_sell_orders(self, symbol):
        #print('Called cancel_all_sell_orders')
        open_orders = await self.exchange_trade.fetch_open_orders(symbol)
        #print("Open orders:", open_orders)
        sell_orders = [order for order in open_orders if order['side'] == 'sell']
        cancelled_order_prices = []
        for order in sell_orders:
            await self.exchange_trade.cancel_order(order['id'], symbol)
            cancelled_order_prices.append(order['price'])
        return cancelled_order_prices

    # Define function to calculate dynamic trailing stop levels
    def calculate_trailing_stop_levels(self, entry_price, significant_ask_levels, optimal_bid_price):
        commission_rate = 0.001  # commission rate as a decimal
        min_sell_price = entry_price * (1 + commission_rate)  # minimum price after accounting for commission

        # Adjust your trailing stop levels calculation to account for commission
        trailing_stop_levels = [max(significant_ask_levels[0], min_sell_price)] 
        trailing_stop_levels += [max(entry_price * (1 + (level - entry_price) / entry_price), min_sell_price) 
                             for level in significant_ask_levels if level >= entry_price]

        retraced_breached_tsl = False
        #print('Retraced Breached TSL:', retraced_breached_tsl)

        # Calculate dynamic buffers based on the distance between consecutive significance levels
        buffers = [trailing_stop_levels[i + 1] - level for i, level in enumerate(trailing_stop_levels[:-1])]
        buffers.append(trailing_stop_levels[-1] * 0.01)  # Fallback to a fixed 1% buffer for the last level

        # Define intentional drawdown buffer % for trailing stop loss
        buffer_percent = 0.0001

        #print(f"Breached TSL levels: {self.breached_tsl_levels}")
        for breached_level in self.breached_tsl_levels:
            retraced_level = breached_level * (1 + buffer_percent)
            #print(f"Price: {self.St}, retraced_level: {retraced_level}")
        
            if self.St <= retraced_level:
                retraced_breached_tsl = True
                #print('Retraced Breached TSL:', retraced_breached_tsl)
                break
    
        # If the current price surpasses the trailing stop level, append it to the surpassed tsl list
        for tsl_price in trailing_stop_levels:
            if self.St > tsl_price:
                self.surpassed_tsls.append(tsl_price)
                print("Surpassed TSLs:", self.surpassed_tsls)

        return trailing_stop_levels, buffers, retraced_breached_tsl

    # Define and execute conditional exit strategy
    async def execute_conditional_exit(self, exchange_trade, symbol, available_base_balance, best_bid_price, best_ask_price, buffer_percent):
        print("called execute_conditional_exit")
        position_closed = False

        buffer_price = self.buy_price * (1 - buffer_percent)  # calculate the buffer price level

        # Cancel all open sell orders for the symbol
        open_orders = await exchange_trade.fetch_open_orders(symbol)
        for order in open_orders:
            if order['side'] == 'sell':
                await exchange_trade.cancel_order(order['id'])
                print("Canceled open sell order:", order['id'])

        # First, try to sell at the buffer price
        exit_order = await exchange_trade.create_limit_sell_order(symbol, available_base_balance, buffer_price)
        print("Exit order placed at buffer price:", exit_order)

        filled_order = await self.wait_for_order_fill(exit_order['id'])

        if filled_order:
            print("Exit order filled at buffer price.")
            self.position_open = False
            position_closed = True
        else:
            print("Exit order not filled at buffer price. Attempting to sell at best ask price.")

            # Cancel the unfilled buffer price order
            await exchange_trade.cancel_order(exit_order['id'])

            # Sell at the best ask price
            exit_order_best_ask = await exchange_trade.create_limit_sell_order(symbol, available_base_balance, best_ask_price)
            print("Exit order placed at best ask price:", exit_order_best_ask)

            filled_order_best_ask = await self.wait_for_order_fill(exit_order_best_ask['id'])

            if filled_order_best_ask:
                print("Exit order filled at best ask price.")
                self.position_open = False
                position_closed = True
            else:
                print("Exit order not filled at best ask price.")
        return position_closed
    
    def check_conditional_exit(self, best_bid_price, buffer_percent):
        #print("Called check_conditional_exit")
        buffer_price = self.buy_price * (1 - buffer_percent)
        print("Buffer price:", buffer_price)
        print("Buy price:", self.buy_price)
        print("Best bid price:", best_bid_price)
        
        return best_bid_price <= buffer_price
        
symbol = 'BTC/USDT'
timeframe = '1m'
limit = 1000

# Enter api keys and secrets and initialize exchange for data account
api_key1 = ''
secret1 = ''
password1 = ''

exchange = ccxtpro.kucoin({
    'apiKey': api_key1,
    'secret': secret1,
    'password': password1,
    'enableRateLimit': True,
    })

# Enter api keys and secrets and initialize exchange for trade account
api_key2 = ''
secret2 = ''
password2 = ''

exchange_trade = ccxtpro.kucoin({
    'apiKey': api_key2,
    'secret': secret2,
    'password': password2,
    'enableRateLimit': True,
    })
async def main(amm, exchange, exchange_trade):

    exchange_id = 'kucoin'
    
    exchange_class = getattr(ccxtpro, exchange_id)
    exchange = exchange_class({
        'apiKey': api_key1,
        'secret': secret1,
        'timeout': 30000,
        'password': password1,
        'enableRateLimit': True,
    })

    exchange_trade_class = getattr(ccxtpro, exchange_id)
    exchange_trade = exchange_trade_class({
        'apiKey': api_key2,
        'secret': secret2,
        'timeout': 30000,
        'password': password2,
        'enableRateLimit': True,
    })

    # Initialize the AdaptiveMarketMaker class
    amm = AdaptiveMarketMaker(exchange, exchange_trade, symbol, timeframe, limit, window_length=30)
    
    # Initialize variables
    lookback_window = 1000
    historical_imbalances = []
    ohlcv_data = []  # Add this line to initialize the ohlcv_data list
    last_timestamp = None  # Add this line to initialize the last_timestamp variable
    adx_threshold = 20  # Set your desired ADX trend strength threshold
    break_even_percent = 0.01  # Set the desired break even percentage here
    trailing_stop_percents = [0.0125, 0.0135]  # Set the desired trailing stop percentage levels here
    profit_percent = 0.015  # Set the desired profit percentages here
    commission_rate = 0.001  # Set the desired commission rate here
    position_opened = False
    atrp_exit_threshold = 0.25  # Set your desired ATRP exit threshold
    obi_exit_threshold = 0.25    # Set your desired OBI exit threshold
    bes_price = None  # Add this line to initialize the bes_price variable
    min_order_size = 0.50  # Set the minimum order size here
    min_spread_percent = 0.001  # Set the desired minimum spread percentage
    order_amount = 20
    last_fetch_time = 0
    csv_filename = 'historical_ohlcv.csv'
    trailing_stop_levels = []
    buffers = []
    retraced_breached_tsl = False
    trailing_stop_calculated = False
    buffer_percent = 0.01
    
    # Initialize the scheduler
    scheduler = BackgroundScheduler()

    # Load historical data
    historical_timeframe = '1m'  # Define your desired timeframe
    since = exchange.parse8601('2023-01-01T00:00:00Z')  # Define the starting date for historical data
    csv_filename = 'historical_ohlcv.csv'
    async def schedule_update_historical_data():
        await amm.update_historical_data(csv_filename, exchange, symbol, historical_timeframe, since) 
    
    # Schedule the update_historical_data function to run every 24 hours
    scheduler.add_job(
    schedule_update_historical_data,
    'interval',
    hours=24,
    executor='asyncio',
    misfire_grace_time=3600,
    )
    # Start the scheduler
    scheduler.start()

# Run the main loop   
    while True:
        try:
           
            data = await amm.watch_trades_and_build_ohlcvc_and_watch_order_book()

            # Append the new ohlcv data to the ohlcv_data list
            orderbook = data['orderbook']
            new_ohlcv_data = data['ohlcv'][-1] if data['ohlcv'] is not None else None

            if new_ohlcv_data is not None:
                if new_ohlcv_data[0] != last_timestamp:
                    last_timestamp = new_ohlcv_data[0]
                    ohlcv_data.append(new_ohlcv_data)
                        
            # Get balance details (UPDATE WITH WS API)
            balance = await exchange_trade.fetch_balance()
            available_quote_balance = balance[symbol.split('/')[1]]['free']  # USDT
            available_base_balance = balance[symbol.split('/')[0]]['free']  # BTC

            # Calculate the minimum order size precision
            market = await exchange_trade.load_markets()
            amount_precision = market[symbol]['precision']['amount']

            # Get orderbook details
            orderbook = await amm.watch_order_book()
            best_bid_price = amm.get_best_bid(orderbook)
            best_ask_price = amm.get_best_ask(orderbook)          

            # Update the current price (St)
            if best_bid_price is not None and best_ask_price is not None:
                St = (best_bid_price + best_ask_price) / 2
                amm.St = St
            
            # Calculate the commission cost and update the cost of liquidity provision (k)
            amm.update_cost_of_liquidity_provision(commission_rate, best_bid_price, best_ask_price, order_amount, St )

            # Calculate the current time since the start of the day (in seconds)
            now = datetime.now()
            midnight = datetime.combine(now.date(), datetime.min.time())
            t = (now - midnight).total_seconds()

            # Update the current time t in amm
            amm.t = t
            
            # Get recent trade data
            trades = await amm.watch_trades()
            new_trade_price = trades[-1]['price']  # Get the price of the last trade

            # Update recent_trade_prices with new trade data
            amm.update_recent_trade_prices(new_trade_price)

            # Call the update_real_time_volatility function
            amm.update_real_time_volatility()
            sigma = amm.sigma

            # Calculate the inventory (IT)
            IT = available_base_balance

            # Load historical data
            historical_ohlcv = await amm.load_historical_data(csv_filename, exchange, symbol, historical_timeframe, since)

            # Calculate trend
            trend = amm.calculate_trend(historical_ohlcv, window_length=amm.window_length)

            # Print the trend value
            #print(f"Trend value: {trend}")

            # Calculate position risk outside of trend condition
            position_risk = amm.calculate_position_risk(historical_ohlcv, window_length=amm.window_length)
            amm.c = position_risk

            # Calculate the inventory-risk-adjusted mid-price Q_mt
            Q_mt = amm.calculate_inventory_risk_adjusted_mid_price(St, IT, amm.t, amm.T, amm.gamma, sigma, amm.k, amm.c)

            optimal_bid_price = amm.calculate_optimal_bid_price(sigma, best_bid_price)

            optimal_ask_price = amm.calculate_optimal_ask_price(sigma, best_ask_price)

            # Initialize the position_opened variable
            #position_opened = False

            # Check if trend is favorable
            if trend > 0.00:  # or whatever threshold you choose for a "favorable" trend
                # Call the function to execute buy order
                await amm.execute_buy_order(exchange_trade, symbol, optimal_bid_price, available_quote_balance, commission_rate)
                position_opened = amm.position_open
                #print("Position opened - Buy Order:", position_opened)

            # Call the function to find significant bid and ask levels
            significant_ask_levels, ask_level_sizes = amm.find_significant_bid_ask_levels(orderbook)

            # Calculate the equity allocation for each level
            allocated_equity = available_quote_balance
            ask_equity_allocations = amm.calculate_dynamic_equity_allocations(allocated_equity, trailing_stop_levels)
            print("Ask equity allocations:", ask_equity_allocations)

            if position_opened:
                #print("Position opened", position_opened)
                # Check if conditions for conditional exit are met
                exit_necessary = amm.check_conditional_exit(best_bid_price, buffer_percent)
                #print("Called check_conditional_exit")
                print("Exit necessary:", exit_necessary)
                if exit_necessary:
                    # If so, execute conditional exit
                    try:
                        position_closed = await amm.execute_conditional_exit(exchange_trade, symbol, available_base_balance, best_bid_price, best_ask_price, buffer_percent)
                        print("Price retraced immediately after opening position. Successfully executed conditional exit strategy and closed the position.")
                    except Exception as e:
                        print("Exception occurred during execute_conditional_exit:", e)
                else:
                    if not trailing_stop_calculated:
                        trailing_stop_levels, buffers, retraced_breached_tsl = amm.calculate_trailing_stop_levels(amm.entry_prices[-1], significant_ask_levels, optimal_bid_price)
                        print("Trailing stop levels:", trailing_stop_levels)
                    
                        # Once calculated, set this to True so it won't calculate again
                        trailing_stop_calculated = False

            # Call the function to execute sell order
            retraced_breached_tsl = await amm.execute_sell_order(exchange_trade, symbol, optimal_ask_price, available_base_balance, available_quote_balance, significant_ask_levels, ask_equity_allocations, amount_precision, trailing_stop_levels, buffers, retraced_breached_tsl)
            position_opened = amm.position_open
            print("Available base balance:", available_base_balance)
            print("Available quote balance:", available_quote_balance)
            print("Significant ask levels:", significant_ask_levels)
            print("Equity allocations:", ask_equity_allocations)
            print("Trailng stop levels:", trailing_stop_levels)
            print("Position Opened:", position_opened)
            print("Retraced Breached TSL:", retraced_breached_tsl)
            
            # Loop through the list of surpassed TSLs
            for tsl in amm.surpassed_tsls:
                # Calculate the retraced level
                retraced_level = tsl * (1 + buffer_percent)
                print(f"Price: {St}, retraced_level: {retraced_level}")

                # If the current price has retraced to or below this level, mark the TSL as breached
                if St <= retraced_level:
                    print(f"Price: {St}, retraced_level: {retraced_level}")
                    retraced_breached_tsl = True
                    amm.breached_tsl_levels.append(tsl)
                    amm.surpassed_tsls.remove(tsl)  # remove the breached TSL from surpassed_tsls
                    print(f"Breached TSL level: {tsl}")
    
                    # If a TSL has been breached, cancel all open sell orders and extend the list of breached TSLs
                    cancelled_order_prices = await amm.cancel_all_sell_orders(symbol) 
                    print("Cancelled order prices:", cancelled_order_prices)
                    amm.breached_tsl_levels.extend(cancelled_order_prices)
                    
                    # Exit the position at the market price
                    await exchange_trade.create_market_sell_order(symbol, available_base_balance)
                    print("Exited the position at the market price.")
        
                    break
            
            # Recalculate significant bid and ask levels
            significant_ask_levels, ask_level_sizes = amm.find_significant_bid_ask_levels(orderbook)

            # Recalculate the equity allocation for each level
            ask_equity_allocations = amm.calculate_dynamic_equity_allocations(available_quote_balance, trailing_stop_levels)

            # Resubmit sell orders at new significance levels
            await amm.execute_sell_order(exchange_trade, symbol, optimal_ask_price, available_base_balance, available_quote_balance, significant_ask_levels, ask_equity_allocations, amount_precision, trailing_stop_levels, buffers, retraced_breached_tsl)

            # Check if the position is closed, and reset position_open to allow new trades
            if not position_opened and amm.position_open:
                amm.position_open = False
                print("Position opened - End of Loop:", position_opened)
                trailing_stop_calculated = True

            # Get the current working directory
            current_working_directory = os.getcwd()
            
            # Set the directory for the .csv file
            csv_directory = os.path.join(current_working_directory, "csv_files")
        
            # Check if the csv_directory exists
            if not os.path.exists(csv_directory):
            # Create the directory if it doesn't exist
                os.makedirs(csv_directory)
            
            # Create the .csv file path
            csv_file_name = os.path.join(csv_directory, "BTC-USDT.csv")
            
            # Data points for csv output file  
            data_points = [ 
                            t,
                            St,
                            available_quote_balance,
                            available_base_balance,
                            sigma,
                            trend,
                            best_bid_price,
                            best_ask_price,
                            optimal_bid_price,
                            optimal_ask_price,
                            significant_ask_levels,
                            ask_equity_allocations,
                            trailing_stop_levels,
                            amm.execute_buy_order,
                            amm.execute_sell_order,
                            ]       
            
            # Header for csv output file
            csv_header = ["Time", "St", "Balance", "Available Quote Balance", "Available Base Balance", "Sigma", "Trend", "Best Bid Price", "Best Ask Price", "Optimal Bid Price", "Optimal Ask Price", "Significant Ask Levels", "Ask Equity Allocations", "Trailing Stop Levels", "Execute Buy Order", "Execute Sell Order"]
            # Call this function inside your main loop to write data points
            amm.write_data_to_csv(csv_file_name, data_points, csv_header)
            
        except Exception as e:
            print(f'Error: {e}')
            traceback.print_exc()  # Print detailed traceback
            await asyncio.sleep(0.01)

if __name__ == '__main__':

    try:
        asyncio.run(main(None, exchange, exchange_trade))

    except KeyboardInterrupt:
        print("Exiting the script gracefully...")
        exchange.close()
        exchange_trade.close()            

