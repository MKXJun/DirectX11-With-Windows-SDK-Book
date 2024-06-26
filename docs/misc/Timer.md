</br>
</br>

# 前言

[GAMES104](https://www.bilibili.com/video/BV12Z4y1B7th?t=3583.3)的王希说过：

> 游戏引擎的世界里，它的核心是靠Tick()函数把这个世界驱动起来。

本来单是一个CPU的计时器是不至于为其写一篇博客的，但把GPU计时器功能加上后就不一样了。在这一篇中，我们将讲述如何使用CPU计时器获取帧间隔，以及使用GPU计时器获取GPU中执行一系列指令的间隔。

# CPU计时器

在游戏中，我们需要用到高精度的计时器。在这里我们直接使用龙书的`GameTimer`，但为了区分后续的GPU计时器，现在将其改名为`CpuTimer`：

```cpp
class CpuTimer
{
public:
    CpuTimer();
 
    float TotalTime()const;     // 返回从Reset()调用之后经过的时间，但不包括暂停期间的
    float DeltaTime()const;		// 返回帧间隔时间

    void Reset();               // 计时开始前或者需要重置时调用
    void Start();               // 在开始计时或取消暂停的时候调用
    void Stop();                // 在需要暂停的时候调用
    void Tick();                // 在每一帧开始的时候调用
    bool IsStopped() const;     // 计时器是否暂停/结束

private:
    double m_SecondsPerCount = 0.0;
    double m_DeltaTime = -1.0;

    __int64 m_BaseTime = 0;
    __int64 m_PausedTime = 0;
    __int64 m_StopTime = 0;
    __int64 m_PrevTime = 0;
    __int64 m_CurrTime = 0;

    bool m_Stopped = false;
};
```

在构造函数中，我们将查询计算机performance counter的频率，因为该频率对于当前CPU是固定的，我们只需要在初始化阶段获取即可。然后我们可以求出单个count经过的时间：

```cpp
CpuTimer::CpuTimer()
{
    __int64 countsPerSec{};
    QueryPerformanceFrequency((LARGE_INTEGER*)&countsPerSec);
    m_SecondsPerCount = 1.0 / (double)countsPerSec;
}
```

在开始使用计数器之前，或者想要重置计时器时，我们需要调用一次`Reset()`，以当前时间作为基准时间。这些`__int64`的类型存储的单位为count：

```cpp
void CpuTimer::Reset()
{
    __int64 currTime{};
    QueryPerformanceCounter((LARGE_INTEGER*)&currTime);

    m_BaseTime = currTime;
    m_PrevTime = currTime;
    m_StopTime = 0;
    m_PausedTime = 0;	// 涉及到多次Reset的话需要将其归0
    m_Stopped  = false;
}
```

然后这里我们先看`Stop()`的实现，就是记录当前Stop的时间和标记为暂停中：

```cpp
void CpuTimer::Stop()
{
    if( !m_Stopped )
    {
        __int64 currTime{};
        QueryPerformanceCounter((LARGE_INTEGER*)&currTime);

        m_StopTime = currTime;
        m_Stopped  = true;
    }
}
```

在调用`Reset()`完成初始化后，我们就可以调用`Start()`启动计时了。当然如果之前调用过`Stop()`的话，将当前`Stop()`和`Start()`经过的暂停时间累加到总的暂停时间：

```cpp
void CpuTimer::Start()
{
    __int64 startTime{};
    QueryPerformanceCounter((LARGE_INTEGER*)&startTime);


    // 累积暂停开始到暂停结束的这段时间
    //
    //                     |<-------d------->|
    // ----*---------------*-----------------*------------> time
    //  m_BaseTime       m_StopTime        startTime     

    if( m_Stopped )
    {
        m_PausedTime += (startTime - m_StopTime);	

        m_PrevTime = startTime;
        m_StopTime = 0;
        m_Stopped  = false;
    }
}
```

然后在每一帧开始之前调用`Tick()`函数，更新当前帧与上一帧之间的间隔时间，该用时通过`DeltaTime()`获取，可以用于物理世界的更新：

```cpp
void CpuTimer::Tick()
{
    if( m_Stopped )
    {
        m_DeltaTime = 0.0;
        return;
    }

    __int64 currTime{};
    QueryPerformanceCounter((LARGE_INTEGER*)&currTime);
    m_CurrTime = currTime;

    // 当前Tick与上一Tick的帧间隔
    m_DeltaTime = (m_CurrTime - m_PrevTime)*m_SecondsPerCount;

    m_PrevTime = m_CurrTime;

    if(m_DeltaTime < 0.0)
    {
        m_DeltaTime = 0.0;
    }
}

float CpuTimer::DeltaTime() const
{
    return (float)m_DeltaTime;
}
```

如果要获取游戏开始到现在经过的时间（不包括暂停期间），可以使用`TotalTime()`：

```cpp
float CpuTimer::TotalTime()const
{
    // 如果调用了Stop()，暂停中的这段时间我们不需要计入。此外
    // m_StopTime - m_BaseTime可能会包含之前的暂停时间，为
    // 此我们可以从m_StopTime减去之前累积的暂停的时间
    //
    //                     |<-- 暂停的时间 -->|
    // ----*---------------*-----------------*------------*------------*------> time
    //  m_BaseTime       m_StopTime        startTime     m_StopTime    m_CurrTime

    if( m_Stopped )
    {
        return (float)(((m_StopTime - m_PausedTime)-m_BaseTime)*m_SecondsPerCount);
    }

    // m_CurrTime - m_BaseTime包含暂停时间，但我们不想将它计入。
    // 为此我们可以从m_CurrTime减去之前累积的暂停的时间
    //
    //  (m_CurrTime - m_PausedTime) - m_BaseTime 
    //
    //                     |<-- 暂停的时间 -->|
    // ----*---------------*-----------------*------------*------> time
    //  m_BaseTime       m_StopTime        startTime     m_CurrTime
    
    else
    {
        return (float)(((m_CurrTime-m_PausedTime)-m_BaseTime)*m_SecondsPerCount);
    }
}
```

总的来说，正常的调用顺序是`Reset()`、`Start()`，然后每一帧调用`Tick()`，并获取`DeltaTime()`。在需要暂停的时候就`Stop()`，恢复用`Start()`。

# GPU计时器

假如我们需要统计某一个渲染过程的用时，如后处理、场景渲染、阴影绘制等，可能有人的想法是这样的：

```cpp
timer.Start();
DrawSomething();
timer.Tick();
float deltaTime = timer.DeltaTime();
```

实际上这样并不能测量，因为CPU跟GPU是异步执行的。设备上下文所调用的大部分方法实际上是向显卡塞入命令然后立刻返回，这些命令被缓存到一个命令队列中等待被消化。

因此，如果要测量GPU中一段执行过程的用时，我们需要向GPU插入两个时间戳，然后将这两个时间戳的Tick Count回读到CPU，最后通过GPU获取这期间的频率来求出间隔。

目前`GpuTimer`放在Common文件夹中，供36章以后的项目使用，后续会考虑放到之前的项目中。

`GpuTimer`类的声明如下：

```cpp
class GpuTimer
{
public:
    GpuTimer() = default;
    
    // recentCount为0时统计所有间隔的平均值
    // 否则统计最近N帧间隔的平均值
    void Init(ID3D11Device* device, ID3D11DeviceContext* deviceContext, size_t recentCount = 0);
    
    // 重置平均用时
    // recentCount为0时统计所有间隔的平均值
    // 否则统计最近N帧间隔的平均值
    void Reset(ID3D11DeviceContext* deviceContext, size_t recentCount = 0);
    // 给命令队列插入起始时间戳
    HRESULT Start();
    // 给命令队列插入结束时间戳
    void Stop();
    // 尝试获取间隔
    bool TryGetTime(double* pOut);
    // 强制获取间隔(可能会造成阻塞)
    double GetTime();
    // 计算平均用时
    double AverageTime()
    {
        if (m_RecentCount)
            return m_AccumTime / m_DeltaTimes.size();
        else
            return m_AccumTime / m_AccumCount;
    }

private:
    
    static bool GetQueryDataHelper(ID3D11DeviceContext* pContext, bool loopUntilDone, ID3D11Query* query, void* data, uint32_t dataSize);
    

    std::deque<double> m_DeltaTimes;    // 最近N帧的查询间隔
    double m_AccumTime = 0.0;           // 查询间隔的累计总和
    size_t m_AccumCount = 0;            // 完成回读的查询次数
    size_t m_RecentCount = 0;           // 保留最近N帧，0则包含所有

    std::deque<GpuTimerInfo> m_Queries; // 缓存未完成的查询
    Microsoft::WRL::ComPtr<ID3D11Device> m_pDevice;
    Microsoft::WRL::ComPtr<ID3D11DeviceContext> m_pImmediateContext;
};
```

其中，`Init()`用于获取D3D设备和设备上下文，并根据`recentCount`确定要统计最近N帧间隔的平均值，还是所有间隔的平均值：

```cpp
void GpuTimer::Init(ID3D11Device* device, ID3D11DeviceContext* deviceContext, size_t recentCount)
{
    m_pDevice = device;
    m_pImmediateContext = deviceContext;
    m_RecentCount = recentCount;
    m_AccumTime = 0.0;
    m_AccumCount = 0;
}
```

在调用`Init()`后，我们就可以开始调用`Start()`来给命令队列插入起始时间戳了。但在此之前，我们需要先介绍我们需要给命令队列插入的具体是什么。

## ID3D11Device::CreateQuery--创建GPU查询

为了创建GPU查询，我们需要先填充`D3D11_QUERY_DESC`结构体：

```cpp
typedef struct D3D11_QUERY_DESC {
  D3D11_QUERY Query;
  UINT        MiscFlags;  // 目前填0
} D3D11_QUERY_DESC;
```

关于枚举类型`D3D11_QUERY`，我们现在只关注其中两个枚举值：

- `D3D11_QUERY_TIMESTAMP`：通过`ID3D11DeviceContext::GetData`返回的`UINT64`表示的是一个时间戳的值。该查询还需要`D3D11_QUERY_TIMESTAMP_DISJOINT`的配合来判断当前查询是否有效。
- `D3D11_QUERY_TIMESTAMP_DISJOINT`：用来确定当前的`D3D11_QUERY_TIMESTAMP`是否返回可信的结果，并可以获取当前流处理器的频率，来允许你将这两个tick变换成经过的时间来求出间隔。该查询只应该在每帧或多帧中执行一次，然后通过`ID3D11DeviceContext::GetData`返回`D3D11_QUERY_DATA_TIMESTAMP_DISJOINT`。

`D3D11_QUERY_DATA_TIMESTAMP_DISJOINT`的结构体如下：

```cpp
typedef struct D3D11_QUERY_DATA_TIMESTAMP_DISJOINT {
  UINT64 Frequency;   // 当前GPU每秒增加的counter数目
  BOOL   Disjoint;    // 仅当其为false时，两个时间戳的询问才是有效的，表明这期间的频率是固定的
                      // 若为true，说明可能出现了拔开笔记本电源、过热、由于节点模式导致的功耗降低等
} D3D11_QUERY_DATA_TIMESTAMP_DISJOINT;
```

由于从GPU回读数据是一件很慢的事情，可能会拖慢1帧到几帧，为此我们需要把创建好的时间戳和频率/连续性查询先缓存起来。这里使用的是`GpuTimerInfo`类

```cpp
struct GpuTimerInfo
{
    D3D11_QUERY_DATA_TIMESTAMP_DISJOINT disjointData {};  // 频率/连续性信息
    uint64_t startData = 0;  // 起始时间戳
    uint64_t stopData = 0;   // 结束时间戳
    Microsoft::WRL::ComPtr<ID3D11Query> disjointQuery;    // 连续性查询
    Microsoft::WRL::ComPtr<ID3D11Query> startQuery;       // 起始时间戳查询
    Microsoft::WRL::ComPtr<ID3D11Query> stopQuery;        // 结束时间戳查询
    bool isStopped = false;                               // 是否插入了结束时间戳
};
```

在`Start()`中我们需要同时创建查询、插入时间戳、开始连续性/频率查询。

```cpp
HRESULT GpuTimer::Start()
{
    if (!m_Queries.empty() && !m_Queries.back().isStopped)
        return E_FAIL;

    GpuTimerInfo& info = m_Queries.emplace_back();
    CD3D11_QUERY_DESC queryDesc(D3D11_QUERY_TIMESTAMP);
    m_pDevice->CreateQuery(&queryDesc, info.startQuery.GetAddressOf());
    m_pDevice->CreateQuery(&queryDesc, info.stopQuery.GetAddressOf());
    queryDesc.Query = D3D11_QUERY_TIMESTAMP_DISJOINT;
    m_pDevice->CreateQuery(&queryDesc, info.disjointQuery.GetAddressOf());

    m_pImmediateContext->Begin(info.disjointQuery.Get());
    m_pImmediateContext->End(info.startQuery.Get());
    return S_OK;
}
```

需要注意的是，`D3D11_QUERY_TIMESTAMP`只通过`ID3D11DeviceContext::End`来插入起始时间戳；`D3D11_QUERY_TIMESTAMP_DISJOINT`则需要区分``ID3D11DeviceContext::Begin`和`ID3D11DeviceContext::End`。

在完成某个特效渲染后，我们可以调用`Stop()`来插入结束时间戳，并完成连续性/频率的查询：

```cpp
void GpuTimer::Stop()
{
    GpuTimerInfo& info = m_Queries.back();
    m_pImmediateContext->End(info.disjointQuery.Get());
    m_pImmediateContext->End(info.stopQuery.Get());
    info.isStopped = true;
}
```

调用`Stop()`后，这时我们还不一定能够拿到间隔。考虑到运行时的性能分析考虑的是多间隔求平均，我们可以接受延迟几帧的回读。为此，我们可以使用`TryGetTime()`，尝试对时间最久远、仍未完成的查询尝试GPU回读：

```cpp
bool GpuTimer::GetQueryDataHelper(ID3D11DeviceContext* pContext, bool loopUntilDone, ID3D11Query* query, void* data, uint32_t dataSize)
{
    if (query == nullptr)
        return false;

    HRESULT hr = S_OK;
    int attempts = 0;
    do
    {
        // 尝试GPU回读
        hr = pContext->GetData(query, data, dataSize, 0);
        if (hr == S_OK)
            return true;
        attempts++;
        if (attempts > 100)
            Sleep(1);
        if (attempts > 1000)
        {
            assert(false);
            return false;
        }
    } while (loopUntilDone && (hr == S_FALSE));
    return false;

bool GpuTimer::TryGetTime(double* pOut)
{
    if (m_Queries.empty())
        return false;

    GpuTimerInfo& info = m_Queries.front();
    if (!info.isStopped) return false;
    if (info.disjointQuery && !GetQueryDataHelper(m_pImmediateContext.Get(), false, info.disjointQuery.Get(), &info.disjointData, sizeof(info.disjointData)))
        return false;
    info.disjointQuery.Reset();

    if (info.startQuery && !GetQueryDataHelper(m_pImmediateContext.Get(), false, info.startQuery.Get(), &info.startData, sizeof(info.startData)))
        return false;
    info.startQuery.Reset();

    if (info.stopQuery && !GetQueryDataHelper(m_pImmediateContext.Get(), false, info.stopQuery.Get(), &info.stopData, sizeof(info.stopData)))
        return false;
    info.stopQuery.Reset();

    if (!info.disjointData.Disjoint)
    {
        double deltaTime = static_cast<double>(info.stopData - info.startData) / info.disjointData.Frequency;
        if (m_RecentCount > 0)
            m_DeltaTimes.push_back(deltaTime);
        m_AccumTime += deltaTime;
        m_AccumCount++;
        if (m_DeltaTimes.size() > m_RecentCount)
        {
            m_AccumTime -= m_DeltaTimes.front();
            m_DeltaTimes.pop_front();
        }
        if (pOut) *pOut = deltaTime;
    }
    else
    {
        double deltaTime = -1.0;
    }

    m_Queries.pop_front();
    return true;
}
```

如果你就是在当前帧获取间隔，可以使用`GetTime()`：

```cpp
double GpuTimer::GetTime()
{
    if (m_Queries.empty())
        return -1.0;

    GpuTimerInfo& info = m_Queries.front();
    if (!info.isStopped) return -1.0;

    if (info.disjointQuery)
    {
        GetQueryDataHelper(m_pImmediateContext.Get(), true, info.disjointQuery.Get(), &info.disjointData, sizeof(info.disjointData));
        info.disjointQuery.Reset();
    }
    if (info.startQuery)
    {
        GetQueryDataHelper(m_pImmediateContext.Get(), true, info.startQuery.Get(), &info.startData, sizeof(info.startData));
        info.startQuery.Reset();
    }
    if (info.stopQuery)
    {
        GetQueryDataHelper(m_pImmediateContext.Get(), true, info.stopQuery.Get(), &info.stopData, sizeof(info.stopData));
        info.stopQuery.Reset();
    }

    double deltaTime = -1.0;
    if (!info.disjointData.Disjoint)
    {
        deltaTime = static_cast<double>(info.stopData - info.startData) / info.disjointData.Frequency;
        if (m_RecentCount > 0)
            m_DeltaTimes.push_back(deltaTime);
        m_AccumTime += deltaTime;
        m_AccumCount++;
        if (m_DeltaTimes.size() > m_RecentCount)
        {
            m_AccumTime -= m_DeltaTimes.front();
            m_DeltaTimes.pop_front();
        }
    }

    m_Queries.pop_front();
    return deltaTime;
}
```



重置GPU计时器的话使用`Reset()`方法：

```cpp
void GpuTimer::Reset(ID3D11DeviceContext* deviceContext, size_t recentCount)
{
    m_Queries.clear();
    m_DeltaTimes.clear();
    m_pImmediateContext = deviceContext;
    m_AccumTime = 0.0;
    m_AccumCount = 0;
    if (recentCount)
        m_RecentCount = recentCount;
}
```



下面的代码展示如何使用GPU计时器：

```cpp
m_GpuTimer.Init(m_pd3dDevice.Get(), m_pd3dImmediateContext.Get());



// ...
m_GpuTimer.Start();
{
    // 一些绘制过程...
}
m_GpuTimer.Stop();

// ...
m_GpuTimer.TryGetTime(nullptr); // 只是为了更新下面的平均值
float avgTime = m_GpuTimer.AverageTime();
```

下面是分块延迟渲染统计各个pass用时的例子：

![image](..\assets\Timer\01.png)

> **注意：**如果游戏开启了垂直同步，那么当前帧中的某一个查询很可能会受到垂直同步的影响被拖长，从而导致原本当前帧GPU计时器的平均用时总和会接近两个垂直同步信号的间隔。

![image](..\assets\Timer\02.png)

