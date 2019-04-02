# <center>地铁乘客流量预测</center>

[TOC]

>  
>
>  <font size=3>**竞赛题目**</font>

&emsp;通过分析地铁站的历史刷卡数据，预测站点未来的客流量变化。开放了20190101至20190125共25天的刷卡记录，共涉及3条线路81个地铁站约7000万条数据作为训练数据`Metro_train.zip`，供选手搭建地铁站点乘客流量预测模型。同时大赛提供了路网地图，即各个地铁站之间的连接关系表，存储在文件`Metro_roadMap.csv`文件中。

&emsp;测试阶段，提供某天所有线路所有站点的刷卡记录数据，预测未来一天00时至24时以10分钟为单位的各时段各站点的进站和出站人次。

&emsp;测试集A集上，提供2019年1月28日的刷卡数据`testA_record_2019-01-28.csv`，选手需对2019年1月29日全天各地铁站以10分钟为单位的人流量进行预测。

&emsp;评估指标采用**平均绝对误差`Mean Absolute Error, MAE`**，分别对入站人数和出站人数预测结果进行评估，然后在对两者取平均，得到最终评分。

&emsp;关于数据的具体描述以及说明[详见天池官网](<https://tianchi.aliyun.com/competition/entrance/231708/information>)。



# <font size=4>1. 赛题分析和前期思路</font>

&emsp;比赛提供了1号到25号共25天的刷卡记录数据，所以第一步就是对每一天的文件进行处理。原始数据集中包含了`time`, `lineID`, `stationID`, `deviceID`, `status`, `userID`, `payType`这几个列，根据题目要求要预测进站和出站的人流量，所以要先统计出每一天的进站和出站流量。

## <font size=3>1.1 数据清洗</font>

>  **提取基础信息**

&emsp;首先对时间信息进行处理，提取出日、周、时、分、秒的信息，由于是按照10分钟为间隔统计，所以在提取分钟信息的时候只需要取整十。接着总计80个站点(除去缺失数据的54站)，每个站点从0点到24点，以10分钟为一次单位，总计144段时间间隔。根据站点、日、时、分进行分组统计，得到每个时段的进站人数和出站人数。

&emsp;经过第一轮处理之后，得到了每一天的文件包含的`columns`有：`stationID`,  `weekday`, `is_holiday`, `day`,  `hour`,  `minute`,  `time_cut`,  `inNums`以及`outNums`这几列。

&emsp;增加了一些和刷卡设备相关的特征，包括`nuni_deveiceID_of_stationID`, `nuni_deviceID_of_stationID_hour`, `nuni_deviceID_of_stationID_hour_minute`。



## <font size=3>1.2 特征工程</font>

> **增加同一站点相邻时间段的进出站流量信息**

&emsp;考虑到当前时刻的流量信息与前后时刻的流量信息存在一定的关系，所以将当前时间段的前两个时段以及后两个时段流量信息作为特征。

&emsp;增加的特征包括`inNums_before1`, `inNums_before2`, `inNums_after1`, `inNums_after2`, `outNums_before1`, `outNums_before2`, `outNums_after1`, `outNums_after2`。



> **增加乘车高峰时段相关的特征**

&emsp;根据杭州地铁的运营时段信息，将高峰时段分为四类，0表示非运营是简单，1表示非高峰是简单，2表示高峰是简单，3表示特殊高峰是简单。由于周末和非周末的高峰时间存在一定的差异，所以需要分别计算。

&emsp;增加了特征`peak_type`。



> **增加同周次的进出站流量信息**

&emsp;均值、最大值以及最小值在一定程度上反映了数据的分布信息，所以增加同周次进站流量和出站流量的均值、最大值以及最小值作为特征。

&emsp;增加的特征包括`inNums_whm_max`, `inNums_whm_min`, `inNums_whm_mean`, `outNums_whm_max`, `outNums_whm_min`, `outNums_whm_mean`, `inNums_wh_max`, `inNums_wh_min`, `inNums_wh_mean`, `outNums_wh_mean`。



> **增加线路信息**

&emsp;根据某一天的刷卡记录表统计每条线路和站点的对应信息，并计算各条线路的站点数，用站点数量代表该站的线路信息。

&emsp;增加特征`line`。



> **增加站点的类型信息**

&emsp;不同的站点属于不同的类型，比如起点站、终点站、换乘站、普通站等，而这些站点的类别信息可以通过邻站点的数量表示，所以根据路网图对邻站点的数量进行统计，表示各个站点的类别。

&emsp;增加特征`station_type`。



> **增加特殊站点的标记**

&emsp;对站点流量的分析过程中，发现第15站的流量与其他站点存在明显的区别，全天都处于高峰状态，因此给15站添加特别的标记。

&emsp;增加特征`is_special`。



> **连接训练特征和目标值**

&emsp;本次建模的思想使用前一天的流量特征和时间特征，以及预测当天的时间特征，来预测进站流量和出站流量。所以要对之前处理好的数据集进行拼接。

&emsp;增加新的特征`yesterday_is_holiday`以及`today_is_holiday`，增加目标值列`inNums`和`outNums`。



> **对时间间隔进行目标编码**

&emsp;考虑时间间隔信息与进站、出站流量的相关性，对时间间隔信息针对`inNums`和`outNums`进行目标编码。(这一步并非必须的，一定程度上可能会导致过拟合，所以可以考虑加入和不加入的情况都测试一下)

&emsp;目标编码后得到了`in_time_cut`和`out_time_cut`。



## <font size=3>1.3 划分训练集和测试集</font>

&emsp;根据上面数据清洗以及特征工程得到的结果对数据集进行划分。



## <font size=3>1.4 搭建模型预测</font>

&emsp;我们使用了LightGBM和CatBoost两个模型预测并取其均值，其实也可以尝试加入XGBoost，然后取3个模型的加权平均，但是我们当时训练时发现XGBoost得到的结果不是很好，所以直接丢掉了。其实，通过加权平均，给XGBoost的结果一个比较好的权重，也有可能会得到比较不错的结果。最后，<font color=blue>**对模型结果平均**</font>。



# <font size=4>**2. 其他的一些想法**</font>

(1) 由于官方给了路网图，所以我们尝试将路网图拼接在特征后面，表示各个站点之间的连接关系，但是这样反而降低了模型最终的性能。

<br/>

(2) 过程中我们一直想充分利用邻站点的信息，想在邻站点上提取尽可能多的特征，包括邻站点相同时刻以及相邻时刻的流量信息，但是这样都会降低模型的性能，也在这上面浪费了不少的时间。

<br/>

(3) 除了从以后的数据中提取特征之外，我们还对提出出来的特征做了一些特征工程，包括计算一些可能有一定关联的特征**计算加减以及比率信息**，但是这些工作都没有能够提升模型的性能。

<br/>

(4) 根据官方交流群的讨论，我们在A榜的时候尝试去掉周末的信息，只讲工作日的信息进行提取和拼接，这在一定程度上提升了模型的效果，后来我们在[鱼的代码](<https://zhuanlan.zhihu.com/p/59998657>)基础上加入了我们之前找到的一些特征。之后，仔细推敲了一下鱼的代码，发现在进行特征`feature`和目标值`target`拼接的时候，和`deviceID`相关的特征使用的是预测当天的，这样一方面会导致leak，另一方面就是最后的测试集这些特征都是`nan`值，也就是说在最终的预测中没有起到作用。于是，我们改变了拼接的方法，再次加入了自己提取的特征，最终在A榜跑出的成绩是`12.99`。注意，**这里使用的数据并不是全部的数据，而只是使用了工作日的数据拼接**。

<br/>

&emsp;在B榜的时候，我们还是希望能够训练一个通用的模型，所以这次把所有的数据放在一起训练，并没有将周末的数据单独提取出来，但是在滑窗的时候使用了`13`和`20`两天的数据作为验证，最终跑出了`12.57`的成绩，说明这种想法是可行的。为了验证一下只提取周末信息的效果，我们也尝试把周末的信息单独提取出来，最后得分一直在`14`以上，可能也是因为我们提的特征不太适用于这种场景。

<br/>

&emsp;最终，我们使用了全部的数据通过Stacking的方法进行了训练，将三个梯度提升树模型进行了堆叠，最后得到的结果也是`14`多一点。



# <font size=4>3. 总结与思考</font>

(1) 首先是对数据的EDA做的不够，包括对各个站点的分析，各个时间段综合分析，对特征重要性的分析等等。主要还是因为经验不够，不知道该怎么做，甚至15站点的特殊性也是从交流群里得到的信息。另外，就是调参做的有问题，反而把模型的性能调低了，说明对参数的理解不够。

<br/>

(2) 第一次团队作战，不知道该怎么协作，分工不是很明确，所以效率不是很高，没能有机会尝试更多的模型。代码写的不规范，导致后面修改的时候浪费了比较多的时间，包括整理也花了不少的时间。

<br/>

(3) 知道的模型太少，只使用了梯度提升树模型，其实还有很多可能有效的模型可以尝试，包括图神经网络，时空模型以及LSTM，但是都因为不够熟悉而无从入手。

<br/>

(4) 总结：

- 了解自己能做的事情，明确分工；
- 做好EDA，做到对数据的充分理解；
- 代码书写规范，每一个功能模块应该定义为一个函数；
- 熟悉不同模型的功能以及试用场景。

