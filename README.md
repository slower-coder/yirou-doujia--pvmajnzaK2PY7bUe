
K，K线，Candle蜡烛图。


T，技术分析，工具平台


L，公式Language语言使用c\+\+14，Lite小巧简易。


项目仓库：[https://github.com/bbqz007/KTL](https://github.com)     


国内仓库：[https://gitee.com/bbqz007/KTL](https://github.com)


当前0\.9\.2新添加功能基于QCharts跟通达信mdt数据文件。使用者能够使用QCharts自定义数据处理图表。


新功能如下：


1\. 数据分析工具，AlgoSysDataTool.cpp添加QCharts的K线图预览通达信日线文件。


2\. 新添加图表分析工具，AlgoSysChartsTool.cpp。使用者可以在功能菜单打开内置编辑器进行修改。


2\.1 解析通达信每日市场全景数据mdt文件。


2\.2 提供每日全景数据星空图。


2\.3 提供每日涨跌分布大饼图。


2\.4 提供每日全景数据列表，特殊列当日K线形状。


2\.5 提供日期区间段全景涨幅汇总，特殊列评级形态。


3\. 通过我的cvtool工具利用opencv找相似K线波段。


4\. 编程平台添加两个可编程模块，QCharts，QtConcurrent。


 


现在可以使用QCharts模块在程序平台中进行编程了。


QCharts提供了多种图表，表现力丰富，十分方便使用。


主要类，QChartView，QChart，Series，Axis。关系如下，QChartView应用QChart绘制图表，QChart使用数据模型Series还有坐标模型Axis。每个Series类型代表一种图表跟数据模型，QChart依照Series的种类绘制不同的图表。QChart对应着直角系坐标的图表，QPolarChart对应极坐标的图表。只要填装好数据模型，QChart就能在QChartView绘制出图表，十分容易使用。


同时也十分挑编程步骤，而且官方文档还没有说明。你不能在QChart没有Series数据模型之前，向QChart添加Axis，包括调用QChart的createDefaultAxes。因为它的内部实现中，添加Axis一定要去依赖Series，即使你没有Attach它们。这使得你的程序在运行期间直接挂掉。对于熟手后就习惯了，但对初使用十分不友好，谁知道你这么挑步骤，而且还不在官方文档中说明。


添加K线预览通达信日线文件，纯粹为了演示。20行代码就能用QCharts写一个K线图，也就只能是好玩演示。按住CTRL，用鼠标可以拖动K线图，滑轮放大缩小坐标比例。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113124659535-1514042379.gif)


另一个添加的编程模块是QtConcurrent。


使用者可以在本程序中使用QtConcurrent更方便地进行多线程处理数据。


我在repacking.patch中提供的代码，包括两个简单map\-reduce例子，以及全景数据重新打包工具。都是使用QtConcurrent进行编写的map\-reduce。


使用者可以套用map\-reduce例子进行编程。


首先介绍一下，QtConcurrent，QFuture，QFutureWatcher。


QtConcurrent::run将lambda运行在多线程，并返回一个QFuture且代表这个异步任务。


同步访问结果，任何线程访问QFuture的result方法都会阻塞直到异步任务结束。


异步完成回调，将QFuture跟QFutureWatcher关联在一起，当异步任务结束时，QFuture发起对QFutureWatcher的Q\_SIGNAL(finished)。


QFutureWatcher是一个QObject对象，Q\_SIGNAL(finished)连接的Q\_SLOT会在它的诞生线程的eventloop中分派，进而完成异步回调。


一般地，QFutureWatcher会使用QApplication的线程，即UI线程。我将这个线程通过QFutureWatcher进行串行的reduce，QtConcurrent使用线程池并行的map。


所以可以得到第一个简单例子




```
auto task = new QFutureWatcher<int>;
// map
auto runnable = [=](int i){
    OutputDebugString(std::to_string(i).c_str());
    return ++i;
};
QObject::connect(task, &QFutureWatcher<int>::finished, 
     // reduce
    [=]{
        auto i = task->result();
        // another map
        if (i < 10)
            task->setFuture(QtConcurrent::run(runnable, i));
    });
task->setFuture(QtConcurrent::run(runnable, 0));
```


上面的例子， 只有一个map任务在同时运行。map任务结束就会通过QFutureWatcher发起signal排队到ui线程，slot在ui线程执行reduce，发出map任务，处理结果。


于是有第二个例子，并行map任务，串行reduce。




```
auto queue = std::make_sharedint> >(0);
for (int i = 0; i < 100; ++i)
{
    queue->emplace_back(i);
}

// map
auto runnable = [=](int i){
    OutputDebugString((std::to_string((int)QThread::currentThreadId()) + "," + std::to_string(i)).c_str());
    return ++i;
};

auto make_reducer = [=]{
    auto task = new QFutureWatcher<int>;
    auto reduce = [=]{
        if (!queue->empty())
        {
            // another map, consume queue
            task->setFuture(QtConcurrent::run(runnable, queue->back()));
            queue->pop_back();
        }
    };
    QObject::connect(task, &QFutureWatcher<int>::finished, reduce);
    reduce();
    return task;
};

// spawn 4 map parallel tasks.
make_reducer();
make_reducer();
make_reducer();
make_reducer();    
```


上面的例子，同时运行4个map任务。当一个map任务结束后，reduce会再向线程池运行一个map任务，确保线程池有4个map任务在同时运行。


一定要指出，map任务运行的时间必须大于reduce运行时间，map\-reduce才有意义。例如O(map) : O(reduce) \= 4 : 1。上面并行4个map的例子就可以变成O(map/4\) : O(reduce) \= 1:1。reduce不用等待map耗时。


如果map任务运行时间小于reduce任务运行时间，那么并行再多的map也只能听从reduce的慢，一堆结果阻塞在队列。


我提供的repacking小工具，将下载到的单日的全景数据zip文件，重新打包成一个zip文件。操作包括unzip，zip。实现方式使用map\-reduce，将unzip运行在map任务，zip运行在reduce任务。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241122200850867-2117585905.gif)


 


