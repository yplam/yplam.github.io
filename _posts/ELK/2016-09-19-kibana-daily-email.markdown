---
title:  "使用phantomjs定期发送Kibana报告"
categories: ELK
---

数据日报（周报）是一种获取应用相关指标的比较原始却直观的方式，Kibana 的UI美观度与易用性的确是没什么可挑剔的，但却缺少定期发送邮件报表功能。虽然 [skedler](https://www.skedler.com/) 有提供商业支持，但其免费版本功能显得有点弱，而收费版对个人用户而言有比较贵。Google一下发现phantomjs可以作为替代。

下面是使用 phantomjs 发送Kibana报告的测试代码。其原理是使用 phantomjs 模拟浏览器访问 Kibana，然后对访问内容进行截图，然后将生成的截图通过 http post的方式调用 sendcloud 接口发送邮件。（发送邮件的调用有点繁琐，因为找不到很好的读取本地文件的方式，而 phantomjs webpage 模块的 renderBuffer 实际上又无法使用）


```

var page = require('webpage').create();
var fs = require('fs');
var url = "***";
page.settings.userName = "***"; // http auth
page.settings.password = "***";
page.viewportSize = { width: 1280, height: 900 };
page.open(url, function(status) {
  if (status !== 'success') {
    console.log('Unable to load the address!');
    phantom.exit(1);
  } else {
    window.setTimeout(function () {
        page.render('abc.png');
        var rawFile = new XMLHttpRequest();
        rawFile.open("GET", 'file://****abc.png', false);
        rawFile.responseType = "blob";
        rawFile.onload = function () {
          var formPost = new FormData();
          var apiUser = '***'; 
          var apiKey = '***';
          var currentDate = new Date();
          formPost.append('apiUser', apiUser);
          formPost.append('apiKey', apiKey);
          formPost.append('from', '***');
          formPost.append('to', '***');
          formPost.append('subject', '***数据日报 - '+currentDate.toString());
          formPost.append('html', '<h1>***数据日报</h1><p>此数据由程序自动生成，如果需要调整相关指标，请反馈给产品经理。</p><p><img src="cid:abc.png"/></p>');
          formPost.append('useAddressList', true);
          formPost.append('attachments', rawFile.response, 'abc.png');
          formPost.append('embeddedImage', rawFile.response, 'abc.png');
          formPost.append('embeddedCid', "abc.png");
          var xhr = new XMLHttpRequest();
          xhr.open('POST', 'http://api.sendcloud.net/apiv2/mail/send', true);
          xhr.send(formPost);
          xhr.onload = function (e) {
            if (xhr.readyState === 4) {
              if (xhr.status === 200) {
                console.log(xhr.responseText);
              } else {
                console.error(xhr.statusText);
              }
            }
          };
          xhr.onerror = function (e) {
            console.error(xhr.statusText);
          };
        };
        rawFile.send(null);

        
        window.setTimeout(function(){
            phantom.exit();
        }, 5000);
    }, 20000);
  }
});


```


将上面内容保存为js文件，然后用下面命令调用

```

phantomjs --web-security=false --local-to-remote-url-access=true ***.js

```
