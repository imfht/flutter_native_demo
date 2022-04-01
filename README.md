# flutter_native_demo

A native flutter plugin project demo.

## Getting Started
# Flutter 使用 dart:ffi 调用 C/C++ 的插件的开发 （支持 windows、macos、ios、android ）

一个各平台调用 C/C++ 源码的例子，如何共享代码，配置相关的编译

官方的例子：[https://docs.flutter.dev/development/platform-integration/c-interop](https://docs.flutter.dev/development/platform-integration/c-interop)

源码地址：[https://github.com/gaoshang212/flutter_native_demo](https://github.com/gaoshang212/flutter_native_demo)
### 创建一个插件

可以执行下面的命令来创建一个插件

```bash
flutter create --template=plugin --platforms=windows,macos,ios,android,linux flutter_native_demo
```

--platforms 可以指定支持哪些平台，如 windows，macos，ios，android，linux

如果没有创建相应平台目录，可以使用下面的命令开启相应的平台

```bash
flutter config --enable-linux-desktop  # 开启linux 桌面
flutter config --enable-macos-desktop  # 开启macos 桌面
flutter config --enable-ios # 开启ios

# 更多的命令可以通过help查看
flutter config --help
```

如果有字符串操作或转换，可以添加 ffi 的包：

```bash
flutter pub add ffi
```

项目结构

![Untitled](./resources/images/tree.png?raw=true)

### 添加 C/C++ 源码文件

很多时候我们各平台是会共用一套C/C++ 源码的，我们先创建一个源码，就按官网的来，但我们创建在一个公共目录（官网创建在IOS/Classes下面）

*libs/native_add/native_add.cpp*

```cpp
#include <stdint.h>

#ifdef WIN32
#define DART_API extern "C" __declspec(dllexport)
#else
#define DART_API extern "C" __attribute__((visibility("default"))) __attribute__((used))
#endif

DART_API int32_t native_add(int32_t x, int32_t y) {
    return x + y;
}
```

### Dart

在 *lib/flutter_native_demo.dart* 中添加动态库的调用代码

```dart
final DynamicLibrary nativeAddLib = Platform.isMacOS || Platform.isIOS
    ? DynamicLibrary.process()
    : DynamicLibrary.open('libNativeAdd.${Platform.isWindows ? 'dll' : 'so'}');

final int Function(int x, int y) nativeAdd = nativeAddLib
    .lookup<NativeFunction<Int32 Function(Int32, Int32)>>('native_add')
    .asFunction();
```

我们改一下 *example/lib/main.dart* 的代码

```dart
// 修改一下 platformVersion 的赋值
platformVersion = nativeAdd(1, 2).toString();
```

### Windows 配置

在 *libs/native_add* 目录中添加一个 CMakeLists.txt ，用来编译 动态库。

```makefile
cmake_minimum_required(VERSION 3.4)

# 项目名称
set(PROJECT_NAME "libNativeAdd")
project(${PROJECT_NAME} LANGUAGES CXX)

# 源文件
add_library(${PROJECT_NAME} SHARED
    "./native_add.cpp"
)

# 动态库的输出目录
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/$<$<CONFIG:DEBUG>:Debug>$<$<CONFIG:RELEASE>:Release>")
# 安装动态库的目标目录
set(INSTALL_BUNDLE_LIB_DIR "${CMAKE_INSTALL_PREFIX}")
# 安装动态库，到执行目录
install(FILES "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/${PROJECT_NAME}.dll" DESTINATION "${INSTALL_BUNDLE_LIB_DIR}" COMPONENT Runtime)
```

在 *windows* 目录下面的  *CMakeLists.txt* 中添加相应的子目录

```makefile
add_subdirectory("../libs/native_add" native_add)
```

在 *example* 目录下面执行下面的命令，来运行程序.

```powershell
cd example
flutter run -d windows -v
```

### Android 配置

安卓的动态库，会自动添加lib头，我们改造一下 *libs/native_add/CMakeLists.txt* 让他兼容windows和 android

```makefile
cmake_minimum_required(VERSION 3.10)

# 项目名称
if (${CMAKE_SYSTEM_NAME} EQUAL "Windows") 
    set(PROJECT_NAME "libNativeAdd")
else()
    set(PROJECT_NAME "NativeAdd")
endif()

project(${PROJECT_NAME} LANGUAGES CXX)

# 源文件
add_library(${PROJECT_NAME} SHARED
    "./native_add.cpp"
)

# Windows 需要把dll拷贝到bin目录
IF (WIN32)
    # 动态库的输出目录
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/$<$<CONFIG:DEBUG>:Debug>$<$<CONFIG:RELEASE>:Release>")
    # 安装动态库的目标目录
    set(INSTALL_BUNDLE_LIB_DIR "${CMAKE_INSTALL_PREFIX}")
    # 安装动态库，到执行目录
    install(FILES "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/${PROJECT_NAME}.dll" DESTINATION "${INSTALL_BUNDLE_LIB_DIR}" COMPONENT Runtime)
ENDIF()
```

在 android/build.gradle 文件中添加 CMakeList.txt 路径

```
android {
	externalNativeBuild {
        // Encapsulates your CMake build configurations.
        cmake {
		        // 指定一个CMake 编译脚本的相对目录。
            path "../libs/native_add/CMakeLists.txt"
        }
   }
}
```

在 *example* 目录下面执行下面的命令，来运行程序

```powershell
cd example
flutter run -d <android> -v
```

说明：可以用 *flutter devices* 查看支持设备，来替换 *<android>*。

### macOS 配置

在 *macos/Classes* 目录中执行下面的命令，给macOS *link* 相关的代码

```bash
ln -s ../../libs/native_add/native_add.cpp ./
```

然后回到 *example* 目录中执行

```bash
flutter run -d macos -v
```

说明：国内使用时会，通过 ****CocoaPods**** 安装包会很慢，可以切换到 [清华的镜像](https://mirrors.tuna.tsinghua.edu.cn/help/CocoaPods/)。设置 *example 目录下macos 的 Podfile。*

### IOS 配置

IOS 和 macOS 的配置基本是一样的，注意一下目录就好了。

### 执行效果

![ios](./resources/images/ios.png?raw=true|width=245px)

![macos](./resources/images/macos.png?raw=true|width=540px)

