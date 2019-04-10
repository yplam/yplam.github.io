---
title:  "Chromium 源码阅读：Could not decode image bug 跟踪"
categories: Chromium, C++
---

偶尔有收到用户关于 "Could not decode image: ***" 的反馈，并且最近频率有增加的趋势，解开
Theme 文件除了图片有点大以外看不出其他问题，所以决定花点时间跟踪一下 Chromium 源码。

本来以为是一个非常简单的问题，只需要找到报错信息，然后打印 backtrace，再断点跟踪一下。然而由于 
Chromium 多线程的特性，以及 crx 文件的安装是在 sandbox 中进行的，断点到出错信息，得到的是下面
的 backtrace：

```
#0  0x0000555557b3e373 in extensions::SandboxedUnpacker::ImageSanitizationDone(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&) (this=0x2f1d9794c020, manifest=..., status=extensions::ImageSanitizer::Status::kDecodingError, file_path_for_error=...) at ../../extensions/browser/sandboxed_unpacker.cc:548
#1  0x0000555557b430f9 in base::internal::FunctorTraits<void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), void>::Invoke<void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), scoped_refptr<extensions::SandboxedUnpacker>, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&>(void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), scoped_refptr<extensions::SandboxedUnpacker>&&, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >&&, extensions::ImageSanitizer::Status&&, base::FilePath const&) (method=
    (void (extensions::SandboxedUnpacker::*)(extensions::SandboxedUnpacker * const, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, const base::FilePath &)) 0x555557b3e340 <extensions::SandboxedUnpacker::ImageSanitizationDone(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&)>, receiver_ptr=..., args=..., args=@0x7fffbb8afb14: extensions::ImageSanitizer::Status::kDecodingError, args=...) at ../../base/bind_internal.h:499
#2  0x0000555557b43020 in base::internal::InvokeHelper<false, void>::MakeItSo<void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), scoped_refptr<extensions::SandboxedUnpacker>, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&>(void (extensions::SandboxedUnpacker::*&&)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), scoped_refptr<extensions::SandboxedUnpacker>&&, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >&&, extensions::ImageSanitizer::Status&&, base::FilePath const&) (functor=
    @0x2f1d9880ebc0: (void (extensions::SandboxedUnpacker::*)(extensions::SandboxedUnpacker * const, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, const base::FilePath &)) 0x555557b3e340 <extensions::SandboxedUnpacker::ImageSanitizationDone(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&)>, args=..., args=..., args=@0x7fffbb8afb14: extensions::ImageSanitizer::Status::kDecodingError, args=...)
    at ../../base/bind_internal.h:599
#3  0x0000555557b42f7d in base::internal::Invoker<base::internal::BindState<void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), scoped_refptr<extensions::SandboxedUnpacker>, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> > >, void (extensions::ImageSanitizer::Status, base::FilePath const&)>::RunImpl<void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), std::__Cr::tuple<scoped_refptr<extensions::SandboxedUnpacker>, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> > >, 0ul, 1ul>(void (extensions::SandboxedUnpacker::*&&)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), std::__Cr::tuple<scoped_refptr<extensions::SandboxedUnpacker>, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> > >&&, std::__Cr::integer_sequence<unsigned long, 0ul, 1ul>, extensions::ImageSanitizer::Status&&, base::FilePath const&) (functor=
    @0x2f1d9880ebc0: (void (extensions::SandboxedUnpacker::*)(extensions::SandboxedUnpacker * const, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, const base::FilePath &)) 0x555557b3e340 <extensions::SandboxedUnpacker::ImageSanitizationDone(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&)>, bound=..., unbound_args=@0x7fffbb8afb14: extensions::ImageSanitizer::Status::kDecodingError, unbound_args=...)
    at ../../base/bind_internal.h:672
#4  0x0000555557b42e5d in base::internal::Invoker<base::internal::BindState<void (extensions::SandboxedUnpacker::*)(std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> >, extensions::ImageSanitizer::Status, base::FilePath const&), scoped_refptr<extensions::SandboxedUnpacker>, std::__Cr::unique_ptr<base::DictionaryValue, std::__Cr::default_delete<base::DictionaryValue> > >, void (extensions::ImageSanitizer::Status, base::FilePath const&)>::RunOnce(base::internal::BindStateBase*, extensions::ImageSanitizer::Status, base::FilePath const&)
    (base=0x2f1d9880eba0, unbound_args=extensions::ImageSanitizer::Status::kDecodingError, unbound_args=...) at ../../base/bind_internal.h:641
#5  0x0000555557345787 in base::OnceCallback<void (unsigned int, std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > const&)>::Run(unsigned int, std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > const&) && (this=0x2f1d982f0418, args=4, args=...) at ../../base/callback.h:97
#6  0x0000555557b11de2 in extensions::ImageSanitizer::ReportError(extensions::ImageSanitizer::Status, base::FilePath const&)
    (this=0x2f1d982f03e0, status=extensions::ImageSanitizer::Status::kDecodingError, path=...) at ../../extensions/browser/image_sanitizer.cc:219
#7  0x0000555557b1203a in extensions::ImageSanitizer::ImageDecoded(base::FilePath const&, SkBitmap const&) (this=0x2f1d982f03e0, image_path=..., decoded_image=...)
    at ../../extensions/browser/image_sanitizer.cc:157
#8  0x0000555557326b0c in base::internal::FunctorTraits<void (OAuth2TokenService::RequestImpl::*)(GoogleServiceAuthError const&, OAuth2AccessTokenConsumer::TokenResponse const&), void>::Invoke<void (OAuth2TokenService::RequestImpl::*)(GoogleServiceAuthError const&, OAuth2AccessTokenConsumer::TokenResponse const&), base::WeakPtr<OAuth2TokenService::RequestImpl>, GoogleServiceAuthError, OAuth2AccessTokenConsumer::TokenResponse>(void (OAuth2TokenService::RequestImpl::*)(GoogleServiceAuthError const&, OAuth2AccessTokenConsumer::TokenResponse const&), base::WeakPtr<OAuth2TokenService::RequestImpl>&&, GoogleServiceAuthError&&, OAuth2AccessTokenConsumer::TokenResponse&&) (method=
    (void (OAuth2TokenService::RequestImpl::*)(OAuth2TokenService::RequestImpl * const, const GoogleServiceAuthError &, const OAuth2AccessTokenConsumer::TokenResponse &)) 0x555557b11fe0 <extensions::ImageSanitizer::ImageDecoded(base::FilePath const&, SkBitmap const&)>, receiver_ptr=..., args=..., args=...) at ../../base/bind_internal.h:499
```

