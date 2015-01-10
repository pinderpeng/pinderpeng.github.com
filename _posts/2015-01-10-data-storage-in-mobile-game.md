---
layout: post
title: 手游开发中的数据存储
categories:
- technology
tags:
- 游戏 存储

---

本文针对手游客户端开发中的数据存储进行总结和讨论。
## 存储方案总结
---

在手游客户端开发时，面对数据存储，我们不得不在响应时间、CPU消耗、复杂度等方面做出决定，一般来说有如下几种方案：

（1）同步文件存储

同步文件存储是指在单进程中将数据存储在本地文件中。在cocos2d-x中，一般是利用UserDefault类将一些数据存储在XML文件中。这种方案的好处是，针对较小的数据，如游戏设置，角色属性等，非常适合，代码实现也较为简单。缺点是，每次更改一项数据，都需要遍历整个XML DOM树，当数据量大的时候，存取时间就比较长了。一个可能的优化是，将不同的数据分别存储在不同的XML文件中。在下面的“文件存储模块”中会进行更详细的设计方法介绍。

（2）异步文件存储

异步文件存储是指创建另外一个线程专门负责数据的存储。在cocos2d-x中，比较常见的是图片等资源的异步加载以及异步保存。例如，在保存某个游戏场景的截图时，在该测试环境下，如果选择同步保存为PNG格式需要4s，如果同步保存为jpg格式需要1s，在移动设备中表现出了明显的卡顿，这是玩法所不能忍受的。如果选择异步保存jpg格式，时间代价就很小了。这种存储方式的弊端是什么呢？lua是不可重入的，也就是不能多个线程同时访问Lua堆栈，如果想要监督异步存储的情况是比较麻烦的。这会在下面的“离线日志模块”进行进一步的讨论。

（3）同步数据库存储

同步数据库存储是指在主进程中创建数据库进行数据存储。在手游中，一般使用sqlite轻量级数据库比较多，这种存储方法适合于存储总量相对较大，单次存取消耗不是很大的情况。

（4）异步数据库存储

异步数据库存储与同步数据库存储类似，但是创建另外线程专门负责数据的存储。这种存储方法适合于存储总量相对较大，单次存取消耗也可能比较大的情况。这种方式也适合可以把所有的存储任务统一交给一个线程来存储。

## 数据存储实践
---

（1）文件存储模块

首先，在这里简要介绍一下存储加密相关的内容。一些手游客户端会存储一些比较敏感的数据，特别是单机、半联网的手游。对于本地文件存储，可以将lua数据项序列化为字符串，然后在数据读取的时候利用RC4等算法进行解密，在数据存储的时候利用RC4等算法进行加密。对于sqlite数据库存储，如果只对数据项加密，每次数据读写都要加解密，并且整个数据库表结构是暴露的，玩家可以复制或删除数据项，所以可以采用加密版本的wxsqlite3进行数据库文件级别的加密。

在lua中可以设计一个专门负责本地文件存储的基类：

{% highlight lua %}
BaseData = class("BaseData")

function BaseData:ctor(dataKey)
    self._rc4Key = "some rc4 key"
    self._dataKey = dataKey
    self.data = {}
end
{% endhighlight %}

数据读取中，首先调用getStringForKey设置数据键值，其次采用RC4进行数据解密，最后反序列化数据内容：

{% highlight lua %}
function BaseData:load()
	local data = cc.UserDefault:getInstance():getStringForKey(self._dataKey, "")
    if data ~= "" then
        data = RC4..decode(data, self._rc4Key)
        self.data = string2table(data)
        return true
     end
     return false
end
{% endhighlight %}

数据存储中，首先序列化数据，其次采用RC4进行数据加密，最后调用setStringForKey设置数据键值。每次修改完数据项，调用dataKey类型Data的save()函数即可。

{% highlight lua %}
function BaseData:save()
    local data = table2string(self.data)
    data = RC4.encode(data, self._rc4Key)
    cc.UserDefault:getInstance():setStringForKey(self._dataKey, data)
end
{% endhighlight %}

（2）离线日志模块

