# pythonst
## 一个小市值策略节选

from jqdata import *
from jqfactor import get_factor_values
import numpy as np
import pandas as pd
import time
from jqlib.technical_analysis  import *
pd.options.display.max_rows = 500
pd.options.display.max_columns = 100
#初始化函数
def initialize(context):
    # 用真实价格交易
    set_option('use_real_price', True)
    # 打开防未来函数
    set_option("avoid_future_data", True)
    # 早上选股
    run_daily(my_trade_check_out_list,  time='7:00', reference_security='000300.XSHG')
    # 早上卖股
    run_daily(my_trade_sell,  time='9:30', reference_security='000300.XSHG')
    # 早上买股
    run_daily(my_trade_buy,  time='9:31', reference_security='000300.XSHG')
   
def after_code_changed(context):
    log.info("after_code_changed_start")
    log.info("已经持有股票：[%s]" % list(context.portfolio.positions))
    set_parameter(context)
    log.info("after_code_changed_end")
    send_message("自己开发的小市值策略开始运行了...")
   
def set_parameter(context):
    log.set_level('order', 'error')
    log.set_level('history', 'error')
    #log.set_level('strategy', 'error')
    log.set_level('system', 'error')
    log.info("initialize")   
    set_benchmark('000300.XSHG')
    # 设置交易成本万分之三
    set_order_cost(OrderCost(close_tax=0.001, open_commission=0.0008, close_commission=0.0003, min_commission=5), type='stock')
    # 持仓数
    g.stock_num = 2
    # 持仓股票的每周卖出时涨停的股票
    g.stock_high_limit = []
    # 选股
    g.check_out_list = []
 
# 卖出股票交易
def my_trade_sell(context):
    check_out_list = []
    current_data = get_current_data()
    for stock in context.portfolio.positions:
        # 涨停就不卖出
        if current_data[stock].last_price >= current_data[stock].high_limit:
            check_out_list.append(stock)
    check_out_list=list(set(check_out_list + g.check_out_list))
    adjust_position_sell(context, check_out_list)
 
 # 买入股票交易
def my_trade_buy(context): 
    adjust_position_buy(context, g.check_out_list)
   

   
# 选股票
def my_trade_check_out_list(context):
    g.check_out_list = []
    g.check_out_list = get_stock_list(context)
    g.check_out_list = g.check_out_list[:g.stock_num]
    desc = '今日自选股:{}'.format(g.check_out_list)
    log.info(desc)
    send_message(desc)

#2-2 选股模块             and pe_ratio < 90 \
def get_stock_list(context):
    yes_day = context.previous_date
    stock_list = get_all_securities(types=['stock'], date=yes_day).index.tolist()
    q = query(valuation.code).filter(
        valuation.code.in_(stock_list),\
        valuation.pb_ratio > 0.01, \
        valuation.pb_ratio < 8, \
        valuation.pe_ratio < 90, \
        valuation.market_cap > 5, \
        valuation.market_cap < 300 \
        )
    df = get_fundamentals(q)
    stock_list = list(df.code)
   
    stock_list = filter_stock_before(context,stock_list)
    stock_list = filter_stock_by_ROC120(context,stock_list)
    stock_list = filter_stock_by_sharpe_ratio_120(context,stock_list)
    if len(stock_list) ==0:
        return stock_list
    df_max_min = get_max_min_closepbmark(stock_list,context)
    stock_list_tmp = []
    for stock in stock_list:
        turnover_ratio = float(df_max_min.loc[stock].turnover_ratio)
        max_close = float(df_max_min.loc[stock].max_close)
        min_close = float(df_max_min.loc[stock].min_close)
        close = float(df_max_min.loc[stock].close)
        max_pb_ratio = float(df_max_min.loc[stock].max_pb_ratio)
        min_pb_ratio = float(df_max_min.loc[stock].min_pb_ratio)
        pb_ratio = float(df_max_min.loc[stock].pb_ratio)
        market_cap = float(df_max_min.loc[stock].market_cap)
        pe_ratio = float(df_max_min.loc[stock].pe_ratio)
        if turnover_ratio < 3 and turnover_ratio > 0.1 \
            and max_close/close < 1.15 and min_close/close > 0.75 \
            and max_pb_ratio/pb_ratio < 1.3 and min_pb_ratio/pb_ratio > 0.6:
            stock_list_tmp.append(stock)
    q = query(valuation.code,valuation.circulating_market_cap).filter(valuation.code.in_(stock_list_tmp)).order_by(valuation.circulating_market_cap.asc())
    df = get_fundamentals(q)
    final_list = list(df.code)
    return final_list

def get_max_min_closepbmark(stock_list,context):
    df = pd.DataFrame(columns = ["day","turnover_ratio","close","max_close","min_close","pb_ratio","max_pb_ratio","min_pb_ratio","market_cap","max_market_cap","min_market_cap","pe_ratio"])
    stock_list_list = split_stock_list(stock_list,lenght = 20)
    for stock_list_tmp in stock_list_list:
        dftmp = get_valuation(stock_list_tmp, count=160,end_date=context.previous_date, fields=['turnover_ratio','pb_ratio','market_cap','pe_ratio']) 
        prices_df = get_bars(stock_list_tmp,count=30,unit='1w', fields=('date','close'),include_now=True, end_dt=context.current_dt,df = True)
        for stock in stock_list_tmp:
            df.loc[stock] = [context.previous_date,dftmp[dftmp.code==stock]['turnover_ratio'].values[-1], \
                        prices_df.loc[stock]['close'].values[-1],np.max(prices_df.loc[stock]['close'].values), np.min(prices_df.loc[stock]['close'].values), \
                        dftmp[dftmp.code==stock]['pb_ratio'].values[-1],np.max(dftmp[dftmp.code==stock]['pb_ratio'].values),np.min(dftmp[dftmp.code==stock]['pb_ratio'].values), \
                        dftmp[dftmp.code==stock]['market_cap'].values[-1],np.max(dftmp[dftmp.code==stock]['market_cap'].values),np.min(dftmp[dftmp.code==stock]['market_cap'].values), \
                        dftmp[dftmp.code==stock]['pe_ratio'].values[-1]]
    return df

