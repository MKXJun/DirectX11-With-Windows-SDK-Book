</br>
</br>

# 前言

写教程到现在，我发现有关纹理资源的一些解说和应用都写的太过分散，导致连我自己找起来都不方便。现在决定把这部分的内容整合起来，覆盖绝大部分内容，其中包括：

1. 纹理映射的基础回顾
2. DirectXTex库中的DDSTextureLoader、WICTextureLoader和ScreenGrab
3. stb读取纹理
4. 2D纹理的创建、使用
5. 2D纹理数组的创建、使用
6. 2D纹理立方体创建、使用
7. 纹理子资源
8. 纹理资源的复制、子资源指定区域的复制
9. 纹理从GPU回读到CPU
10. 使用内存初始化纹理
11. 使用多重采样纹理

# 纹理映射的基础回顾

由于内容重复，这里只给出跳转链接：

[纹理坐标系](part1/09?id=纹理坐标系)

[过滤器](part1/09?id=过滤器)

[对纹理进行采样](part1/09?id=对纹理进行采样)

# DDSTextureLoader和WICTextureLoader库

## DDS位图和WIC位图

DDS是一种图片格式，是DirectDraw Surface的缩写，它是DirectX纹理压缩（DirectX Texture Compression，简称DXTC）的产物。由NVIDIA公司开发。大部分3D游戏引擎都可以使用DDS格式的图片用作贴图，也可以制作法线贴图。其中dds格式支持1D纹理、2D纹理、2D纹理数组、2D纹理立方体、3D纹理，支持mipmaps，而且你还可以进行纹理压缩。

WIC（Windows Imaging Component）是一个可以扩展的平台，为数字图像提供底层API，它可以支持bmp、dng、ico、jpeg、png、tiff等格式的位图的编码与解码。

## 如何添加进你的项目

