# <font color="FireBrick"><div style="text-align: center">实习工作中的对一些问题的思考</div></font>

## <font color="Chocolate"><div style="text-align: center">1.数据仓库的学习和了解</div></font>
## <font color="Darkorange"><div style="text-align: center">2.业务中需要的问题和思考<div></font>
## <font color="Coral"><div style="text-align: center">3.技术的应用(学习过程不详细记录了)</div></font>
### <font color="Peru"><div style="text-align: right">备注:日记形式记录 </div></font>



## <font color="HotPink">1.10,订单表事实表重构问题</font>
### 工作内容
* 现在的订单事实表不是很好用,订单事实表就是简单的将卖家单和买家单以及订单关在在一起.与其说是事实表,并没有按照星型模型来建立,更像是上一层的聚合表.

### 对订单事实表的思考
* 因为更像是聚合表,不如就完全按照聚合表的形式去做.
	1. 卖家上货生成卖家单,主要包括
		* 卖家信息:用户ID,用户名,卖家地址
		* 货物信息:商品ID,商品名,品牌名,sku,尺码,价钱,库存分类(全新,二手,预售)
		* 订单信息:卖家订单ID,卖家订单状态
		* 审核信息:保证金,审核信息,审核不通过理由
		* 时间:卖家订单添加时间/更新时间
		* 卖家单不足
			* 添加商品类型分类,虽然说这并不是订单表的一部分,但是也是常用的货物信息.可以作为冗余数据添加进来,免的sql总是join商品信息的维度表.
			* 添加库存数量(通过对在售状态的商品统计),虽然说这样统计的库存不是很准确,但是平时做需求的时候也是借鉴这个库存数量的.相比较而言,因为目前的每天销售单量还不是很大,日延迟不太会对库存深度有很大影响.以后日销售单量大的时候,就不太好了,只能缩短这个订单事实表的更新时间.一个小时跑一次.
			* 调价信息,当卖家进行调价的时候会更换一个订单号.我觉得这样做非常不合理,虽然这样做在统计整体调价次数上很方便,但是再精细到上次和下一次的调价统计就不是很好做,因为涉及到了上次调价和下次调价的订单匹配(能通过原始订单单号匹配).采用以append的方式做一个调价记录表我觉得任何需求都能很好的解决.
				* 调价记录表内容(自设计)
					* 调价单号(跟最原始单号一样,唯一值,不产生变化)
					* 调节前的价格,调节后的价格
					* 订单添加时间,订单调价时间
				* 这样订单调价时间始终是向前的,匹配前后调价的时候根据时间大小和调价前就能关联上.而且在统计整体或者某个维度的调价次数的时候也很好处理.上调/下调通过比较new_price & old_price就可以了,就不用关心订单关联了
				* 在卖家单的事实表上加个调价信息:调价次数上调次数.
	2. 买家购买生成买家单,主要包括
		* 买家信息:ID,用户名,买家地址
		* 支付信息:支付类型,支付状态,支付时间
		* 订单信息:买家订单状态
			* 这里买家订单状态跟卖家订单状态是不一样的.
				* 卖家是从商品上架到成交再到发货
				* 买家是从商品锁定到付款再到收货
		* 时间:订单创建时间/订单更新时间
		* 货物信息:跟卖家的货物信息相同
	3. 交易订单:卖家上货,买家锁定交易货物(未付款但减库存的状态)生成交易订单
		* 订单信息:订单id,订单状态,订单是否有效,成交价格(很有必要)
		* 时间:创建时间/更新时间
		* 不足
			* 关于订单整体表,可以说交易订单起到了中间轴的作用.交易订单的状态流转也是重中之重.但是现在没有收录交易订单的历史流转信息,只有现在的最终结果.在底层交易订单的事实表(只有add没有update,订单状态变一次就加一条记录)是可以查到的.但是现在需求方面来说,关于流转状态的需求和最终订单的统计的需求可以说一半一半吧.所以现在重点要加上交易订单的流转状态的记录

### 设计想法
* 本次更新算是更新订单的聚合表,不算是设计事实表

1. 设计多维模型,要从交易流程来进行设计
	* 卖家上架->买家购买->卖家发货到NICE->物流->鉴定->NICE发货买家->物流->买家收货->售后(->退货)
2. 给订单状态加标记位,这样如果后期修改(添加删除)状态,数据层的sql不用改变
	* 买家未付款标价位  ->  GMV,销售单量等一般都知道买家付钱即可
	* 卖家发货标记位
	* 买家到货标记位
	* 后边两个标记位目前没有想好,感觉范围比较大.
3. 以后自己抽象设计多维模型,注意多维模型也是有粒度的,也可以存在冗余数据.

## <font color="HotPink">1.12,维度模型的事实表</font>
* 什么叫事实表?
	* 事实表是事实事务产生的数据.
	* 什么叫事实数据呢?
		* 交易的交易订单
		* 入库的入库单据
		* 快递的快递凭证
		* 直播的场次记录
		* 上面列举的例子,我们发现都是我们行为直接产生的数据,是事实发生的.
	* 什么叫事务数据呢?
		* 这里的事务跟关系型数据库中的事务是同一个事务.主要特点是要么都发生要么都不发生
		* 交易中分为买家付款/卖家收款/买家拿货等不同步骤.这些行为的整体为一个事务,要么都成功,要么都失败.
			* 买家付款了,卖家必须要收到款,买家必须要拿到货物.
			* 其中一个环节出错,这个交易就失败了.买家没付款成功,卖家就不卖.卖家没收到钱也不可能让买家拿走货物.买家付款了必须要得到货物.
