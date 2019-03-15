---
title:  "Chromium 源码阅读：chrome/browser/ui/views 入门概览"
categories: Chromium
---

从上一篇知道 chrome/browser/themes/ 的数据流为从 Extension Service 得到 Theme 安装信息，并且通过 BrowserThemeProvider 为 UI 层提供 Theme UI 信息。本篇分析 UI 的 views 层，不过由于 views 层代码量实在庞大，以及对 Chromium 内部实现机制欠缺了解，所以本文只能作为一个入门概览。

Chromium 官方有提供一个关于 Views 层调试的文档: [https://chromium.googlesource.com/chromium/src/+/master/docs/graphical_debugging_aid_chromium_views.md](https://chromium.googlesource.com/chromium/src/+/master/docs/graphical_debugging_aid_chromium_views.md) ,不过我用 gdb 的 viewg 始终没有合法的输出，所以最终使用直接代码输出 Log 的方式：

**2019-03-16 更新：** 可能是编写 viewg 脚本的开发者粗心或者脚本没及时更新，发现是错把 views 写成 view 了，于是我给官方提交了个 bug fix，然后成了 Chromium 代码贡献量最少的开发者（一个字母，哈哈）。

GDB 加载完成后使用此命令加载 viewg:

```
source tools/gdb/viewg.gdb
```

然后添加断点，只要是继承自 View 的类都可以，非静态方法:

```
break BrowserView::OnActiveTabChanged(content::WebContents*, content::WebContents*, int, int)
```

切换浏览器标签进入断点后，使用 viewg 命令，就会在个人目录生成 state.svg 文件，拖到浏览器就可以访问。

**以下为旧版本**

~~BrowserView(chrome/browser/ui/views/frame/browser_view.cc) 算是浏览器窗口 view 的根（继承自views::View），通过 BrowserView 可以打印出整个浏览器 views 树的结构。在 BrowserView 的非静态方法中加入：~~
```
LOG(WARNING) << "" << std::endl <<views::PrintViewGraph(this) << std::endl;
```
~~然后编译运行 chrome 就可以在终端日志中得到 views 树输出的 dot 文件内容，将其保存到 ”~/state.dot“，然后通过下面指令生成 svg 图片：~~
```
dot -Tsvg -o ~/state.svg ~/state.dot
```
**END旧版本**

注：图片如下，宽度超标，请拖到新窗口查看原图。

↓↓↓↓↓↓↓↓↓↓

![state.svg]({{ site.url }}/assets/2019/state.svg "state.svg" ){:class="img-responsive"}

↑↑↑↑↑↑↑↑↑↑

其中 bounds: (4, 5), (913x1021) 的含义是 x=4，y=5，width=912，height=1021，而那些处于隐藏状态的 view 这几个值中的一个或多个为 0。


