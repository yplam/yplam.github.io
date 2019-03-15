---
title:  "Chromium 源码阅读：chrome/browser/themes"
categories: Chromium
---

前言：最近花了几天时间解决 Chromium CRX3 文件格式编码问题，然后决定用部分业余时间学习 Chromium 源码，从 UI/Theme/Extension 开始。（因为7,8年前曾经马马虎虎地看了一遍 Theme 以及其渲染过程的代码，并且写了一个简单的[ Theme 打包工具](https://www.themebeta.com/chrome-theme-creator-online.html)，所以现在决定还是以此作为入口。）

**chrome/browser/themes/**

chrome/browser/themes/ 目录包含跟 Theme 格式定义相关代码，Theme 加载相关代码，以及全局的 ThemeService，但并不包含关于 Theme 如何渲染浏览器外观的代码,不包含关于 CRX 打包解包相关代码。

### 文件概览

**CustomThemeSupplier**
```
custom_theme_supplier.cc
custom_theme_supplier.h
```
CustomThemeSupplier 是 Theme 表示的基类，并没有提供有效的实现。

**BrowserThemePack**
```
browser_theme_pack.cc
browser_theme_pack.h
browser_theme_pack_unittest.cc
```
继承自 CustomThemeSupplier 使用DataPack优化过的Theme表示方式，在Theme安装的时候对图片等进行预处理，然后保存成mmappable文件，用来提升性能。

**IncreasedContrastThemeSupplier**
```
increased_contrast_theme_supplier.cc
increased_contrast_theme_supplier.h
```
继承自 CustomThemeSupplier，提供高对比度的 Theme

**ThemeProperties**
```
theme_properties.cc
theme_properties.h
theme_properties_unittest.cc
```
ThemeProperties 提供 Theme 相关的属性定义，**并提供默认值**，Static only class for querying which properties / images are themeable and the defaults of these properties.

**ThemeService**
```
theme_service.h
theme_service.cc
theme_service_unittest.cc
theme_service_browsertest.cc
```
ThemeService 为 Theme 服务类，负责加载用户设置的 Theme，删除无用的 Theme（监听 extensions::NOTIFICATION_EXTENSIONS_READY_DEPRECATED， 并且延时后），启动 Theme 同步服务。
并且响应 OnExtensionLoaded 事件，更新 Theme。同时为 UI 层提供 ThemeProvide。

**ThemeServiceWin**
```
theme_service_win.cc
theme_service_win.h
```
ThemeServiceWin 继承自ThemeService，针对 Windows 10 的颜色做相关处理。

**ThemeServiceAuraX11**
```
theme_service_aurax11.cc
theme_service_aurax11.h
```
ThemeServiceAuraX11 继承自ThemeService，提供原生的 X11 theme 支持。

**ThemeServiceFactory**
```
theme_service_factory.cc
theme_service_factory.h
```
ThemeServiceFactory 为工厂模式单例，使用者通过 ThemeServiceFactory::GetForProfile 获取当前使用的 ThemeService。

**ThemeSyncableService**
```
theme_syncable_service.cc
theme_syncable_service.h
theme_syncable_service_unittest.cc
```
ThemeSyncableService 为 Theme 同步服务，本地安装新 Theme 后同步到后端，并且响应后端同步信息的改变。


### 一些流程

**为 UI 层提供数据**

浏览器 UI 层，如 chrome/browser/ui/views/bookmarks/bookmark_bar_view.cc 均继承自 views::View (ui/views/view.h) 所有 View 都可以通过 ui::ThemeProvider* GetThemeProvider() ，获取 ThemeProvider 句柄，然后通过 ThemeProvider 获取经过 Theme 后的 UI 颜色、图片等。GetThemeProvider 最终会调用 ThemeService::GetThemeProviderForProfile 或者 ThemeService::GetDefaultThemeProviderForProfile，返回的为 ThemeService 的内部类： 
```
class BrowserThemeProvider : public ui::ThemeProvider
```
在此完成 ThemeService 与 ui::ThemeProvider 的解耦与调用。

**Theme 安装流程**

ThemeService 通过加载安装的 Theme，通过 BrowserThemeProvider 内部类提供 UI 层所需的数据，然后通过内部类：
```
class ThemeService::ThemeObserver
    : public extensions::ExtensionRegistryObserver
```
与 Extension 子系统的的 ExtensionRegistry 关联，当安装新扩展时，ExtensionRegistry 服务就会回调所有 ExtensionRegistryObserver 的 OnExtensionLoaded 方法，如果此扩展为 Theme，则通过 theme service 更新。

ThemeService::ThemeObserver::OnExtensionLoaded 调用回溯：

```
#0  0x000055555916362b in ThemeService::ThemeObserver::OnExtensionLoaded(content::BrowserContext*, extensions::Extension const*)
    (this=0x3eac7df9acd0, browser_context=0x3eac7ca81ba0, extension=0x3eac7ebd1fe0) at ../../chrome/browser/themes/theme_service.cc:264
#1  0x000055555794596c in extensions::ExtensionRegistry::TriggerOnLoaded(extensions::Extension const*) (this=0x3eac7cad7a20, extension=0x3eac7ebd1fe0) at ../../extensions/browser/extension_registry.cc:64
#2  0x0000555557941c1e in extensions::ExtensionRegistrar::ActivateExtension(extensions::Extension const*, bool) (this=0x3eac7c3b2a60, extension=0x3eac7ebd1fe0, is_newly_added=true)
    at ../../extensions/browser/extension_registrar.cc:450
#3  0x00005555579419df in extensions::ExtensionRegistrar::AddNewExtension(scoped_refptr<extensions::Extension const>) (this=0x3eac7c3b2a60, extension=...)
    at ../../extensions/browser/extension_registrar.cc:140
#4  0x0000555557940bf1 in extensions::ExtensionRegistrar::AddExtension(scoped_refptr<extensions::Extension const>) (this=0x3eac7c3b2a60, extension=...)
    at ../../extensions/browser/extension_registrar.cc:105
#5  0x000055555a1f88d3 in extensions::ExtensionService::AddExtension(extensions::Extension const*) (this=0x3eac7c3b2820, extension=0x3eac7ebd1fe0)
    at ../../chrome/browser/extensions/extension_service.cc:1188
#6  0x000055555a1fad83 in extensions::ExtensionService::FinishInstallation(extensions::Extension const*) (this=0x3eac7c3b2820, extension=0x3eac7ebd1fe0)
    at ../../chrome/browser/extensions/extension_service.cc:1617
#7  0x000055555a1f90c5 in extensions::ExtensionService::AddNewOrUpdatedExtension(extensions::Extension const*, extensions::Extension::State, int, syncer::Ordinal<syncer::StringOrdinalTraits> const&, std::__
Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > const&, base::Optional<int> const&)
    (this=0x3eac7c3b2820, extension=0x3eac7ebd1fe0, initial_state=extensions::Extension::ENABLED, install_flags=8, page_ordinal=..., install_parameter=..., dnr_ruleset_checksum=...)
    at ../../chrome/browser/extensions/extension_service.cc:1557
#8  0x000055555a1fa268 in extensions::ExtensionService::OnExtensionInstalled(extensions::Extension const*, syncer::Ordinal<syncer::StringOrdinalTraits> const&, int, base::Optional<int> const&)
    (this=0x3eac7c3b2820, extension=0x3eac7ebd1fe0, page_ordinal=..., install_flags=8, dnr_ruleset_checksum=...) at ../../chrome/browser/extensions/extension_service.cc:1484
#9  0x000055555a1ba42a in extensions::CrxInstaller::ReportSuccessFromUIThread() (this=0x3eac7f4ed020) at ../../chrome/browser/extensions/crx_installer.cc:956

```



