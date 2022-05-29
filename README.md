##### 用户行为数仓需求

```
需求一：每日新增用户相关指标

新增用户(新增设备)：指第一次安装并且使用app的用户，后期卸载之后再使用就不算新用户

1：每日新增用户量
ods层的表名为：ods_user_active
dwd层的表名为：dwd_user_active

第一步：我们基于清洗之后打开app上报的数据创建一个历史表，这个表里面包含的有xaid字段，针对每天的数据基于xaid进行去重
第二步：如果我们要计算2026年2月1日的新增用户量，就拿这一天上报的打开app的数据，和前面的历史表进行left join，使用xaid进行关联，关联不上的数据则为新增数据

举个例子：
(1) 第一步会产生一个历史表，dws_user_active_history，这个表中有一个xaid字段
dws_user_active_history
xaid
a1
b1
c1
d1

(2) 第二步会产生一个临时表，表里面包含的是那一天上报的打开app的数据
dws_user_active_20260201_tmp
xaid
a1
b1
x1
y1
z1

(3) 对这两个表进行left join
dws_user_active_20260201_tmp	dws_user_active_history
xaid							xaid
a1								a1
b1								b1
x1								null
y1								null
z1								null

此时：dws_user_active_history.xaid 为null的数据条数即为当日新增用户数

第三步：将计算出来的每日新增用户信息保存到表dws_user_new_item 表中，这个表按照天作为分区，便于后期其它需求使用这个表

第四步：基于dws_user_new_item对数据进行聚合，将计算出来的新增用户数量保存到结果表app_user_new_count

注意：在这里处理完之后，还需要将dws_user_active_20260201_tmp这个临时表中的数据insert到dws_user_active_history这个历史表中
最后删除这个临时表即可。

2：每日新增用户量的日环比和周同比
同比：一般是指本期统计数据和往年的同时期的统计数据比较
环比：一般是指本期统计数据和上一期的统计数据作比较

日环比=(本期的数据-上一期的数据)/上一期的数据
日环比中的时间单位是天

周同比=(本期的数据-上一期的数据)/上一期的数据
周同比中的时间单位是周(7天)

实现思路：
直接基于app_user_new_count进行统计即可，可以统计出来某一天的日环比和周同比
生成一个新表app_user_new_count_ratio
里面包含日期、新增用户量、日环比、周同比




总结：我们最终需要在dws层创建三个表
dws_user_active_20260201_tmp
dws_user_active_history
dws_user_new_item

在app层需要创建两个表
app_user_new_count
app_user_new_count_ratio

针对dws层抽取脚本
1：表初始化脚本(初始化执行一次)
dws_mall_init_table_1.sh

2：添加区分数据脚本(每天执行一次)
dws_mall_add_partition_1.sh


针对app层抽取脚本
1：表初始化脚本(初始化执行一次)
app_mall_init_table_1.sh.sh

2：添加区分数据脚本(每天执行一次)
app_mall_add_partition_1.sh



计算2026-02-01~2026-02-09

sh dws_mall_add_partition_1.sh 20260201 - 20260228

sh app_mall_add_partition_1.sh 20260201 - 20260228


合并分区后sqoop导出
create table app_user_new_count_tmp as select * from app_user_new_count;


查看数据库连接状态
sqoop list-databases --connect jdbc:mysql://82.157.51.166:3306/ --username root --password mysqlmm

sqoop export \
--connect jdbc:mysql://82.xxx.xx.xx:3306/app_mall \
--username root \
--password xxx \
--table app_user_new_count \
--fields-terminated-by "\001" \
--export-dir '/user/hive/warehouse/app_mall.db/app_user_new_count_tmp' \
--input-null-non-string '\\N' \
--input-null-string '\\N' \
--m 1

sqoop export \
--connect jdbc:mysql://82.xxx.xx.xx:3306/app_mall \
--username root \
--password xxx \
--table app_user_new_count_ratio \
--fields-terminated-by "\001" \
--export-dir '/user/hive/warehouse/app_mall.db/app_user_new_count_ratio_tmp' \
--input-null-non-string '\\N' \
--input-null-string '\\N' \
--m 1
```