* 事实表有什么特点?
	1. 列少行多.事实数据表通常包含少量的属性列,大量的数据行。
		* 在数据表中,一个描述性数据为一个属性.而事实表中的描述信息基本都已维度表的形式存在,在事实表中则只有一个连接维度表的外键.
		 	* 例如,订单事实表关于卖家信息只有卖家ID,不能有卖家姓名等描述信息.除了标识卖家的ID信息以为,其他卖家信息(用户名/性别/年龄/手机号/地址/邮箱/...)都包含在了维度表中.
	2. 事实数据表的主要特点是包含数字数据（事实），并且这些数字信息可以汇总，以提供有关单位作为历史的数据.
	3. 事实表没有描述性数据,描述性数据以维度表存在,维度表和事实表之间通过外键关联.
		* 例如,订单事实表中的卖家ID,在事实表中为卖家信息的外键,在关联的卖家信息维度表为主键.
		* 其实这条在数据仓库中并不是那么严格.可以允许在事实表中添加少量经常使用的描述性信息.避免连接维度表产生的性能和时间的消耗.
* 怎么理解事实表和维度表的关系
	* 事实表应该本身就是一个事实,实际存在的,而维度表则是通过事务的不同角度去分析一个事务的度量表.
* 事实表是维度模型的基本表，存放有大量的业务性能度量值。事实表的一行对应一个度量值，一个度量值就是事实表的一行(可以为关联维度表的外键),事实表的所有度量值必须具有相同的粒度.

* 事实表的种类
	1. 事务事实表
		* 事务事实表记录的事务层面的事实，保存的是最原子的数据，也称“原子事实表”。事务事实表中的数据在事务事件发生后产生，数据的粒度通常是每个事务一条记录。一旦事务被提交，事实表数据被插入，数据就不再进行更改，其更新方式为增量更新。
		* 例如,订单的事务事实表,订单状态每变化一次就产生一条数据.这条数据直接添加在了订单的事务事实表中(append),不能更新事务事实表(update).虽然只有一个属性值状态变化,却要新存储一条关于这个订单数据所有数据.
		* 事务事实表记录了事实的所有历史,所有需求都能满足.但是同样冗余数据比较多.
		* 一般事务事实表是没有主键的.
		* 事务事实表的日期维度记录的是事务发生的日期，它记录的事实是事务活动的内容。用户可以通过事务事实表对事务行为进行所有特别详细的分析。

	* 通过事务事实表，还可以建立聚集事实表，为用户提供高性能的分析。
	2. 周期性事实表
		* 周期快照事实表以具有规律性的、可预见的时间间隔来记录事实，时间间隔如每天、每月、每年等等。典型的例子如销售日快照表、库存日快照表等,数据仓库中常见的按照日期分区的表一般都是具有周期性的,存储的是该时期的快照.
		* 周期快照事实表的粒度是每个时间段一条记录，通常比事务事实表的粒度要粗，是在事务事实表之上建立的聚集表。周期快照事实表的维度个数比事务事实表要少，但是记录的事实要比事务事实表多。
		* 周期快照事实表的日期维度通常是记录时间段的终止日，记录的事实是这个时间段内一些聚集事实值。事实表的数据一旦插入即不能更改，其更新方式为增量更新。
	3. 累计性事实表
		* 累积快照事实表和周期快照事实表有些相似之处，它们存储的都是事务数据的快照信息。但是它们之间也有着很大的不同，周期快照事实表记录的确定的周期的数据，而累积快照事实表记录的不确定的周期的数据。
			* 可以理解为累计性事实表记录的是当前时间的最终状态快照,但是这个状态是什么时候变化的没有统一规定.
			* 一般周期性快照事实表是insert到数据仓库的响应的周期下的.一个周期一个存储地方,不删除.而累积快照事实表则insert overwrite覆盖的,也就是把之前的订单表内容全部删除,然后收集新的数据(当前时间的最终快照)插入(数据仓库插入方式).要是在数据库中就是直接更新.
		* 累积快照事实表代表的是完全覆盖一个事务或产品的生命周期的时间跨度，它通常具有多个日期字段，用来记录整个生命周期中的关键时间点。另外，它还会有一个用于指示最后更新日期的附加日期字段。由于事实表中许多日期在首次加载时是不知道的，所以必须使用代理关键字来处理未定义的日期，而且这类事实表在数据加载完后，是可以对它进行更新的，来补充随后知道的日期信息。


## <font color="HotPink">1.15,订单表维度表的设计</font>
### 工作内容
* 之前订单事实表重建工作也接近尾声,实际上重构的订单事实表并没有按照维度模型重建,而是相当于订单相关的聚合表微重构,添加了订单流程状态变换的收录(收录到一个map<status,time>中),并没有添加订单状态的标识,因为这样设计需要更改底层结构,有点费事.
* 之前的参与过好赞的仓库搭建工作,但是表基本上是从nice上平移过来的,需要设计的部分也被大哥们设计好了,我只需要完成灌数据的工作就ok了.
* 现在我试着学学设计表,从订单维度表开始.(最初的设计,肯定漏洞百出)以后会逐步完善的.

### 维度模型
* 首先需要明白什么是维度模型,维度模型包括 1张事实表 + N张维度表.
* 事实表是维度模型中最最最重要的部分了.
* 本次要做的是订单维度表的设计,选什么当事实表呢?正如事实表这个名字,订单维度表,当然事实是订单.所以应该以订单为事实表来构建维度模型.

## <font color="HotPink">3.15,商品ctr的排序问题</font>
### 工作内容
* 制作一张关于全新商品的ctr报表
* 需要内容:商品ID,商品名,sku,展示次数,展示人数,点击次数,点击人数,ctr-pv,ctr-uv
* 按照ctr转化率排序给出TOP200
* pass掉展示次数小于1000的
* 按照天为单位

