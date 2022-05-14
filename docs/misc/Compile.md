# 前言

**本教程不考虑Effects11(FX11)，而是基于原始的HLSL。**

目前编译与加载着色器的方法如下：

1. 使用Visual Studio中的HLSL编译器，随项目编译期间一同编译，并生成`.cso`(Compiled Shader Object)对象文件，在运行期间加载该文件以读取字节码。
2. 使用Visual Studio中的HLSL编译器，随项目编译期间一同编译，并生成`.inc`或`.h`的头文件，着色器字节码在编译期间就可以确定。
3. 在程序运行期间编译着色器代码，并读取生成的字节码。

在个人的DX11项目中，使用的是方法1(优先)和方法3的混合形式。尽管方法2是最近了解到的，但个人目前并不考虑更换为该方法。

# 与着色器相关的文件扩展名

为了符合微软的约定，需要为你的着色器代码使用下面的扩展名(有所修改)：

1. 扩展名为`.hlsl`的文件用于编写HLSL的源代码，参与编译
2. 扩展名为`.hlsli`的文件作为HLSL的标头文件，不参与编译
3. 扩展名为`.cso`的文件作为已编译的着色器对象(Compiled Shader Object)
4. 扩展名为`.inc`或`.h`的文件是C++的头文件，但它的内部包含了着色器的字节码，使用`BYTE`数组来记录

# 方法1：编译期产生对象文件，并在运行期加载

现在以Rendering a Triangle项目为例，现在我们已经编写好的着色器文件有`Triangle.hlsli`, `Triangle_VS.hlsl`, `Triangle_PS.hlsl`这三个，它们存放项目在HLSL文件夹内。现在你可以将它拉进项目当中。

![](..\assets\Compile\04.png)

其中`Triangle.hlsli`作为HLSL的头文件默认不参与项目的编译过程。

而对于`Triangle_VS.hlsl`和`Triangle_PS.hlsl`，则在项目属性要这样设置：

![](..\assets\Compile\05.png)

![](..\assets\Compile\06.png)

其中入口点名称指的是该着色器阶段最先开始调用的函数名。比如在C/C++/新建的.hlsl文件中，默认的入口点名称是main。而上面的例子中，我们希望让顶点着色器从`VS`函数开始运行，则需要指定入口点为`VS`。

关于着色器模型，因为假定用户的显卡已经支持特性等级11.0，这里使用的是`Shader Model 5.0`，如果你的显卡不支持特性等级11.0，则需要将特性等级降为10.0/10.1，分别对应能使用的着色器模型为4.0/4.1

生成项目后，需要留意在输出窗口(生成)中是否出现了下面的内容：

![](..\assets\Compile\07.png)

**只有出现了上述内容，才说明成功编译出对象文件，否则说明没有被编译出来。**如果你之前已经编译出对象文件，再编译时没有出现该输出结果，可能需要先删除之前编译出来的对象文件再试一次。

## D3DReadFileToBlob函数--读取编译好的着色器二进制信息

对着色器代码或文件的相关操作位于头文件`d3dcompiler.h`，而且你还需要添加静态库`d3dcompiler.lib`

接下来，我们使用下面的函数来读取编译好的着色器二进制信息：

```cpp
HRESULT D3DReadFileToBlob(LPCWSTR pFileName,    // [In].cso文件名
                  ID3DBlob** ppContents);       // [Out]获取二进制大数据块
```

**注意：如果你的项目中不存在该函数，说明你可能预先包含了DX SDK，然而该教程使用的是Windows SDK，该函数位于D3DCompiler >= 46的版本，因此你需要剔除DX SDK的包含路径和库路径。**

使用方式也十分简单(以创建顶点着色器和顶点布局为例)：