#工具函数-开盘前过滤股票
def filter_stock_before(context,stock_list):
    # 过滤科创、创业板、ST、停牌、近一段时间内卖出过的股票
    curr_data = get_current_data()
    stock_list = [stock for stock in stock_list if not (
            curr_data[stock].paused or  # 停牌
            curr_data[stock].is_st or  # ST
            ('ST' in curr_data[stock].name) or
            ('*' in curr_data[stock].name) or
            ('退' in curr_data[stock].name) or
            (stock.startswith('300')) or  # 创业
            (stock.startswith('301')) or  # 创业
            (stock.startswith('688')) or  # 科创
            (stock.startswith('689'))   # 科创
    )]

    # 过滤36天内有过停牌记录的股票
    yesterday = context.previous_date
    prices_df = get_price(stock_list, count=36,end_date=yesterday,  frequency="daily", fields=["money"])['money']
    stock_list = [stock for stock in stock_list if (not prices_df[stock].isnull().values.any()) and (prices_df[stock][prices_df[stock]==0].count() == 0)]
    # 过滤上市日期不足250日股票后的列表
    stock_list =[stock for stock in stock_list if not (yesterday - get_security_info(stock).start_date) < datetime.timedelta(days=250)]
    return stock_list

# 120日变动速率
def filter_stock_by_ROC120(context,stock_list):
    final_list = []
    stock_list_list = split_stock_list(stock_list,lenght = 100)
    for stock_list_tmp in stock_list_list:
        factor_data_ROC120 = get_factor_values(securities=stock_list_tmp, factors=["ROC120"], count=10, end_date=context.current_dt) 
        for stock in stock_list_tmp:
            ROC120_ma10 = np.mean(factor_data_ROC120["ROC120"][stock].values)
            ROC120 = factor_data_ROC120["ROC120"][stock].values[-1]
            if ROC120 > 20 and ROC120 > ROC120_ma10 and ROC120 > factor_data_ROC120["ROC120"][stock].values[-2]:
                final_list.append(stock)
    return final_list
   
# 120日夏普比率
def filter_stock_by_sharpe_ratio_120(context,stock_list):
    final_list = []
    stock_list_list = split_stock_list(stock_list,lenght = 100)
    for stock_list_tmp in stock_list_list:
        factor_data_sharpe_ratio_120 = get_factor_values(securities=stock_list_tmp, factors=["sharpe_ratio_120"], count=10, end_date=context.current_dt)
        for stock in stock_list_tmp:
            sharpe_ratio_120_ma10 = np.mean(factor_data_sharpe_ratio_120["sharpe_ratio_120"][stock].values)
            sharpe_ratio_120 = factor_data_sharpe_ratio_120["sharpe_ratio_120"][stock].values[-1]
            if sharpe_ratio_120 > 1 and sharpe_ratio_120 > sharpe_ratio_120_ma10:
                final_list.append(stock)
    return final_list 

# 分割list   
def split_stock_list(stock_list,lenght = 10):
    len_all = len(stock_list)
    stock_list_tmp = []
    if len_all <= lenght:
        stock_list_tmp.append(stock_list)
    else:
        for i in range(0,len_all,lenght):
            stock_list_tmp.append(stock_list[i:(i+lenght)])
    return stock_list_tmp
   

#交易模块-开仓
#买入指定价值的证券,报单成功并成交(包括全部成交或部分成交,此时成交量大于0)返回True,报单失败或者报单成功但被取消(此时成交量等于0),返回False
def open_position(security, value):
    order = order_target_value(security, value)
    if order != None and order.filled > 0:
        return True
    return False

#交易模块-平仓
#卖出指定持仓,报单成功并全部成交返回True，报单失败或者报单成功但被取消(此时成交量等于0),或者报单非全部成交,返回False
def close_position(position):
    security = position.security
    order = order_target_value(security, 0)  # 可能会因停牌失败
    if order != None:
        if order.status == OrderStatus.held and order.filled == order.amount:
            return True
    return False

                   
#交易模块-买入股票                   
def adjust_position_buy(context, buy_stocks):
    buy_stocks = [stock for stock in buy_stocks if stock not in context.portfolio.positions]
    position_count = len(context.portfolio.positions)
    if g.stock_num > position_count:
        value = context.portfolio.cash / (g.stock_num - position_count)
        for stock in buy_stocks:
            if context.portfolio.positions[stock].total_amount == 0:
                if open_position(stock, value):
                    if len(context.portfolio.positions) >= g.stock_num:
                        break   
#交易模块-卖出股票                       
def adjust_position_sell(context, buy_stocks):
    for stock in context.portfolio.positions:
        if stock not in buy_stocks and context.portfolio.positions[stock].closeable_amount > 0:
            position = context.portfolio.positions[stock]
            close_position(position)
    stock_list = [stock for stock in context.portfolio.positions if stock in buy_stocks] 
    if len(stock_list) > 0:
        log.info("%s已有股票继续持有" % (stock_list))