### 内容解读
* ctr就是展示到点击的转化率,ctr = 点击数/展示数
* ctr能够体现出一款商品的吸引力.
* 报表制作过程没有什么问题,有相关数据统计,直接统计数据请求次数就ok了.
	* 之前ctr采用的请求接口数量,但是这样存在很大的误差.有时候甚至点击量比展示量还要多
	* 现在给展示以及点击都加了数据埋点,而且埋点的规则很严格,基本上误差可以限制在很小

### 关于ctr排序问题的思考
* 开始接到这个需求的时候,是按照ctr转化率来排序的,这很正常.
* 后来修改为按照展示人数排序.
* 按照ctr/展示人数排序的区别
	* 哪一款商品有热度有吸引力(ctr转化率高)
	* 但是容易受到商品其他因素的影响,例如商品封面的吸引力,商品价格等等
* 后来经过商量改为按照展示数uv来进行排序.为什么?
	* 以实习生出数据角度,无非是看个当前热度比较高的商品罢了.但是既然要出这个报表就说明这排名是由可挖掘的东西存在的,仔细想一想有什么东西可以从ctr排名中得出呢?
	* 显然我们只能看到热度商品的热度而已,影响因素一个也看不到.
	* 如果以展示uv来排序的话,某一款商品存在问题就一目了然.因为展示人数多,被很多人看到,但是ctr转化率很低一下子就说明出商品本身存在问题.然后我们锁了商品,去找原因.比如没有细节图或者细节图更少;售价高让人望而却步;封面丑没有点进去的欲望.
	* 可能你也会想到按照ctr排序,排序低的不也能看到吗?其实不然,ctr排序低的商品不一定存在问题,因为可能存在流量问题,也就是说没有获得更多的曝光.得到曝光缺不吸引人的商品是不是问题更大呢?
	
* 那么,按照ctr排序就不好吗?
	* 不是的,要根据实际业务需求来确定.
	* 当你们要搞活动大促销的时候,希望知道哪些商品热度高来参加活动.或者我们想给一些虽然ctr低,但是有实质的商品更多的曝光的时候,按照ctr排序更好
	* 如果要挖掘当前曝光产品问题就需要按照曝光人数排序.

## <font color="Coral">3.19,按直播场次为单位完美转化成天为单位统计数据</font>
### 工作内容
* 日常需求,统计主播数据(直播天数,直播时长,累计直播收益)
* 数据基础有个dw层的微聚合表,以直播场次为单位记录该场次相关的数据(主播信息:id/name/ip/位置,直播状态,设备信息:app版本/os/os版本/设备类型/,开始时间,结束时间,直播时长,累计观众数,最大在线观看数,观看时长,点赞次数,分享次数:站内/站外,评论数,评论人数,新增粉丝数,收益数,送礼人数).
* 数据需求:2月份整月收益流水达5000之上的主播相关信息.

### 内容解读
* 可以从dw层的直播相关表中就能直接获取大量的数据.
* 根据直播时间来筛选出在2月份的开播记录,按照主播分组,sum(直播时长),sum(收益)
* 注意筛选的时候,那种跨天级的直播要如何处理?最开始做需求的时候,采用的是start_time为标准,该场直播就划分在start_date中,不考虑跨天级别.数据也好处理.但是这种方式在记录主播开播天数存在的问题比较大.以及以往直播相关的需求也基本上都是按照天数为单位来统计的,如何才能减小误差呢?

### 关于跨天级的直播处理

1. 从底层ods表再以天为单位抽象出一个天为单位的表来,忽略直播场次.
	* 这种方法在计算收益以/观众数uv/分享次数uv等等还是可以的.以这些记录的add_time来划分每一天,然后根据主播来分组,直接忽略掉直播ID这个中间单位.但是在计算观看时长,以及本次这种按月统计天数就显得力不从心,因为底层数据未免有些太大了.
	* 而且统计的误差场次也就是那种跨天的直播场次,没跨天的直播没有任何影响,未免有些小题大做.
	* 如果需求是就要知道当天看过该主播直播的uv,或者分享的uv,那没有办法,就需要从底层ods层取细粒度数据统计.如果是上述的需求按月份统计直播天数就不太好.

2. 在场的基础表上抽象出天的直播
	* 目标是跨天的直播(造成误差的也是跨天的直播),不意味改整体表结构,需要处理的是开始时间和结束时间. 
	* 需要将跨天的直播拆成两个直播.

```sql
	select lid,start_date as live_date --正常不跨天的直播,start_date = end_date
	from live_table 
	where start_date = end_date
	
	union all
	
	select lid,start_date as live_date --跨天的直播,取开始天部分的直播
	from live_table
	where start_date != end_date
	
	union all
	
	select lid,end_date as live_date --不要忘了还要去结束天部分的直播
	from live_table
	where start_date != end_date
	 
```

* 这样虽然dw层是场次为单位的记录方式,但是也被我们拆成天数来看待,然后我们用live-date来做数据统计就ok啦.

### 问题解决
* 知道如何将按场次统计转化为按天统计,那么我们上面遇到的需求该怎么做呢?
1. 要筛选出2月份的直播,考虑跨天问题
	* 起始时间不在2月,结束时间在2月.
	* 起始时间在2月,结束时间超出2月.
	* 其实,也就是只要开始时间或者结束时间有一个是在2月份的直播.

```sql

	select lid,... --筛选出的基础表
	from live_table
	where (start_date between 20190201 and 20190228)
	  or  (end_date between 20190201 and 20190228)
```

2. 以上述基础表来划分天来统计
	* 不跨天的,正常统计开始时间即可.
	* 跨天的需要按照天拆分,拆分方式如上

3. 按照主播为单位分组,count(distinct live_date).

### 额外思考
* 因为跨直播天数最多也就横跨2天,所以只需要划分成两段就好了.
* 但是如果存在那么一种情况,横跨好多天,天数都不确定,那你怎么才能正确划分天呢?
	* 这个只能写UDTF函数了,自动生成数据.给个func(start-date,end-date)生成期间的每一天.
	* UDTF函数需要配合lateral view来使用才能关联上原表的数据.