```cpp
ComPtr<ID3DBlob> blob;
HR(D3DReadFileToBlob(L"HLSL\\Triangle_VS.cso", blob.GetAddressOf()));
HR(md3dDevice->CreateVertexShader(blob->GetBufferPointer(), blob->GetBufferSize(), nullptr, m_pVertexShader.GetAddressOf()));
// 创建顶点布局
HR(md3dDevice->CreateInputLayout(VertexPosColor::inputLayout, ARRAYSIZE(VertexPosColor::inputLayout),
    blob->GetBufferPointer(), blob->GetBufferSize(), m_pVertexLayout.GetAddressOf()));
```

然后就可以拿获取到的`ID3DBlob`来创建着色器了。创建着色器和顶点布局的部分在本文不进行讨论，请回到教程02继续查看。

该方法的特点是会在你的项目文件夹中产生编译好的着色器二进制文件，并且需要你在程序运行的时候直接读进来。

# 方法2：编译器产生头文件，并在项目中包含该文件

对于`Triangle_VS.hlsl`和`Triangle_PS.hlsl`，在项目属性要这样设置：

![](..\assets\Compile\01.png)

![](..\assets\Compile\02.png)

这里关于头文件的名称以及内部的全局变量名可以自行决定。

头文件 经过编译后会在HLSL文件夹产生`Triangle_VS.inc`和`Triangle_PS.inc`两个文件，观察里面的代码你可以发现里面有汇编部分(不会包含进代码中)和一个全局变量，在`Triangle_VS.inc`中产生的是全局变量`gTriangle_VS`，而在`Triangle_PS.inc`中产生的是全局变量`gTriangle_PS`。这两个变量都是`BYTE`数组，里面的内容正是编译好的字节码。

现在需要在你需要编写创建着色器相关代码的源文件上面包含这两个头文件：

```cpp
#include "HLSL/Triangle_VS.inc"
#include "HLSL/Triangle_PS.inc"
```

然后创建顶点着色器和顶点布局的代码变成了这样：

```cpp
// 创建顶点着色器
HR(m_pd3dDevice->CreateVertexShader(gTriangle_VS, sizeof(gTriangle_VS), nullptr, m_pVertexShader.GetAddressOf()));
// 创建并绑定顶点布局
HR(m_pd3dDevice->CreateInputLayout(VertexPosColor::inputLayout, ARRAYSIZE(VertexPosColor::inputLayout),
    gTriangle_VS, sizeof(gTriangle_VS), m_pVertexLayout.GetAddressOf()));
```

接下来就可以生成整个项目了，需要留意是否有红色部分的输出，否则可能没有成功编译出`.inc`文件(这可能会在已有`.inc`文件再次编译的时候导致出现问题，需要删除原来的`.inc`文件)。

![](..\assets\Compile\03.png)

由于上述两个头文件的产生(即着色器的编译)先于项目的编译，在没有产生这两个头文件的时候，你也可以忍着编译错误先把上述代码添加进去，然后编译的时候就一切正常了。

该方法的特点是所有的过程均在编译期完成，着色器字节码镶嵌在了你的应用程序内部，可能会导致应用程序变大。


# 方法3：运行期间编译着色器代码，生成字节码

现在你需要了解这些函数

## D3DCompileFromFile函数--运行期编译.hlsl文件

```cpp
HRESULT D3DCompileFromFile(
    LPCWSTR pFileName,                  // [In]要编译的.hlsl文件
    CONST D3D_SHADER_MACRO* pDefines,   // [In_Opt]忽略
    ID3DInclude* pInclude,              // [In_Opt]如何应对#include宏
    LPCSTR pEntrypoint,                 // [In]入口函数名
    LPCSTR pTarget,                     // [In]使用的着色器模型
    UINT Flags1,                        // [In]D3DCOMPILE系列宏
    UINT Flags2,                        // [In]D3DCOMPILE_FLAGS2系列宏
    ID3DBlob** ppCode,                  // [Out]获得着色器的二进制块
    ID3DBlob** ppErrorMsgs);            // [Out]可能会获得错误信息的二进制块
```