在[DirectXTex](https://github.com/Microsoft/DirectXTex)中打开`DDSTextureLoader`文件夹和`WICTextureLoader`文件夹，分别找到对应的头文件和源文件(带11的)，并加入到你的项目中

## DDSTextureLoader

### CreateDDSTextureFromFile函数--从文件读取DDS纹理

```cpp
HRESULT CreateDDSTextureFromFile(
    ID3D11Device* d3dDevice,                // [In]D3D设备
    const wchar_t* szFileName,              // [In]dds图片文件名
    ID3D11Resource** texture,               // [Out]输出一个指向资源接口类的指针，也可以填nullptr
    ID3D11ShaderResourceView** textureView, // [Out]输出一个指向着色器资源视图的指针，也可以填nullptr
    size_t maxsize = 0,                     // [In]限制纹理最大宽高，若超过则内部会缩放，默认0不限制
    DDS_ALPHA_MODE* alphaMode = nullptr);  	// [In]忽略
	
HRESULT CreateDDSTextureFromFile(
    ID3D11Device* d3dDevice,                 // [In]D3D设备
    ID3D11DeviceContext* d3dContext,         // [In]D3D设备上下文
    const wchar_t* szFileName,               // [In]dds图片文件名
    ID3D11Resource** texture,                // [Out]输出一个指向资源接口类的指针，也可以填nullptr
    ID3D11ShaderResourceView** textureView,  // [Out]输出一个指向着色器资源视图的指针，也可以填nullptr
    size_t maxsize = 0,                      // [In]限制纹理最大宽高，若超过则内部会缩放，默认0不限制
    DDS_ALPHA_MODE* alphaMode = nullptr);    // [In]忽略
```

第二个重载版本用于为DDS位图生成mipmaps，但大多数情况下你能载入的DDS位图本身都自带mipmaps了，与其运行时生成，不如提前为它制作mipmaps。

### CreateDDSTextureFromFileEx函数--从文件读取DDS纹理的增强版

上面两个函数都使用了这个函数，而且如果你想要更强的扩展性，就可以了解一下：

```cpp
HRESULT CreateDDSTextureFromFileEx(
    ID3D11Device* d3dDevice,                // [In]D3D设备
    const wchar_t* szFileName,              // [In].dds文件名
    size_t maxsize,                         // [In]限制纹理最大宽高，若超过则内部会缩放，默认0不限制
    D3D11_USAGE usage,                      // [In]使用D3D11_USAGE枚举值指定数据的CPU/GPU访问权限
    unsigned int bindFlags,                 // [In]使用D3D11_BIND_FLAG枚举来决定该数据的使用类型
    unsigned int cpuAccessFlags,            // [In]D3D11_CPU_ACCESS_FLAG枚举值
    unsigned int miscFlags,                 // [In]D3D11_RESOURCE_MISC_FLAG枚举值
    bool forceSRGB,                         // [In]强制使用SRGB，默认false
    ID3D11Resource** texture,               // [Out]获取创建好的纹理(可选)
    ID3D11ShaderResourceView** textureView, // [Out]获取创建好的纹理资源视图(可选)
    DDS_ALPHA_MODE* alphaMode = nullptr);   // [Out]忽略(可选)
	
HRESULT CreateDDSTextureFromFileEx(
    ID3D11Device* d3dDevice,                // [In]D3D设备
    ID3D11DeviceContext* d3dContext,        // [In]D3D设备上下文
    const wchar_t* szFileName,              // [In].dds文件名
    size_t maxsize,                         // [In]限制纹理最大宽高，若超过则内部会缩放，默认0不限制
    D3D11_USAGE usage,                      // [In]使用D3D11_USAGE枚举值指定数据的CPU/GPU访问权限
    unsigned int bindFlags,                 // [In]使用D3D11_BIND_FLAG枚举来决定该数据的使用类型
    unsigned int cpuAccessFlags,            // [In]D3D11_CPU_ACCESS_FLAG枚举值
    unsigned int miscFlags,                 // [In]D3D11_RESOURCE_MISC_FLAG枚举值
    bool forceSRGB,                         // [In]强制使用SRGB，默认false
    ID3D11Resource** texture,               // [Out]获取创建好的纹理(可选)
    ID3D11ShaderResourceView** textureView, // [Out]获取创建好的纹理资源视图(可选)
    DDS_ALPHA_MODE* alphaMode = nullptr);   // [Out]忽略(可选)
```

### CreateDDSTextureFromMemory函数--从内存创建DDS纹理

这里我只介绍简易版本的，因为跟上面提到的函数差别只是读取来源不一样，其余参数我就不再赘述：

```cpp
HRESULT CreateDDSTextureFromMemory(
	ID3D11Device* d3dDevice,                 // [In]D3D设备
	const uint8_t* ddsData,                  // [In]原dds文件读取到的完整二进制流
	size_t ddsDataSize,                      // [In]原dds文件的大小
	ID3D11Resource** texture,                // [Out]获取创建好的纹理(可选)
	ID3D11ShaderResourceView** textureView,  // [Out]获取创建好的纹理资源视图(可选)
	size_t maxsize = 0,                      // [In]限制纹理最大宽高，若超过则内部会缩放，默认0不限制
	DDS_ALPHA_MODE* alphaMode = nullptr);    // [Out]忽略(可选)
```

如果你需要生成mipmaps，就使用带D3D设备上下文的重载版本。

## WICTextureLoader

### CreateWICTextureFromFileEx

由于用法上和`DDSTextureLoader`大同小异，我这里也只提`CreateWICTextureFromFileEx`函数：

```cpp
HRESULT CreateWICTextureFromFileEx(
    ID3D11Device* d3dDevice,                    // [In]D3D设备
    const wchar_t* szFileName,                  // [In]位图文件名
    size_t maxsize,                             // [In]限制纹理最大宽高，若超过则内部会缩放，默认0不限制
    D3D11_USAGE usage,                          // [In]使用D3D11_USAGE枚举值指定数据的CPU/GPU访问权限
    unsigned int bindFlags,                     // [In]使用D3D11_BIND_FLAG枚举来决定该数据的使用类型
    unsigned int cpuAccessFlags,                // [In]D3D11_CPU_ACCESS_FLAG枚举值
    unsigned int miscFlags,                     // [In]D3D11_RESOURCE_MISC_FLAG枚举值
    unsigned int loadFlags,                     // [In]默认WIC_LOADER_DEAULT
    ID3D11Resource** texture,                   // [Out]获取创建好的纹理(可选)
    ID3D11ShaderResourceView** textureView);    // [Out]获取创建好的纹理资源视图(可选)
```

# ScreenGrab库

`ScreenGrab`既可以用于屏幕截屏输出，也可以将你在程序中制作好的纹理输出到文件。

在[DirectXTex](https://github.com/Microsoft/DirectXTex)中找到`ScreenGrab`文件夹，将`ScreenGrab11.h`和`ScreenGrab11.cpp`加入到你的项目中即可使用。

但为了能保存WIC类别的位图，还需要包含头文件`wincodec.h`以使用里面一些关于WIC控件的GUID。`ScreenGrab`的函数位于名称空间`DirectX`内。

## SaveDDSTextureToFile函数--以.dds格式保存纹理

```cpp
HRESULT SaveDDSTextureToFile(
    ID3D11DeviceContext* pContext,  // [In]设备上下文
    ID3D11Resource* pSource,        // [In]必须为包含ID3D11Texture2D接口类的指针
    const wchar_t* fileName );      // [In]输出文件名
```

理论上它可以保存纹理、纹理数组、纹理立方体。

## SaveWICTextureToFile函数--以指定WIC型别的格式保存纹理

```cpp
HRESULT SaveWICTextureToFile(
    ID3D11DeviceContext* pContext,  // [In]设备上下文
    ID3D11Resource* pSource,        // [In]必须为包含ID3D11Texture2D接口类的指针
    REFGUID guidContainerFormat,    // [In]需要转换的图片格式对应的GUID引用
    const wchar_t* fileName,        // [In]输出文件名
    const GUID* targetFormat = nullptr,       // [In]忽略
    std::function<void(IPropertyBag2*)> setCustomProps = nullptr );	// [In]忽略
```

下表给出了常用的GUID：

| GUID                       | 文件格式 |
| -------------------------- | -------- |
| `GUID_ContainerFormatPng`  | png      |
| `GUID_ContainerFormatJpeg` | jpg      |
| `GUID_ContainerFormatBmp`  | bmp      |
| `GUID_ContainerFormatTiff` | tif      |

这里演示了如何保存后备缓冲区纹理到文件：

```cpp
ComPtr<ID3D11Texture2D> backBuffer;
// 输出截屏
mSwapChain->GetBuffer(0, __uuidof(ID3D11Texture2D), reinterpret_cast<void**>(backBuffer.GetAddressOf()));
HR(SaveDDSTextureToFile(md3dImmediateContext.Get(), backBuffer.Get(), L"Screenshot\\output.dds"));
HR(SaveWICTextureToFile(md3dImmediateContext.Get(), backBuffer.Get(), GUID_ContainerFormatPng, L"Screenshot\\output.png"));
```

如果输出的dds文件打开后发现图像质量貌似有问题，你可以检验输出纹理的`alpha`值(关闭Alpha通道查看位图通常可以恢复正常)，也可以尝试用DDSView程序来打开文件观看(图像本身可能没有问题但程序不能完美预览高版本产生的dds文件)。

# 使用stb读取纹理数据并创建纹理

`WICTextureLoader`的优点是读取方便，直接生成D3D资源，但只能读取有限格式的纹理。

而stb除了支持jpg, png, tga, bmp外，还支持psd, gif, hdr, pic, ppm binary格式。只不过读进来的只有数据，我们还需要用它来创建2D纹理。

在[github](https://github.com/nothings/stb/blob/master/stb_image.h)找到stb的`stb_image.h`并下载，然后就可以直接添加进你的项目里。

然后找一个要用到`stb_image.h`的源文件，或者单独起一个新的cpp，在下面添加：

```cpp
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```

这段代码目前放入到`TextureManager.cpp`内。

然后为了读取文件，我们使用下面的代码：

```cpp
int texWidth, texHeight, numComponents;
stbi_uc* img_data = stbi_load(filename, &texWidth, &texHeight, &numComponents, STBI_rgb_alpha);
```

这里使用`STBI_rgb_alpha`是为了在图片只有RGB数据的情况下也强制生成alpha通道，从而让一个像素占4字节

然后我们就可以创建纹理，把数据传上去：

```cpp
if (img_data)
{
    CD3D11_TEXTURE2D_DESC texDesc(forceSRGB ? DXGI_FORMAT_R8G8B8A8_UNORM_SRGB : DXGI_FORMAT_R8G8B8A8_UNORM, 
        width, height, 1,
        enableMips ? 0 : 1, 
        D3D11_BIND_SHADER_RESOURCE | (enableMips ? D3D11_BIND_RENDER_TARGET : 0),
        D3D11_USAGE_DEFAULT, 0, 1, 0, 
        enableMips ? D3D11_RESOURCE_MISC_GENERATE_MIPS : 0);
    Microsoft::WRL::ComPtr<ID3D11Texture2D> tex;
    HR(m_pDevice->CreateTexture2D(&texDesc, nullptr, tex.GetAddressOf()));
    // 上传纹理数据
    m_pDeviceContext->UpdateSubresource(tex.Get(), 0, nullptr, img_data, width * sizeof(uint32_t), 0);
    // 创建SRV
    CD3D11_SHADER_RESOURCE_VIEW_DESC srvDesc(D3D11_SRV_DIMENSION_TEXTURE2D, 
                forceSRGB ? DXGI_FORMAT_R8G8B8A8_UNORM_SRGB : DXGI_FORMAT_R8G8B8A8_UNORM);
    HR(m_pDevice->CreateShaderResourceView(tex.Get(), &srvDesc, res.ReleaseAndGetAddressOf()));
    // 生成mipmap
    if (enableMips)
        m_pDeviceContext->GenerateMips(res.Get());
}
```

用`UpdateSubresource`是因为如果纹理带mipmap，在初始化的时候还要提供完整的mipmap初始化数据，但现在我们只有图片的数据，操作起来会比较麻烦。

生成纹理后，读取的数据需要释放掉：

```cpp
stbi_image_free(img_data);
```

至于HDR纹理的读取等后面接触了再回头补充。



# 常用显存操作函数

## ID3D11DeviceContext::UpdateSubresource函数[2]--将内存数据先上传到不可见的临时显存，再拷贝到目标子资源

这个函数在之前我们主要是用来将内存数据拷贝到常量缓冲区中，现在我们也可以用它将内存数据间接拷贝到纹理的子资源当中：

```cpp
void ID3D11DeviceContext::UpdateSubresource(
  ID3D11Resource  *pDstResource,    // [In]目标资源对象
  UINT            DstSubresource,   // [In]对于2D纹理来说，该参数为指定Mip等级的子资源
  const D3D11_BOX *pDstBox,         // [In]这里通常填nullptr，或者拷贝的数据宽高比当前子资源小时可以指定范围	
  const void      *pSrcData,        // [In]用于拷贝的内存数据
  UINT            SrcRowPitch,      // [In]该2D纹理的 宽度*数据格式的位数
  UINT            SrcDepthPitch     // [In]对于2D纹理来说并不需要用到该参数，因此可以任意设置
);

```

## ID3D11DeviceContext::Map函数--创建一片内存用于映射GPU显存资源，并拒绝GPU对该子资源的访问

```cpp
HRESULT ID3D11DeviceContext::Map(
    ID3D11Resource           *pResource,          // [In]包含ID3D11Resource接口的资源对象
    UINT                     Subresource,         // [In]子资源索引
    D3D11_MAP                MapType,             // [In]D3D11_MAP枚举值，指定读写相关操作
    UINT                     MapFlags,            // [In]填0，忽略
    D3D11_MAPPED_SUBRESOURCE *pMappedResource     // [Out]获取到的已经映射到内存的子资源
);
```

D3D11_MAP枚举值类型的成员如下：

| D3D11_MAP成员                | 含义                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| D3D11_MAP_READ               | 映射到内存的资源用于读取。该资源在创建的时候必须绑定了</br>D3D11_CPU_ACCESS_READ标签 |
| D3D11_MAP_WRITE              | 映射到内存的资源用于写入。该资源在创建的时候必须绑定了</br>D3D11_CPU_ACCESS_WRITE标签 |
| D3D11_MAP_READ_WRITE         | 映射到内存的资源用于读写。该资源在创建的时候必须绑定了</br>D3D11_CPU_ACCESS_READ和D3D11_CPU_ACCESS_WRITE标签 |
| D3D11_MAP_WRITE_DISCARD      | 映射到内存的资源用于写入，之前的资源数据将会被抛弃。该</br>资源在创建的时候必须绑定了D3D11_CPU_ACCESS_WRITE和</br>D3D11_USAGE_DYNAMIC标签 |
| D3D11_MAP_WRITE_NO_OVERWRITE | 映射到内存的资源用于写入，但不能复写已经存在的资源。</br>该枚举值只能用于顶点/索引缓冲区。该资源在创建的时候需要</br>有D3D11_CPU_ACCESS_WRITE标签，在Direct3D 11不能用于</br>设置了D3D11_BIND_CONSTANT_BUFFER标签的资源，但在</br>11.1后可以。具体可以查阅MSDN文档 |

## ID3D11DeviceContext::Unmap函数--结束内存映射(完成读写)并重新启用GPU对该资源的访问权限

```cpp
void ID3D11DeviceContext::Unmap(
    ID3D11Resource *pResource,      // [In]包含ID3D11Resource接口的资源对象
    UINT           Subresource      // [In]需要取消的子资源索引
);
```

在Map和Unmap之间，你可以对map出来的内存区域进行读写。对于写入，在Unmap后会将写入的数据上传到GPU。

## ID3D11DeviceContext::CopyResource方法--将GPU资源拷贝到目标GPU资源

该方法通过GPU将一份完整的源资源复制到目标资源：

```cpp
void ID3D11DeviceContext::CopyResource(
	ID3D11Resource *pDstResource,	// [InOut]目标资源
	ID3D11Resource *pSrcResource	// [In]源资源
);
```

但是需要注意：

1. 不支持以`D3D11_USAGE_IMMUTABLE`创建的目标资源
2. 两者资源类型要一致
3. 两者不能是同一个指针
4. 要有一样的维度(包括宽度，高度，深度，大小)
5. 要有兼容的DXGI格式，两者格式最好是能相同，或者至少是相同的组别，比如`DXGI_FORMAT_R32G32B32_FLOAT`,`DXGI_FORMAT_R32G32B32_UINT`和`DXGI_FORMAT_R32G32B32_TYPELESS`相互间就可以复制。
6. 两者任何一个在调用该方法的时候不能被映射(先前调用过`ID3D11DeviceContext::Map`方法又没有`Unmap`)
7. 允许深度/模板缓冲区作为源或目标资源

## ID3D11DeviceContext::CopySubresourceRegion方法--从指定GPU资源选取区域复制到目标GPU资源特定区域

```cpp
void ID3D11DeviceContext::CopySubresourceRegion(
	ID3D11Resource  *pDstResource,	// [In/Out]目标资源
	UINT            DstSubresource,	// [In]目标子资源索引
	UINT            DstX,			// [In]目标起始X值
	UINT            DstY,			// [In]目标起始Y值
	UINT            DstZ,			// [In]目标起始Z值
	ID3D11Resource  *pSrcResource,	// [In]源资源
	UINT            SrcSubresource,	// [In]源子资源索引
	const D3D11_BOX *pSrcBox		// [In]指定复制区域
);
```

### D3D11_BOX结构体

现在我们需要对源位图进行节选，但节选之前，首先我们需要了解定义3D盒的结构体`D3D11_BOX`：

```cpp
typedef struct D3D11_BOX {
	UINT left;	
	UINT top;
	UINT front;
	UINT right;
	UINT bottom;
	UINT back;
} D3D11_BOX;
```

3D box使用的是下面的坐标系，和纹理坐标系很像：

![](..\assets\Texture2D\07.png)

由于选取像素采用的是半开半闭区间，如`[left, right)`，在指定left, top, front的值时会选到该像素，而不对想到right, bottom, back对应的像素。

对于1D纹理来说，是没有Y轴和Z轴的，因此需要令`front=0, back=1, top=0, bottom=1`才能表示当前的1D纹理，如果出现像back和front相等的情况，则不会选到任何的纹理像素区间。

而2D纹理没有Z轴，在选取像素区域前需要置`front=0, back=1`。

3D纹理(体积纹理)可以看做一系列纹理的堆叠，因此`front`和`back`可以用来选定哪些纹理需要节选。

# Texture2D--2D纹理

Direct3D 11允许我们创建1D纹理、2D纹理、3D纹理，分别对应的接口为`ID3D11Texture1D`, `ID3D11Texture2D`和`ID3D11Texture3D`。创建出来的对象理论上不仅在内存中占用了它的实现类所需空间，还在显存中占用了一定空间以存放纹理的实际数据。

由于实际上我们最常用到的就是2D纹理，因此这里不会讨论1D纹理和3D纹理的内容。

首先让我们看看D3D11对一个2D纹理的描述：

```cpp
typedef struct D3D11_TEXTURE2D_DESC
{
    UINT Width;                     // 纹理宽度
    UINT Height;                    // 纹理高度
    UINT MipLevels;                 // 允许的Mip等级数
    UINT ArraySize;                 // 可以用于创建纹理数组，这里指定纹理的数目，单个纹理使用1
    DXGI_FORMAT Format;             // DXGI支持的数据格式
    DXGI_SAMPLE_DESC SampleDesc;    // MSAA采样级别
    D3D11_USAGE Usage;              // 使用D3D11_USAGE枚举值指定数据的CPU/GPU访问权限
    UINT BindFlags;                 // 使用D3D11_BIND_FLAG枚举来决定该数据的使用类型
    UINT CPUAccessFlags;            // 使用D3D11_CPU_ACCESS_FLAG枚举来决定CPU访问权限
    UINT MiscFlags;                 // 使用D3D11_RESOURCE_MISC_FLAG枚举给纹理标明其它可能用途
}   D3D11_TEXTURE2D_DESC;

typedef struct DXGI_SAMPLE_DESC
{
    UINT Count;                     // MSAA采样数
    UINT Quality;                   // MSAA质量等级
} DXGI_SAMPLE_DESC;
```

**这里特别要讲一下`MipLevels`：**

1. 如果你希望它不产生mipmap，则应当指定为1(只包含最大的位图本身)
2. 如果你希望它能够产生完整的mipmap，可以指定为0，这样你就不需要手工去算这个纹理最大支持的mipmap等级数了，在创建好纹理后，可以再调用`ID3D11Texture2D::GetDesc`来查看实际的`MipLevels`值是多少
3. 如果你指定的是其它的值，这里举个例子，该纹理的宽高为`400x400`，mip等级为3时，该纹理会产生`400x400`，`200x200`和`100x100`的mipmap

**紧接着是`DXGI_FORMAT`：**

它用于指定纹理存储的数据格式，最常用的就是`DXGI_FORMAT_R8G8B8A8_UNORM`了。这种格式在内存的排布可以用下面的结构体表示：

```cpp
struct {
	uint8_t r;
	uint8_t g;
	uint8_t b;
	uint8_t a;
};
```

了解这个对我们后期通过内存填充纹理十分重要。

**然后是`Usage`：**

| D3D11_USAGE           | CPU读 | CPU写 | GPU读 | GPU写 |
| --------------------- | ----- | ----- | ----- | ----- |
| D3D11_USAGE_DEFAULT   |       |       | √     | √     |
| D3D11_USAGE_IMMUTABLE |       |       | √     |       |
| D3D11_USAGE_DYNAMIC   |       | √     | √     |       |
| D3D11_USAGE_STAGING   | √     | √     | √     | √     |

如果一个纹理以`D3D11_USAGE_DEFAULT`的方式创建，那么它可以使用下面的这些方法来更新纹理：

1. `ID3D11DeviceContext::UpdateSubresource`（CPU->GPU临时空间->目标GPU资源）
2. `ID3D11DeviceContext::CopyResource`
3. `ID3D11DeviceContext::CopySubresourceRegion`

通过`DDSTextureLoader`或`WICTextureLoader`创建出来的纹理默认都是这种类型

而如果一个纹理以`D3D11_USAGE_IMMUTABLE`的方式创建，则必须在创建阶段就完成纹理资源的初始化。此后GPU只能读取，也无法对纹理再进行修改

`D3D11_USAGE_DYNAMIC`创建的纹理通常需要频繁从CPU写入，使用`ID3D11DeviceContext::Map`方法创建一片内存来映射对应的显存，经过修改后再调用`ID3D11DeviceContext::Unmap`方法应用更改。而且它对纹理有诸多的要求，直接从下面的ERROR可以看到：
`D3D11 ERROR: ID3D11Device::CreateTexture2D: A D3D11_USAGE_DYNAMIC Resource must have ArraySize equal to 1. [ STATE_CREATION ERROR #101: CREATETEXTURE2D_INVALIDDIMENSIONS]`
`D3D11 ERROR: ID3D11Device::CreateTexture2D: A D3D11_USAGE_DYNAMIC Resource must have MipLevels equal to 1. [ STATE_CREATION ERROR #102: CREATETEXTURE2D_INVALIDMIPLEVELS]`

上面说到，动态纹理不能是数组，且mip等级只能是1，即不能有mipmaps

而`D3D11_USAGE_STAGING`则完全允许在CPU和GPU之间的数据传输，但它只能作为一个类似中转站的资源，而不能绑定到渲染管线上，即你也不能用该纹理生成mipmaps。比如说有一个`D3D11_USAGE_DEFAULT`你想要从显存拿到内存，只能通过它以`ID3D11DeviceContext::CopyResource`或者`ID3D11DeviceContext::CopySubresourceRegion`方法来复制一份到本纹理，然后再通过`ID3D11DeviceContext::Map`方法取出到内存。

**现在来到`BindFlags`：**

以下是和纹理有关的`D3D11_BIND_FLAG`枚举成员：

| D3D11_BIND_FLAG             | 描述                                                        |
| --------------------------- | ----------------------------------------------------------- |
| D3D11_BIND_SHADER_RESOURCE  | 纹理可以作为着色器资源绑定到渲染管线                        |
| D3D11_BIND_STREAM_OUTPUT    | 纹理可以作为流输出阶段的输出点                              |
| D3D11_BIND_RENDER_TARGET    | 纹理可以作为渲染目标的输出点，并且指定它可以用于生成mipmaps |
| D3D11_BIND_DEPTH_STENCIL    | 纹理可以作为深度/模板缓冲区                                 |
| D3D11_BIND_UNORDERED_ACCESS | 纹理可以绑定到无序访问视图作为输出                          |

**再看看`CPUAccessFlags`：**

| D3D11_CPU_ACCESS_FLAG  | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| D3D11_CPU_ACCESS_WRITE | 允许通过映射方式从CPU写入，它不能作为管线的输出，且只能用于`D3D11_USAGE_DYNAMIC`和`D3D11_USAGE_STAGING`绑定的资源 |
| D3D11_CPU_ACCESS_READ  | 允许通过映射方式给CPU读取，它不能作为管线的输入或输出，且只能用于`D3D11_USAGE_STAGING`绑定的资源 |

可以用按位或的方式同时指定上述枚举值，如果该flag设为0可以获得更好的资源优化操作。

**最后是和纹理相关的`MiscFlags`：**

| D3D11_RESOURCE_MISC_FLAG          | 描述                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| D3D11_RESOURCE_MISC_GENERATE_MIPS | 允许通过`ID3D11DeviceContext::GenerateMips`方法生成mipmaps   |
| D3D11_RESOURCE_MISC_TEXTURECUBE   | 允许该纹理作为纹理立方体使用，要求必须是至少包含6个纹理的`Texture2DArray` |

## ID3D11Device::CreateTexture2D--创建一个2D纹理

填充好`D3D11_TEXTURE2D_DESC`后，你才可以用它创建一个2D纹理：

```cpp
HRESULT ID3D11Device::CreateTexture2D( 
    const D3D11_TEXTURE2D_DESC *pDesc,          // [In] 2D纹理描述信息
    const D3D11_SUBRESOURCE_DATA *pInitialData, // [In] 用于初始化的资源
    ID3D11Texture2D **ppTexture2D);             // [Out] 获取到的2D纹理
```

过程我就不再赘述了。

## 2D纹理的资源视图(以着色器资源视图为例)

创建好纹理后，我们还需要让它绑定到资源视图，然后再让该资源视图绑定到渲染管线的指定阶段。

`D3D11_SHADER_RESOURCE_VIEW_DESC`的定义如下：

```cpp
typedef struct D3D11_SHADER_RESOURCE_VIEW_DESC
    {
    DXGI_FORMAT Format;
    D3D11_SRV_DIMENSION ViewDimension;
    union 
        {
        D3D11_BUFFER_SRV Buffer;
        D3D11_TEX1D_SRV Texture1D;
        D3D11_TEX1D_ARRAY_SRV Texture1DArray;
        D3D11_TEX2D_SRV Texture2D;
        D3D11_TEX2D_ARRAY_SRV Texture2DArray;
        D3D11_TEX2DMS_SRV Texture2DMS;
        D3D11_TEX2DMS_ARRAY_SRV Texture2DMSArray;
        D3D11_TEX3D_SRV Texture3D;
        D3D11_TEXCUBE_SRV TextureCube;
        D3D11_TEXCUBE_ARRAY_SRV TextureCubeArray;
        D3D11_BUFFEREX_SRV BufferEx;
        } 	;
    } 	D3D11_SHADER_RESOURCE_VIEW_DESC;
};
```

其中`Format`要和纹理创建时的`Format`一致，对于2D纹理来说，应当指定`D3D11_SRV_DIMENSION`为`D3D11_SRV_DIMENSION_TEXTURE2D`。

然后`D3D11_TEX2D_SRV`结构体定义如下：

```cpp
typedef struct D3D11_TEX2D_SRV
{
    UINT MostDetailedMip;
    UINT MipLevels;
} 	D3D11_TEX2D_SRV;
```

通过`MostDetailedMap`我们可以指定开始使用的纹理子资源，`MipLevels`则指定使用的子资源数目。如果要使用完整mipmaps，则需要指定`MostDetailedMap`为0， `MipLevels`为`(UINT)-1`. 

>  注意：如果你希望使用当前mip或当前arraySlice之后的所有元素，则可以指定元素数为`(UINT)-1`

例如我想像下图那样使用mip等级为1到2的纹理子资源，可以指定`MostDetailedMip`为1，`MipLevels`为2.

![](..\assets\Texture2D\01.png)

创建着色器资源视图的演示如下：

```cpp
D3D11_SHADER_RESOURCE_VIEW_DESC srvDesc;
srvDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
srvDesc.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2D;
srvDesc.Texture2D.MipLevels = 1;
srvDesc.Texture2D.MostDetailedMip = 0;
HR(md3dDevice->CreateShaderResourceView(tex.Get(), &srvDesc, texSRV.GetAddressOf()));
```

## ID3D11DeviceContext::*SSetShaderResources方法--设置着色器资源

我们创建着色器资源的目的就是以它作为媒介，传递给着色器使用。上面打*意味着渲染管线的所有可编程着色器阶段都有该方法。

此外，着色器资源视图不仅可以绑定纹理资源，还可以绑定缓冲区资源。

目前在`DDSTextureLoader`和`WICTextureLoader`中，我们只需要用到纹理的着色器资源。这里以`ID3D11DeviceContext::PSSetShaderResources`为例：

```cpp
void ID3D11DeviceContext::PSSetShaderResources(
	UINT StartSlot,	// [In]起始槽索引，对应HLSL的register(t*)
	UINT NumViews,	// [In]着色器资源视图数目
	ID3D11ShaderResourceView * const *ppShaderResourceViews	// [In]着色器资源视图数组
);
```

## 纹理子资源(Texture Subresources)

通常我们将可能包含mipmaps的纹理称作**纹理**，那么**纹理子资源**实际上指的就是其中的一个mip等级对应的2维数组(针对2D纹理数组来说，即`Texture2D[arrayIdx][mipIdx]`)。比如512x512的纹理加载进来包含的mipmap等级数(Mipmap Levels)为10，包含了从512x512, 256x256, 128x128...到1x1的10个二维数组颜色数据，这十个纹理子资源在纹理中的内存是相对紧凑的。

例如：上述纹理(R8G8B8A8格式) mip等级为1的纹理子资源首元素地址 为 从mip等级为0的纹理子资源首元素地址再偏移512x512x4字节的地址。

Direct3D API使用Mip切片(Mip slice)来指定某一mip等级的纹理子资源，也有点像索引。比如mip slice值为0时，对应的是512x512的纹理，而mip slice值1对应的是256x256，以此类推。

### 描述一个纹理子资源的两种结构体：D3D11_SUBRESOURCE_DATA 和 D3D11_MAPPED_SUBRESOURCE 

如果你想要为2D纹理进行初始化，那么你要接触到的结构体类型为`D3D11_SUBRESOURCE_DATA`。定义如下：

```cpp
typedef struct D3D11_SUBRESOURCE_DATA
{
    const void *pSysMem;	// 用于初始化的数据
    UINT SysMemPitch;		// 当前子资源一行所占的字节数(2D/3D纹理使用)
    UINT SysMemSlicePitch;	// 当前子资源一个完整切片所占的字节数(仅3D纹理使用)
} 	D3D11_SUBRESOURCE_DATA;
```

![](..\assets\Texture2D\02.png)

而如果你使用的是`ID3D11DeviceContext::Map`方法来获取一个纹理子资源，那么获取到的是`D3D11_MAPPED_SUBRESOURCE`，其定义如下：

```cpp
typedef struct D3D11_MAPPED_SUBRESOURCE {
    void *pData;		// 映射到内存的数据or需要提交的地址范围
    UINT RowPitch;		// 当前子资源一行所占的字节数(2D/3D纹理有意义)
    UINT DepthPitch;	// 当前子资源一个完整切片所占的字节数(仅3D纹理有意义)
} D3D11_MAPPED_SUBRESOURCE;
```

若一张512x512的纹理(R8G8B8A8)，那么它的`RowPitch`为512x4=2048字节，同理在初始化一个512x512的纹理(R8G8B8A8)，它的`RowPitch`有可能为512x4=2048字节。

> **注意：在运行的时候，`RowPitch`和`DepthPitch`有可能会比你所期望的值更大一些，因为在每一行的数据之间有可能会填充数据进去以对齐。**

通常情况下我们希望读出来的RGBA是连续的，然而下述映射回内存的做法是错误的，因为每一行的数据都有填充，读出来的话你可能会发现图像会有错位：

```cpp
std::vector<unsigned char> imageData;
m_pd3dImmediateContext->Map(texOutputCopy.Get(), 0, D3D11_MAP_READ, 0, &mappedData);
memcpy_s(imageData.data(), texWidth * texHeight * 4, mappedData.pData, texWidth * texHeight * 4);
m_pd3dImmediateContext->Unmap(texOutputCopy.Get(), 0);
```

**正确的做法应该是逐行读取，利用rowPitch跳行：**

```cpp
std::vector<unsigned char> imageData;
m_pd3dImmediateContext->Map(texOutputCopy.Get(), 0, D3D11_MAP_READ, 0, &mappedData);
unsigned char* pData = reinterpret_cast<unsigned char*>(mappedData.pData);
for (UINT i = 0; i < texHeight; ++i)
{
    memcpy_s(&imageData[i * texWidth * 4], texWidth * 4, pData, texWidth * 4);
    pData += mappedData.RowPitch;
}
m_pd3dImmediateContext->Unmap(texOutputCopy.Get(), 0);
```

## 纹理从GPU回读到CPU

通常这种资源的类型有可能是`D3D11_USAGE_IMMUTABLE`或者`D3D11_USAGE_DEFAULT`。我们需要按下面的步骤进行：

1. 创建一个`D3D11_USAGE_STAGING`的纹理，指定CPU读取权限，纹理宽高一致，Mip等级和数组大小都为1；
2. 使用`ID3D11DeviceContext::CopyResource`方法拷贝一份到我们新创建的STAGING纹理，然后对其进行内存映射，注意需要严格按照上面提到的方式逐行读取，最后解除映射。

## 通过内存初始化纹理

现在我们尝试通过纯代码的形式来创建一个纹理(以项目09作为修改)，代码如下：

```cpp
uint32_t ColorRGBA(uint8_t r, uint8_t g, uint8_t b, uint8_t a)
{
	return (r | (g << 8) | (b << 16) | (a << 24));
}


bool GameApp::InitResource()
{
	uint32_t black = ColorRGBA(0, 0, 0, 255), orange = ColorRGBA(255, 108, 0, 255);

	// 纹理内存映射，用黑色初始化
	std::vector<uint32_t> textureMap(128 * 128, black);
	uint32_t(*textureMap)[128] = reinterpret_cast<uint32_t(*)[128]>(textureArrayMap.data());

	for (int y = 7; y <= 17; ++y)
		for (int x = 25 - y; x <= 102 + y; ++x)
			textureMap[y][x] = textureMap[127 - y][x] = orange;

	for (int y = 18; y <= 109; ++y)
		for (int x = 7; x <= 120; ++x)
			textureMap[y][x] = orange;

	// 创建纹理
	D3D11_TEXTURE2D_DESC texDesc;
	texDesc.Width = 128;
	texDesc.Height = 128;
	texDesc.MipLevels = 1;
	texDesc.ArraySize = 1;
	texDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
	texDesc.SampleDesc.Count = 1;		// 不使用多重采样
	texDesc.SampleDesc.Quality = 0;
	texDesc.Usage = D3D11_USAGE_DEFAULT;
	texDesc.BindFlags = D3D11_BIND_SHADER_RESOURCE;
	texDesc.CPUAccessFlags = 0;
	texDesc.MiscFlags = 0;	// 指定需要生成mipmap

	D3D11_SUBRESOURCE_DATA sd;
	uint32_t * pData = textureMap.data();
	sd.pSysMem = pData;
	sd.SysMemPitch = 128 * sizeof(uint32_t);
	sd.SysMemSlicePitch = 128 * 128 * sizeof(uint32_t);


	ComPtr<ID3D11Texture2D> tex;
	HR(m_pd3dDevice->CreateTexture2D(&texDesc, &sd, tex.GetAddressOf()));

	D3D11_SHADER_RESOURCE_VIEW_DESC srvDesc;
	srvDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
	srvDesc.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2D;
	srvDesc.Texture2D.MipLevels = 1;
	srvDesc.Texture2D.MostDetailedMip = 0;
	HR(m_pd3dDevice->CreateShaderResourceView(tex.Get(), &srvDesc, m_pTexSRV.GetAddressOf()));
	
	// ...
}
```

其它部分的代码修改就不讲了，最终效果如下：

![](..\assets\Texture2D\03.png)

但是如果你想要以初始化的方式来创建带mipmap的`Texture2D`纹理，则在初始化的时候需要提供`D3D11_SUBRESOURCE_DATA`数组，元素数目为`MipLevels`.

再或者如果你是要以初始化的方式来创建带mipmap的`Texture2D`纹理数组，则提供的元素数目为`MipLevels * ArraySize`.



## 2D纹理在HLSL的采样和索引

对于2D纹理，最常用的采样函数是`Sample`，而采样点的数目与所使用的`SamplerState`有关。使用点采样的时候只会选择距离最近的像素；而每当纹理的坐标轴从点采样替换为双线性采样时，采样点的个数要乘以2；三线性采样(U、V、Mipmap)需要采样的像素个数为8.其中，UV方向的双线性采样行为会采样与当前纹理坐标邻近的四个像素。

> 注意：由于只有在像素着色阶段会计算ddx和ddy，且`Sample`需要用到ddx和ddy信息计算mipmap等级，故它只能在像素着色器中使用。

如果想在别的着色器阶段采样纹理，则可以使用`SampleLevel`方法。

此外，纹理除了采样，还可以通过索引的方式来访问：

```hlsl
float4 color = g_Tex[posH.xy];
```

# Texture2DArray--2D纹理数组

之前提到，`D3D11_TEXTURE2D_DESC`中可以通过指定`ArraySize`的值来将其创建为纹理数组。

## HLSL中的2D纹理数组

首先来到HLSL代码，如果尝试使用下面的方式：

```hlsl
Texture2D g_TexArray[3] : register(t0);
```

实际上是等价于：

```hlsl
Texture2D g_TexArray[0] : register(t0);
Texture2D g_TexArray[1] : register(t1);
Texture2D g_TexArray[2] : register(t2);
```

因此在像素着色器阶段我们不能这样写：

```hlsl
float4 PS(VertexPosHTex pIn) : SV_Target
{
    float4 texColor = g_TexArray[g_TexIndex].Sample(g_Sam, float2(pIn.Tex));
    return texColor;
}
```

HLSL编译器会产生报错：`sampler array index must be a literal experssion`，即`g_TexIndex`的值也必须是个字面值，而不是变量。

但我们还是想要能够根据变量取对应纹理的能力。正确的做法应当是声明一个`Texture2DArray`变量，然后在C++中使用纹理数组：

```hlsl
Texture2DArray g_TexArray : register(t0);
```

`Texture2DArray`同样也具有`Sample`方法，用法示例如下：

```hlsl
// 像素着色器
float4 PS(VertexPosHTex pIn) : SV_Target
{
    float4 texColor = g_TexArray.Sample(g_Sam, float3(pIn.Tex, g_TexIndex));
    return texColor;
}
```

Sample方法的第一个参数依然是采样器

而第二个参数则是一个3D向量，其中x与y的值对应的还是纹理坐标，而z分量即便是个`float`，主要是用于作为索引值选取纹理数组中的某一个具体纹理。同理索引值0对应纹理数组的第一张纹理，1对应的是第二张纹理等等...

使用纹理数组的优势是，我们可以一次性预先创建好所有需要用到的纹理，并绑定到HLSL的纹理数组中，而不需要每次都重新绑定一个纹理。然后我们再使用索引值来访问纹理数组中的某一纹理。

## D3D11CalcSubresource函数--计算子资源的索引值

对于纹理数组，每个元素都会包含同样的mip等级数。Direct3D API使用数组切片(array slice)来访问不同纹理，也是相当于索引。这样我们就可以把所有的纹理资源用下面的图来表示。该纹理数组有4个纹理，每个纹理包含3个mipmap：

![](..\assets\Texture2D\04.png)

然后给定当前纹理数组每个纹理的mipmap等级数(Mipmap Levels)，数组切片(Array Slice)和Mip切片(Mip Slice)，我们就可以用下面的函数来求得指定子资源的索引值：

```cpp
inline UINT D3D11CalcSubresource(UINT MipSlice, UINT ArraySlice, UINT MipLevels )
{ return MipSlice + ArraySlice * MipLevels; }
```

## 将多个Texture2D的数据复制到Texture2DArray

现在我们手头上仅有的就是`DDSTextureLoader11.h`和`WICTextureLoader11.h`中的函数，但这里面的函数每次都只能加载一张纹理。为了能把这些纹理装载进纹理数组，具体的操作顺序如下：

1. 读取第一个存有纹理的文件，得到`ID3D11Texture2D`对象，并通过`GetDesc`获取信息；
2. 创建一个`ID3D11Texture2D`对象，它同时也是一个纹理数组；
3. 读取下一个纹理文件，然后检查宽高、数据格式等属性是否相同，再将其复制到该纹理数组中。重复直到所有纹理读取完毕；
4. 为该纹理数组对象创建创建一个纹理资源视图（Shader Resource View）。

现在我们可以尝试将6张同样宽高的纹理复制到一个纹理数组：

```cpp
ComPtr<ID3D11Texture2D> pTexArray;
// 省略CreateTexture2D过程，arraySize为6，mipLevels=1
for (uint32_t i = 0; i < 6; ++i)
{
    m_pd3dImmediateContext->CopySubresourceRegion(pTexArray.Get(), 
                                                  D3D11CalcSubresource(0, i, 1), 0, 0, 0, 
                                                  pTex.Get(), 0, nullptr);
}
```

### D3D11_TEX2D_ARRAY_SRV结构体

在创建着色器目标视图时，你还需要填充共用体中的`D3D11_TEX2D_ARRAY_SRV`结构体：

```cpp
typedef struct D3D11_TEX2D_ARRAY_SRV
{
    UINT MostDetailedMip;   // 指定起始使用的mipLevel
    UINT MipLevels;         // (UINT)-1表示访问MostDetailedMip开始往后的所有mips
    UINT FirstArraySlice;   // 指定起始使用的纹理
    UINT ArraySize;         // (UINT)-1表示访问FirstArraySlice开始往后的所有slice
} 	D3D11_TEX2D_ARRAY_SRV;
```

![](..\assets\Texture2D\05.png)

例如我想指定像上面那样的范围，可以指定`FirstArraySlice`为1，`ArraySize`为2，`MostDetailedMip`为1，`MipLevels`为2.

### ID3D11DeviceContext::GenerateMips--为纹理资源视图绑定的所有纹理创建完整的mipmap链

由于通过该函数读取进来的纹理mip等级可能只有1，如果还需要创建mipmap链的话，还需要用到下面的方法。

```cpp
void ID3D11DeviceContext::GenerateMips(
  ID3D11ShaderResourceView *pShaderResourceView	// [In]需要创建mipamp链的SRV
);
```

比如一张1024x1024的纹理，经过该方法调用后，就会生成剩余的512x512, 256x256 ... 1x1的子纹理资源，加起来一共是11级mipmap。

在调用该方法之前，需要做好大量的准备：

1. 在创建2D纹理资源时，`Usage`为`D3D11_USAGE_DEFAULT`以允许GPU写入，`BindFlags`要绑定`D3D11_BIND_RENDER_TARGET`和`D3D11_BIND_SHADER_RESOURCE`，`MiscFlags`设置`D3D11_RESOURCE_MISC_GENERATE_MIPS`枚举值，`mipLevels`设置为0使得在创建纹理的时候会自动预留出其余mipLevel所需要用到的内存大小。
2. 如果是2D纹理，将图片的RGBA数据写入到子资源0中。此时创建好的纹理，子资源0为图片内容，其余子资源为黑色。如果是2D纹理数组，你可以利用`D3D11CalcSubresource`为所有纹理元素的首mipLevel来填充图片。
3. 为该2D纹理资源创建着色器资源视图，指定`MostDetailedMip`为0，`MipLevels`为-1以访问完整mipmaps。

做好这些准备后你才可以调用`GenerateMips`，否则可能会产生异常。

# TextureCube--纹理立方体

**纹理立方体**的底层资源实际上是**2D纹理数组资源**，然后在此基础上通过着色器资源视图指定出来的，在SRV中需要确定哪6个连续的纹理作为纹理立方体。这也意味着你可以在一个2D纹理数组上创建多个纹理立方体。

![](..\assets\Texture2D\06.png)

Direct3D提供了枚举类型`D3D11_TEXTURECUBE_FACE`来标识立方体某一表面：

```cpp
typedef enum D3D11_TEXTURECUBE_FACE {
	D3D11_TEXTURECUBE_FACE_POSITIVE_X = 0,
	D3D11_TEXTURECUBE_FACE_NEGATIVE_X = 1,
	D3D11_TEXTURECUBE_FACE_POSITIVE_Y = 2,
	D3D11_TEXTURECUBE_FACE_NEGATIVE_Y = 3,
	D3D11_TEXTURECUBE_FACE_POSITIVE_Z = 4,
	D3D11_TEXTURECUBE_FACE_NEGATIVE_Z = 5
} D3D11_TEXTURECUBE_FACE;
```

可以看出:

1. 索引0指向+X表面;
2. 索引1指向-X表面;
3. 索引2指向+Y表面;
4. 索引3指向-Y表面;
5. 索引4指向+Z表面;
6. 索引5指向-Z表面;

使用立方体映射意味着我们需要使用3D纹理坐标进行寻址，通过向量的形式来指定使用立方体某个表面的其中一点。

在HLSL中，立方体纹理用`TextureCube`来表示。

## 创建一个纹理立方体

对于创建好的DDS立方体纹理，我们只需要使用`DDSTextureLoader`就可以很方便地读取进来：

```cpp
HR(CreateDDSTextureFromFile(
	device.Get(), 
	cubemapFilename.c_str(), 
	nullptr, 
	textureCubeSRV.GetAddressOf()));
```

然而从网络上能够下到的天空盒资源大多是六张天空盒的正方形贴图，用DXTex导入还是比较麻烦的一件事情。我们也可以自己编写代码来构造立方体纹理。在创建资源的时候，需要将`MiscFlags`标记为`D3D11_RESOURCE_MISC_TEXTURECUBE`

将六张天空盒的正方形贴图转换成立方体需要经历这4个步骤:

1. 读取这六张正方形贴图
2. 创建包含6个纹理的数组
3. 将这六张贴图完整地拷贝到该数组中，和前面纹理数组的拷贝过程是相同的
4. 创建立方体纹理的SRV

具体操作可以直接参考22-23章`GameApp`初始化资源的部分。

# Texture2DMS--多重采样纹理

## 多重采样

首先复习一下**超采样（SSAA）**。现渲染WxH大小的屏幕，为了降低走样/锯齿的影响，可以使用2倍宽高的后备缓冲区及深度缓冲区，光栅化的操作量增大到了4倍左右，同样像素着色器的执行次数也增大到了原来的4倍左右。每个2x2的像素块对应原来的1个像素，通过对这2x2的像素块求平均值得到目标像素的颜色。

**多重采样（MSAA）**采用的是一种较为折中的方案。以4xMSAA为例，指的是单个像素需要按特定的模式对周围采样4次：

![](..\assets\Texture2D\08.png)

假定当前像素位置为(x+0.5,y+0.5)，在D3D11定义的采样模式下，这四个采样点的位置为：(x+0.375, y+0.125)、(x+0.875, y+0.375)、(x+0.125, y+0.625)、(x+0.625, y+0.875)。具体我们可以通过HLSL中的`Texture2DMS::GetSamplePosition`方法进行验证。

在光栅化阶段，在一个像素区域内使用4个子采样点，但每个像素仍只执行1次像素着色器的计算。这4个子采样点都会计算自己的深度值，然后根据深度测试和三角形覆盖性测试来决定是否复制该像素的计算结果。为此**深度缓冲区**和**渲染目标**需要的空间依然为原来的**4倍**。

假设三角形1覆盖了2-3号采样，且通过了深度测试，则2-3号采样复制像素着色器得到的红色。此时渲染三角形2覆盖了1号采样，且通过了深度测试，则1号采样复制像素着色器得到的蓝色。在渲染完三角形后，对这四个采样点的颜色进行一个resolve操作，可以得到当前像素最终的颜色。这里我们可以认为是对这四个采样点的颜色求平均值。

![](..\assets\Texture2D\09.png)

## 读取多重采样资源

前面我们已经知道了如果创建多重采样资源并作为渲染目标使用。而为了能够在着色器使用多重采样纹理，需要在HLSL声明如下类型的资源：

```hlsl
Texture2DMS<type, sampleCount> g_Texture : register(t*)
```

由于是模板类型，sampleCount必须为字面值。非多重采样的纹理设置的sampleCount是1，除了`Texture2D`类型，还可以传入到`Texture2DMS<float4, 1>`中使用。但在创建着色器资源视图时，我们需要指定`ViewDimension`为`D3D11_SRV_DIMENSION_TEXTURE2DMS`

`Texture2DMS`类型只支持读取，不支持采样：

```hlsl
T Texture2DMS<T>::Load(
    in int2 coord,
    in int sampleIndex
);
```

## ID3D11DeviceContext::ResolveSubresource方法--将多重采样的资源复制到非多重采样的资源

```cpp
void ID3D11DeviceContext::ResolveSubresource(
    ID3D11Resource *pDstResource,   // [In]目标非多重采样资源
    UINT           DstSubresource,  // [In]目标子资源索引
    ID3D11Resource *pSrcResource,   // [In]源多重采样资源
    UINT           SrcSubresource,  // [In]源子资源索引
    DXGI_FORMAT    Format           // [In]解析成非多重采样资源的格式
);
```

该API最常用于需要将当前pass的渲染目标结果作为下一个pass的输入。源/目标资源必须拥有相同的资源类型和相同的维度。此外，它们必须拥有兼容的格式：

- 已经确定的类型则要求必须相同：比如两个资源都是`DXGI_FORMAT_R8G8B8A8_UNORM`，在Format中一样要填上
- 若有一个类型确定但另一个类型不确定：比如两个资源分别是`DXGI_FORMAT_R32_FLOAT`和`DXGI_FORMAT_R32_TYPELESS`，在Format参数中必须指定`DXGI_FORMAT_R32_FLOAT`
- 两个不确定类型要求必须相同：如`DXGI_FORMAT_R32_TYPELESS`，那么在Format参数中可以指定`DXGI_FORMAT_R32_FLOAT`或`DXGI_FORMAT_R32_UINT`等

若后备缓冲区创建的时候指定了多重采样，正常渲染完后到呈现(Present)时会自动调用`ResolveSubresource`方法得到用于显示的非多重采样纹理。当然这仅限于交换链使用的是**BLIT模型**而非FLIP模型，因为**翻转模型的后备缓冲区不能使用多重采样纹理**，要使用MSAA我们需要另外新建一个等宽高的MSAA纹理先渲染到此处，然后`ResolveSubresource`到后备缓冲区。

当然还有一种办法是在渲染的时候自己手动在着色器做Resolve：

```cpp
float4 outputColor = 0.0f;
for (int i = 0; i < sampleCount; ++i)
{
    outputColor += g_Texture.load(coord, i);
}
outputColor /= sampleCount;
```