## <font color="Coral">3.20,二手商品ctr统计</font>
### 工作内容
* 统计二手商品的ctr
* 有部分二手分类页面的ctr打点数据,而且从打点信息统计的展示和点击数据是准确的.
* 但是在活动界面的二手商品没有打点,所以不会产生任何打点数据.只能采用接口来统计.

### 困难点
* 通过接口数据统计展示信息是不太准确的.
	* 因为你每次在外部界面滑动的时候是存在预加载机制的,一次就会产生10条加载的商品,而你真正看到的可能就最头部的两张.
	* 通过打点改善后,在手机界面展示超过80%的图片进行打点,这样就不会收到加载机制的影响.
* 活动页面没有打点信息怎么办?用接口信息凑活一下?没有意义,因为数据的产生来源不一样,根本没有可比意义.

### 思考
* 做数据一定要保证数据来源一致.
* 在数据仓库中,有不同来源的数据汇聚到一起.同一个数据需求可能存在各种各样的算法,算法多不要紧,只要算法正确,同一个数据源的结果肯定是一样的.如果采用不用的数据源,计算结果不一样,那么你们数据部门给出的数据就没有可信度.你想,同一样的部门左右工位的员工同一个需求给出的数据是两样的数据,算法可能都一样,不存在谁对谁错,但是谁会信你们.
* 所以做数据最重要的要保证数据源头的一致性,以及数据口径的一致性.


## <font color="Coral">3.21,关于商品上架流程打点的感想</font>
### 工作内容
* 商品上架流程漏斗的需求
* 查看实际流程转化率的话以pv为准,查看实际流失多少人的话uv为准

### 内容解读
* 因为每个步骤每个按钮都有详细的打点信息
	* 点击信息:点击的按钮,点击包含的信息(这里是商品相关的信息)
	* 界面信息:当前页面,上一个界面(from一定要有)
* from很重要这样才能走通流程
* 打点产生数据要严格,一定要确认点击了后再产生打点数据,避免误差.
* 每条流程一个漏斗,有很多分支.

### 思考
* 之前也做过类似的商品流程转化率的需求,但是没有做打点.是用接口信息做的.
* 接口数据并没有上下操作的关联信息,以及并没有做严格的唯一性.
* 所以在走流程的时候需要找关联,之前找的关联是通过接口的商品进行关联的.这样有个问题你没法确认是有限关联,也就是时间上怎么认证有效呢?老大说15分钟内算有效.这样存在的问题是重复关联,实际只有一条转化了,但是因为时间范围大,把上一条也关联上了.这样产生的数据误差很大很大.
* 现在有打点数据,每条都有from信息,直接统计总的,并不用考虑关联节省了很多事情.以及减少了多个误差.


## <font color="Coral">3.22,关于hive传参问题</font>
### 传参的几种方式
1. python脚本 

```sql
	hql = """
		select...where col = %d 
	""" % (parameter)
```

2. hiveconf
	* 在hql脚本开头设置参数 ``` set parameter='val1' ```
	* 在执行脚本时设置参数 ``` hive -hiveconf parameter='val1' -f test.hql ```
	* 在脚本中使用参数 ``` ${hiveconf:parameter} ```
	* 示例

```sql

-- 查询标签名
set tag_name='#YEEZY冲不冲#';
-- 开始日期
set start_date='20190329';
-- 结束日期
set end_date='20190330';

select t.partition_date,
       sum(detail_pv),
       round(sum(like_pv)/sum(detail_pv), 2),
       round(sum(comment_pv)/sum(like_pv), 2)
from(
    select partition_date,
           request_body_params['sid'] sid, 
           sum(if(request_path = '/show/getdetail', 1, 0)) as detail_pv,
           sum(if(request_path = '/show/like', 1, 0)) as like_pv,
           sum(if(request_path = '/show/comment', 1, 0)) as comment_pv
    from dw.dw_ml_access
    where partition_date between ${hiveconf:start_date} and ${hiveconf:end_date}
      and request_path in('/show/getdetail', '/show/like', '/show/comment')
    group by 1,2
) t
join(
    select distinct id as sid
    from kkgoo.user_show_sid
    where from_unixtime(add_time,'yyyyMMdd') between ${hiveconf:start_date} and ${hiveconf:end_date}
    and content rlike ${hiveconf:tag_name}
) s on t.sid = s.sid
group by 1

```


## <font color="Coral">3.23,关于UGC的思考</font>
### 工作内容
* 最近一直在出UGC相关的数据报表,数据主要是UGC的ctr.

### 内容解读
* UGC是什么?PGC是什么?
	* UGC是指用户的原创内容,例如:小红书上的各种推荐帖子,微博上各种教程,B站的创作视频,抖音的短视频.
	* PGC是指专业人士生产内容.例如:廖雪峰的python学习网站.
