# 风险及免责提示：该策略由聚宽用户在聚宽社区分享，仅供学习交流使用。
# 原文一般包含策略说明，如有疑问请到原文和作者交流讨论。
# 原文网址：https://www.joinquant.com/post/35737
# 标题：折价基金统计套利
# 作者：Gyro

# 克隆自聚宽文章：https://www.joinquant.com/post/31201
# 标题：折价基金
# 作者：KaiXuan

# 克隆自聚宽文章：https://www.joinquant.com/post/31107
# 标题：折价基金投资策略-Gyro版
# 作者：Gyro

from jqdata import *

def initialize(context):
    # 开启异步报单
    set_option('async_order', True)
    set_option('use_real_price', True)
    log.set_level('order', 'error')
    # set_benchmark('161005.XSHE')#设置基准
    set_order_cost(OrderCost(close_tax=0.001, open_commission=0.00015, close_commission=0.00015, min_commission=0), type='stock')

def after_code_changed(context):
    unschedule_all()
    # run_daily(keepalive,time='10:00', reference_security='000300.XSHG')
    # run_daily(keepalive,time='14:00', reference_security='000300.XSHG')
    
    run_weekly(adjust, 1,time='open', reference_security='000300.XSHG')
    # run_weekly(rebuy, 1,time='9:35', reference_security='000300.XSHG')
    # run_weekly(rebuy, 1,time='9:40', reference_security='000300.XSHG')
    
    run_weekly(sell, 5,time='14:25', reference_security='000300.XSHG')
    # run_weekly(resell, 5,time='14:30', reference_security='000300.XSHG')
    # run_weekly(resell, 5,time='14:35', reference_security='000300.XSHG')
    
    g.num = 30
    g.buy_list = []

def keepalive(context):
    # log.info('当前时间是%s，程序运行正常'%str(context.current_dt.time()))
    # log.info('当前可用资金%s'%context.portfolio.available_cash)
    # log.info('当前持仓%s'%context.portfolio.positions.keys())
    log.info('当前程序运行正常')
    
def process_initialize(context):
    #　全部LOF、ETF、FJA列表
    g.all_lof = get_all_securities(['lof', 'etf', 'fja'])
    # g.all_lof = get_all_securities(['lof', 'etf'])#, 'fja'
    log.info('lof 总数 %s 个'%len(g.all_lof)) 

def adjust(context):
    # 获得当前列表
    lofs = g.all_lof
    lofs = lofs[lofs['start_date'] < context.previous_date]
    lofs = lofs[lofs['end_date'] > context.current_dt.date()]
    lofs = lofs.index.tolist()
    # 基金净值
    net_value=get_extras('unit_net_value', lofs, end_date=context.previous_date, df=True, count=1).T
    net_value.columns=['unit_net_value']
    # 基金价格
    close = history(count=1, unit='1d', field="close", security_list=lofs).T
    close = close.dropna()
    # close = close.sort_index(by="close",ascending=False)
    close = close[close < 10] #不买reits
    # log.info('close \n ',close)
    
    volume=history(count=1, unit='1d', field="volume", security_list=lofs).T
    volume = volume.dropna()
    volume.columns = ["vol"]
    volume = volume.sort_index(by="vol",ascending=False)

    # volume = volume[:int(len(g.all_lof) * 0.618)]
    volume = volume[:int(len(g.all_lof) * 0.382)]

    # volume = volume[(volume.vol>500000)]
    # log.info('volume \n',volume)
    log.info('成交量过滤之后还有%s个'%len(volume))
    
    close = close[close.index.isin(volume.index)]
    close.columns=['close']
    # score = 价格/净值
    val=net_value.join(close)
    val['score'] = val['close'] / val['unit_net_value']
    # 过滤
    val = val[(val['score'] > 0)]
    val = val[(val['score'] < 0.99)]
    # val = val[(val['score'] < 1)]
    # 排序
    buy_count = g.num
    #20 1.32 2.221
    #30 1.15 2.647 
    #10 1.6 2.34
    val = val.sort_index(by='score', ascending = True).head(buy_count)
    log.info('待买列表 \n ',val)
    buy_list = val.index.tolist()
    g.buy_list = buy_list   #g.buy_list留着周五卖出时候用
    # log.info('etf折价策略前%s：\n%s'%(g.num,g.buy_list))

    # 买入
    if buy_list and context.portfolio.available_cash > 500:
        buy_value = context.portfolio.total_value / len(buy_list)
        # buy_value = context.portfolio.available_cash / len(buy_list)
        for f in buy_list:
            order_target_value(f, buy_value)
            log.info('正在买入%s，买入金额%s元'%(f,buy_value))
    else:
        if context.portfolio.available_cash < 500:
            log.info('可用金额小于500元，停止买入')
        if len(buy_list) == 0 :
            log.info('buy_list为空，停止买入')

def sell(context):    # 卖出
    for f in context.portfolio.positions.keys():
        if f not in g.buy_list:
            log.info('正在清仓%s'%f)
            order_target_value(f, 0)
            
def resell(context):
    orders = get_open_orders()  #获取未成卖单
    # 循环，撤销订单
    if orders:  #如果有待成交的卖单
        log.info('当前有未成交卖单')
        for _order in orders.values():
            cancel_order(_order)
            log.info('正在撤单 %s'%_order)  # \n 
        for stock in set(context.portfolio.positions.keys()):
            if stock not in g.buy_list:
                log.info('再次卖出股票%s'%stock)
                order_target_value(stock, 0)
            if stock in g.buy_list:
                print('已经持有的股票%s这次又选出来了，不卖'%stock)
    else:
        log.info('所有卖单都已成交')
    
def rebuy(context):
    orders = get_open_orders()  #获取未成下单
    # 循环，撤销订单
    if orders:  #如果有待成交的单
        log.info('当前有未成交买单')
        
        for _order in orders.values():
            cancel_order(_order)
            log.info('正在撤单 %s'%_order)  # \n
            
        if g.buy_list and context.portfolio.available_cash > 500:
            buy_value = context.portfolio.total_value / len(g.buy_list)
            # buy_value = context.portfolio.available_cash / len(g.buy_list)
            for f in g.buy_list:
                order_target_value(f, buy_value)
                log.info('再次买入%s，应买入金额%s元'%(f,buy_value))
        else:
            if context.portfolio.available_cash < 500:
                log.info('可用金额小于500元，停止买入')
            if len(buy_list) == 0 :
                log.info('buy_list为空，停止买入')
    else:
        log.info('所有买单都已成交')

# end