离线日志的应用场景是针对非强联网手游。在联网时，将游戏中的日志信息立即发回给服务器。在非联网时，将生成的日志保存到本地，当联网时再将数据发回给服务器。其中，为防止日志丢失，离线的日志应该立即保存。由于存在玩家很长时间没联网、或者一次性产生了大量数据等情况，离线数据可能会积累较多，所以采用异步数据库存储的方式。设计方法如下：

1）在C++引擎端，创建离线日志类，开一个线程进行数据存取，读写sqlite数据库。数据库Protocol表包含id、pid、content、time几个字段，分别代表自增id、协议序号、协议内容和协议时间。C++主线程和存取线程利用线程安全的队列进行数据交互。为避免消耗过多的CPU，不推荐使用无锁队列，可以使用条件变量来实现该队列。

{% highlight cpp %}
class OfflineLogger : public cocos2d::Ref  { 
private:
    sqlite3 *_pdb;
    threadsafe_queue<MessageItem> _taskQueue; 
    bool _isLoaded;
    std::list<std::string> _logList;
private:
    void connectDB(const std::string &filename);
    void createTable(); 
    void closeDB();
    void insertToTable(int pid, const std::string &content, int time); 
    void delFromTable(int pid, int time); 
    void loadDataFromDB(); 
    void addTask(bool insertFlag, int pid, const std::string &content, int time); 
public:
    static OfflineLogger *create(const std::string &filename); 
    OfflineLogger(const std::string &filename);
    ~OfflineLogger();
 
    void process(const std::string &filename); 
    void addTask(int pid, const std::string &content, int time); 
    void removeTask(int pid, int time);
    bool isLogLoaded();
    std::list<std::string> &getLoadedLogs();  
};
{% endhighlight %}

2）当客户端启动时，异步加载数据库中的离线日志数据。但是，主线程如何知道加载完成呢？如果是同步存储的话，我们可以利用观察者模式，在lua中注册C++的回调函数，当同步存储完成进行相应的调用。但是异步存储这样就不可行了，可行的办法是利用线程间的共享内存，在lua中利用cocos2d-x的scheduleScriptFunc每隔一段时间询问共享内存中的标记值。由于lua每隔固定时间就要访问C++接口，可能会产生一些消耗，所以一个优化是联网之后才去询问这个标记值。

{% highlight lua %}
self.protocolSendNum = self.protocolSendNum + 1
if OFFLINE_LOGGER and self.connected and self.offlineLogger:canSendLogs() then
    local logs = self.offlineLogger:sendLogs()
end
{% endhighlight %}

3）在每个网络数据包中，增加connected、num和time字段，connected表示是否为离线数据包、num表示本地客户端启动后数据包的编号、time为数据包的产生时间。当有新的数据包时，经过序列化和加密处理后，在数据库表中填写num、content和time三个字段，增加一条记录放入数据队列。

{% highlight lua %}
local protocol = {
     prot = prot,
     args = args,
     time = os.time(),
     dataBuffer = nil,
     isLogin = isLogin,
     connected = ret,
     num = self.protocolSendNum
 }
 {% endhighlight %}

4）当联网成功发送数据包时，根据数据包的connected判断是否是离线日志，再根据num和time删除数据表中的相应记录。

另外，离线日志还有一种可能的设计方式是同步数据库存储，即利用cocos2d-x的scheduleScriptFunc每隔一段时间自动加载限制数量的离线日志，然后插入发送队列中。当离线日志全部加载完成，再停止scheduleScriptFunc。具体实现方式可以根据游戏实际情况选取。

## 一些讨论
---

（1）数据库lua接口导出

对于同步数据库存储，可以从cocos2d-x引擎中导出wxsqlite3的lua接口，供lua引擎端灵活调用，如可进行创建数据库表、插入删除等操作。

（2）异步数据库存储

上述的离线日志系统是针对特定业务场景定制的异步数据库存储方案，我们还可以继续进行拓展，实现普遍意义上的异步数据库存储方案，其实现方法也是类似的。

但也依然存在lua不可重入的问题，一种解决方法是，专门创建一个scheduler模块，统一负责检查所有注册的标记值，监听任务完成并进行通知。

（3）存储方案选择

不同的游戏类型、不同的数据类型都可能决定存储方案的选择，可以根据实际的情况来选取，也可组合多种数据存储方案，没有必要过度、过早优化。