**再次注意：如果你的项目中不存在该函数，说明你可能预先包含了DX SDK，然而该教程使用的是Windows SDK，该函数位于D3DCompiler >= 46的版本，因此你需要剔除DX SDK的包含路径和库路径。**

其中`pInclude`用于决定如何处理包含文件。如果设为`nullptr`，则编译的着色器代码包含`#include`时会引发编译器报错。如果你需要使用`#include`，可以传递`D3D_COMPILE_STANDARD_FILE_INCLUDE`宏，这是一个默认的包含句柄，可以按该着色器代码所处的相对路径去搜索对应的头文件并包含进来。

```cpp
#define D3D_COMPILE_STANDARD_FILE_INCLUDE ((ID3DInclude*)(UINT_PTR)1)
```

## D3DWriteBlobToFile函数--将编译好的着色器二进制信息写入文件

```cpp
HRESULT D3DWriteBlobToFile(
    ID3DBlob* pBlob,    // [In]编译好的着色器二进制块
    LPCWSTR pFileName,  // [In]输出文件名
    BOOL bOverwrite);   // [In]是否允许覆盖
```

对于`bOverwrite`来说，无论是`TRUE`还是`FALSE`都无关紧要，因为我们只有在检测到没有编译好的着色器文件时才会启动运行期编译，然后再保存到文件。

具体用法已经集成在下面的`CreateShaderFromFile`函数中了

## CreateShaderFromFile函数的实现

下面是`CreateShaderFromFile`函数的实现，**现在该函数已经放到了d3dUtil.h中**：

```cpp
// 安全COM组件释放宏
#define SAFE_RELEASE(p) { if ((p)) { (p)->Release(); (p) = nullptr; } }

// ------------------------------
// CreateShaderFromFile函数
// ------------------------------
// [In]csoFileNameInOut 编译好的着色器二进制文件(.cso)，若有指定则优先寻找该文件并读取
// [In]hlslFileName     着色器代码，若未找到着色器二进制文件则编译着色器代码
// [In]entryPoint       入口点(指定开始的函数)
// [In]shaderModel      着色器模型，格式为"*s_5_0"，*可以为c,d,g,h,p,v之一
// [Out]ppBlobOut       输出着色器二进制信息
HRESULT CreateShaderFromFile(const WCHAR * csoFileNameInOut, const WCHAR * hlslFileName,
    LPCSTR entryPoint, LPCSTR shaderModel, ID3DBlob ** ppBlobOut)
{
    HRESULT hr = S_OK;

    // 寻找是否有已经编译好的顶点着色器
    if (csoFileNameInOut && D3DReadFileToBlob(csoFileNameInOut, ppBlobOut) == S_OK)
    {
        return hr;
    }
    else
    {
        DWORD dwShaderFlags = D3DCOMPILE_ENABLE_STRICTNESS;
#ifdef _DEBUG
        // 设置 D3DCOMPILE_DEBUG 标志用于获取着色器调试信息。该标志可以提升调试体验，
        // 但仍然允许着色器进行优化操作
        dwShaderFlags |= D3DCOMPILE_DEBUG;

        // 在Debug环境下禁用优化以避免出现一些不合理的情况
        dwShaderFlags |= D3DCOMPILE_SKIP_OPTIMIZATION;
#endif
        ID3DBlob* errorBlob = nullptr;
        hr = D3DCompileFromFile(hlslFileName, nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE, entryPoint, shaderModel,
            dwShaderFlags, 0, ppBlobOut, &errorBlob);
        if (FAILED(hr))
        {
            if (errorBlob != nullptr)
            {
                OutputDebugStringA(reinterpret_cast<const char*>(errorBlob->GetBufferPointer()));
            }
            SAFE_RELEASE(errorBlob);
            return hr;
        }

        // 若指定了输出文件名，则将着色器二进制信息输出
        if (csoFileNameInOut)
        {
            return D3DWriteBlobToFile(*ppBlobOut, csoFileNameInOut, FALSE);
        }
    }

    return hr;
}
```

