</br>
</br>

# 前言

提供键鼠输入可以说是一个游戏的必备要素。在这里，我们不使用`DirectInput`，而是使用Windows消息处理机制中的`Raw Input`，不过要从头开始实现会让事情变得很复杂。DXTK提供了鼠标输入的`Mouse.h`和键盘输入的`Keyboard.h`（**现在已经单独抽离出来使用**），对消息处理机制进行了封装，使用`Mouse`类和`Keyboard`类可以让我们的开发效率事半功倍。

对`Raw Input`有兴趣的同学，你可以看键鼠类的内部实现，也可以看MSDN文档： [Raw Input](https://docs.microsoft.com/en-us/windows/desktop/inputdev/raw-input)

`Mouse`类和`Keyboard`类都在名称空间`DirectX`内。

# 鼠标输入

`Mouse`类是一个单例类，我们可以通过`Mouse::Get()`静态方法来获取该实例：

```cpp
static Mouse& Mouse::Get();
```

也可以在应用程序类中直接作为成员使用，因为它提供了默认构造函数。不过为了异常安全，建议使用智能指针：

```cpp
std::unique_ptr<DirectX::Mouse> m_pMouse;
m_pMouse = std::make_unique<DirectX::Mouse>();
```

要想使用鼠标类，我们还需要完成三件事情：

1.在初始化阶段给鼠标类设置要绑定的窗口句柄，使用`Mouse::SetWindow`方法：

```cpp
void Mouse::SetWindow(HWND window);
```

2.设置鼠标模式，需要使用`Mouse::SetMode`方法：

```cpp
void Mouse::SetMode(Mouse::Mode mode);
```

鼠标模式有下面两种：

```cpp
enum Mode
{
    MODE_ABSOLUTE = 0,  // 绝对坐标模式，每次状态更新xy值为屏幕像素坐标，且鼠标可见
    MODE_RELATIVE,      // 相对运动模式，每次状态更新xy值为每一帧之间的像素位移量，且鼠标不可见
};
```

可以加在`D3DApp::Init`方法上：

```cpp
bool D3DApp::Init()
{
    m_pMouse = std::make_unique<DirectX::Mouse>();
    m_pKeyboard = std::make_unique<DirectX::Keyboard>();

    if (!InitMainWindow())
        return false;

    if (!InitDirect3D())
        return false;

    return true;
}
```


3.在消息处理的回调函数上进行修改，在接收到下面这些消息后，需要调用`Mouse::ProcessMessage`方法：

```cpp
WM_ACTIVATEAPP WM_INPUT
WM_LBUTTONDOWN WM_MBUTTONDOWN WM_RBUTTONDOWN WM_XBUTTONDOWN
WM_LBUTTONUP WM_MBUTTONUP WM_RBUTTONUP WM_XBUTTONUP    
WM_MOUSEWHEEL WM_MOUSEHOVER WM_MOUSEMOVE
```

具体的修改的部分在最后会展示。

完成这些操作后，我们的程序就可以开始使用鼠标了。
      
对于每一帧，我们可以通过`Mouse::GetState`方法获取当前帧下鼠标的运动状态：

```cpp
Mouse::State Mouse::GetState() const;
```

`Mouse::State`包含如下成员：

```cpp
struct State
{
    bool    leftButton;         // 鼠标左键被按下
    bool    middleButton;       // 鼠标滚轮键被按下
    bool    rightButton;        // 鼠标右键被按下
    bool    xButton1;           // 忽略
    bool    xButton2;           // 忽略
    int     x;                  // 绝对坐标x或相对偏移量
    int     y;                  // 绝对坐标y或相对偏移量
    int     scrollWheelValue;   // 滚轮滚动累积值
    Mode    positionMode;       // 鼠标模式
};
```

对于剩下的方法：

`Mouse::ResetScrollWheelValue`方法可以清空滚轮的滚动累积值

`Mouse::IsConnected`方法则可以检验鼠标是否连接

`Mouse::SetVisible`方法设置鼠标是否可见

`Mouse::IsVisible`方法可以检验鼠标是否可见


## 鼠标状态追踪

`Mouse::ButtonStateTracker`类提供了更高级的功能，通过根据上一次的鼠标事件和当前鼠标事件的对比，来判断鼠标的状态。它有两个重要方法：

```cpp
void Mouse::ButtonStateTracker::Update( const Mouse::State& state );
// 在每一帧的时候应提供Mouse的当前状态去更新它

State Mouse::ButtonStateTracker::GetLastState() const;
// 获取上一帧的鼠标事件，应当在Update之前使用，否则变为获取当前帧的状态
```

然后还有5个重要公共成员：

```cpp
ButtonState leftButton;     // 鼠标左键状态
ButtonState middleButton;   // 鼠标滚轮按键状态
ButtonState rightButton;    // 鼠标右键状态
ButtonState xButton1;       // 忽略
ButtonState xButton2;       // 忽略
```

> 注意: 这里要区分`State`和`ButtonState`类型的五个同名成员，含义不同。

枚举量`ButtonState`含义：

```cpp
enum ButtonState
{
    UP = 0,         // 按钮未被按下
    HELD = 1,       // 按钮长按中
    RELEASED = 2,   // 按钮刚被放开
    PRESSED = 3,    // 按钮刚被按下
};

```

由于鼠标状态追踪类不是单例，而且它存有上一帧的鼠标状态，不应该作为一个临时变量，而是也应该像鼠标类一样在整个程序生命周期内都存在。

在绝对模式下，我们也可以获取两帧之间的鼠标相对位移量：

```cpp
Mouse::State mouseState = m_pMouse->GetState();
Mouse::State lastMouseState = mMouseTracker.GetLastState();
int dx = mouseState.x - lastMouseState.x, dy = mouseState.y - lastMouseState.y;
```

虽然这样子第一帧的时候，鼠标状态追踪类并没有开始记录，而鼠标已经记录下了第一帧的状态，但由于开始运行的第一帧通常都很快，等到我们第一次使用鼠标的时候已经经过了一段时间，所以并不会产生什么问题。


然后在更新了跟踪器状态后，就可以判断鼠标状态做进一步操作了。以渲染立方体的项目为例，这里打算通过鼠标拖动产生旋转，如：

```cpp
// 更新鼠标按钮状态跟踪器，仅当鼠标按住的情况下才进行移动
mMouseTracker.Update(state);
if (mouseState.leftButton == true && mMouseTracker.leftButton == mMouseTracker.HELD)
{
    // 旋转立方体
    cubeTheta -= (mouseState.x - lastMouseState.x) * 0.01f;
    cubePhi -= (mouseState.y - lastMouseState.y) * 0.01f;
}

mCBuffer.world = XMMatrixRotationY(cubeTheta) * XMMatrixRotationX(cubePhi);
```

# 键盘输入

`Keyboard`类也是一个单例类，我们可以通过`Keyboard::Get()`静态方法来获取该实例：

```cpp
static Keyboard& Keyboard::Get();
```

也可以在应用程序类中直接作为成员使用，因为它提供了默认构造函数。不过为了异常安全，建议使用智能指针：

```cpp
std::unique_ptr<DirectX::Keyboard> m_pKeyboard;
m_pKeyboard = std::make_unique<DirectX::Keyboard>();
```

要想使用键盘类，我们还需要完成一件或两件事情：

1.如果`Keyboard`类内有`SetWindow`方法，则需要调用以初始化，否则不需要：

```cpp
void Keyboard::SetWindow(ABI::Windows::UI::Core::ICoreWindow* window);
```

2.在消息处理的回调函数上进行修改，在接收到下面这些消息后，需要调用`Keyboard::ProcessMessage`方法：

```cpp
WM_ACTIVATEAPP
WM_KEYDOWN WM_SYSKEYDOWN WM_KEYUP WM_SYSKEYUP
```

具体的修改的部分在最后会展示。

完成上述操作我们就可以使用键盘了。

对于每一帧，我们可以通过`Keyboard::GetState`方法获取当前帧下键盘所有按键的状态：

```cpp
Keyboard::State Keyboard::GetState() const;
```

`Keyboard::State`结构体记录了按键信息，而`Keyboard::Keys`枚举量定义了有哪些按键。获取了键盘按键状态后，我们要关注的是`Keyboard::State`内的方法：

`Keyboard::State::IsKeyDown`方法判断按键是否被按下

```cpp
bool Keyboard::State::IsKeyDown(Keyboard::Keys key) const;
```

`Keyboard::State::IsKeyUp`方法判断按键是否没有按下

```cpp
bool Keyboard::State::IsKeyUp(Keyboard::Keys key) const;
```

下面演示的是键盘连续操作，其中dt是两帧之间的时间间隔

```cpp
if (keyState.IsKeyDown(Keyboard::W))
    cubePhi += dt * 2;
if (keyState.IsKeyDown(Keyboard::S))
    cubePhi -= dt * 2;
if (keyState.IsKeyDown(Keyboard::A))
    cubeTheta += dt * 2;
if (keyState.IsKeyDown(Keyboard::D))
    cubeTheta -= dt * 2;

mCBuffer.world = XMMatrixRotationY(cubeTheta) * XMMatrixRotationX(cubePhi);
```

> 注意：对于鼠标和键盘的拖动距离，推荐鼠标用偏移量，而推荐键盘用按压持续时间


## 键盘状态追踪

如果要判断按键是刚按下还是刚放开，则需要`Keyboard::KeyboardStateTracker`类帮助

`Keyboard::KeyboardStateTracker::Update`方法需要接受当前帧的State以进行更新：

```cpp
void Keyboard::KeyboardStateTracker::Update(const Keyboard::State& state);
```

然后就可以使用它的`IsKeyPressed`或`IsKeyReleased`方法来进行判断键盘按键是否刚按下，或者刚释放了，同样需要接受`Keyboard::Keys`枚举量

于是我们可以尝试让前面的立方体通过键盘或鼠标动起来。

# D3DApp::MsgProc方法的变化

这里展示了消息处理部分的变化：

```cpp
LRESULT D3DApp::MsgProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
        // 省略原有的部分...
        
        // 监测这些键盘/鼠标事件
    case WM_INPUT:

    case WM_LBUTTONDOWN:
    case WM_MBUTTONDOWN:
    case WM_RBUTTONDOWN:
    case WM_XBUTTONDOWN:

    case WM_LBUTTONUP:
    case WM_MBUTTONUP:
    case WM_RBUTTONUP:
    case WM_XBUTTONUP:

    case WM_MOUSEWHEEL:
    case WM_MOUSEHOVER:
    case WM_MOUSEMOVE:
        m_pMouse->ProcessMessage(msg, wParam, lParam);
        return 0;

    case WM_KEYDOWN:
    case WM_SYSKEYDOWN:
    case WM_KEYUP:
    case WM_SYSKEYUP:
        m_pKeyboard->ProcessMessage(msg, wParam, lParam);
        return 0;

    case WM_ACTIVATEAPP:
        m_pMouse->ProcessMessage(msg, wParam, lParam);
        m_pKeyboard->ProcessMessage(msg, wParam, lParam);
        return 0;
    }

    return DefWindowProc(hwnd, msg, wParam, lParam);
}
```

补充说明：当你打开`Mouse.cpp`和`Keyboard.cpp`查看源码的时候，大概率会遇到下面的报错：

![](..\assets\Mouse-And-Keyboard\01.png)

你可以直接无视该错误继续编译，即便到VS2019这个情况依然存在。

现在来看看当前章节对应的项目。该程序使用键盘的WSAD四个键控制立方体旋转，或者鼠标拖动旋转。

![](..\assets\Mouse-And-Keyboard\02.gif)

