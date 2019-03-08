---
title:  "Chromium 编译与调试笔记"
categories: Chromium
---

最近手上有个小项目需要兼容 Chrome CRX3 版本的扩展格式，Google 了一下竟然完全没有相关介绍文档，于是决定啃一下相关代码。

本文会持续更新，最后更新时间： 2019-03-06，如有问题欢迎联系 yplam(at)yplam.com。

上次编译 Chromium 已经是 N 年前的事情，那时伟大的墙还没现在那么大存在感，按官方步骤编译就可以，然而现时（2019年2月）按官方文档竟然连代码都下不下来。本文作为此过程的记录（主要是解决墙的问题）。

环境：P52(CPU i7-8850H + 16GB内存 + 250GB SSD) + Manjaro 18 Linux（如果你使用的是 Windows 环境，可以参考本文末尾关于 Windows 下的编译方法）。

环境要求：
* 64位系统，需要 8GB 内存（建议16G以上）跟 100GB 硬盘，经测试编译后大概用掉60GB空间，但按默认选项用掉的空间应该会多点。
* 需要保证有一个高速的科学上网环境，并且需要提供 http proxy，源码大概有 30GB。
* git
* python2，经测试 virtualenv 下的 python2 无法使用，可能是因为某些脚步会调用 bash，跳出 virtualenv 的环境。（如果您解决了这个问题，欢迎反馈，谢谢）

### 编译 Chromium

基本上按照[官方文档](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md),只是开始前解决 proxy 跟 python 版本问题。

**注意：下面过程可能会对您操作系统造成不可知的错误，请仅作为参考**

先查看系统 python 版本：

```
python --version
```

如果不是 2.x 则可以通过建立软链的方式切换到 Python2（或者使用你系统自带的切换工具）

```
cd $(dirname $(which python)) && sudo rm python && sudo ln -s python2 python
```

假设 http proxy 端口为 8123，创建 boto.cfg 文件，记下保存路径：

```
[Boto]
proxy=127.0.0.1
proxy_port=8123
```

配置 proxy 相关内容：

```
export http_proxy="127.0.0.1:8123"
export https_proxy="127.0.0.1:8123"
export NO_AUTH_BOTO_CONFIG={your path}/boto.cfg
git config --global http.proxy 127.0.0.1:8123
```

至此，可以按照[官方文档](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md)进行编译。

下面为相关命令，运行前需要先保证有稳定的网络，以及接上外部电源，在我的 P52 上下载源码大概需要 2小时，编译大概需要 3 小时：

**注意：下面为Manjaro下的编译方式，其他系统请按需修改或者参考官方文档。**

```
sudo pacman -S --needed python perl gcc gcc-libs bison flex gperf pkgconfig nss alsa-lib glib2 gtk3 nspr ttf-ms-fonts freetype2 cairo dbus libgnome-keyring
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH:/path/to/depot_tools"
mkdir chromium && cd chromium
fetch --nohooks chromium
cd src
gclient runhooks
gn gen --enable_nacl=false --symbol_level=1 --remove_webcore_debug_symbols=true out/Default
autoninja -C out/Default chrome
git config --global --unset-all http.proxy
cd $(dirname $(which python)) && sudo rm python && sudo ln -s python3 python

```

如无意外会生成 out/Default/chrome ,尝试一下是否可以正常运行 ./out/Default/chrome --no-sandbox

### 更新代码

更新代码的方式与上面类似，主要是解决 proxy 问题：

```
cd $(dirname $(which python)) && sudo rm python && sudo ln -s python2 python
export http_proxy="127.0.0.1:8123"
export https_proxy="127.0.0.1:8123"
export NO_AUTH_BOTO_CONFIG=/home/yplam/work/chromium/boto.cfg
git config --global http.proxy 127.0.0.1:8123
git rebase-update
gclient sync
git config --global --unset-all http.proxy
cd $(dirname $(which python)) && sudo rm python && sudo ln -s python3 python
```

### 编译与运行TEST

使用 gn ls 命令可以列出所有可编译的 target，譬如，我想跑跑 ui/gfx/color_utils_unittest.cc ，那么可以使用类似命令找出来:
```
gn ls out/Default | grep test | grep ui | grep gfx
```

可以看到 //ui/gfx:gfx_unittests 的存在， gfx_unittests 定义在 ui/gfx/BUILD.gn，包含 color_utils_unittest.cc。

可以通过下面命令编译与运行：

```
ninja -C out/Default ui/gfx:gfx_unittests
out/Default/gfx_unittests --gtest_filter="ColorUtils.*"
```

因为编译 TEST 比编译整个 Chromium 要快得多，可以通过此方式比较简单的了解与跟踪某些代码的运行。

### GDB 调试 Chromium

