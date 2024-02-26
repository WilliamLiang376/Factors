# 风险及免责提示：该策略由聚宽用户在聚宽社区分享，仅供学习交流使用。
# 原文一般包含策略说明，如有疑问请到原文和作者交流讨论。
# 原文网址：https://www.joinquant.com/post/32383
# 标题：致敬市场(1)，探讨指数化策略投资
# 作者：Gyro

import pandas as pd
import datetime as dt

def initialize(context):
    # setting parameters
    g.index = '399317.XSHE' #国证1000
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
    # only run on the first trading-day of May
    if context.current_dt.month != 5:
        return
    # Begin to work
    log.info('Hello, every year')
    # choice stocks
    stocks = choice_stocks(context, 150)
    # trading on choice
    trader(context, stocks, 100)
    # report porfolio
    report_porfolio(context)

def choice_stocks(context, choice_num):
    # init
    stocks = get_index_stocks(g.index)
    last_year = context.current_dt.year - 1
    log.info('stocks pool size', len(stocks))
    # get annual report
    q = query(
        indicator.code,
        indicator.pubDate,
        indicator.eps,
      ).filter(
          income.code.in_(stocks)
      )
    df = get_fundamentals(q, statDate=str(last_year)).dropna().set_index('code')
    log.info('Total annual report of the year', last_year, len(df))
    # sort
    df = df.sort_values(by='eps', ascending=False).head(choice_num)
    # report choice
    cdata = get_current_data()
    df['name'] = [cdata[s].name for s in df.index]
    pd.set_option('display.max_rows', len(df))
    log.info('Choice stocks\n', df)
    return df.index.tolist()

def trader(context, stocks, position_num):
    cdata = get_current_data()
    # Sell
    for s in context.portfolio.positions:
        if s not in stocks:
            log.info('sell', s, cdata[s].name)
            order_target(s, 0)
            g.sell_tounter = g.sell_tounter + 1
    # Buy
    poisition_size = context.portfolio.total_value / position_num
    for s in stocks:
        if context.portfolio.available_cash < poisition_size:
            break
        if s not in context.portfolio.positions:
            log.info('buy', s, cdata[s].name)
            order_value(s, poisition_size)
            g.buy_counter = g.buy_counter + 1

def report_porfolio(context):
    # table of positions
    cdata = get_current_data()
    tvalue = context.portfolio.total_value
    ptable = pd.DataFrame(columns=['name', 'volume', 'weight'])
    for s in context.portfolio.positions:
        ps = context.portfolio.positions[s]
        ptable.loc[s] = [cdata[s].name, ps.total_amount, 100*ps.value/tvalue]
    # sort
    ptable = ptable.sort_values(by='weight', ascending=False)
    # report
    pd.set_option('display.max_rows', len(ptable))
    log.info('Holding positions\n', ptable)
    log.info('Total positions', len(ptable))

def on_strategy_end(context):
    log.info('... ... ... Trading Report ... .... ...')
    log.info('total return', context.portfolio.returns)
    log.info('total buy', g.buy_counter)
    log.info('total sell', g.sell_tounter)
    report_porfolio(context) # all postion
    log.info('... ... ... END ... ... ...')
# end