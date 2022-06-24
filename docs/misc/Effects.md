</br>
</br>

# 前言

如果之前你是跟随本教程系列学习的话，应该能够初步了解[Effects11(现FX11)](https://github.com/Microsoft/FX11)的实现机制，并且可以[编写一个简易的特效管理框架](https://www.cnblogs.com/X-Jun/p/9665452.html)，但是随着特效种类的增多，要管理的着色器、资源等也随之变多。如果写了一套由多个HLSL着色器组成特效，就仍需要在C++端编写与HLSL相对应的特效框架，这样写起来依然是十分繁杂。以前学习龙书的DirectX11时，里面使用的正是Effects11框架，不得不承认用它实现C++跟HLSL的交互的确方便了许多，但是时过境迁，微软将会逐渐抛弃fx_5_0，且目前FX11也已经列为Archived，不再更新。都说如果要实现一个3D引擎的话，必须要有一个属于自己的特效管理框架。

本文假定读者已经读过至少前13章的内容，或者有较为丰富的DirectX 11开发经历。

学习目标：

1. **熟悉着色器反射机制**
2. **实现一个复杂Effects框架，了解该框架的使用**

# 先从Effects11(FX11)谈起

DirectX的特效是包含管线状态和着色器的集合，而Effects框架则正是用于管理这些特效的一套API。如果使用Effects11(FX11)框架的话，那么在HLSL中除了本身的语法外，还支持Effects特有的语法，这些语法大部分经过解析后会转化为在C++中使用Direct3D的API。

知己知彼，才能百战不殆。要想写好一个特效管理框架，首先要把Effects框架与C++的关系给分析透彻。下面的内容也会引用FX11的少量源码来佐证。



## Pass、Technique11、Group

**Pass**：一个Pass由一组需要用到的着色器和一些渲染状态组成。通常情况下，我们至少需要一个顶点着色器和一个像素着色器。如果是要进行流输出，则至少需要一个顶点着色器和一个几何着色器。而通用计算则需要的是计算着色器。除此之外，它在HLSL还支持一些额外的函数，用以改变一些渲染状态。

**Technique11**：一个Technique由一个或多个Pass组成，用于创建一个渲染技术。有时候为了实现一种特效，需要历经多个Pass的处理才能实现，我们称之为**多通道渲染**。比如实现OIT（顺序无关透明度），第一趟Pass需要完成透明像素的收集，第二趟Pass则是将收集好的像素按深度排序，并将透明混合的结果渲染到目标。

**Group**：一个Group由一个或多个Technique组成。

下面展示了一份比较随性的fx5.0代码的部分**（注意：下面的代码不属于HLSL的语法！）**：

```cpp
// 存在部分省略

GeometryShader pGSComp = CompileShader(gs_5_0, gsBase());
GeometryShader pGSwSO = ConstructGSWithSO(pGSComp, "0:Position.xy; 1:Position.zw; 2:Color.xy", 
                                                   "3:Texcoord.xyzw; 3:$SKIP.x;", NULL, NULL, 1);

// 此处省略着色器函数...

technique11 T0
{
    pass P0
    {
        SetVertexShader(CompileShader(vs_5_0, VS()));
        SetGeometryShader(NULL);
        SetPixelShader(CompileShader(ps_5_0, PS(true, false, true)));

        SetRasterizerState(g_NoCulling);
        SetDepthStencilState(NULL, 0);
        SetBlendState(EnableAlphaBlending, (float4)0, 0xFFFFFFFF);
    }
    
    Pass P1
    {
        SetVertexShader(CompileShader(vs_5_0, VS()));
        SetGeometryShader(pGSwSO);
        SetPixelShader(NULL);
    }
}

```



这里面的函数调用大部分实际上都是在C++完成的，因此在Direct3D API中可以找到对应的原型：

```cpp
SetVertexShader()	// 等价于ID3D11DeviceContext::VSSetShader
SetGeometryShader()	// 等价于ID3D11DeviceContext::GSSetShader
SetPixelShader()	// 等价于ID3D11DeviceContext::PSSetShader

SetRasterizerState()	// 等价于ID3D11DeviceContext::RSSetState
SetDepthStencilState()	// 等价于ID3D11DeviceContext::OMSetDepthStencilState
SetBlendState()			// 等价于ID3D11DeviceContext::OMSetBlendState

ConstructGSWithSO()		// 等价于ID3D11Device::CreateGeometryShaderWithStreamOutput
```

而像`VertexShader`、`PixelShader`这些仅存在于fx5.0的语法，在C++中对应的是`ID3D11VertexShader`、`ID3D11PixelShader`等等。

至于`CompileShader`，我们可以猜测内部使用的是类似`D3DCompile`这样的函数，只不过这份源码肯定是需要经过特殊处理才能变成原生的HLSL代码。

在C++端，编译fx5.0可以使用[**D3DCompile**](https://docs.microsoft.com/en-us/windows/win32/api/d3dcompiler/nf-d3dcompiler-d3dcompile)或[**D3DCompileFromFile**](https://docs.microsoft.com/zh-cn/windows/win32/api/d3dcompiler/nf-d3dcompiler-d3dcompilefromfile)，然后再使用[**D3DX11CreateEffectFromMemory**](https://docs.microsoft.com/zh-cn/windows/win32/direct3d11/d3dx11createeffectfrommemory)创建出Effects。只不过会收到这样的警告：

`X4717: Effects deprecated for D3DCompiler_47`

## 渲染状态、采样器状态

在fx5.0中能够创建出`SamplerState`、`RasterizerState`、`BlendState`和`DepthStencilState`，并且还能预先设置好内部的各项参数，就像下面这样**（注意：下面的代码不属于HLSL的语法！）**：

```cpp
SamplerState g_SamAnisotropic
{
	Filter = ANISOTROPIC;
	MaxAnisotropy = 4;

	AddressU = WRAP;
	AddressV = WRAP;
	AddressW = WRAP;
};

RasterizerState g_NoCulling
{
    FillMode = Solid;
    CullMode = None;
    FrontCounterClockwise = false;
}
```

实际上，采样器的状态和渲染状态都是在C++中完成的，上面的代码翻译成C++则变成类似这样：

```cpp
// g_SamAnisotropic
CD3D11_SAMPLER_DESC sampDesc(CD3D11_DEFAULT());
sampDesc.Filter = D3D11_FILTER_ANISOTROPIC;
sampDesc.MaxAnisotropy = 4;
sampDesc.AddressU = D3D11_TEXTURE_ADDRESS_WRAP;
sampDesc.AddressV = D3D11_TEXTURE_ADDRESS_WRAP;
sampDesc.AddressW = D3D11_TEXTURE_ADDRESS_WRAP;
device->CreateSamplerState(&sampDesc, SSAnistropicWrap.GetAddressOf());

// g_NoCulling
CD3D11_RASTERIZER_DESC rasterizerDesc(CD3D11_DEFAULT());
rasterizerDesc.FillMode = D3D11_FILL_SOLID;
rasterizerDesc.CullMode = D3D11_CULL_NONE;
rasterizerDesc.FrontCounterClockwise = false;
device->CreateRasterizerState(&rasterizerDesc, RSNoCull.GetAddressOf()));
```



## 常量缓冲区

以前在用fx5.0写常量缓冲区的时候是这样的：

```cpp
cbuffer cbPerFrame
{
	DirectionalLight gDirLights[3];
	float3 gEyePosW;

	float  gFogStart;
	float  gFogRange;
	float4 gFogColor;
};

cbuffer cbPerObject
{
	float4x4 gWorld;
	float4x4 gWorldInvTranspose;
	float4x4 gWorldViewProj;
	float4x4 gTexTransform;
	Material gMaterial;
}; 
```

在你声明了cbuffer后，Effects11(FX11)会在C++端创建出对应的常量缓冲区：

```cpp
D3D11_BUFFER_DESC cbd;
ZeroMemory(&cbd, sizeof(cbd));
cbd.Usage = D3D11_USAGE_DYNAMIC;	// FX11内部使用的是D3D11_USAGE_DEFAULT
cbd.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
cbd.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;	// FX11内部是0
cbd.ByteWidth = byteWidth;
return device->CreateBuffer(&cbd, nullptr, cBuffer.GetAddressOf());
```

### 隐藏指定寄存器槽的问题

已知常量缓冲区有16个寄存器槽，那么，怎么确定cbuffer当前使用的是哪个槽呢？

1. 有通过register(b#)指定寄存器槽位的`cbuffer`优先占用
2. 除去那些显式指定槽位的`cbuffer`，如果`cbuffer`里面的成员有被当前着色器使用过，将会根据声明顺序按空余槽位从小到大的顺序占用

根据上面的例子，cbPerFrame将使用slot(b0)，而cbPerObject将使用slot(b1)。

现在让我们省略所有的花括号，观察下面的代码，根据下面两种情况，问那三个未指定寄存器槽的cbuffer分别占用了哪个slot？

1. 顶点着色器使用过第1、3、4、5个cbuffer里面的变量
2. 像素着色器使用过第2、3、4、6个cbuffer里面的变量

```cpp
cbuffer CBChangesEveryInstanceDrawing : register(b0) { ... }
cbuffer CBChangesEveryObjectDrawing { ... }
cbuffer CBChangesEveryFrame { ... }
cbuffer CBDrawingStates { ... }
cbuffer CBChangesOnResize : register(b2) { ... }
cbuffer CBChangesRarely : register(b3) { ... }
```

答案如下：

1. CBChangesEveryFrame占用了slot(b1)，CBDrawingStates占用了slot(b4)
2. CBChangesEveryObjectDrawing占用了slot(b1)，CBChangesEveryFrame占用了slot(b4)，CBDrawingStates占用了slot(b5)

不仅是寄存器槽cb#，其余的如t#、u#、s#等也是一样的道理。

**只要当前资源没有标定寄存器槽，并且没有被着色器使用过，编译后它们不会占用寄存器槽。**

### 常量缓冲区的更新

在Effects11的C++端创建了常量缓冲区的同时，还会创建一份与cbuffer等大的内存副本，这么做是为了减少常量缓冲区的更新次数（即CPU→GPU的写入）。并且每个副本还要设置一个脏标记，即只有在数据发生变化的时候才会进行实际的提交。

在Effects11中，更新常量初值的方式如下：

```cpp
m_pFX->GetVariableByName("gWorld")->AsMatrix()->SetMatrix((float*)&M);
```

这里实际上就是更新所属常量缓冲区的内存副本中`gWorld`所属的内存区域，然后将脏标记设置为`true`。

所有的更新结束后，通过调用`ID3DX11EffectPass::Apply`来执行实际的常量缓冲区更新：

```cpp
m_pTech->GetPassByIndex(p)->Apply(0, m_pd3dImmediateContext);
```

在完成更新后，Apply便会将常量缓冲区绑定到渲染管线上，例如执行下面的语句：

```cpp
m_pd3dImmediateContext->VSSetConstantBuffers(0, 1, &pCB->pD3DObject);
```

不仅是常量缓冲区，Apply操作还会绑定着色器、着色器资源（SRV）、可读写资源（UAV）、采样器、各种渲染状态等。

翻看FX11的源码，我们可以找到更新常量缓冲区的地方。该函数会在Apply后调用：

```cpp
inline void CheckAndUpdateCB_FX(ID3D11DeviceContext *pContext, SConstantBuffer *pCB)
{
    if (pCB->IsDirty && !pCB->IsNonUpdatable)
    {
        // CB out of date; rebuild it
        pContext->UpdateSubresource(pCB->pD3DObject, 0, nullptr, pCB->pBackingStore, pCB->Size, pCB->Size);
        pCB->IsDirty = false;
    }
}
```

当然，如果cbuffer用的是DYNAMIC更新，则需要改为Map与UnMap的更新方式。



### 默认常量缓冲区(Default Constant Buffer)

如果一个变量没有`static`或`const`修饰符，那么编译器将会认为它是属于名为`$Globals`的默认常量缓冲区的一员。类似的，着色器入口点的`uniform`形参将会被认为是属于另一个名为`$Params`的默认常量缓冲区。

考虑下面一段代码：

```cpp
uniform bool g_FogEnable;	// 属于$Gbloals

cbuffer CB0 : register(b0) { ... }
cbuffer CB1 : register(b1) { ... }
cbuffer CB2 { ... }

float4 PS(
    PIN pin, 
	uniform int numLights /* 属于$Params */
) : SV_Target
{
    ...
}
```

对于常量缓冲区槽位的安排，最终会按如下顺序安排：

1. 有指定寄存器槽位的`cbuffer`优先占用
2. 其次是`$Globals`占用空余槽位中值最小的那个
3. 紧接着`$Params`占用空余槽位中最小的那个
4. 剩余有被该着色器使用的`cbuffer`按空余槽位从小到大的顺序占用

因此，编译器会这样解释：

```cpp
cbuffer CB0 : register(b0) { ... }
cbuffer CB1 : register(b1) { ... }
cbuffer $Globals : register(b2) { bool g_FogEnable; }
cbuffer $Params : register(b3) { int numLights; }
cbuffer CB2 : register(b4) { ... }
```

当然，直接声明`$Globals`或`Globals`是不可能编译通过的。

这就能解释的通，为什么我们在编译HLSL代码时，b#的最大值只能到13（即我们只能指定14个自定义的常量缓冲区），但在头文件`d3d11.h`却又说有16个寄存器槽位了。因为剩余的两个槽位要让位于`$Globals`和`$Params`这两个默认常量缓冲区。

# 着色器反射

编译好的着色器二进制数据中蕴含着丰富的信息，我们可以通过着色器反射机制来获取自己所需要的东西，然后构建一个属于自己的Effects类。

## D3DReflect函数--获取着色器反射对象

在调用该函数之前需要使用`D3DCompile`或`D3DCompileFromFile`产生编译好的着色器二进制对象`ID3DBlob`：

```cpp
HRESULT D3DReflect(
	LPCVOID pSrcData,		// [In]编译好的着色器二进制信息
	SIZE_T  SrcDataSize,	// [In]编译好的着色器二进制信息字节数
	REFIID  pInterface,		// [In]COM组件的GUID
	void    **ppReflector	// [Out]输出的着色器反射借口
);
```

其中`pInterface`为`__uuidof(ID3D11ShaderReflection)`时，返回的是`ID3D11ShaderReflection`接口对象；而`pInterface`为`__uuidof(ID3D12ShaderReflection)`时，返回的是`ID3D12ShaderReflection`接口对象。

`ID3D11ShaderReflection`提供了大量的方法给我们获取信息，其中我们比较感兴趣的主要信息有：

1. 着色器本身的信息
2. 常量缓冲区的信息
3. 采样器、资源的信息

## D3D11_SHADER_DESC结构体--着色器本身的信息

通过方法`ID3D11ShaderReflection::GetDesc`，我们可以获取到`D3D11_SHADER_DESC`对象。这里面包含了大量的基础信息：

```cpp
typedef struct _D3D11_SHADER_DESC {
  UINT                             Version;						// 着色器版本、类型信息
  LPCSTR                           Creator;						// 是谁创建的着色器
  UINT                             Flags;						// 着色器编译/分析标签
  UINT                             ConstantBuffers;				// 实际使用到常量缓冲区数目
  UINT                             BoundResources;				// 实际用到绑定的资源数目
  UINT                             InputParameters;				// 输入参数数目(4x4矩阵为4个向量形参)
  UINT                             OutputParameters;			// 输出参数数目
  UINT                             InstructionCount;			// 指令数
  UINT                             TempRegisterCount;			// 实际使用到的临时寄存器数目
  UINT                             TempArrayCount;				// 实际用到的临时数组数目
  UINT                             DefCount;					// 常量定义数目
  UINT                             DclCount;					// 声明数目(输入+输出)
  UINT                             TextureNormalInstructions;	// 未分类的纹理指令数目
  UINT                             TextureLoadInstructions;		// 纹理读取指令数目
  UINT                             TextureCompInstructions;		// 纹理比较指令数目
  UINT                             TextureBiasInstructions;		// 纹理偏移指令数目
  UINT                             TextureGradientInstructions;	// 纹理梯度指令数目
  UINT                             FloatInstructionCount;		// 实际用到的浮点数指令数目
  UINT                             IntInstructionCount;			// 实际用到的有符号整数指令数目
  UINT                             UintInstructionCount;		// 实际用到的无符号整数指令数目
  UINT                             StaticFlowControlCount;		// 实际用到的静态流控制指令数目
  UINT                             DynamicFlowControlCount;		// 实际用到的动态流控制指令数目
  UINT                             MacroInstructionCount;		// 实际用到的宏指令数目
  UINT                             ArrayInstructionCount;		// 实际用到的数组指令数目
  UINT                             CutInstructionCount;			// 实际用到的cut指令数目
  UINT                             EmitInstructionCount;		// 实际用到的emit指令数目
  D3D_PRIMITIVE_TOPOLOGY           GSOutputTopology;			// 几何着色器的输出图元
  UINT                             GSMaxOutputVertexCount;		// 几何着色器的最大顶点输出数目
  D3D_PRIMITIVE                    InputPrimitive;				// 输入装配阶段的图元
  UINT                             PatchConstantParameters;		// 待填坑...
  UINT                             cGSInstanceCount;			// 几何着色器的实例数目
  UINT                             cControlPoints;				// 域着色器和外壳着色器的控制点数目
  D3D_TESSELLATOR_OUTPUT_PRIMITIVE HSOutputPrimitive;			// 镶嵌器输出的图元类型
  D3D_TESSELLATOR_PARTITIONING     HSPartitioning;				// 待填坑...
  D3D_TESSELLATOR_DOMAIN           TessellatorDomain;			// 待填坑...
  UINT                             cBarrierInstructions;		// 计算着色器内存屏障指令数目
  UINT                             cInterlockedInstructions;	// 计算着色器原子操作指令数目
  UINT                             cTextureStoreInstructions;	// 计算着色器纹理写入次数
} D3D11_SHADER_DESC;
```

其中，成员`Version`不仅包含了着色器版本，还包含着色器类型。下面的枚举值定义了着色器的类型，并通过宏`D3D11_SHVER_GET_TYPE`来获取：

```cpp
typedef enum D3D11_SHADER_VERSION_TYPE
{
    D3D11_SHVER_PIXEL_SHADER    = 0,
    D3D11_SHVER_VERTEX_SHADER   = 1,
    D3D11_SHVER_GEOMETRY_SHADER = 2,
    
    // D3D11 Shaders
    D3D11_SHVER_HULL_SHADER     = 3,
    D3D11_SHVER_DOMAIN_SHADER   = 4,
    D3D11_SHVER_COMPUTE_SHADER  = 5,

    D3D11_SHVER_RESERVED0       = 0xFFF0,
} D3D11_SHADER_VERSION_TYPE;

#define D3D11_SHVER_GET_TYPE(_Version) \
    (((_Version) >> 16) & 0xffff)
```

即：

```cpp
auto shaderType = static_cast<D3D11_SHADER_VERSION_TYPE>(D3D11_SHVER_GET_TYPE(sd.Version));
```

## D3D11_SHADER_INPUT_BIND_DESC结构体--描述着色器资源如何绑定到着色器输入

为了获取着色器程序内声明的一切给着色器使用的对象，从这个结构体入手是一种十分不错的选择。我们将使用`ID3D11ShaderReflection::GetResourceBindingDesc`方法，和枚举显示适配器那样从索引0开始枚举一样的做法，只要当前的索引值获取失败，说明已经获取完所有的输入对象：

```cpp
for (UINT i = 0;; ++i)
{
	D3D11_SHADER_INPUT_BIND_DESC sibDesc;
	hr = pShaderReflection->GetResourceBindingDesc(i, &sibDesc);
	// 读取完变量后会失败，但这并不是失败的调用
	if (FAILED(hr))
		break;
    
    // 根据sibDesc继续分析...
}
```

> 注意：那些在着色器代码中从未被当前着色器使用过的资源将不会被枚举出来，并且在着色器调试和着色器反射的时候看不到它们，而反汇编中也许能够看到该变量被标记为unused。

现在先来看该结构体的成员：

```cpp
typedef struct _D3D11_SHADER_INPUT_BIND_DESC {
	LPCSTR                   Name;			// 着色器资源名
	D3D_SHADER_INPUT_TYPE    Type;			// 资源类型
	UINT                     BindPoint;		// 指定的输入槽起始位置
	UINT                     BindCount;		// 对于数组而言，占用了多少个槽
	UINT                     uFlags;		// D3D_SHADER_INPUT_FLAGS枚举复合
	D3D_RESOURCE_RETURN_TYPE ReturnType;	// 
	D3D_SRV_DIMENSION        Dimension;		// 着色器资源类型
	UINT                     NumSamples;	// 若为纹理，则为MSAA采样数，否则为0xFFFFFFFF
} D3D11_SHADER_INPUT_BIND_DESC;
```

其中成员`Name`帮助我们使用着色器反射按名获取资源，而成员`Type`帮助我们确定资源类型。这两个成员一旦确定下来，对我们开展更详细的着色器反射和实现自己的特效框架提供了巨大的帮助。具体枚举如下：

```cpp
typedef enum _D3D_SHADER_INPUT_TYPE {
  D3D_SIT_CBUFFER,
  D3D_SIT_TBUFFER,
  D3D_SIT_TEXTURE,
  D3D_SIT_SAMPLER,
  D3D_SIT_UAV_RWTYPED,
  D3D_SIT_STRUCTURED,
  D3D_SIT_UAV_RWSTRUCTURED,
  D3D_SIT_BYTEADDRESS,
  D3D_SIT_UAV_RWBYTEADDRESS,
  D3D_SIT_UAV_APPEND_STRUCTURED,
  D3D_SIT_UAV_CONSUME_STRUCTURED,
  D3D_SIT_UAV_RWSTRUCTURED_WITH_COUNTER,
  // ...
} D3D_SHADER_INPUT_TYPE;
```

根据上述枚举可以分为常量缓冲区、采样器、着色器资源、可读写资源四大类。对于采样器、着色器资源和可读写资源我们只需要知道它设置在哪个slot即可，但对于常量缓冲区，我们还需要知道其内部的成员和位于哪一段内存区域。

## D3D11_SHADER_BUFFER_DESC结构体--描述一个着色器的常量缓冲区

在通过上面提到的枚举值判定出来是常量缓冲区后，我们就可以通过`ID3D11ShaderReflection::GetConstantBufferByName`迅速拿下常量缓冲区的反射，然后再获取`D3D11_SHADER_BUFFER_DESC`的信息：

```cpp
ID3D11ShaderReflectionConstantBuffer* pSRCBuffer = pShaderReflection->GetConstantBufferByName(sibDesc.Name);
// 获取cbuffer内的变量信息并建立映射
D3D11_SHADER_BUFFER_DESC cbDesc{};
hr = pSRCBuffer->GetDesc(&cbDesc);
if (FAILED(hr))
	return hr;
```

>  注意：ID3D11ShaderReflectionConstantBuffer并不是COM组件，因此不能用ComPtr存放。

该结构体定义如下：

```cpp
typedef struct _D3D11_SHADER_BUFFER_DESC {
	LPCSTR           Name;		// 常量缓冲区名称
	D3D_CBUFFER_TYPE Type;		// D3D_CBUFFER_TYPE枚举值
	UINT             Variables;	// 内部变量数目
	UINT             Size;		// 缓冲区字节数
	UINT             uFlags;	// D3D_SHADER_CBUFFER_FLAGS枚举复合
} D3D11_SHADER_BUFFER_DESC;
```

根据成员`Variables`，我们就可以确定查询变量的次数。

## D3D11_SHADER_VARIABLE_DESC结构体--描述一个着色器的变量

虽然有点想吐槽，常量缓冲区里面存的是变量这个说法，但还是得这样来看待：常量缓冲区内的数据是可以改变的，但是在着色器运行的时候，`cbuffer`内的任何变量就不可以被修改了。因此对C++来说，它是可变量，但对着色器来说，它是常量。

好了不扯那么多，现在我们用这样一个循环，通过`ID3D11ShaderReflectionVariable::GetVariableByIndex`来逐一枚举着色器变量的反射，然后获取`D3D11_SHADER_VARIABLE_DESC`的信息：

```cpp
// 记录内部变量
for (UINT j = 0; j < cbDesc.Variables; ++j)
{
    ID3D11ShaderReflectionVariable* pSRVar = pSRCBuffer->GetVariableByIndex(j);
    D3D11_SHADER_VARIABLE_DESC svDesc;
    hr = pSRVar->GetDesc(&svDesc);
    if (FAILED(hr))
        return hr;
    // ...
}
```

`ID3D11ShaderReflectionVariable`不是COM组件，因此无需管释放。

那么`D3D11_SHADER_VARIABLE_DESC`的定义如下：

```cpp
typedef struct _D3D11_SHADER_VARIABLE_DESC {
	LPCSTR Name;			// 变量名
	UINT   StartOffset;		// 起始偏移
	UINT   Size;			// 大小
	UINT   uFlags;			// D3D_SHADER_VARIABLE_FLAGS枚举复合
	LPVOID DefaultValue;	// 用于初始化变量的默认值
	UINT   StartTexture;	// 从变量开始到纹理开始的偏移量[看不懂]
	UINT   TextureSize;		// 纹理字节大小
	UINT   StartSampler;	// 从变量开始到采样器开始的偏移量[看不懂]
	UINT   SamplerSize;		// 采样器字节大小
} D3D11_SHADER_VARIABLE_DESC;
```

其中前三个参数是我们需要的，由此我们就可以构建出根据变量名来设置值和获取值的一套方案。

讲到这里其实已经满足了我们构建一个最小特效管理类的需求。但你如果想要获得更详细的变量信息，则可以继续往下读，这里只会粗略讲述。

## D3D11_SHADER_TYPE_DESC结构体--描述着色器变量类型

现在我们已经获得了一个着色器变量的反射，那么可以通过`ID3D11ShaderReflectionVariable::GetType`获取着色器变量类型的反射，然后获取`D3D11_SHADER_TYPE_DESC`的信息：

```cpp
ID3D11ShaderReflectionType* pSRType = pSRVar->GetType();
D3D11_SHADER_TYPE_DESC stDesc;
hr = pSRType->GetDesc(&stDesc);
if (FAILED(hr))
	return hr;
```

`D3D11_SHADER_TYPE_DESC`的定义如下：

```cpp
typedef struct _D3D11_SHADER_TYPE_DESC {
	D3D_SHADER_VARIABLE_CLASS Class;		// 说明它是标量、矢量、矩阵、对象，还是类型
	D3D_SHADER_VARIABLE_TYPE  Type;			// 说明它是BOOL、INT、FLOAT，还是别的类型
	UINT                      Rows;			// 矩阵行数
	UINT                      Columns;		// 矩阵列数
	UINT                      Elements;		// 数组元素数目
	UINT                      Members;		// 结构体成员数目
	UINT                      Offset;		// 在结构体中的偏移，如果不是结构体则为0
	LPCSTR                    Name;			// 着色器变量类型名，如果变量未被使用则为NULL
} D3D11_SHADER_TYPE_DESC;
```

如果它是个结构体，就还能通过`ID3D11ShaderReflectionType::GetMemberTypeByIndex`方法继续获取子类别。。。

# 实现一个复杂Effects框架需要考虑到的问题

在设计一个Effects框架时，你需要考虑这些问题：

1. 是使用常规HLSL代码，然后通过着色器反射来实现；还是像Effects11那样，混杂着自定义语法，自己做代码分析
2. 如果是前者，那HLSL代码有什么施加约束（如常量缓冲区、全局变量的约束）
3. 你的Effects允许塞入一个着色器，还是六种着色器各一个，又还是任意数目的着色器
4. 你希望你的框架能提供多么复杂的功能（取决于你想获取多么详细的着色器反射信息），以及 缓存哪些信息
5. 常量缓冲区使用DYNAMIC更新还是DEFAULT更新
6. 你如何定义一个Effect Pass（是否每个Effect Pass都需要提供独立的形参存储空间），它能够管理哪些资源

因为不同的引擎对此需求可能有所不同，这取决于你怎么去设计。

# EffectHelper类的使用

目前本人实现了一个功能尽可能简化，但能够满足基本需求的`EffectHelper`类。它的功能和限制如下：

1. 支持原生HLSL代码
2. 允许塞入任意数目的着色器，但要求这些着色器在常量缓冲区和全局变量的定义上没有冲突。一种明智的做法是把所有用到的常量缓冲区、采样器、着色器资源、可读写资源、全局变量都放在同一个头文件，然后每个着色器文件都包含这个头文件来使用；又或者是把所有着色器都写到同一个文件上
3. 该框架允许按名添加着色器，以及按名添加通道，在创建通道时按名指定使用哪些着色器
4. 和Effects11一样，通过名称来获取HLSL常量缓冲区的变量，然后设置和获取值
5. 每个通道需要单独设置着色器形参（按名获取），并且可以独立设置光栅化状态、混合状态、深度/模板状态，不设置则使用默认状态。通过Apply应用当前通道。不支持Technique和Group这种形式
6. 类内部全局设置和缓存采样器状态、着色器资源、可读写资源

本文并不打算写实现细节，整个框架源码在1500行以内，你可以观察内部实现原理。现在主要介绍如何使用。

## EffectHelper::AddShader方法--添加着色器

在C++端，首先编译着色器代码，得到编译好的着色器二进制信息，然后通过`EffectHelper::AddShader`添加着色器：

```cpp
m_pEffectHelper = std::make_unique<EffectHelper>();

ComPtr<ID3DBlob> blob;

// 创建顶点着色器(3D)
HR(CreateShaderFromFile(L"HLSL\\Basic_VS_3D.cso", L"HLSL\\Basic_VS_3D.hlsl", "VS_3D", "vs_5_0", blob.ReleaseAndGetAddressOf()));
HR(m_pEffectHelper->AddShader("Basic_VS_3D", m_pd3dDevice.Get(), blob.Get()));
// 创建顶点布局(3D)
HR(m_pd3dDevice->CreateInputLayout(VertexPosNormalTex::inputLayout, ARRAYSIZE(VertexPosNormalTex::inputLayout),
                                   blob->GetBufferPointer(), blob->GetBufferSize(), m_pVertexLayout3D.GetAddressOf()));

// 创建像素着色器(3D)
HR(CreateShaderFromFile(L"HLSL\\Basic_PS_3D.cso", L"HLSL\\Basic_PS_3D.hlsl", "PS_3D", "ps_5_0", blob.ReleaseAndGetAddressOf()));
HR(m_pEffectHelper->AddShader("Basic_PS_3D", m_pd3dDevice.Get(), blob.Get()));

```

## EffectHelper::AddEffectPass方法--添加渲染通道

在创建好着色器后，我们就可以添加渲染通道。首先要填充通道信息，结构体`EffectPassDesc`定义如下：

```cpp
// 渲染通道描述
// 通过指定添加着色器时提供的名字来设置着色器
struct EffectPassDesc
{
	LPCSTR nameVS = nullptr;
	LPCSTR nameDS = nullptr;
	LPCSTR nameHS = nullptr;
	LPCSTR nameGS = nullptr;
	LPCSTR namePS = nullptr;
	LPCSTR nameCS = nullptr;
};
```

如果不需要使用某一着色器阶段，则需指定为`nullptr`。通过设置`AddShader`使用的名称来指定使用哪个着色器，然后就可以创建通道了：

```cpp
// 添加渲染通道
EffectPassDesc epDesc;
epDesc.nameVS = "Basic_VS_3D";
epDesc.namePS = "Basic_PS_3D";
HR(m_pEffectHelper->AddEffectPass("Basic_3D", m_pd3dDevice.Get(), &epDesc));
```

## 设置采样器状态、着色器资源、可读写资源

`EffectHelper`提供了按名设置和按槽设置两种方式：

```cpp
class EffectHelper
{
  public:
    // ...
    
	// 按槽设置采样器状态
	void SetSamplerStateBySlot(UINT slot, ID3D11SamplerState* samplerState);
	// 按名设置采样器状态(若存在同槽多名称则只能使用按槽设置)
	void SetSamplerStateByName(LPCSTR name, ID3D11SamplerState* samplerState);
	
	// 按槽设置着色器资源
	void SetShaderResourceBySlot(UINT slot, ID3D11ShaderResourceView* srv);
	// 按名设置着色器资源(若存在同槽多名称则只能使用按槽设置)
	void SetShaderResourceByName(LPCSTR name, ID3D11ShaderResourceView* srv);

	// 按槽设置可读写资源
	void SetUnorderedAccessBySlot(UINT slot, ID3D11UnorderedAccessView* uav, UINT initialCount);
	// 按名设置可读写资源(若存在同槽多名称则只能使用按槽设置)
	void SetUnorderedAccessByName(LPCSTR name, ID3D11UnorderedAccessView* uav, UINT initialCount);
    
    // ...
};
```

## 通过IEffectConstantBufferVariable设置常量缓冲区变量

`EffectHelper`通过HLSL定义的常量缓冲区内变量的名称来获取可用于读写的接口：

```cpp
std::shared_ptr<IEffectConstantBufferVariable> pWorld = m_pEffectHelper->GetConstantBufferVariable("g_World");
```

接口类`IEffectConstantBufferVariable`定义如下：

```cpp
// 常量缓冲区的变量
// 非COM组件
struct IEffectConstantBufferVariable
{
	// 设置无符号整数，也可以为bool设置
	virtual void SetUInt(UINT val) = 0;
	// 设置有符号整数
	virtual void SetSInt(INT val) = 0;
	// 设置浮点数
	virtual void SetFloat(FLOAT val) = 0;

	// 设置无符号整数向量，允许设置1个到4个分量
	// 着色器变量类型为bool也可以使用
	// 根据要设置的分量数来读取data的前几个分量
	virtual void SetUIntVector(UINT numComponents, const UINT data[4]) = 0;

	// 设置有符号整数向量，允许设置1个到4个分量
	// 根据要设置的分量数来读取data的前几个分量
	virtual void SetSIntVector(UINT numComponents, const INT data[4]) = 0;

	// 设置浮点数向量，允许设置1个到4个分量
	// 根据要设置的分量数来读取data的前几个分量
	virtual void SetFloatVector(UINT numComponents, const FLOAT data[4]) = 0;

	// 设置无符号整数矩阵，允许行列数在1-4
	// 要求传入数据没有填充，例如3x3矩阵可以直接传入UINT[3][3]类型
	virtual void SetUIntMatrix(UINT rows, UINT cols, const UINT* noPadData) = 0;

	// 设置有符号整数矩阵，允许行列数在1-4
	// 要求传入数据没有填充，例如3x3矩阵可以直接传入INT[3][3]类型
	virtual void SetSIntMatrix(UINT rows, UINT cols, const INT* noPadData) = 0;

	// 设置浮点数矩阵，允许行列数在1-4
	// 要求传入数据没有填充，例如3x3矩阵可以直接传入FLOAT[3][3]类型
	virtual void SetFloatMatrix(UINT rows, UINT cols, const FLOAT* noPadData) = 0;

	// 设置其余类型，允许指定设置范围
	virtual void SetRaw(const void* data, UINT byteOffset = 0, UINT byteCount = 0xFFFFFFFF) = 0;

	// 获取最近一次设置的值，允许指定读取范围
	virtual HRESULT GetRaw(void* pOutput, UINT byteOffset = 0, UINT byteCount = 0xFFFFFFFF) = 0;
};
```

前面的矩阵可以这样设置：

```cpp
XMMATRIX Eye = XMMatrixIdentity();
pWorld->SetFloatMatrix(4, 4, (const FLOAT*)&Eye);
```

要注意这样的设置并不是立即生效到着色器内的。

## IEffectPass接口类

在完成各种资源绑定后，就可以来到渲染通道这边了。`IEffectPass`定义如下：

```cpp
// 渲染通道
// 非COM组件
struct IEffectPass
{
	// 设置光栅化状态
	virtual void SetRasterizerState(ID3D11RasterizerState* pRS) = 0;
	// 设置混合状态
	virtual void SetBlendState(ID3D11BlendState* pBS, const FLOAT blendFactor[4], UINT sampleMask) = 0;
	// 设置深度混合状态
	virtual void SetDepthStencilState(ID3D11DepthStencilState* pDSS, UINT stencilValue) = 0;
	// 获取顶点着色器的uniform形参用于设置值
	virtual std::shared_ptr<IEffectConstantBufferVariable> VSGetParamByName(LPCSTR paramName) = 0;
	// 获取域着色器的uniform形参用于设置值
	virtual std::shared_ptr<IEffectConstantBufferVariable> DSGetParamByName(LPCSTR paramName) = 0;
	// 获取外壳着色器的uniform形参用于设置值
	virtual std::shared_ptr<IEffectConstantBufferVariable> HSGetParamByName(LPCSTR paramName) = 0;
	// 获取几何着色器的uniform形参用于设置值
	virtual std::shared_ptr<IEffectConstantBufferVariable> GSGetParamByName(LPCSTR paramName) = 0;
	// 获取像素着色器的uniform形参用于设置值
	virtual std::shared_ptr<IEffectConstantBufferVariable> PSGetParamByName(LPCSTR paramName) = 0;
	// 获取计算着色器的uniform形参用于设置值
	virtual std::shared_ptr<IEffectConstantBufferVariable> CSGetParamByName(LPCSTR paramName) = 0;
	// 应用着色器、常量缓冲区(包括函数形参)、采样器、着色器资源和可读写资源到渲染管线
	virtual void Apply(ID3D11DeviceContext* deviceContext) = 0;
};
```

可见每个渲染通道有自己独立的三个渲染状态，并存储着着色器uniform形参的信息允许用户设置。

最后绘制前，我们要应用当前的渲染通道：

```cpp
m_pCurrEffectPass->Apply(m_pd3dImmediateContext.Get());
```