* UGC有什么用?
	* UGC一般都是用户自己产生的内容,一句诗歌,一张图片,一篇教程,一个短视频等等.现在流行的社交软件基本上都是以UGC为主,PGC/运营为引导的.你看大家所熟知的小红书,女生们一想买什么东西都会去上边找推荐,各种事情都有人写教程.之前看到过一个女生吐槽说小红书身上拉屎都有人教你.
	* 优质的UGC内容可以带来很多的流量,同时也会带来价值.流量的印证无非就是展示量和点击量,就是所谓的ctr.侧面一点的就是产生该UGC的用户的粉丝量.价值体现了所带来的流量多少,带来的流量越多,打条广告,做个宣传所能产生的价值越多.站在总体角度上考虑就是,你社交软件UGC约优质约符合大众,吸引的用户就多,所带来的商业价值也就越高.所以要想搞好一个社交软件,优化用户UGC产出以及内容这项工作必不可少.之前和一个搞市场的人聊过天,他说过在抖音这种日活量上亿的平台上,找一个100w粉丝的用户做个短视频产品宣传一般要花10w块.B站这种视频中插播一条广告也要7.8万.
	* UGC多,用户也多就一定好吗?事情都是有两面性的.有的时候UGC多也并不一定是好事,首先我们要明确的是优质UGC.微博这个例子就很典型,现在你去看看微博可能10个号中就有4个是营销号,3个是吃瓜不发东西的用户,2个是喷子,1个只发发生活感触.而这些营销号他们目的是粉丝量,然后打广告.他们能产生好的UGC吗,不一定.我想大多数都是靠转发什么吸引别人眼球的新闻来吸引粉丝,有的甚至拿过来自编自演,恶意诽谤来博得眼球.但是也不是说没有正经靠创作优质内容来的大V,那样始终是在少数.
	* 所以作为社交软件的运营,一定要注意UGC这方面的运营.
		1. 优化UGC内容
		2. 刺激更多的人创作UGC
		3. 选取更好话题来引导用户创作(PGC/运营引导)

### 思考
* 我记得之前做过数据需求,关于nice用户画像,有2/3仅仅是社交用户,1/3才是电商用户.既存在购买行为又有UGC产出的用户占电商用户的4/5.当时我就很奇怪,因为我周围人知道nice都是因为他搞潮鞋,但是电商用户竟然这么少,一度以为数据错了.但是事实就是这样.
* 现在想想,明白了原因.毕竟nice之前就是做社交的(类似ins),有用户基础.但是最近时间发展的不好,所以算是一款过时的社交软件,所以才选在转型做电商.
* 回到数据上来,我们看到虽然电商用户占少数,但是电商中的大部分人都是有UGC产出的.这点还是很不错的.现在公司主要目标是要发展电商了,那么用户UGC需要放弃吗?说实话,之前重心一直一直在电商上,运营想方设法的做电商活动,也很少有人管用户UGC的相关事情了.以至于之前我没有收到任何的数据需求.这样做就等于放弃了2/3的用户,这未免有点过.
* 当然我的业务水平在运营眼里就是小渣渣,我能想到的他们也一定能想到.那么就是如果把UGC往电商上靠拢.用UGC推动销售.可以说nice上还有的2/3的社交用户,他们都是资深潮人了吧(因为nice在2014-2016年很火),老用户/资深潮人产出的UGC自然有质量保证.有基础,有方向该怎么做呢?
	* 球鞋开箱/测评(短视频形式),这个我觉得应该重点去发展.
	* 穿搭(之前社交用户比较感兴趣的方面),指定好主题就ok
	* 新品上身
	* 行情分析
* 数据方面该做的支持有哪些?
	* 内容ctr数据,主要体现了UGC的热度,筛选出大家比较感兴趣的话题
	* 穿搭和新品上身,都是以照片形式存在的.之前也有给照片打商品标签,通过标签就能点到商品的购买界面.这个是有基础的.但是没有数据支持.目前也没有响应的接口数据和打点数据.如果有这些数据的记录,我们很清楚的知道什么类型的UGC能产生更多次的商品曝光,更方便指定活动.尤其是开箱,这个很重要.有的人看了开箱视频就说明他有倾向去购买,但是有些犹豫(可能是不了解具体样子,不了解具体材质),通过开箱视频就能获取到他所疑惑的信息,再加上标签,很容易就促进商品的曝光的.
	* 其他也同理,抽象一点就是,我们需要做的是加上曝光的来源UGC数据.然后做出报表来支持运营进行活动内容优化.
		* 报表维度:商品,时间
		* 报表度量:来源UGC(可分小部分),展示PV/UV,点击PV/UV,交易成功数


## <font color="Coral">3.28,数据集市和数据仓库</font>
### 数据集市理解
* 之前看了<数据仓库>那本书,看完理解的数据集市是多维模型.当然每个多维模型面向一个主题.是dw层上更近的一层dm层.
* 今天看到去哪er一个数据仓库大佬写的博客说,数据集市无非就是面向主题的宽表.也就是说我目前在用的dw层.跟我的理解有出入,我认为dw层是宽表,但并不是数据集市.
	* 宽表:就是上面订单宽表.订单表中不单包含订单表中的内容,还聚合了一些经常用到的信息:例如,卖家单的信息,买家单的信息,货物信息等.算是一种聚合表.

### 数据集市概念
* 数据集市(Data Mart) ，也叫数据市场，数据集市就是满足特定的部门或者用户的需求，按照多维的方式进行存储，包括定义维度、需要计算的指标、维度的层次等，生成面向决策分析需求的数据立方体.   ---来自百度百科.
	* 面向主题业务
	* 多维方式存储
	* 有维度/粒度
	* 面向决策分析
* 数据集市中数据的结构通常被描述为星型结构或雪花结构。一个星型结构包含两个基本部分——一个事实表和各种支持维表。这里说的星型结构/雪花结构就是多维模型.
* 数据集市就是企业级数据仓库的一个子集，他主要面向部门级业务，并且只面向某个特定的主题。为了解决灵活性与性能之间的矛盾，数据集市就是数据仓库体系结构中增加的一种小型的部门或工作组级别的数据仓库.
	* 数据集市相当于面向单一业务主题的小型数据仓库

### 数据仓库特性
#### 数据仓库四大特点
* 数据仓库是决策支持系统（dss）和联机分析应用数据源的结构化数据环境。
	* 数据仓库不是面向财务的,而是面向DSS的.
	* 联机分析主要分为联机事务处理(OLTP)和联机分析处理(OLAP).
		* OLTP是面向事务型的数据库的
		* OLAP则是面向数据仓库的.