官方文档在此：[https://chromium.googlesource.com/chromium/src/+/master/docs/linux_debugging.md](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_debugging.md)

由于GDB启动较慢，如果只是想简单的跟踪一下程序运行信息，可以直接使用 base/logging.h 中定义的方法，如：
```
LOG(INFO) << "Found " << num_cookies << " cookies";
```

再结合上面提到的 TEST，用来跟踪一些小功能的代码比较有效率。当然，如果要对浏览器运行时的状态进行细致一点的跟踪，那么应该还是用 gdb 好点。

譬如，chrome/browser/themes/theme_service.cc 提供 Theme 服务类，通过阅读代码，可以知道对 Theme 的更新是通过外部调用 ThemeObserver 的 OnExtensionLoaded 方法更新的，但对于是如何调用 OnExtensionLoaded 的
却不是那么容易可以看出来，那么我们可以简单的使用 GDB 的 backtrace 来查看。

在开始调试前请保证你的电脑有足够内存（应该需要 10GB+ 的内存，反正我 16GB 内存如果运行其他应用的话有时会内存耗尽），在 src 目录运行下面命令：

```
gdb -tui --args out/Default/chrome --disable-seccomp-sandbox http://google.com
```

加载需要点时间，加载后设置断点，在 theme_service.cc 的 264 行设置断点：

```
b theme_service.cc:264
```

然后运行

```
r
```

启动过程中可能遇到其他断点，使用

```
c
```

继续运行。

然后进入 extension 页面，打开 developer 模式，拖一个 crx Theme 扩展进去，此时会进入我们的断点，然后

```
bt
```

或者如果只需要看最近8个 backtrace：

```
bt 8
```

得到下面关于 ThemeService::ThemeObserver::OnExtensionLoaded 的调用回溯：
```
#0  0x000055555916362b in ThemeService::ThemeObserver::OnExtensionLoaded(content::BrowserContext*, extensions::Extension const*)
    (this=0x30e87097be90, browser_context=0x30e86f3abca0, extension=0x30e871bd6260) at ../../chrome/browser/themes/theme_service.cc:264
#1  0x000055555794596c in extensions::ExtensionRegistry::TriggerOnLoaded(extensions::Extension const*) (this=0x30e86f412a20, extension=0x30e871bd6260) at ../../extensions/browser/extension_registry.cc:64
#2  0x0000555557941c1e in extensions::ExtensionRegistrar::ActivateExtension(extensions::Extension const*, bool) (this=0x30e86f9dda60, extension=0x30e871bd6260, is_newly_added=true)
    at ../../extensions/browser/extension_registrar.cc:450)] 244
#3  0x00005555579419df in extensions::ExtensionRegistrar::AddNewExtension(scoped_refptr<extensions::Extension const>) (this=0x30e86f9dda60, extension=...)
    at ../../extensions/browser/extension_registrar.cc:140
#4  0x0000555557940bf1 in extensions::ExtensionRegistrar::AddExtension(scoped_refptr<extensions::Extension const>) (this=0x30e86f9dda60, extension=...)
    at ../../extensions/browser/extension_registrar.cc:105
#5  0x000055555a1f88d3 in extensions::ExtensionService::AddExtension(extensions::Extension const*) (this=0x30e86f9dd820, extension=0x30e871bd6260)
    at ../../chrome/browser/extensions/extension_service.cc:1188
#6  0x000055555a1fad83 in extensions::ExtensionService::FinishInstallation(extensions::Extension const*) (this=0x30e86f9dd820, extension=0x30e871bd6260)
    at ../../chrome/browser/extensions/extension_service.cc:1617
#7  0x000055555a1f90c5 in extensions::ExtensionService::AddNewOrUpdatedExtension(extensions::Extension const*, extensions::Extension::State, int, syncer::Ordinal<syncer::StringOrdinalTraits> const&, std::_
_Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > const&, base::Optional<int> const&)
    (this=0x30e86f9dd820, extension=0x30e871bd6260, initial_state=extensions::Extension::ENABLED, install_flags=0, page_ordinal=..., install_parameter=..., dnr_ruleset_checksum=...)

```

我们继续执行：

```
c
```
完成关闭浏览器退出，在 gdb 输入：

```
q
```
退出。

因为对 GDB 也不熟悉，上面只演示了一个基础操作。（ps: GDB的官方文档较枯燥，在网上找找别人的使用心得分享，譬如[100-gdb-tips](https://www.kancloud.cn/itfanr/i-100-gdb-tips/81849), [gdb Debugging Full Example](http://www.brendangregg.com/blog/2016-08-09/gdb-example-ncurses.html)）

### 关于CRX3格式的调试结果记录

CRX3格式相关代码位于：

```
components/crx_file/crx_creator.cc
```

line 62：

```
CreatorResult Create(const base::FilePath& output_path,
                     const base::FilePath& zip_path,
                     crypto::RSAPrivateKey* signing_key) 
```

跟 CRX2 最大的区别在于 CrxFileHeader 部分现在不再是简单的 length+data 的格式，而是使用 Protobuf 编码后的二进制数据，Protobuf格式为 crx3.proto 所示：

```
// Copyright 2017 The Chromium Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file

syntax = "proto2";

option optimize_for = LITE_RUNTIME;

package crx_file;

// A CRX₃ file is a binary file of the following format:
// [4 octets]: "Cr24", a magic number.
// [4 octets]: The version of the *.crx file format used (currently 3).
// [4 octets]: N, little-endian, the length of the header section.
// [N octets]: The header (the binary encoding of a CrxFileHeader).
// [M octets]: The ZIP archive.
// Clients should reject CRX₃ files that contain an N that is too large for the
// client to safely handle in memory.

message CrxFileHeader {
  // PSS signature with RSA public key. The public key is formatted as a
  // X.509 SubjectPublicKeyInfo block, as in CRX₂. In the common case of a
  // developer key proof, the first 128 bits of the SHA-256 hash of the
  // public key must equal the crx_id.
  repeated AsymmetricKeyProof sha256_with_rsa = 2;

  // ECDSA signature, using the NIST P-256 curve. Public key appears in
  // named-curve format.
  // The pinned algorithm will be this, at least on 2017-01-01.
  repeated AsymmetricKeyProof sha256_with_ecdsa = 3;

  // The binary form of a SignedData message. We do not use a nested
  // SignedData message, as handlers of this message must verify the proofs
  // on exactly these bytes, so it is convenient to parse in two steps.
  //
  // All proofs in this CrxFile message are on the value
  // "CRX3 SignedData\x00" + signed_header_size + signed_header_data +
  // archive, where "\x00" indicates an octet with value 0, "CRX3 SignedData"
  // is encoded using UTF-8, signed_header_size is the size in octets of the
  // contents of this field and is encoded using 4 octets in little-endian
  // order, signed_header_data is exactly the content of this field, and
  // archive is the remaining contents of the file following the header.
  optional bytes signed_header_data = 10000;
}

message AsymmetricKeyProof {
  optional bytes public_key = 1;
  optional bytes signature = 2;
}

message SignedData {
  // This is simple binary, not UTF-8 encoded mpdecimal; i.e. it is exactly
  // 16 bytes long.
  optional bytes crx_id = 1;
}


```

其中 crx_id 为 RSA public key 的 sha256 编码后的前 16 byte。signed_header_data 为 "CRX3 SignedData\x00" + signed_header_size + signed_header_data + zip archive 的 OPENSSL_ALGO_SHA256 编码（使用私钥）

关于 Protobuf 可以使用 Google 官方库，然而由于在此只使用了 bytes 数据类型，操作起来比较简单，可以参考 [https://developers.google.com/protocol-buffers/docs/encoding](https://developers.google.com/protocol-buffers/docs/encoding) 直接对二进制流进行读写操作。

### Windows 环境下编译 Chromium

下面附带分享一下 Windows 环境编译 Chromium 的附加内容。（2019年1月经测试可以正常下载代码与编译）

首先第一步是能够解决科学上网，并且网速要够快。

下载解压 depot_tools 后，先配置proxy，用管理员打开命令行

```
netsh winhttp set proxy 127.0.0.1:1080
```

修改depot_tools 目录中的 cipd.ps1 文件，配置 System.Net.WebClient 使用代理

```
...
$proxyAddr = "http://127.0.0.1：1080"
$proxy = new-object System.Net.WebProxy
$proxy.Address = $proxyAddr
...
$wc.proxy = $proxy
...
```

可能还需要添加这两个命令行变量，不然初次运行gclient可能会报错：

```
set HTTP_PROXY=http://127.0.0.1:1080
set HTTPS_PROXY=%HTTP_PROXY%
```

按官方文档继续配置 depot_tools 

结束后复位一下代理：

```
netsh winhttp reset proxy
```

### 后记

折腾了几天 Chromium，解决了 CRX3 文件格式编码问题，突然有点不想就此打住的念头。Chromium 算是自己能遇到的最复杂最优秀的开源项目以及代码库了吧，难得有机会，要不继续学习下去？ Why not？

### 参考资料

* [Checking out and building Chromium on Linux](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md)
* [Tips for debugging on Linux](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_debugging.md)
* [Sync Chromium src behind proxy](https://colinxu.wordpress.com/2018/04/28/sync-chromium-src-behind-proxy/)
* [http://blog.gclxry.com/](http://blog.gclxry.com/)



