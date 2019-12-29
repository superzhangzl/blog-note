# Java之JNA

## 一、记录背景	

​		关于项目内容还是需要保密的，只描述一下使用到JNA技术的需求点。在Windows平台下，获取指定应用程序的窗体信息，例如长宽、坐标等，具体获取后用途在下一篇文章中说吗，此处仅描述在实现这个需求时做的一些记录。

​		这个需求一听到时候就会下意识的觉得使用C/C++更合适一些，但部门技术栈为Java，这个项目也需要Web交互，相当于把这个功能嵌套在Web服务中提供使用。自然而然想到了JNA，让我自己用JNI去封装是不可能的。



## 二、JNA介绍

​		JNA（Java Native Access ）提供封装好的java函数用JNI来调用本地共享文件**.dll/.so**中的函数，极大的简化了我们的开发工作量。

​		下面的介绍引用自JNA的[官网](https://github.com/java-native-access/jna)：

> JNA provides Java programs easy access to native shared libraries without writing anything but Java code - no JNI or native code is required. This functionality is comparable to Windows' Platform/Invoke and Python's ctypes.
>
> JNA allows you to call directly into native functions using natural Java method invocation. The Java call looks just like the call does in native code. Most calls require no special handling or configuration; no boilerplate or generated code is required.
>
> JNA uses a small JNI library stub to dynamically invoke native code. The developer uses a Java interface to describe functions and structures in the target native library. This makes it quite easy to take advantage of native platform features without incurring the high overhead of configuring and building JNI code for multiple platforms. Read this [more in-depth description](https://github.com/java-native-access/jna/blob/master/www/FunctionalDescription.md).
>
> In addition, JNA includes a platform library with many native functions already mapped as well as a set of utility interfaces that simplify native access.



​		简单来说就是引入jar包后就可以直接使用java代码来操作本来需要通过JNI封装后才可以进行的操作。同样想到一个之前用过的比较类似的Java工具包[bytedeco](https://github.com/bytedeco/javacpp-presets)，封装了多种非Java平台的但是英语极为广泛的工具，例如：OpenCV、FFmpeg等。说句题外话，这些工具都很好用，拿OpenCV来说，直接引入官方包时，Windows开发时需要将本地dll文件放置在系统Path下，导入对应版本的jar包，再通过`System.loadLibrary()`引入。当部署到服务器时需要将opencv换成Linux下的so文件，非常的麻烦。直接引入bytedeco下对应的包会自动识别所在平台导入引入对应的文件。个人在使用期间感觉区别就是编写习惯不同，官网的jar包更贴近Java的编码和命名习惯，bytedeco更贴近c++的习惯些，其实影响都不大，反而bytedeco在部署期间更方便些，就是打包后的文件比较大，可以自己定制。

​	说远了，通过引入这些第三方的工具包确实能提高我们的开发效率，简化部署操作。







windowinfo 类的文档
https://docs.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-windowinfo

HWND 文档
https://support.microsoft.com/en-ie/help/124103/how-to-obtain-a-console-window-handle-hwnd

Windows编程之hdc和hwnd的区别
https://blog.csdn.net/wumenglu1018/article/details/52832321