* 数据仓库研究和解决从数据库中获取信息的问题。
	* 数据仓库可以包含多个数据源头的数据,也就是不同的数据库.
	* 数据仓库虽然包含n多数据源,但是数据输出口径要一致.
* 数据仓库的特征在于面向主题、集成性、稳定性和时变性。
	* 面向主题:数据仓库中的数据是按照一定的主题域进行组织。主题是一个抽象的概念，是指用户使用数据仓库进行决策时所关心的重点方面，一个主题通常与多个操作型信息系统相关。而操作型数据库的数据组织面向事务处理任务，各个业务系统之间各自分离。面向事务的也就是过在同一个事务操作下,涉及到的数据可能包含好多主题的业务,在这样的数据进入数据仓库过程中需要根据主题进行拆分.
	* 集成性:数据仓库中的数据是在对原有分散的数据库数据抽取、清理的基础上经过系统加工、汇总和整理得到的，必须消除源数据中的不一致性，以保证数据仓库内的信息是关于整个企业的一致的全局信息。面向事务处理的操作型数据库通常与某些特定的应用相关，数据库之间相互独立，并且往往是异构的。
	* 稳定性:数据仓库的数据主要供企业决策分析之用，所涉及的数据操作主要是数据查询，一旦某个数据进入数据仓库以后，一般情况下将被长期保留，也就是数据仓库中一般有大量的查询操作，但修改和删除操作很少，通常只需要定期的加载、刷新。操作型数据库中的数据通常实时更新，数据根据需要及时发生变化。
	* 反应历史变化:数据仓库中的数据通常包含历史信息，系统记录了企业从过去某一时点(如开始应用数据仓库的时点)到目前的各个阶段的信息，通过这些信息，可以对企业的发展历程和未来趋势做出定量分析和预测。而操作型数据库主要关心当前某一个时间段内的数据。

## <font color="Coral">3.31,数据仓库架构</font>
### 数据仓库架构

