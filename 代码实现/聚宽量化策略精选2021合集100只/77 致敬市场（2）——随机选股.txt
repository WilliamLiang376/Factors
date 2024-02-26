# 风险及免责提示：该策略由聚宽用户在聚宽社区分享，仅供学习交流使用。
# 原文一般包含策略说明，如有疑问请到原文和作者交流讨论。
# 原文网址：https://www.joinquant.com/post/32870
# 标题：致敬市场（2）——随机选股
# 作者：Gyro

import pandas as pd
import numpy as np

def initialize(context):
    # setting parameters
    g.index = '399311.XSHE' # 国证1000
    g.position_num = 100 # num >= 100
    # setting system
    set_benchmark(g.index)
    log.set_level('order', 'error')
    set_option('use_real_price', True)
    set_option('avoid_future_data', False) # 提取指定年份年报不能开启
    # setting strategy
    run_monthly(iHandle, 1, '14:50')
    # setting variables
    g.buy_counter = 0 # buy counter
    g.sell_tounter = 0 # sell counter

def iHandle(context):
    # one trading-day one year
    if context.current_dt.month not in [1, 7]:
        return
    # Begin to work
    log.info('Hello, working')
    # stocks
    cdata = get_current_data()
    stocks = get_index_stocks(g.index)
    # Sell
    for s in context.portfolio.positions:
        if s not in stocks:
            log.info('sell', s, cdata[s].name)
            order_target(s, 0)
            g.sell_tounter = g.sell_tounter + 1
    # Buy
    N = len(stocks)
    poisition_size = context.portfolio.total_value / g.position_num
    while context.portfolio.available_cash > poisition_size:
        k = np.random.randint(N)
        s = stocks[k]
        if s not in context.portfolio.positions:
            log.info('buy', s, cdata[s].name)
            order_value(s, poisition_size)
            g.buy_counter = g.buy_counter + 1

def on_strategy_end(context):
    log.info('... ... ... Trading Report ... .... ...')
    log.info('total return', context.portfolio.returns)
    log.info('total buy', g.buy_counter)
    log.info('total sell', g.sell_tounter)
    log.info('... ... ... END ... ... ...')
# end
