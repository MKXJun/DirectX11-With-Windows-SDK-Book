</br>
</br>

# 综述

**参考文章：https://blogs.msdn.microsoft.com/chuckw/2012/04/24/wheres-dxerr-lib/**

在龙书11中所使用的`HR`宏和`dxerr`库是一个比较实用的错误原因追踪工具。D3D中的某些函数拥有返回值`HRESULT`，通过`dxerr`库，可以将错误码转换成错误详细信息的字符串。

在DirectX SDK中，包含了头文件`dxerr.h`和库文件`dxerr.lib`，在以往的做法包含了DX SDK后，就可以直接使用`dxerr`了。但如果是要编写基于Windows SDK的Direct3D程序，在Windows SDK 8.0以上已经没有了`dxerr`库。

此时此刻，你仍然有两种选择来脱离对DirectX SDK的依赖：
1. **寻找较新的`dxerr.h`和`dxerr.cpp`源码来编译出`dxerr.lib`，或者直接加入你的项目当中；**
2. **直接抛弃`dxerr`库**

## 新的dxerr源码

微软已经将`dxerr`库开源了，下面的链接可以下载，如果不放心的话，你也可以到上面的参考文章去下载。

[dxerr_nov2015.zip下载地址](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Components.PostAttachments/00/10/29/73/07/dxerr_nov2015.zip)

在我以往的DirectX11项目中，则是从DXUT中拉过来的`dxerr`：

[DXUT Github/Core](https://github.com/Microsoft/DXUT/tree/master/Core)

但要注意的是，由于新的`dxerr.h`仅提供了`DXTrace`的Unicode字符集版本，需要将原来的`__FILE__`替换为`__FILEW__`，并在项目属性页中将**字符集设置为Unicode**。

![](..\assets\HR\01.png)

## 抛弃dxerr库

自Windows SDK 8.0起，`HRESULT`值关于DirectX图形API的错误消息字符串映射已经加入到`FormatMessage`函数中。我们可以直接脱离对`dxerr`的依赖，并使用该函数来直接获取错误消息字符串。因此，`dxerr`库也就没有必要在Windows SDK 8.0以上版本保留了。

### FormatMessageW函数--获取格式化消息字符串

鉴于我们只是要获取错误码对应的字符串信息，这里就简单提及一下该函数的部分用法：

```cpp
DWORD FormatMessageW(
  DWORD   dwFlags,			// [In]FORMAT_MESSAGE系列宏
  LPCVOID lpSource,			// [In]直接填NULL
  DWORD   dwMessageId,		// [In]传入函数异常时返回的HRESULT
  DWORD   dwLanguageId,		// [In]语言ID
  LPTSTR  lpBuffer,			// [In]用于输出消息字符串的缓冲区
  DWORD   nSize,			// [In]WCHAR缓冲区可容纳元素个数
  va_list *Arguments		// [In]直接填NULL
);
```

### DXTraceW函数

这里我将`dxerr`中`DXTraceW`函数的实现进行了修改，由于现在错误码信息为中文，为此也顺便把错误窗口和输出也汉化了。只需要包含`Windows.h`和`sal.h`就可以使用。
函数原型：
```cpp
// ------------------------------
// DXTraceW函数
// ------------------------------
// 在调试输出窗口中输出格式化错误信息，可选的错误窗口弹出(已汉化)
// [In]strFile			当前文件名，通常传递宏__FILEW__
// [In]hlslFileName     当前行号，通常传递宏__LINE__
// [In]hr				函数执行出现问题时返回的HRESULT值
// [In]strMsg			用于帮助调试定位的字符串，通常传递L#x(可能为NULL)
// [In]bPopMsgBox       如果为TRUE，则弹出一个消息弹窗告知错误信息
// 返回值: 形参hr
HRESULT WINAPI DXTraceW(_In_z_ const WCHAR* strFile, _In_ DWORD dwLine, _In_ HRESULT hr, _In_opt_ const WCHAR* strMsg, _In_ bool bPopMsgBox);
```

函数实现：
```cpp
HRESULT WINAPI DXTraceW(_In_z_ const WCHAR* strFile, _In_ DWORD dwLine, _In_ HRESULT hr,
	_In_opt_ const WCHAR* strMsg, _In_ bool bPopMsgBox)
{
	WCHAR strBufferFile[MAX_PATH];
	WCHAR strBufferLine[128];
	WCHAR strBufferError[300];
	WCHAR strBufferMsg[1024];
	WCHAR strBufferHR[40];
	WCHAR strBuffer[3000];

	swprintf_s(strBufferLine, 128, L"%lu", dwLine);
	if (strFile)
	{
		swprintf_s(strBuffer, 3000, L"%ls(%ls): ", strFile, strBufferLine);
		OutputDebugStringW(strBuffer);
	}

	size_t nMsgLen = (strMsg) ? wcsnlen_s(strMsg, 1024) : 0;
	if (nMsgLen > 0)
	{
		OutputDebugStringW(strMsg);
		OutputDebugStringW(L" ");
	}
	// Windows SDK 8.0起DirectX的错误信息已经集成进错误码中，可以通过FormatMessageW获取错误信息字符串
	// 不需要分配字符串内存
	FormatMessageW(FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
		nullptr, hr, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
		strBufferError, 256, nullptr);

	WCHAR* errorStr = wcsrchr(strBufferError, L'\r');	
	if (errorStr)
	{
		errorStr[0] = L'\0';	// 擦除FormatMessageW带来的换行符(把\r\n的\r置换为\0即可)
	}

	swprintf_s(strBufferHR, 40, L" (0x%0.8x)", hr);
	wcscat_s(strBufferError, strBufferHR);
	swprintf_s(strBuffer, 3000, L"错误码含义：%ls", strBufferError);
	OutputDebugStringW(strBuffer);

	OutputDebugStringW(L"\n");

	if (bPopMsgBox)
	{
		wcscpy_s(strBufferFile, MAX_PATH, L"");
		if (strFile)
			wcscpy_s(strBufferFile, MAX_PATH, strFile);

		wcscpy_s(strBufferMsg, 1024, L"");
		if (nMsgLen > 0)
			swprintf_s(strBufferMsg, 1024, L"当前调用：%ls\n", strMsg);

		swprintf_s(strBuffer, 3000, L"文件名：%ls\n行号：%ls\n错误码含义：%ls\n%ls您需要调试当前应用程序吗？",
			strBufferFile, strBufferLine, strBufferError, strBufferMsg);

		int nResult = MessageBoxW(GetForegroundWindow(), strBuffer, L"错误", MB_YESNO | MB_ICONERROR);
		if (nResult == IDYES)
			DebugBreak();
	}

	return hr;
}
```


## HR宏
现在的HR宏变成了这样：
```cpp
// ------------------------------
// HR宏
// ------------------------------
// Debug模式下的错误提醒与追踪
#if defined(DEBUG) | defined(_DEBUG)
	#ifndef HR
	#define HR(x)												\
	{															\
		HRESULT hr = (x);										\
		if(FAILED(hr))											\
		{														\
			DXTraceW(__FILEW__, (DWORD)__LINE__, hr, L#x, true);\
		}														\
	}
	#endif
#else
	#ifndef HR
	#define HR(x) (x)
	#endif 
#endif
```

测试效果如下：

![](..\assets\HR\02.png)

在调试输出窗口也可以看到：

![](..\assets\HR\03.png)

