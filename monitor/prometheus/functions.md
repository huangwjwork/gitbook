# function函数

[TOC]
## abs
求绝对值
## absent
检测是否为空
## ceil
四舍五入
## changes
一定时间范围内改变的次数
## clamp_max
设置上限，最大值不大于设置的上限
```shell
clamp_max(v instant-vector,max scalar)
```
## clamp_min
设置下限，最小值不小于设置的下限
```shell
clamp_min(v instant-vector,min scalar)
```
## day_of_month
## day_of_week
## days_in_month
每个月的天数  28-31
## delta
计算区间开始时与当前值的差值
```shell
delta(cpu_temp_celsius{host="zeus"}[2h])
```
## deriv
deriv（v range-vector）使用简单线性回归计算范围向量v中时间序列的每秒导数。  
仅限于gauge数据
## exp
## floor
取整  
0.6 --> 0   
## histogram_quantile
## holt_winters
## hour
返回小时数，0-23
## idelta
## increase
一定时间内的增长量，比如CPU时间，访问量(counter数据类型)
## irate
一定时间区间内，两个相邻的值之间的变化率（对应整体的变化量）
## label_join
添加标签
## label——replace
替换标签
## ln
自然对数
## log10
十进制对数
## minute
返回分钟（0-59）
## mouth
返回月份（1-12） 
## predict_linear
简单线性回归
## rate
计算一段时间内美妙的平均增长率（与irate类似但又不同）
## resets
返回一定范围内呗reset的次数，仅适用count  
resets(up[1y])  查看一年内exporter重启次数
## round
## scalar
向量转换为标量  vector -> scalar  
## sort
按照升序返回向量元素
## sort_desc
降序返回向量元素
## sqrt
计算向量的平方根
## time
返回Unix时间
## timestamp
返回时间戳
## vector
讲标量座位没有标签的向量返回