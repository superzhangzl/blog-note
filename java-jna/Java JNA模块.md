# Java之JNA

## 一、记录背景	

​		关于项目内容还是需要保密的，只描述一下使用到JNA技术的需求点。在Windows平台下，获取指定应用程序的窗体信息，例如长宽、坐标等，具体获取后用途在下一篇文章中说明，此处仅描述在实现这个需求时做的一些记录。

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



​		简单来说就是引入jar包后就可以直接使用java代码来操作本来需要通过JNI封装后才可以进行的操作。同样想到一个之前用过的比较类似的Java工具包集合[bytedeco](https://github.com/bytedeco/javacpp-presets)，封装了多种非Java平台的但是英语极为广泛的工具，例如：OpenCV、FFmpeg等。说句题外话，这些工具都很好用，拿OpenCV来说，直接引入官方包时，Windows开发时需要将本地dll文件放置在系统Path下，导入对应版本的jar包，再通过`System.loadLibrary()`引入。当部署到服务器时需要将opencv换成Linux下的so文件，非常的麻烦。直接引入bytedeco下对应的包会自动识别所在平台导入引入对应的文件。个人在使用期间感觉区别就是编写习惯不同，官网的jar包更贴近Java的编码和命名习惯，bytedeco更贴近c++的习惯些，其实影响都不大，反而bytedeco在部署期间更方便些，就是打包后的文件比较大，可以自己定制。

​	说远了，通过引入这些第三方的工具包确实能提高我们的开发效率，简化部署操作。

### 2.1 引入

**说明：** 代码或配置均基于`v5.3.1`版本，在windows平台下通过测试，其他平台的适用性不保证。

Maven项目在`pom.xml`中引入如下配置即可：

```xml
<dependency>
  <groupId>net.java.dev.jna</groupId>
  <artifactId>jna-platform</artifactId>
  <version>5.3.1</version>
</dependency>
<dependency>
  <groupId>net.java.dev.jna</groupId>
  <artifactId>jna</artifactId>
  <version>5.3.1</version>
</dependency>
```

若是普通的java项目只需要将上述两个jar包添加到classpath中即可。



## 三、示例一：获取进程号

需求：根据接口请求打开对应的应用程序，获取对应的pid，并在其他接口操作对应的pid。

在java中执行命令时，通常可以使用`Runtime.getRuntime().exec(cmd)`的多个重载方法，获得`Process`进程句柄。当需要获得该进程的进程号时，该类并没有提供对应的方法。通过JNA提供的jar包可以使用如下的方法获取指定`Process`的进程号：

```java
/**
 * need jna-platform.jar
 *
 * @param  process
 * @return pid
 */
public static Long getPid(Process process) {
  if (process.getClass().getName().equals("java.lang.Win32Process")
      || process.getClass().getName().equals("java.lang.ProcessImpl")) {
    /* determine the pid on windows plattforms */
    try {
      Field f = process.getClass().getDeclaredField("handle");
      f.setAccessible(true);
      long handl = f.getLong(process);
      Kernel32 kernel = Kernel32.INSTANCE;
      WinNT.HANDLE handle = new WinNT.HANDLE();
      handle.setPointer(Pointer.createConstant(handl));
      int ret = kernel.GetProcessId(handle);
      return (long) ret;
    } catch (Throwable e) {
      e.printStackTrace();
    }
  }
  return null;
}
```



顺便提供一个windows平台下判断进程是否存在的工具方法：

```java
public static void checkProcess(String processName, long pid) {
  try {
    // windows下的命令，类似于linux下的ps
    String[] cmd = {"tasklist"};
    // 格式：
    // processName   pid     another
    Process process = Runtime.getRuntime().exec(cmd);
    InputStream inputStream = process.getInputStream();
    List<String> lines = new BufferedReader(new InputStreamReader(inputStream))
      .lines()
      .collect(Collectors.toList());
    for (String line : lines) {
      // 在此处执行判断操作，此处的进程名称为具体的可执行文件的名称，而不是应用程序名称
      if (line.matches(...)) {
        // 截取pid
        Long pid = Long.parseLong(line.replaceAll("\\s+", " ").split(" ")[1]);
        // 可以对匹配的程序执行kill操作，也可以在判断成功以后返回boolean类型的状态
        String[] kill = {"taskkill", "/F", "/PID", pid.toString()};
        Process killProcess = Runtime.getRuntime().exec(kill);
      }
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```



## 四、示例二：获取窗体信息

项目需求：获取指定应用程序窗体的信息，包括标题、实际长宽、标题栏高度（因为其他程序需要使用这些参数）。同时设置该应用程序窗体移动到左上角的位置并且置顶。

```java
public class WindowInfo {
    private String title;
    private Integer width;
    private Integer height;
    private Integer titleHeight;

    //设置一个空的类，只是为了防止出现NPE
    private static final WindowInfo empty = new WindowInfo("", 0, 0, 0);

    public static WindowInfo getEmpty() {
        return empty;
    }
    // ...
}
```

```java
/**
 * 获取windows窗口title的工具类
 * 仅在windows平台下单个显示器进行了测试
 *
 * @author zzl
 */
public class GetWindowsTitleUtil {
    private static Log logger = LogFactory.getLog(GetWindowsTitleUtil.class);
		// 指定窗体移动的位置坐标
    private static final int fixedX = 0;
    private static final int fixedY = 0;
    // 预定义一些使用到的方法，此接口定义其实可以省略，直接使用User32中的方法，作为第二种方法记录一下
    protected interface User32 extends StdCallLibrary {
        User32 INSTANCE = (User32) Native.loadLibrary("user32", User32.class);

        boolean EnumWindows(WinUser.WNDENUMPROC lpEnumFunc, Pointer arg);

        boolean ShowWindow(WinDef.HWND hwnd, int flag);

        int GetWindowTextA(WinDef.HWND hWnd, byte[] lpString, int nMaxCount);

        void GetWindowInfo(WinDef.HWND hwnd, WinUser.WINDOWINFO windowinfo);

        void GetWindowPlacement(WinDef.HWND hwnd, WinUser.WINDOWPLACEMENT wPlacement);

        // 0 0   0x0040 | 0x0001
        // 最后三个参数作用不明，暂时按照上面注释里的三个参数传输
        void SetWindowPos(WinDef.HWND hwnd, WinDef.HWND newHwnd, int left, int top, int x1, int x2, int x3);
    }

    //默认为空对象，否则会在加锁的时候出现npe
    private static volatile WindowInfo info = WindowInfo.getEmpty();

    /**
     * 支持子串匹配，若无则返回空对象，不至于让******那边出错，
     * 但是为了防止出现空指针异常的情况，就返回为空对象
     *
     * @param name
     * @return
     */
    public static WindowInfo getTitleByName(String name) {
        //对title加锁，判断之前设为null，判断之后设为null
        //不过按照当前使用场景是应该不会出现并发问题，但是作为工具还是要处理一下
        //title必须设为static，不然匿名类内部无法设置到
        synchronized (info) {
            info = WindowInfo.getEmpty();
            final User32 user32 = User32.INSTANCE;
            // 遍历任务栏中存在的窗体，不论是否最小化。后台任务不计算，
            user32.EnumWindows((hWnd, arg1) -> {
                byte[] windowText = new byte[512];
                // 若使用User32内置方法，则需要预先定义为char[]类型
                user32.GetWindowTextA(hWnd, windowText, 512);
                String wText = null;
                wText = Native.toString(windowText, Charset.forName("gbk"));
                if (wText.isEmpty()) {
                    return true;
                }
                // 当进程名字有重复的情况，按照系统读取的最后一个
                if (wText.contains(name) &&
                        //以下为不包含需要过滤的情况，比如出现了浏览器的情况
                        !wText.contains("启动") &&
                        !wText.contains("搜索") &&
                        !wText.contains("=") &&
                        !wText.contains("Firefox") &&
                        !wText.contains("Chrome")
                ) {
                    // 当应用窗口可能最小化时，窗体的长宽是读取不到的，而且录屏也是检测不到的
                    // 此时就需要将窗体设置为前台显示状态且置顶
                    // 注：此时可能出现，java使用非管理员指定，而指定任务使用管理员权限，会出现无法设置为前台运行的状况
                    // 解决方法：使用管理员权限执行该任务即可！！！！
                    int show = WinUser.SW_RESTORE;
                    user32.ShowWindow(hWnd, show);

                    WinUser.WINDOWINFO windowinfo = new WinUser.WINDOWINFO();
                    user32.GetWindowInfo(hWnd, windowinfo);
                    // 运行窗体展示的长宽，不包括标题栏和border(不可见)
                    int width = windowinfo.rcClient.right - windowinfo.rcClient.left;
                    int height = windowinfo.rcClient.bottom - windowinfo.rcClient.top;
                    int fullHeight = windowinfo.rcWindow.bottom - windowinfo.rcWindow.top;
                    // 计算标题栏高度
                    int titleHeight = fullHeight - height - windowinfo.cyWindowBorders;
                    info = new WindowInfo(wText, width, height, titleHeight);
                    logger.info(info);

//                    user32.ShowWindow(hWnd, 1); //显示窗体，适用于最小化，或者处于z轴后面的窗体置前 //获取位置
                    WinUser.WINDOWPLACEMENT wPlacement = new WinUser.WINDOWPLACEMENT();
                    user32.GetWindowPlacement(hWnd, wPlacement);
                    // 移动位置到左上角，不会出现窗体局部隐藏的情况
                    user32.SetWindowPos(hWnd, new WinDef.HWND(new Pointer(-1)), fixedX, fixedY, 0, 0, 0x0040 | 0x0001);
                    return true;
                }
                return true;
            }, null);
            WindowInfo result = info;
            info = WindowInfo.getEmpty();
            return result;
        }
    }

    public static void main(String[] args) {
        WindowInfo name = getTitleByName("应用程序");
        System.out.println(name);
    }
}

```



// TODO 画图表示窗体信息



## 五、示例三：获取显示设备信息💻

// TODO 





// TODO 将demo项目过滤后打包发布到Github

​		上面的工具方法仅仅只是展示了JNA一个简单的应用，实际上能操作的更多，桌面应用程序开发有关的都可以使用，JNA官网中也有一些具体的示例及应用(比如我大IDEA😊)。

​		需要注意的是使用上的**习惯**，java调用时是将参数传入方法中，然后通过return获得结果。而JNA中大多（项目中使用到的）是预先定义好结果变量（指针），再将该变量一起作为参数传入方法中，方法的结果会赋值给该指针所指向的变量。所以在思维上需要进行一定的转变，要理解C系中指针这一重要概念。



参考文档：

- [WinUser.WINDOWINFO 类的文档](https://docs.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-windowinfo)

- [HWND 文档](https://support.microsoft.com/en-ie/help/124103/how-to-obtain-a-console-window-handle-hwnd)

- [Windows编程之hdc和hwnd的区别](https://blog.csdn.net/wumenglu1018/article/details/52832321)

- [JNA—JNI终结者](https://blog.csdn.net/jiangyu1013/article/details/80409897)

- [使用JNA访问WindowsAPI操作Windows窗口元素](https://www.cnblogs.com/yumingle/p/6776258.html)

- [利用jna-platform.jar调取窗口及安装键盘钩子](https://blog.csdn.net/Shenpibaipao/article/details/79220794?utm_source=blogxgwz9)