除了知道报错是在 extensions::SandboxedUnpacker::ImageSanitizationDone 以外，其他都是一些
复杂的函数指针。

参考 Chromium 关于[线程与任务](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md)
，[回调](https://chromium.googlesource.com/chromium/src/+/master/docs/callback.md)
的文档。扩展安装属于发送到线程上的序列化任务，底层通过偏函数，并且由于使用 mojo，使得对调用流程的跟踪
异常麻烦。

最终只能通过阅读源码 + dump 的方式反推整个流程如下：

* DownloadItemImpl::OnDownloadCompleting 下载完成
* DownloadFileImpl::RenameAndAnnotate 重命名，然后回调
* DownloadItemImpl::OnDownloadRenamedToFinalName 下载完成并且完成重命名
* ChromeDownloadManagerDelegate::ShouldOpenDownload 检查到是 crx
* download_crx_util::OpenChromeExtension 打开 crx
* CrxInstaller::InstallCrx 安装 crx
* extensions::CrxInstaller::InstallCrxFile
* SandboxedUnpacker::StartWithCrx
* SandboxedUnpacker::Unzip
* SandboxedUnpacker::Unpack
* SandboxedUnpacker::ReadManifestDone
* ImageSanitizer::CreateAndStart
* ImageSanitizer::Start
* ImageSanitizer::ImageFileRead
* data_decoder::ImageDecoderImpl::DecodeImage

**结果：**

由于 extensions/browser/image_sanitizer.cc 中定义 const int kMaxImageCanvas = 4096 * 4096。
并且该值限制了所有 extension 中图片的大小，当图片大小 

```
x*y*4+(skia::mojom::Bitmap::Data_) > 4096 * 4096
```

时就会报错，然而由于现在 2k 甚至 4k 显示器的使用越来越多，当其使用一张全屏图作为主题背景时就会超出
此限制值。

ps:跟 Chromium 的开发人员反馈了一下，然而对方认为只是简单的增大 kMaxImageCanvas 并不合适，可能
需要分场景考虑，所以估计此问题短时间内不会解决。

**技巧：**

因为 Chromium 源码分层比较清晰，查找调用时只在模块内部查找可以提高效率。

如果源码中有调用 mojo 定义的服务，可以先查看 mojo 定义文件如：services/data_decoder/public/mojom/image_decoder.mojom
然后查看生成的接口定义 gen/services/data_decoder/public/mojom/image_decoder.mojom.h，
然后查找 include 了 image_decoder.mojom.h 的文件，一般可以找到该 mojo 服务的实现。


##一些 backtrace

由于 Chromium 多线程特性，在 gdb 中跟踪流程变得比较困难，下面是一些断点的 backtrace。

**extensions::CrxInstaller::InstallCrxFile**

```
#0  0x000055555a367bcd in extensions::CrxInstaller::InstallCrxFile(extensions::CRXFileInfo const&) (this=0x164ebed6d5a0, source_file=...) at ../../chrome/browser/extensions/crx_installer.cc:180
#1  0x000055555a367b6a in extensions::CrxInstaller::InstallCrx(base::FilePath const&) (this=0x164ebed6d5a0, source_file=...) at ../../chrome/browser/extensions/crx_installer.cc:176
#2  0x00005555590ae5f6 in download_crx_util::OpenChromeExtension(Profile*, download::DownloadItem const&) (profile=0x164ebcaae020, download_item=...)
    at ../../chrome/browser/download/download_crx_util.cc:117
#3  0x0000555558d483ba in ChromeDownloadManagerDelegate::ShouldOpenDownload(download::DownloadItem*, base::RepeatingCallback<void (bool)> const&) (this=0x164ebf98cde0, item=0x164ebecf3e20, callback=...)
    at ../../chrome/browser/download/chrome_download_manager_delegate.cc:561
#4  0x00007ffff2c33102 in content::DownloadManagerImpl::ShouldOpenDownload(download::DownloadItemImpl*, base::RepeatingCallback<void (bool)> const&) (this=0x164ebcafc020, item=0x164ebecf3e20, callback=...)
    at ../../content/browser/download/download_manager_impl.cc:468
#5  0x00007fffea0adad7 in download::DownloadItemImpl::OnDownloadRenamedToFinalName(download::DownloadInterruptReason, base::FilePath const&)
    (this=0x164ebecf3e20, reason=download::DOWNLOAD_INTERRUPT_REASON_NONE, full_path=...) at ../../components/download/internal/common/download_item_impl.cc:1788
#6  0x00007fffea0b9b8c in base::internal::FunctorTraits<void (download::DownloadItemImpl::*)(download::DownloadInterruptReason, base::FilePath const&), void>::Invoke<void (download::DownloadItemImpl::*)(download::DownloadInterruptReason, base::FilePath const&), base::WeakPtr<download::DownloadItemImpl> const&, download::DownloadInterruptReason, base::FilePath const&>(void (download::DownloadItemImpl::*)(download::DownloadInterruptReason, base::FilePath const&), base::WeakPtr<download::DownloadItemImpl> const&, download::DownloadInterruptReason&&, base::FilePath const&) (method=
    (void (download::DownloadItemImpl::*)(download::DownloadItemImpl * const, download::DownloadInterruptReason, const base::FilePath &)) 0x7fffea0ad520 <download::DownloadItemImpl::OnDownloadRenamedToFinalName(download::DownloadInterruptReason, base::FilePath const&)>, receiver_ptr=..., args=@0x7fffffffb574: download::DOWNLOAD_INTERRUPT_REASON_NONE, args=...) at ../../base/bind_internal.h:499
#7  0x00007fffea0b9ad4 in base::internal::InvokeHelper<true, void>::MakeItSo<void (download::DownloadItemImpl::* const&)(download::DownloadInterruptReason, base::FilePath const&), base::WeakPtr<download::DownloadItemImpl> const&, download::DownloadInterruptReason, base::FilePath const&>(void (download::DownloadItemImpl::* const&)(download::DownloadInterruptReason, base::FilePath const&), base::WeakPtr<download::DownloadItemImpl> const&, download::DownloadInterruptReason&&, base::FilePath const&) (functor=
    @0x164ec0be71c0: (void (download::DownloadItemImpl::*)(download::DownloadItemImpl * const, download::DownloadInterruptReason, const base::FilePath &)) 0x7fffea0ad520 <download::DownloadItemImpl::OnDownloadRenamedToFinalName(download::DownloadInterruptReason, base::FilePath const&)>, weak_ptr=..., args=@0x7fffffffb574: download::DOWNLOAD_INTERRUPT_REASON_NONE, args=...) at ../../base/bind_internal.h:619
#8  0x00007fffea0b9a2c in base::internal::Invoker<base::internal::BindState<void (download::DownloadItemImpl::*)(download::DownloadInterruptReason, base::FilePath const&), base::WeakPtr<download::DownloadItemImpl> >, void (download::DownloadInterruptReason, base::FilePath const&)>::RunImpl<void (download::DownloadItemImpl::* const&)(download::DownloadInterruptReason, base::FilePath const&), std::__Cr::tuple<base::WeakPtr<download::DownloadItemImpl> > const&, 0ul>(void (download::DownloadItemImpl::* const&)(download::DownloadInterruptReason, base::FilePath const&), std::__Cr::tuple<base::WeakPtr<download::DownloadItemImpl> > const&, std::__Cr::integer_sequence<unsigned long, 0ul>, download::DownloadInterruptReason&&, base::FilePath const&) (functor=
    @0x164ec0be71c0: (void (download::DownloadItemImpl::*)(download::DownloadItemImpl * const, download::DownloadInterruptReason, const base::FilePath &)) 0x7fffea0ad520 <download::DownloadItemImpl::OnDownloadRenamedToFinalName(download::DownloadInterruptReason, base::FilePath const&)>, bound=..., unbound_args=@0x7fffffffb574: download::DOWNLOAD_INTERRUPT_REASON_NONE, unbound_args=...)
    at ../../base/bind_internal.h:672
```

**extensions::SandboxedUnpacker::ReadManifestDone**
```
#0  0x0000555557b3d20d in extensions::SandboxedUnpacker::ReadManifestDone(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&) (this=0x29624dac2820, manifest=..., error=...) at ../../extensions/browser/sandboxed_unpacker.cc:437
#1  0x0000555557b429b7 in base::internal::FunctorTraits<void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), void>::Invoke<void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), scoped_refptr<extensions::SandboxedUnpacker>, base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&>(void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), scoped_refptr<extensions::SandboxedUnpacker>&&, base::Optional<base::Value>&&, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&) (method=
    (void (extensions::SandboxedUnpacker::*)(extensions::SandboxedUnpacker * const, base::Optional<base::Value>, const base::Optional<std::__Cr::basic_string<char> > &)) 0x555557b3d1e0 <extensions::SandboxedUnpacker::ReadManifestDone(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&)>, receiver_ptr=..., args=..., args=...) at ../../base/bind_internal.h:499
#2  0x0000555557b428eb in base::internal::InvokeHelper<false, void>::MakeItSo<void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), scoped_refptr<extensions::SandboxedUnpacker>, base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&>(void (extensions::SandboxedUnpacker::*&&)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), scoped_refptr<extensions::SandboxedUnpacker>&&, base::Optional<base::Value>&&, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&) (functor=
    @0x29624e775dd0: (void (extensions::SandboxedUnpacker::*)(extensions::SandboxedUnpacker * const, base::Optional<base::Value>, const base::Optional<std::__Cr::basic_string<char> > &)) 0x555557b3d1e0 <extensions::SandboxedUnpacker::ReadManifestDone(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&)>, args=..., args=..., args=...) at ../../base/bind_internal.h:599
#3  0x0000555557b4285c in base::internal::Invoker<base::internal::BindState<void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), scoped_refptr<extensions::SandboxedUnpacker> >, void (base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&)>::RunImpl<void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), std::__Cr::tuple<scoped_refptr<extensions::SandboxedUnpacker> >, 0ul>(void (extensions::SandboxedUnpacker::*&&)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), std::__Cr::tuple<scoped_refptr<extensions::SandboxedUnpacker> >&&, std::__Cr::integer_sequence<unsigned long, 0ul>, base::Optional<base::Value>&&, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&) (functor=
    @0x29624e775dd0: (void (extensions::SandboxedUnpacker::*)(extensions::SandboxedUnpacker * const, base::Optional<base::Value>, const base::Optional<std::__Cr::basic_string<char> > &)) 0x555557b3d1e0 <extensions::SandboxedUnpacker::ReadManifestDone(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&)>, bound=..., unbound_args=..., unbound_args=...) at ../../base/bind_internal.h:672
#4  0x0000555557b427de in base::internal::Invoker<base::internal::BindState<void (extensions::SandboxedUnpacker::*)(base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&), scoped_refptr<extensions::SandboxedUnpacker> >, void (base::Optional<base::Value>, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&)>::RunOnce(base::internal::BindStateBase*, base::Optional<base::Value>&&, base::Optional<std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > > const&) (base=0x29624e775db0, unbound_args=..., unbound_args=...) at ../../base/bind_internal.h:641
#5  0x0000555557758b38 in base::OnceCallback<void (mojo::InlinedStructPtr<media::mojom::CdmPromiseResult>, std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > const&)>::Run(mojo::InlinedStructPtr<media::mojom::CdmPromiseResult>, std::__Cr::basic_string<char, std::__Cr::char_traits<char>, std::__Cr::allocator<char> > const&) && (this=0x29624ea9d348, args=..., args=...)
    at ../../base/callback.h:97
#6  0x0000555557b9a041 in data_decoder::mojom::JsonParser_Parse_ForwardToCallback::Accept(mojo::Message*) (this=0x29624ea9d340, message=0x7fffbb0af7d0)
    at gen/services/data_decoder/public/mojom/json_parser.mojom.cc:194
#7  0x00007ffff7b7fb9d in mojo::InterfaceEndpointClient::HandleValidatedMessage(mojo::Message*) (this=0x29624f2fe560, message=0x7fffbb0af7d0)
    at ../../mojo/public/cpp/bindings/lib/interface_endpoint_client.cc:428
#8  0x00007ffff7b7f631 in mojo::InterfaceEndpointClient::HandleIncomingMessageThunk::Accept(mojo::Message*) (this=0x29624f2fe590, message=0x7fffbb0af7d0)
    at ../../mojo/public/cpp/bindings/lib/interface_endpoint_client.cc:133
#9  0x00007ffff7b7e39b in mojo::FilterChain::Accept(mojo::Message*) (this=0x29624f2fe5a0, message=0x7fffbb0af7d0) at ../../mojo/public/cpp/bindings/lib/filter_chain.cc:40
```