使用方式如下：

```cpp
ComPtr<ID3DBlob> blob;

// 创建顶点着色器
HR(CreateShaderFromFile(L"HLSL\\Triangle_VS.cso", L"HLSL\\Triangle_VS.hlsl", "VS", "vs_5_0", blob.ReleaseAndGetAddressOf()));
HR(m_pd3dDevice->CreateVertexShader(blob->GetBufferPointer(), blob->GetBufferSize(), nullptr, m_pVertexShader.GetAddressOf()));
// 创建并绑定顶点布局
HR(m_pd3dDevice->CreateInputLayout(VertexPosColor::inputLayout, ARRAYSIZE(VertexPosColor::inputLayout),
    blob->GetBufferPointer(), blob->GetBufferSize(), m_pVertexLayout.GetAddressOf()));
```

# 方法4：使用fxc命令行程序+编写批处理+利用VS项目预先编译

>  **注意：初学者只需要会用方法1或方法3来编译着色器即可，这条需要熟悉命令行，能尝试编写批处理，并且可能会修改到.vcxproj。**

因为我一直挺好奇DirectXTK是怎么做到没有在项目属性中添加配置却能够在编译程序之前先调用命令行程序编译着色器，现在大概是找到门路了。

## fxc命令行程序

首先了解怎么使用fxc命令行程序来编译着色器，找到64位fxc.exe，有可能放在`C:\Program Files (x86)\Windows Kits\10\bin\10.0.XXXXX.0\x64\`，或者利用Everything等工具来寻找。

打开命令行，跳转到fxc.exe所在位置，然后运行：

```bat
fxc /?
```

然后可以看到fxc编译选项的含义，下面列出一些常用的：

| 选项             | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| `/T <profile>`   | 着色器模型，常用的`<profile>`有：`cs_4_0`, `vs_5_0`, `ps_5_0`等 |
| `/E <name>`      | 入口点名称，以哪个函数名作为shader的入口                     |
| `/I <include>`   | 附加包含路径                                                 |
| `/Od`            | 禁用优化（调试着色器必须）                                   |
| `/O{0,1,2,3}`    | 优化级别0到3，默认是1                                        |
| `/Zi`            | 启用调试信息（调试着色器必须）                               |
| `/Fo <file>`     | 输出编译好的二进制对象文件（通常指cso文件）                  |
| `/Fh <file>`     | 输出包含二进制对象编码的头文件（如方法二）                   |
| `/P <file>`      | 将预处理阶段后的文本保存到文件（必须单独使用）               |
| `/D <id>=<text>` | 预定义宏                                                     |
| `/Ges`           | 强制严格编译，可能不允许使用旧语法。                         |
| `/nologo`        | 编译的时候不输出版权信息                                     |
| `/WX`            | 警告视为错误                                                 |

以第二章的HLSL文件为例，编译`Triangle_VS.hlsl`输出`Triangle_VS.cso`

在Release模式下，与方法1等价的命令行为：

``` bat
fxc "path/to/Triangle_VS.hlsl" /E VS /Fo "path/to/Triangle_VS.cso" /T vs_5_0 /nologo
```

在Debug模式下，与方法1等价的命令行为：

```bat
fxc "path/to/Triangle_VS.hlsl" /Zi /Od /E VS /Fo "path/to/Triangle_VS.cso" /T vs_5_0 /nologo
```

而前面提到的`D3DCOMPILE_ENABLE_STRICTNESS`对应`/Ges`选项

## 批处理



# 参考文献

[Compiling Shaders](https://docs.microsoft.com/zh-cn/windows/desktop/direct3dhlsl/dx-graphics-hlsl-part1#compiling-with-d3dcompilefromfile)

[How To: Compile a Shader](https://docs.microsoft.com/zh-cn/windows/desktop/direct3d11/how-to--compile-a-shader)

