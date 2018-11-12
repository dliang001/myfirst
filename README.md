chrome浏览器插件开发经验（一）
最近在进行chrome浏览器插件的开发，一些小的经验总结随笔。

1、首先，推荐360的chrome插件开发文档：http://open.chrome.360.cn/extension_dev/overview.html

2、从chrome18开始往后，chrome浏览器插件开发的 manifest.json 文件中的 "manifest_version": 2 属性就必须被显式（固定）的声明了。

3、chrome插件开发，很大程度上需要chrome.* API 的支持，附上chrome历史版本的API更新记录：http://lmk123.duapp.com/chrome/extensions/whats_new.html

4、如果需要下载不同的chrome版本进行安装测试，推荐一个下载网址：http://www.mykurong.com/chrome/

5、为chrome网页添加右键选项：

　　首先，需要在 manifest.json 文件中添加权限支持：

　　"permissions": [

　　...

　　"contextMenus",

　　...

　　]

　　

复制代码
chrome.contextMenus.create({
"title" : "菜单项文字",
"type" : "normal", //菜单项类型 "checkbox", "radio","separator"
"contexts" : ["frame"], //菜单项影响的页面元素 "anchor","image"
"documentUrlPatterns":["http://*.163.com/*"],//iframe的src匹配
"targetUrlPatterns" : ["http://*.163.com/*"],//href的匹配
"onclick" : changeSCHandler() //单击时的处理函数
});
复制代码
6、插件通信：

　　6.1 background.js 和 content_script.js 通信推荐使用 chrome.extension.sendRequest()、chrome.extension.onRequest.addListener(function(request, sender, sendRequest){}); 的形式。

　　6.2 其他页面调用 background.js 里的函数和变量时推荐在其他页面使用 var backgroundObj = chrome.extension.getBackgroundPage(); if(backgroundObj){ backgroundObj.func(param); }的形式。

　　6.3 如果插件运行中会有多个tab页同时打开和加载，则需要注意通信过程中使用 tab.id 参数，因为每个加载插件的tab页都会保留自己的一个 content_script.js 运行，所以和 content_script.js 通信时需要指定是向哪个tab页进行通信；获取当前打开的 tab 页的 id 可以使用 chrome.tabs.getSelected(function(tab){current_tab_id = tab.id;}); 的形式。

7、关于 xmlhttprequest

　　在chrome插件中可能会用到ajax请求，以及跨域请求的出现，推荐使用 xmlhttprequest，会比较稳定。但使用 xmlhttprequest 会有一个不完善的地方，就是在 chrome 中，xmlhttprequest 请求的HTTP requestHeaders 头不包含 Referer 数据，如果需要这个字段就必须对 chrome 的 xmlhttprequest 请求进行监听和修改，具体如下：

　　

首先，需要在 manifest.json 文件中添加权限支持：

　　"permissions": [

　　...

　　"webRequest", "webRequestBlocking",  //用于获取 xmlhttprequest 以及对 xmlhttprequest 进行 block 操作

　　...

　　]

然后使用如下方式：

复制代码
var wR=chrome.webRequest||chrome.experimental.webRequest; //兼容17之前版本的chrome，若需要使用chrome.experimental，需要在 about:flags 中“启用“实验用。。API”
if(wR){
    wR.onBeforeSendHeaders.addListener(
        function(details) {
            if (details.type === 'xmlhttprequest') {
                var exists = false;
                for (var i = 0; i < details.requestHeaders.length; ++i) {
                    if (details.requestHeaders[i].name === 'Referer') {
                        exists = true;
                        
                        break;
                    }
                }
                if (!exists) {//不存在 Referer 就添加
                    details.requestHeaders.push({ name: 'Referer', value: 'http://www.yourname.com'});
                }
                return { requestHeaders: details.requestHeaders };
            }
},
        {urls: ["https://*.google.com/*","http://*.google.com/*"]},//匹配访问的目标url
        ["blocking", "requestHeaders"]
    );
}
复制代码
8、题外：如何在页面中插入包含透明图片的顶层div

复制代码
var topDiv = document.createElement('div');
        topDiv.style.width=document.documentElement.scrollWidth+"px";
        topDiv.style.height=document.documentElement.scrollHeight+"px";
        topDiv.style.position="absolute";
        topDiv.style.left=0;
        topDiv.style.top=0;
        topDiv.style.zIndex = 999;
        var title = document.createElement('a');
        var img = document.createElement('img');
        img.src = "http://.../.../transparent.gif";
        img.setAttribute("width","100%");
        img.setAttribute("height","100%");
        title.appendChild(img);
        topDiv.appendChild(title);
        document.getElementsByTagName('body')[0].insertBefore(topDiv,document.getElementsByTagName('body')[0].childNodes[0]);
复制代码
在document中创建和body同样宽度、高度的div，然后在其中添加透明图片，最后将div的zIndex设为最大，并添加到 body 的子节点序列中即可。