![数据仓库架构](https://github.com/zzhangyuhang/thinking-in-my-work/blob/master/数据仓库架构.jpg)

* 上面是目前我了解的我们公司的数据仓库的基本架构,可能我列的不太全.但是基本框架都列出来了.
* ODS层是细节数据层,是从数据源直接过来的数据(日志).基本上不做任何处理(只去重去脏),是最细粒度的数据.
* 其实浏览了网上的一些分层概念跟我们目前使用的架构有些出入.网上一部分帖子说dw层为准备数据层,也就说是为维度模型和宽表准备的细节数据层,也叫准备数据层.也就是从mysql剔除无用的数据和对日志数据进行处理的数据.我们这里就直接归入了ods层.dw层是数据聚合层,根据不同的主题划分成为不同的宽表.
* dm层也就是数据集市层,也就是多维模型.也是根据不同主题确定的
* 其实我们这里的多维模型建设的不太好.因为一般的需求都从dw层的聚合表就基本可以满足了.

### 对架构的理解
* 其实我对目前公司的架构有些疑惑的.主要疑惑是在于dw和dm层的关系.
* 经过我在网上了解的分层和对公司的分层坐了一个自我理解.
	* 公司dw的面向主题的聚合宽表,不应该放在dw层中.应该是属于dm层.
	* ods层的数据没有变化,就是直接收录的日志数据和从mysql导入的数据.(可能经过ETL)
	* dw层究竟应该放什么样的数据呢?
		* 我认为dw层全名叫数据仓库层(data warehouse),应该放入的是从ods层过来的经过划分主题的细粒度的多维模型数据.
		* 每个主题一个事实表n张维度表.
		* 我们需要注意的是 多维模型也是有粒度之分的,但是事实表和维度表之间的粒度应该统一.
	* dw放了维度模型,dm层应该放什么呢?
		* 从dw层的细节多维模型抽取聚合成为粗粒度的面向用户的多维模型.也就是说对dw的多维模型根据业务需求进行一定的聚合完成的聚合多维模型.
		* 以及聚合的宽表. 


![整体架构](https://github.com/zzhangyuhang/thinking-in-my-work/blob/master/整体架构.png)

![数据仓库架构](https://github.com/zzhangyuhang/thinking-in-my-work/blob/master/数据仓库架构的理解.jpg)

## <font color="Coral">4.1,如何做好数据仓库</font>
### 思考源头
* 如何做好数据仓库?这是一个比较空泛的问题?怎么是好?
* 其实可以理解为如何更好的支持业务.从开始实习到现在,每天都在处理需求.天天做需求,有的时候真的挺烦的.为什么需求一直做不完呢?最近总有这种困扰.

### 自我理解
* 数据仓库怎么才叫好呢?
	* 数据仓库需要能够支持各种各样的数据需求.
	* 数据仓库数据分类要明确.
	* 数据仓库能快速支持数据需求.
* 首先数能够支持各种数据需求,我们现在的数据仓库并不能很好的做好这一点.主要原因是自我定位的问题.我刚来的时候是我总以自己是实习生的身份来做任何事情,以为分配我的工作就是辅助全职来处理平时的数据需求工作.其实不然,我的工作应该是从处理需求过程中来推动数据仓库建设的.其实在每天的渊源不断的数据需求中,大部分的需求都是相同的.我们需要把重复的/每天都要看的,做成报表或者以数据可视化的形式展现出去,而不是等着业务方来索要数据.做数据需要从被动转化为主动.从平时需求来抽象成日常数据完成角色转换.以及转变思想到他们想要什么?有的运营人员并不是很专业,他们想要查看的数据也可能表达不清楚.在做数据需求前需要明白他究竟想要看什么?而不是完全照做.
* 数据仓库分类和快速支持数据需求是分不开的.分类明确完好就能快速支持数据需求.仓库分层明确很重要,我们目前的分层策略还可以,但是有待完善.有待完善的是需要按照主题来分配需求的去向.我们现在的数据只是分层了并没有特别明确的面向主题.相关的维度模型还没有构建好.这块确实是一个需要后续完善的地方.

### 认识
* 数据仓库是为了支持决策制定的.
	* 不是面向财务,允许有误差,要能解释清楚.
	* 输出口径要严格一致.
* 数据仓库管理者的责任
	* 理解业务用户的责任/目标/目的/任务
	* 业务的高质量的理解/分析
		* 一般理解/分析业务要从流程入手,从不同流程/不同角度来理解/分析业务
		* 例如购买流程:用户访问首页 -> 用户搜索 -> 用户到list页面 -> 用户到detail页面 -> 下单 -> 成单 -> 利润.
			* 管理层看用户行为数据，那么一定是看关键路径的指标，一般不会看每个路径的转化，因为他的kpi是提升整体的收入或者整体的新用户数
			* 负责用户行为的产品看数据，那么一定是看每个路径的转化指标，因为他的kpi就是提升整体的转化，细化之后就是每个关键节点的转化率
	* 完善数据仓库的功能,更好解决数据需求
* 在做数据仓库一定要从业务流程出发/面向不同的主题/面向不同决策人员来理解/分析数据,建立数据仓库的dw/dm层.无需按照所有维度来给出数据(随便一个数据需求就能碰撞出n多维度),这样费力还不能满足别人需求(决策人员也就想看其中一个维度数据).


## <font color="Coral">4.3,如何计算在线时长</font>

### 工作内容
* 计算某个用户某台设备的在线时长

### 内容解读
* 数据中并没有统计累计在线时长的,所以需要自己计算.
* 计算在线时长需要定义怎么才算在线?
	* 如果在线肯定会有接口请求数据
	* 在线的隐式含义是连续的接口请求,如何才算连续?
		* 定义上下两次接口之间的间隔时间大小才认定是否有效连续.
			* 间隔时间长,认为断开重连,不是本次在线间断->结束
			* 间隔时间短,可认为在线,可认为是连续请求.我们这定义为300秒,也就是5分钟.
	* 计算本次在线最开始接口请求的时间和最后接口请求的时间之差就是在线时长.
* 缺点
	* 有的中途切换并不能认证为同一次在线,导致中间间隔时间多加.
	* 有的页面停留时间过长,导致认为两次在线,导致中间间隔时间没加.
	* 总之跟定义连续接口访问的时间限制大小有很大关系.
		* 因为我们现在app的ugc内容以图片为主,所以停留时间不长,5分钟已经扩的很开.
		* 需要根据业务来指定

### 思路
1. hive本身没有类似UDF可以使用,所以需要自己定义函数来完成计算.
2. 定义什么函数?计算在线时长是聚合操作,以用户分组.
	* UDAF -- java
	* transform -- python
3. 根据同一用户,同一台设备分组.
4. 计算在定义的有效时间内,前后次请求的差值.可以认为这个差值就是本次点击的在线时长
5. 遇到超出有效时间的连续请求可认定为断开连接,本次在线结束.
6. 结束时,汇总所有接口之间差值就是在线时长.

### 代码
* 因为我现在公司的技术栈没有java,则采用transform的方式来计算.
* 下面是transform的具体用法sql部分
* transform语法,hql部分

```sql
	add file transform1.py;
	add file transform2.py;
	-- 插入新表
	from(
	 	select ... from table where ...
	) tmp
	insert overwrite table1 partition(dt = '')
	select transform(tmp.*) -- 使用transform,tmp.*是传入的数据
	using 'python transform1.py' -- 执行transform的文件
	as( -- transform输出结果
		col1 int,
		col2 string,
		...
	)
	row format -- 定义transform输出格式
	delimited fields terminated by '\001'
	collection items terminated by '\002'
	map keys terminated by '\003'
	
	insert overwrite table1 partition(dt = '')
	select transform(tmp.*) -- 使用transform,tmp.*是传入的数据
	using 'python transform2.py' -- 执行transform的文件
	...


	-- 数据转换	
	select transform(t.*) -- 使用transform,tmp.*是传入的数据
	using 'python transform.py' -- 执行transform的文件
	as( -- transform输出结果
		col1 int,
		col2 string,
		...
	)
	from(
		select ... from table where ...
	) t
		
```

* user_time.hql

``` sql

-- user_time.hql
ADD FILE transform.py; --添加transform脚本

FROM
    (
        SELECT
            device_id,
            uid,
            path,
            request_timestamp
        FROM ods.api_access_log acl
        WHERE partition_date={{DATE}} --{{DATE}}是参数
        DISTRIBUTE BY device_id --用设备ID分桶
        SORT BY device_id, request_timestamp --排序
) tmp --基表,用于收集要处理的数据
INSERT OVERWRITE TABLE dw.dw_user_use_time --要插入的表partition(partition_date={{DATE}}) --要插入表的分区
SELECT TRANSFORM(tmp.*) -- transform语法,传入tmp的所有数据
USING 'python transform.py' -- transform.py 和 hql 在一个文件夹下
AS( -- transform输出
    device_id string,
    user_id bigint,
    use_time int
)
ROW FORMAT -- transform输出格式
DELIMITED FIELDS TERMINATED BY '\001'
COLLECTION ITEMS TERMINATED BY '\002'
MAP KEYS TERMINATED BY '\003'

```
  
* transform.py内容

``` python

import os
import sys

	for line in sys.stdin: 
		# sys.stdin 是select transform(tmp.*)传输进来的
		
		line = line.rstrip('\n') # 去除行末尾的换行符
		col1,col2,col3 = line.split()
		
	... # 处理过程
	
	# transform输出  必须要用print输出
	print '\t'.join(map(str,[col1, col2, col3]))


```

* 问题解决的transform.py

```python 

#!/usr/bin/python
#encoding:utf-8
# transform.py

import sys
import time

define USER_LEAVE_INTERVAL = 300 # 定义有效时间

def process(data):
    total_cost_time = 0 # 总的使用时间
    prev_request_time = None # 上一个请求时间
    user_id = None # 记录当前处理的用户ID
    device_id = None # 记录当前处理的设备ID
    
    for parts in data:
        try:
            device_id, uid, path, request_time = parts
            request_time = long(request_time)
            
            if prev_request_time is None: # 初始化
                prev_request_time = request_time 
                user_id = uid 
                device_id = device_id
                continue
           
            cost = request_time - prev_request_time # 计算前后两个请求时间差值
            prev_request_time = request_time # 更新上一个请求时间为当前时间
            if cost > 0 and cost < USER_LEAVE_INTERVAL: # 差值时间有效
                total_cost_time += cost # 加入总的时间
        except:
            continue
    if total_cost_time > 0: # 输出
        print "\001".join(map(str, [device_id, uid, total_cost_time]))
        
def main():

    old_device_id = None # 用于标识上一条数据的设备ID
    buffers = [] # 用户保存要传入处理过程的列表
    for line in sys.stdin: # 获取transform的输出
        line = line.rstrip('\n') 
        if not line: # 空行略过
            continue
        parts = line.split() # 切分数据
        if len(parts) != 4: # 如果传入的数据为脏数据,略过
            continue
        device_id = parts[0] # 获取设备ID

        if device_id == "": # 设备ID为空,略过
            continue

        if device_id != old_device_id: # 如果当前设备ID 不等于 上一条数据的设备ID
            if old_device_id: # 且上一条数据设备ID不为None,说明上个设备数据取完(tmp.*传入的数据是按照设备ID排序的)
                process(buffers) # 对这个设备ID的数据进行处理
            old_device_id = device_id # 更新设备ID为当前设备ID
            buffers = [] # 置空

        buffers.append(parts) # 设备ID相同时直接加入buffer
    else:
        if buffers:
            process(buffers)

if __name__ == "__main__":
    main()

```

## <font color="Coral">4.9,ODS和数据仓库</font>
### 问题
* 今天在浏览一个数据仓库大佬的博客的时候他阐述他们公司现在存在一个问题是关于订单的历史数据的问题.他们采用的是直接导订单表的方式到数据仓库中,中途没有做任何处理.这样导致产出报表的时候以及临时需求的时候不能满足对历史状态变更的需求.
* 上面我有提到数据仓库的四个特性
	1. 面向主题
	2. 集成
	3. 反应历史变化
	4. 相对稳定
* 我们可以看到大佬公司目前的数据仓库的状态并不满足第三点:反应历史变化.

### 解决
* 这里需要引入ODS.ODS是操作型的数据集合,数据粒度很细,可以理解为原始数据集.
* ODS数据的特点
	1. 即时性
	2. 操作型
	3. 集成了全体信息(该次操作的所有信息)
* ODS中的数据是操作一条,产生一条数据,保存一条数据.ODS中保存了所有的历史数据.
* 引入ODS后,就可以用ODS来满足历史性数据需求.
* 通过ODS来完成对历史操作的记录,产出对历史数据的数据需求.
* 注意:一般不采用接口性/点击性数据来完成对订单历史数据的需求,因为这两种即时性数据产生的误差相对较大.对于订单这种状态流转最好在后台中记录一下.然后导入数据仓库中.

### 架构

* 并行架构

![并行架构](https://github.com/zzhangyuhang/thinking-in-my-work/blob/master/ODS和DW并行.jpg)

* 这个架构可以理解为点击日志一些操作型数据导入ODS,从后端关系型数据库产生的数据直接导入DW.真的很垃圾.对于并行架构来说就舍弃了分层的概念.数据从数据源头直接进入ODS/DW.历史数据需求采用ODS的数据来完成,最终累计型数据需求采用DW的数据来完成.虽然可以完成任何数据需求,但是这样就面临了一种新的问题就是数据源不一样产生的数据结果不同的问题.例如,订单相关的问题.如果采用ODS来统计历史时刻的GMV和DW来统计历史时刻的GMV肯定是不一样的.这个问题可以统一数据源来完成.如果需要用历史数据和累计数据进行对接呢?能对上吗?数据仓库数据口径不统一,可信度差,这个问题相对于无法做历史数据需求这个问题可严重的多.而且放弃了分层概念导致数据之间的耦合度很高.我觉得这个架构真的不好.

* 分层架构

![分层架构](https://github.com/zzhangyuhang/thinking-in-my-work/blob/master/ODS和DW分层.jpg)

* 这个是分层架构,我们可以看到所有数据源头都汇总到ODS层.然后统一由ODS层向上提取出数据仓库DW层的数据,解决了数据源头不统一的问题.分层方便数据的管理.虽然这种分层架构所有的数据都要进入ODS,导致ODS的灵活性下降.但是试想一下有多少需求需要从ODS中产出呢?10个里面有1个就不错了.而且逐层产出的数据仓库层/数据集市层,里面的多维模型下的事实表也分很多种.事务型事实表完全能够满足对历史数据的需求,又何必再到ODS来产出呢?

### 思考
* 平心而论,对于对于历史订单状态流转问题的解决办法挺多的.实际解决并不是添加ODS层来完成.只不过大佬的公司可能并没有做数据仓库分层.导致所有数据直接导入数据仓库.
0. 前提,后台记录状态流转.不要从点击流/接口流数据中取得.(累计状态不是从这两个流中取得导致数据源头不一致)
1. 在订单宽表中收录状态流转的信息.状态流转以map<status,time>形式存在.
2. 在多维模型中,采用订单流转来做订单的事务型事实表.然后向上抽取出累计型的订单事实表.