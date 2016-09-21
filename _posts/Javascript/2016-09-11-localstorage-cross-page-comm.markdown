---
title:  "使用localStorage进行跨标签页通讯"
categories: Javascript HTML5
---

localStorage 对象用来在浏览器本地存取数据，并且数据不因浏览器的关闭而丢失。浏览器中同一个域下的窗口可以共享 localStorage 数据。因此我们可以利用这一特性进行跨浏览器内的跨页的事件发送，从而进行通讯。

下面介绍一个简单的应用场景：用户打开了我们网站的多个页面，其状态为未登录，那么可能所有页面上都会有登录按钮或者登录框，评论框也会提示登录，而当他在某一个页面登陆后，我们希望同时更新其他非活动页面的UI，这样当他返回到其他在登录前已打开的页面时，可以以已登录状态进行相关操作（至少不会再出现登录框）。


```
    // isLogin 由后台模板生成，标识当前页用户的登录状态
    var isLogin = true;
    
    var isLoginLocal = localStorage.getItem('isLogin');
    if(isLogin && (isLoginLocal=='no' || isLoginLocal==null)){
        localStorage.setItem('isLogin', 'yes');
    }
    else if( !isLogin && (isLoginLocal=='yes' || isLoginLocal==null)){
        localStorage.setItem('isLogin', 'no');
    }
    window.addEventListener("storage", function(e){
        if(e.key == 'isLogin'){
            // 用户在其他页面进行登录或者登出操作，当前页面ui需要做相应的调整
            if(e.newValue == 'yes'){
                $('.showLogin').show();
                $('.hideLogin').hide();
            }
            else{
                $('.showLogin').hide();
                $('.hideLogin').show();
            }
        }
    }, false);
    
```