然后是主要新增功能，市场全景图。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113125018373-1744278603.png)


星空图（直角坐标系）有三个维度，横坐标振幅，纵坐标是涨幅。点的粗细代表成交量。以斜率1：1跟1：\-1可以划分出两个区域，位于斜线左侧的，表示振幅小于涨幅绝对值，如果是涨的就是高开了，如果是跌的就是低开了。斜线右侧的，表示振幅大于涨幅绝对值，发生过最高转低或最低转高。按住CTRL通过鼠标滑轮放大缩小坐标。


星空图（极坐标系）同样是三个维度，角坐标是涨幅，半径坐标是振幅。点的粗细代表成交量。左侧代表涨，右侧代表跌。靠近下方表示涨跌得轻，靠近上方表示涨跌得猛。越往外环振幅越大，可能越激，越往内环也就振幅越小没什么起伏。按住CTRL通过鼠标滑轮放大缩小半径坐标。左右拖动调整角坐标位置。


成交量过滤器，可以过滤掉不同等级的成交量的点。使用者可以根据需要，自己修改代码，调整等级划分，或者改用成交额，加权等其它数据。


涨跌家数。这里有4个比例大饼图。市场当日全部交易个股按涨幅等级归类，形成比例大饼。从10%到\-10%划分十个等级。并且颜色从红到绿。按绝对值大小分深浅色。越红涨得多，越绿跌得惨。第一个图是涨幅，下面的图是对比今开的收盘的涨幅，右面的图是今开相对昨收的幅度，反映的昨天市场情绪的延续，高开低开的分布比例。右下图是振幅相关，振幅集中于几多。


全景行情列表， 提供日K线形状列。通过符号文字来画出。可以通过排序来归类出相同K线形状的个股。振幅线用\=号表示，开收线用\+或\*号表示，\+表示收高于开，\*表示收低于开。HL表示最高最低即图的方向。画到列表上就是一根横躺的日K线。双击树型全景数据文件名，就可以打开这个文件的全景数据，生成星空图，大饼图，还有列表。


双击.cod打开A股（包含中小板，创业科创），双击.mdt打开B股。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113134606936-927001841.png)


多日全景汇总列表。 需要选定一个全景文件，右键菜单打开。以这个全景文件的日期为界限，汇总之前的全部文件，或是之后的全部文件。汇总对象是每一个个股的每日涨幅。并且日期区间的起始价跟结束价，以及终止涨幅。最后是一个特殊列，形态评级。将一个个股的每日涨幅按等级评分，10%到\-10%划分成 A\-J 十个等级，A\-E是对应正数，F\-J是对应负数。A是涨停级别，J则是跌停级别。最后汇总并集。使用者可以按自己的需要，制作自己想要的评级形态。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113135730139-551808401.png)


排序后


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113141454040-365332158.png)


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113144225954-2029377224.png)


 


发布提供两个数据样本，shmdt.zip包含240926到241111的数据，对应于10到11月本轮牛市数据。shmdt2\.zip包含24年6月熊市数据。可以对比。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113143924403-105113031.gif)


 


下面就是介绍如何使用我的cvtool工具，利用opencv找相似K线。


![](https://img2024.cnblogs.com/blog/665551/202411/665551-20241113144407913-427181765.png)


步骤：


1\. 将你要分析的K线图，使用白底配色方案保存成带数字编号结尾的图片，如001\.png， 002\.png。


2\. cvtool 你的图片名 match。执行这个命令打开你的图片。


3\. 在select窗口，用鼠标框选出一段K线，然后在鼠标滑轮键双击提交。


4\. 结果会显示在match窗口。


5\. 在match窗口调整算法参数，推荐SQIFF NORMED，即第一条bar。第二条bar调整阀值，值越大越多错误结果，越小越少甚至过滤成无。


6\. 空格键打开下一编号的图，并应用当前match。


因为match算法不是基于ML，所以只能娱乐一下。opencv到底能不能够满足你的要求，自己来调教一下吧。


我的cvtool工具，原本只是用来调试调教opencv参数。介绍地址在[https://github.com/bbqzsl/p/13992225\.html](https://github.com):[wgetCloud机场](https://tabijibiyori.org)。


后续可能会将opencv也整合进来。


投资一定有输赢。股市是一个财富再分配工具，你有机会从别人口袋中再分配到财富，也同样有机会将财富再分配给别人。赢的钱不会凭空生出来。


 


另外，KTL这个工具还可以通过编程扩展你需要的任何功能。


例如我在上一个版本提供了两个小功能patch代码。


用sqlchiper浏览微信数据库，解析protobuf数据文件。详细在 [逆向WeChat(七)](https://github.com/bbqzsl/p/18423518 "发布于 2024-10-07 21:52")


本篇结束。


 


mdt数据文件可以在通达信官网下载。


 