```

需求二：每日活跃用户(主活)相关指标

(1)：每日主活用户量
 直接使用dws层的dws_user_active_history这个表，直接求和即可获取到当日的主活用户量，将最终的结果保存到app层的app_user_active_count表中
(2)：每日主活用户量的日环比和周同比
这个指标直接基于每日主活用户量的表(app_user_active_count)进行计算即可，把最终的结果保存到app层的app_user_active_count_ratio表中



针对app层抽取脚本
1：表初始化脚本(初始化执行一次)
app_mall_init_table_2.sh

2：添加区分数据脚本(每天执行一次)
app_mall_add_partition_2.sh


sh app_mall_add_partition_2.sh 20260201 - 20260209

for((i=1;i<=9;i++))
do
    if [ $i -lt 10 ]
	then
	    dt="2026020"$i
	else
        dt="202602"$i	
	fi
	
	echo "app_mall_add_partition_2.sh" ${dt}
	sh app_mall_add_partition_2.sh ${dt}
done


如何统计每周主活，每月主活
周：按照自然周，每周一凌晨计算上一周的主活
月：按照自然月，每月1号计算上一个月的主活

```

##### 商品数仓需求

```
商品订单数据数仓开发

表名				说明				导入方式
ods_user			用户信息表			全量
ods_user_extend		用户扩展表			全量
ods_user_addr		用户收货地址表		全量
ods_goods_info		商品信息表			全量
ods_category_code	商品类目码表		全量
ods_user_order		订单表				增量
ods_order_item		订单商品表			增量
ods_order_delivery	订单收货表			增量
ods_payment_flow	支付流水表			增量
```

```
需求一：用户信息宽表

用户信息宽表包括服务端中的user表，user_extend表


实现思路：
对dwd_user表和dwd_user_extend表执行left join操作，通过user_id进行关联即可，将结果数据保存到dws_user_info_all表中


针对dws层抽取脚本
1：表初始化脚本(初始化执行一次)
dws_mall_init_table_1.sh

2：添加分区数据脚本(每天执行一次)
dws_mall_add_partition_1.sh
sh dws_mall_add_partition_1.sh 20260228
```

```
需求二：电商GMV
GMV：指一定时间段内的成交总金额
GMV多用于电商行业，实际指的是拍下的订单总金额，包含付款和未付款的部分。

我们在统计的时候就可以将订单表中的每天的所有订单金额全部累加起来就可以获取到当天的GMV了

实现思路：
对dwd_user_order表中的数据进行统计即可，通过order_money字段可以计算出来GMV
将结果数据保存到表app_gmv中


针对gmv字段的类型可以使用double或者decimal(10,2)都可以
decimal(10,2)可以更方便的控制小数位数，数据看起来更加清晰


针对app层抽取脚本
1：表初始化脚本(初始化执行一次)
sh app_mall_init_table_2.sh

2：添加分区数据脚本(每天执行一次)
sh app_mall_add_partition_2.sh

sh app_mall_add_partition_2.sh 20260222 - 20260228
```

```
需求三：商品相关指标
1：商品的销售情况(商品名称、一级类目、订单总量、销售额)
订单中的详细信息是在dwd_order_item表中，需要关联dwd_goods_info和dwd_category_cpde获取商品名称和商品一级类目信息
通过这个表可以统计出 前十名的销售商品

2：商品品类偏好Top10(商品一级类目、订单总量) 但是数据不多 就只是统计了前三类别
这个指标可以在第一个指标的基础之上，根据一级类目进行分组，按照类目下的订单总量排序，取Top10，保存到表app_category_top10

针对dws层抽取脚本
1：表初始化脚本(初始化执行一次)
sh dws_mall_init_table_3.sh

2：添加分区数据脚本(每天执行一次)
sh dws_mall_add_partition_3.sh
sh dws_mall_add_partition_3.sh 20260228


针对app层抽取脚本
1：表初始化脚本(初始化执行一次)
sh app_mall_init_table_3.sh

2：添加分区数据脚本(每天执行一次)
sh app_mall_add_partition_3.sh
sh app_mall_add_partition_3.sh 20260228
```

