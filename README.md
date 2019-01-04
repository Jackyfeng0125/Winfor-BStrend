# Winfor-BStrend
&emsp;&emsp;BStrend属于<b>*BS系列*</b>项目的分支，简化了主程序的一部分功能，并主要用于回测和在日K线的级别上对于股票趋势的更新和报告。本项目基于Python语言开发，要求版本在3.6及以上，打包的可执行文件适用于Windows操作系统。本项目的主要设计组件包括如下部分：
+ **策略设计**：核心策略算法为高低点算法和基于高低点之上测趋势算法。本项目中的`高低点`与``趋势``继承主程序的定义及算法，在之后部分（[1.1高低点](#hl)）会予以说明。
+ **数据来源**：本项目对于股票历史数据存在较频繁的读写和遍历。数据主要为股票在回测阶段的收盘价、最高价与最低价，通过Wind数据库的量化接口API导入。
+ **数据存储**：本项目并不直接存储原始股票历史数据，而是将回测后的主要结果（历史高低点，以及当前策略运行产生的中间运算结果）存储在`sqlite`这一python轻量级数据库中；并在之后的每日运行中通过读取数据库获取股票的策略历史运算记录，并结合当前最新数据（当日最高价、最低价及收盘价）进行运算结果的更新，将更新结果写入数据库覆盖之前结果。
+ **图像绘制**：本项目的图像可视化基于`matplotlib.pyplot`这一python二维科学绘图常用工具包，仅在股票回测结束和之后股票趋势发生更新的当日收盘后会更新图像，图像包含了股票截止目前的高低点信息、历史趋势类型。由于图像绘制较为耗时且占用内存较高，因此设置了只绘制部分回测图像的控制参数以节约时空成本。具体设置请见后续部分。
+ **任务调度**：本项目中的任务调度框架采用`APschedular`，主要任务为交易日定时工作及运行日志管理。以A股市场交易时间为基准，用户可设定在每日收盘后（应当考虑到港股收盘时间较晚和WIND数据库数据更新需要一定延迟时间），设定当日更新全体股票的时刻，调度会记录任务堆栈中的定时，并在指定时间触发任务执行的信号。
+ **自动发信**：用户可选择“每日发信”模式和仅当有目标股票发生趋势改变时发信两种模式。美股信息单独发信，A股和港股合并发信。发信内容会提示股票变动，以及基准市场指数的趋势及图像，当前所有股票的趋势类型。
<!-- TOC START min:1 max:6 link:true update:false -->
- [Winfor-BStrend](#winfor-bstrend)  
  - [1.策略设计](#1策略设计)  
    - [1.1高低点](#11高低点)  
    - [1.3高低点和趋势的初始化设定](#13高低点和趋势的初始化设定)  
  - [2.用户导引](#2用户导引)  
    - [2.1文件配置](#21文件配置)  
    - [2.2参数释义](#22参数释义)  
<!-- TOC END -->



### 1.策略设计
#### 1.1高低点 {#hl}
&emsp;&emsp;需要明确的是，高点和低点是间隔交叉出现的。在文档及代码中，请注意以下表示高低点的用语指向同一意思表示：


|             高点             |  高点index |             低点             |  低点index |
|:----------------------------:|:----------:|:----------------------------:|:----------:|
| H,h, hpoint, Hpoint highpoint | hindex, hpi | L，l, lpoint, Lpoint lowpoint | lindex lpi |

**高点**
> i）前低点已确认的情况下，回调达到`THRESH_D`个交易区间单位时，确认此区间的最高点为当前级别高点
ii）前低点已确认的情况下，回调幅度超过前`AVG_N`次回调的均值（可选`AVG_BUFFER`），确认当前区间最高点为当前级别高点
iii）前低点已确认的情况下，回调跌破前低点，即确认当前区间最高点为当前级别高点。

**低点**
> i）前高点已确认的情况下，上涨达到`THRESH_D`个交易区间单位时，确认此区间的最低点为当前级别低点
ii）前低点已确认的情况下，上涨幅度超过前`AVG_N`次上涨的均值（可选`AVG_BUFFER`），确认当前区间最低点为当前级别低点
iii）前低点已确认的情况下，上涨超过前高点，即确认当前区间最低点为当前级别低点。
#### 1.2趋势
&emsp;&emsp;本项目中定义三种趋势类型：
* `up`上升趋势
* `down`下跌趋势
* `consd`盘整趋势

**上涨**
> *上涨趋势的确认*：
i）出现连续两个低点抬高和高点抬高
 $h_i,l_i,h_{i+1},l_{i+1};h_{i+1}>h_{i}, l_{i+1}>l_{i} $
 $l_i,h_i,l_{i+1},h_{i+1};l_{i+1}>l_{i},;h_{i+1}>h_{i}$
 ii）当最高点距离前日线低点`TREND_REV`个交易日时，则确认为日线上涨趋势
*上涨趋势的延续*：低点依次抬高
*上涨趋势结束*：
i）转入盘整趋势
 $h_i,l_i,h_{i+1},l_{i+1};h_{i+1}>h_{i} \ and \ l_{i+1}<l_i$
 ii）转入下跌趋势
 $h_i,l_i,h_{i+1},l_{i+1};h_{i+1}<h_i \ and \ l_{i+1}<l_i$

 **下跌**
 > *下跌趋势的确认*：
 i）出现连续两个低点降低和高点降低
  $h_i,l_i,h_{i+1},l_{i+1};h_{i+1}<h_{i} \ and\ l_{i+1}<l_{i} $
  $l_i,h_i,l_{i+1},h_{i+1};l_{i+1}<l_{i}\ and\ h_{i+1}<h_{i}$
  ii）当最低点距离前日线低点`TREND_REV`个交易日时，则确认为日线下跌趋势
 *下跌趋势的延续*：高点依次降低
 *下跌趋势结束*：
 i）转入盘整趋势
  $h_i,l_i,h_{i+1},l_{i+1};h_{i+1}<l_{i},l_{i+1}>l_i$
  ii）转入上涨趋势
  $h_i,l_i,h_{i+1},l_{i+1};h_{i+1}>h_i,l_{i+1}>l_i$

  **盘整**
  > *盘整趋势的确认*：
  > 由以上上涨趋势和下跌趋势确认转入盘整趋势的
  在序列最初高低点较少时，无法被归类于上涨或下跌趋势的，暂时归类为盘整趋势
  *盘整趋势的结束*：满足上涨或下跌趋势的确认定义，结束盘整趋势而转入上涨或下跌趋势。

#### 1.3高低点和趋势的初始化设定
&emsp;&emsp;在回测时，需要解决在遍历数据列表之前对于高低点和趋势的初始化问你。由于策略算法基于迭代规则，因此初值是必要的。初始化的原则是尽可能耗费较少的数据选择出合适的初值。幸运的是，当回测时期较长的时候，初值对于之后策略表现（主要是判断准确度）的影响会渐渐衰减。
<b>*高低点的初始化设定*</b>
&emsp;&emsp;从序列首开始遍历日线数据，如果找到第一个分型类型是底分型，那么设该底K线为待判定低点，转而进入低点判定分支；如果第一个分型是顶分型，那么设定该顶K线为待判定高点，转而进入高点判定分支。关于分型的具体定义如下：
> *顶分型*：
$K_{i-1}.h\le K_i.h\ and\ K_i.h\ge K_{i+1}.h \ and \ not \ K_{i-1}.h=K_i.h=K_{i+1}.h:称K_i$为该顶分型的顶
*底分型*：
$K_{i-1}.l\ge K_i.l\ and\ K_i.l\le K_{i+1}.l\ and \ not \ K_{i-1}.l=K_i.l=K_{i+1}.l:称K_i$为该底分型的底

<b>*趋势的初始化设定*</b>
&emsp;&emsp;趋势判别在回测算法中需要在遍历数据列表判定完回测区间内所有高低点后进行。趋势初始化需要至少确认4个高点或低点，枚举所有组合可能：
- $l_0, h_0, l_1, h_1; l_1>l_0\Rightarrow$`up`
- $l_0, h_0, l_1, h_1; l_1<l_0\Rightarrow$`consd`
- $h_0, l_0, h_1, l_1; h_0<h_1\Rightarrow$`down`
- $h_0, l_0, h_1, l_1; h_0>h_1\Rightarrow$`consd`
此外在第4个高点或者低点被确认之前的K线趋势都设置为确实类`None`。

### 2.用户导引
#### 2.1文件配置
&emsp;&emsp;用户使用本程序，可直接运行`monitor_s.exe`可执行程序，不需要本地python环境，但是以下支持文件务必完成配置。
- 下载`monitor_s.exe`到本地任意路径 ./directory/BStrend。
- 在同一文件夹下，新建`WindPy.pth`文件，并在文件中写入本机Wind安装地址，例如`C:\Wind\Wind.NET.Client\WindNET\x64`
- 进入Wind界面，在量化接口中修复python插件
- 在`.txt`文件中，第一列写入需要进行回测的Wind股票代码
- 下载config.conf到./directory/BStrend，直接在文件中修改对应的参数，注意注释中的解释和格式要求，并保存。参数具体含义请参考之后具体释义
#### 2.2参数释义

