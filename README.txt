第五期|前端搞监控：开发的角度去理解监控和埋点，多端错误监控实现
自述
先说一下这次大会很长，我最后两场没听了。不过比较劲爆的是知道了阿里的钉钉是用的c++的框架开发的。果然前端做客户端方面还是有所欠缺啊。

这次大会是8场，从早上10点到下午6点。分别有



我个人听的迷迷糊糊，其中我记得贝贝集团讲的非常干货。几乎是纯技术角度分享了自己遇到的问题。我个人对这次大会主要是一个个人总结。

记得比较惨痛的是千米网面试的时候问我埋点。唉，那会还是太年轻

这个大哥的文章很不错，比我有见解：https://juejin.im/post/5ea4dc1ee51d4546c72e2ece

一、为什么需要监控和埋点
首先为什么我们需要埋点和监控？能带来什么好处？解决什么业务痛点？

1、技术不难
首先我必须声明一下，初级的监控这块其实大家都会写。说的简单点，无非就是全局监控error和代码侵入式的监控。所以从技术难点上，监控的技术难点不大。

2、请思考你们的业务
首先C端这块其实在代码监控上也许不是非常迫切，但是埋点应该是比较需要的。

好处：监控用户行为，进行数据分析，加速卖货，引导客户等等，等等

B端：对于用户的行为分析意义也许不大，但是B端更追求稳定，所以错误监控方面更加需要

这些都是我个人比较片面的一些理解

3、ppt实例参考
个人见解有限，我放点截图让大家发散下思维





二、该如何设计监控，你需要什么数据
在明确业务场景之后，那么就该开始设计你的数据结构和需要的数据了

首先我个人其实在C端经历了三年，从软件实施到前端开发，C端是我比较熟悉的

1、设计举例
我需要什么数据，从常规的C端需求出发。设想我是一个产品，我现在需要一个数据分析收集系统，得到数据进行后期的分析使用

从常规出发，页面点击量，停留时间，访问人次，用户的操作流程，谁访问了，用户地区等等。我发一张截图



对你的数据进行分级




2、数据结构体
这里我还是偷懒用一下政采云上面的ppt图片

感觉他们的格式非常好



二、你该怎么上报你的数据（埋点）
1、请求用img的src进行请求数据上报
为什么用 1x1 像素的 gif 图

1、没有跨域问题 

2、发 GET 请求之后不需要获取和处理数据、服务器也不需要发送数据 

3、不会携带当前域名 cookie！ 

4、不会阻塞页面加载，影响用户的体验，只需 new Image 对象

 5、相比于 BMP/PNG 体积最?小，可以节约 41% / 35% 的网络资源大小

注意：埋点也许是有差异的，但是从错误监控等等方面，这样是没什么问题的

2、事件拦截和代理
事件拦截：委托到dom上面，利用img进行上报数据，navigator.sendBeacon方法

mousedown，touch，scroll，keydown

页面进入离开：onload，berorOnLoad

这块是埋点部分，通常能收集到用户的操作行为，页面停留时间，离开等等

navigator.sendBeacon会有兼容问题，所以是做兼容性处理，具体我忘了。大家百度吧

三、错误监控（SDK）宋小菜
写到这里我感觉有点能力不足了，实在是埋点和错误监控方面不只是纯粹的技术，和业务是非常紧密相关的。下面我会对ppt里面的各路大神提供的错误监控代码进行copy总结

1、RNSDK（reactNative）（宋小菜：Jimmy）
JS 端 

? 错误捕获 

    ? ErrorUtils.setGlobalHandler 类似于 window.onerror 

    ? Promise.rejectionTracking 类似于 Unhandledrejection 

    ? ?网络请求：替换 XMLHttpRequest，代理理其 send/open/onload 等方法

    ? ?页?面跳转：react-navigation 的 onStateChange 或者在 redux 集成方式中使用 screenTracking 

? Native 端

   ? iOS 使?用 KSCrash 进?行行?日志采集，可以本地进?行行符号化 
   ? 存储捕获到的数据（包括 JS 端和 native 端）统?一上报
const tracking = require("promise/setimmediate/rejection-tracking");
tracking.disable();
tracking.enable({
allRejections: true,
onHandled: () => {
// We do nothing
},
onUnhandled: (id: any, error: any) => {
const stack = computeStackTrace(error);
stack.mechanism = 'unhandledrejection';
stack.data = {
id
}
// tslint:disable-next-line: strict-type-predicates
if (!stack.stack) {
stack.stack = [];
}
Col.trackError(stringifyMsg(stack))
}
});复制代码
2、小程序（宋小菜）
? 网络请求: 代理全局对象 wx 的 wx.request ?方法 

? ?面跳转: 覆写 Page 对象，代理其生命 周期方法

import "miniprogram-api-typings"
export const wrapRequest = () => {
const originRequest = wx.request;
wx.request = function(...args: any[]): WechatMiniprogram.RequestTask {
// get request data
return originRequest.apply(this, ...args);
//
}
}复制代码
3、小程序SDK实现（宋小菜）
/* global Page Component */
function captureOnLoad(args) {
console.log('do what you want to do', args)
}
function captureOnShow(args) {
console.log('do what you want to do', args)
}
function colProxy(target, method, customMethod) {
const originMethod = target[method]
if(target[method]){
target[method] = function() {
customMethod(…arguments)
originMethod.apply(this, arguments)
}
}
}复制代码
// Page
const originalPage = Page
Page = function (opt) {
colProxy(opt.methods, 'onLoad', captureOnLoad)
colProxy(opt.methods, 'onShow', captureOnShow)
originalPage.apply(this, arguments)
}
// Component
const originalComponent = Component
Component = function (opt) {
colProxy(opt.methods, 'onLoad', captureOnLoad)
colProxy(opt.methods, 'onShow', captureOnShow)
originalComponent.apply(this, arguments)
}复制代码
四、错误监控：贝贝集团：Allan：《React+Redux前端开发实战》作者
贝贝集团非常的干货，这里贴一下，所面临的的环境是具有非常多的端，80+工程项目。

下面都是我ppt里面原原本本摘抄下来，并且加上个人部分个人理解。主要是贝贝集团的分享太干货了。

直接从错误捕获说吧

1、错误捕获机制
 ? window.onerror：运行时错误捕获 

? window.addEventListener(‘unhandledrejection’)：promise 没有 catch 错误 

? try/catch 处理跨域脚本错误 

? 其他技术栈中（Vue、React）的错误捕获

 ? window.addEventListener(‘error’)：资源加载错误 

? … …

2、监听window.onerror


当发?生 JavaScript 运行时错误（包括处理程序中引发的语法错误和异常）时，使?接口 ErrorEvent 的 error 事件将在 window 被触发，并被 window.onerror() 调用

3、监听 unhandledrejection 事件


当 Promise 被 reject 并且没有得到处理的时候，会触发 unhandledrejection 事件。所以可以对此事件进行监听，将错误信息捕获上报。

4、跨域脚本错误：Script error.
【方案1】：后端配置 Access-Control-Allow-origin、前端在 script 标签配置 crossorigin 【方案2】：劫持原?方法，使?用 try/catch 绕过，将错误抛出

使用的是方案2：好处是不会直接报错，可以通过try/catch方式捕获错误，并且不会影响js的运行。我个人在代码编辑处理的时候也非常推荐。这个方式还是写eggjs的时候学过来的。配合primose方式能大大简化代码



5、其它技术栈——Vue.js
劫持 Vue.config.errorHandler （errorCaptured），当 Vue 的项 目中发生错误时，将错误捕获上报



6、其它技术栈——React.js
监听 componentDidCatch，当 React 的项目中发生错误时，将错 误捕获上报



7、用户行为收集
这里其实很简单，很多时候错误的产品是需要复现方式的。就像我们开发的时候测试会给你复现方式。如果你不知道什么原因产品的bug，那么你也无法正对性的去修复问题。也许修复了bug，反而导致系统崩溃了呢？

用户行为方面我们可以分为：用户行为，浏览器行为，控制台打印行为

针对的我们可以下面这样做

【1】点击行为 （用户行为）
使用 addEventListener 全局监听点击事件， 将用户行为（click、input）和 dom 元素名字 收集。 当错误发生将错误和行为一起上报。



【2】发送请求行为 （浏览器?行为）
监听 XMLHttpRequest 对象的 onreadystatechange 回调 函数，在回调函数执行时收集数据。



【3】页面跳转 （浏览器?行为）
监听 window.onpopstate，页面跳转的时会 触发此方法，将信息收集



【4】控制台打印 （控制台行为）
改写 console 对象的 info、warn、error 方 法，在 console 执行时将信息收集。



五、错误或者埋点数据的处理
这里比较关键，就不怎么贴ppt了。而且还是有一点个人理解的

1、埋点
埋点属于业务分析行为，更多的是基于业务本身方向进行数据分析。

埋点的关键是保证你的数据是一个连贯额闭环，能够充分的分析出用户从进入到离开页面所产生的一系列用户行为。

政采云团队在做的时候基本上做到了每个按钮都会进行埋点。

最后会生成一个热力分析图。（相当牛逼）来标识页面中高频操作的地方

2、错误监控存储
这块是重点讲解部分

首先我在参与视频学习的过程中了解到几个点

1、ES（elasticsearch）

下面说明来自简书：elasticsearch简写es，es是一个高扩展、开源的全文检索和分析引擎，它可以准实时地快速存储、搜索、分析海量的数据。

简单来说就像是一个百度搜索引擎，快速准确

主要理解点：

为什么要用ES，因为他是全文索引的，效率非常快，但是mysql差了很多。

ES用来做数据一次处理，不作为长期存储，一般只会存留一个月的数据

2、数据库存储（MySQl）

当前期数据处理完成之后再进行存入数据库，后续进行分析处理

3、存储流程

错误上报——数据清洗——数据持久化——数据可视化——监控告警

4、关于数据清洗

关键在于分析数据，提取聚合重复错误，整理为更小的数据，然后存储

同时也是为了减小服务器压力，提供中转站。

比如：贝贝和阿里钉钉都提到了需要对数据进行削峰处理，这个大家字面意思应该就能理解

贝贝集团的处理方式是：

1、每分钟处理一次

2、每分钟获取数据10000条：超过就采样入库

3、同类型错误大于200条只记录数量

4、处理无用的信息等等（上传的数据因为是需要进行string处理，会出现乱码或者无意义的字符）

核心就是：剔除重复，留下唯一

5、告警

告警的目的是能够快速通知负责人，对系统进行修复处理。同时告警，大家需要根据错误的情况进行分析，归类。如：一般错误，功能错误，页面错误，系统错误等等的一个升级过程

比如：一般错误，功能错误可以发送工作群，邮件

但是系统级别错误那么就是发短信了



六、其他端的一个错误捕获处理
这里主要是摘抄自贝贝集团的ppt，基本上是截图

1、node错误监控
1、初始化

node启动的时候，需要获取当前node一个初始id等信息。如果是集群化处理那么比如贝贝集团的情况ZooKeeper里面去获取对 应的业务 id 实现天网的初始化

2、错误捕获机制

Node 端使?用 process 对象监听 uncaughtException、unhandledRejection 事件， 捕获未处理理的 JS 异常和 Promise 异常



3、错误信息收集

捕获到错误对象后，使?了第三方库 stack- trace 进行解析，该库会将错误堆栈解析成 数 组 并获取堆栈调用链路路 中应用内文件的源码。

主要是用第三方库实现错误信息的一个可视化展示

2、weex错误监控
1、多端兼容

1）Weex 工程打包有两套 Webpack 配置，但业 务中统一引用的是 @weex/skynet 

2）针对 Web 端打包的时候使用 replace-loader 这个 loader 将包替换为 @fe-base/skynet



2、错误捕获

1）前端为客户端提供业务的模块名

 2）通过 hybrid 接口将模块名传给客户端 

3）客户端在捕获到 weex 的错误发生时上报



3、小程序监控
小程序监控我不截图了。写太多了。我直接总结

首先小程序的app启动的时候有一个错误捕获函数：onError，通过这个能捕获全局存在的错误

然后因为是C端，你需要获取当前启动的小程序的一个唯一id，除了appid之外就是和用户信息有关的id，作为记录，然后上报为本次的一个错误记录起始。

4、客户端监控
1、Android错误上报机制

使用系统提供的机制，实现 Thread.UncaughtExceptionHandler 接口，通过 uncau第五期|前端搞监控：开发的角度去理解监控和埋点，多端错误监控实现
自述
先说一下这次大会很长，我最后两场没听了。不过比较劲爆的是知道了阿里的钉钉是用的c++的框架开发的。果然前端做客户端方面还是有所欠缺啊。

这次大会是8场，从早上10点到下午6点。分别有



我个人听的迷迷糊糊，其中我记得贝贝集团讲的非常干货。几乎是纯技术角度分享了自己遇到的问题。我个人对这次大会主要是一个个人总结。

记得比较惨痛的是千米网面试的时候问我埋点。唉，那会还是太年轻

这个大哥的文章很不错，比我有见解：https://juejin.im/post/5ea4dc1ee51d4546c72e2ece

一、为什么需要监控和埋点
首先为什么我们需要埋点和监控？能带来什么好处？解决什么业务痛点？

1、技术不难
首先我必须声明一下，初级的监控这块其实大家都会写。说的简单点，无非就是全局监控error和代码侵入式的监控。所以从技术难点上，监控的技术难点不大。

2、请思考你们的业务
首先C端这块其实在代码监控上也许不是非常迫切，但是埋点应该是比较需要的。

好处：监控用户行为，进行数据分析，加速卖货，引导客户等等，等等

B端：对于用户的行为分析意义也许不大，但是B端更追求稳定，所以错误监控方面更加需要

这些都是我个人比较片面的一些理解

3、ppt实例参考
个人见解有限，我放点截图让大家发散下思维





二、该如何设计监控，你需要什么数据
在明确业务场景之后，那么就该开始设计你的数据结构和需要的数据了

首先我个人其实在C端经历了三年，从软件实施到前端开发，C端是我比较熟悉的

1、设计举例
我需要什么数据，从常规的C端需求出发。设想我是一个产品，我现在需要一个数据分析收集系统，得到数据进行后期的分析使用

从常规出发，页面点击量，停留时间，访问人次，用户的操作流程，谁访问了，用户地区等等。我发一张截图



对你的数据进行分级




2、数据结构体
这里我还是偷懒用一下政采云上面的ppt图片

感觉他们的格式非常好



二、你该怎么上报你的数据（埋点）
1、请求用img的src进行请求数据上报
为什么用 1x1 像素的 gif 图

1、没有跨域问题 

2、发 GET 请求之后不需要获取和处理数据、服务器也不需要发送数据 

3、不会携带当前域名 cookie！ 

4、不会阻塞页面加载，影响用户的体验，只需 new Image 对象

 5、相比于 BMP/PNG 体积最?小，可以节约 41% / 35% 的网络资源大小

注意：埋点也许是有差异的，但是从错误监控等等方面，这样是没什么问题的

2、事件拦截和代理
事件拦截：委托到dom上面，利用img进行上报数据，navigator.sendBeacon方法

mousedown，touch，scroll，keydown

页面进入离开：onload，berorOnLoad

这块是埋点部分，通常能收集到用户的操作行为，页面停留时间，离开等等

navigator.sendBeacon会有兼容问题，所以是做兼容性处理，具体我忘了。大家百度吧

三、错误监控（SDK）宋小菜
写到这里我感觉有点能力不足了，实在是埋点和错误监控方面不只是纯粹的技术，和业务是非常紧密相关的。下面我会对ppt里面的各路大神提供的错误监控代码进行copy总结

1、RNSDK（reactNative）（宋小菜：Jimmy）
JS 端 

? 错误捕获 

    ? ErrorUtils.setGlobalHandler 类似于 window.onerror 

    ? Promise.rejectionTracking 类似于 Unhandledrejection 

    ? ?网络请求：替换 XMLHttpRequest，代理理其 send/open/onload 等方法

    ? ?页?面跳转：react-navigation 的 onStateChange 或者在 redux 集成方式中使用 screenTracking 

? Native 端

   ? iOS 使?用 KSCrash 进?行行?日志采集，可以本地进?行行符号化 
   ? 存储捕获到的数据（包括 JS 端和 native 端）统?一上报
const tracking = require("promise/setimmediate/rejection-tracking");
tracking.disable();
tracking.enable({
allRejections: true,
onHandled: () => {
// We do nothing
},
onUnhandled: (id: any, error: any) => {
const stack = computeStackTrace(error);
stack.mechanism = 'unhandledrejection';
stack.data = {
id
}
// tslint:disable-next-line: strict-type-predicates
if (!stack.stack) {
stack.stack = [];
}
Col.trackError(stringifyMsg(stack))
}
});复制代码
2、小程序（宋小菜）
? 网络请求: 代理全局对象 wx 的 wx.request ?方法 

? ?面跳转: 覆写 Page 对象，代理其生命 周期方法

import "miniprogram-api-typings"
export const wrapRequest = () => {
const originRequest = wx.request;
wx.request = function(...args: any[]): WechatMiniprogram.RequestTask {
// get request data
return originRequest.apply(this, ...args);
//
}
}复制代码
3、小程序SDK实现（宋小菜）
/* global Page Component */
function captureOnLoad(args) {
console.log('do what you want to do', args)
}
function captureOnShow(args) {
console.log('do what you want to do', args)
}
function colProxy(target, method, customMethod) {
const originMethod = target[method]
if(target[method]){
target[method] = function() {
customMethod(…arguments)
originMethod.apply(this, arguments)
}
}
}复制代码
// Page
const originalPage = Page
Page = function (opt) {
colProxy(opt.methods, 'onLoad', captureOnLoad)
colProxy(opt.methods, 'onShow', captureOnShow)
originalPage.apply(this, arguments)
}
// Component
const originalComponent = Component
Component = function (opt) {
colProxy(opt.methods, 'onLoad', captureOnLoad)
colProxy(opt.methods, 'onShow', captureOnShow)
originalComponent.apply(this, arguments)
}复制代码
四、错误监控：贝贝集团：Allan：《React+Redux前端开发实战》作者
贝贝集团非常的干货，这里贴一下，所面临的的环境是具有非常多的端，80+工程项目。

下面都是我ppt里面原原本本摘抄下来，并且加上个人部分个人理解。主要是贝贝集团的分享太干货了。

直接从错误捕获说吧

1、错误捕获机制
 ? window.onerror：运行时错误捕获 

? window.addEventListener(‘unhandledrejection’)：promise 没有 catch 错误 

? try/catch 处理跨域脚本错误 

? 其他技术栈中（Vue、React）的错误捕获

 ? window.addEventListener(‘error’)：资源加载错误 

? … …

2、监听window.onerror


当发?生 JavaScript 运行时错误（包括处理程序中引发的语法错误和异常）时，使?接口 ErrorEvent 的 error 事件将在 window 被触发，并被 window.onerror() 调用

3、监听 unhandledrejection 事件


当 Promise 被 reject 并且没有得到处理的时候，会触发 unhandledrejection 事件。所以可以对此事件进行监听，将错误信息捕获上报。

4、跨域脚本错误：Script error.
【方案1】：后端配置 Access-Control-Allow-origin、前端在 script 标签配置 crossorigin 【方案2】：劫持原?方法，使?用 try/catch 绕过，将错误抛出

使用的是方案2：好处是不会直接报错，可以通过try/catch方式捕获错误，并且不会影响js的运行。我个人在代码编辑处理的时候也非常推荐。这个方式还是写eggjs的时候学过来的。配合primose方式能大大简化代码



5、其它技术栈——Vue.js
劫持 Vue.config.errorHandler （errorCaptured），当 Vue 的项 目中发生错误时，将错误捕获上报



6、其它技术栈——React.js
监听 componentDidCatch，当 React 的项目中发生错误时，将错 误捕获上报



7、用户行为收集
这里其实很简单，很多时候错误的产品是需要复现方式的。就像我们开发的时候测试会给你复现方式。如果你不知道什么原因产品的bug，那么你也无法正对性的去修复问题。也许修复了bug，反而导致系统崩溃了呢？

用户行为方面我们可以分为：用户行为，浏览器行为，控制台打印行为

针对的我们可以下面这样做

【1】点击行为 （用户行为）
使用 addEventListener 全局监听点击事件， 将用户行为（click、input）和 dom 元素名字 收集。 当错误发生将错误和行为一起上报。



【2】发送请求行为 （浏览器?行为）
监听 XMLHttpRequest 对象的 onreadystatechange 回调 函数，在回调函数执行时收集数据。



【3】页面跳转 （浏览器?行为）
监听 window.onpopstate，页面跳转的时会 触发此方法，将信息收集



【4】控制台打印 （控制台行为）
改写 console 对象的 info、warn、error 方 法，在 console 执行时将信息收集。



五、错误或者埋点数据的处理
这里比较关键，就不怎么贴ppt了。而且还是有一点个人理解的

1、埋点
埋点属于业务分析行为，更多的是基于业务本身方向进行数据分析。

埋点的关键是保证你的数据是一个连贯额闭环，能够充分的分析出用户从进入到离开页面所产生的一系列用户行为。

政采云团队在做的时候基本上做到了每个按钮都会进行埋点。

最后会生成一个热力分析图。（相当牛逼）来标识页面中高频操作的地方

2、错误监控存储
这块是重点讲解部分

首先我在参与视频学习的过程中了解到几个点

1、ES（elasticsearch）

下面说明来自简书：elasticsearch简写es，es是一个高扩展、开源的全文检索和分析引擎，它可以准实时地快速存储、搜索、分析海量的数据。

简单来说就像是一个百度搜索引擎，快速准确

主要理解点：

为什么要用ES，因为他是全文索引的，效率非常快，但是mysql差了很多。

ES用来做数据一次处理，不作为长期存储，一般只会存留一个月的数据

2、数据库存储（MySQl）

当前期数据处理完成之后再进行存入数据库，后续进行分析处理

3、存储流程

错误上报——数据清洗——数据持久化——数据可视化——监控告警

4、关于数据清洗

关键在于分析数据，提取聚合重复错误，整理为更小的数据，然后存储

同时也是为了减小服务器压力，提供中转站。

比如：贝贝和阿里钉钉都提到了需要对数据进行削峰处理，这个大家字面意思应该就能理解

贝贝集团的处理方式是：

1、每分钟处理一次

2、每分钟获取数据10000条：超过就采样入库

3、同类型错误大于200条只记录数量

4、处理无用的信息等等（上传的数据因为是需要进行string处理，会出现乱码或者无意义的字符）

核心就是：剔除重复，留下唯一

5、告警

告警的目的是能够快速通知负责人，对系统进行修复处理。同时告警，大家需要根据错误的情况进行分析，归类。如：一般错误，功能错误，页面错误，系统错误等等的一个升级过程

比如：一般错误，功能错误可以发送工作群，邮件

但是系统级别错误那么就是发短信了



六、其他端的一个错误捕获处理
这里主要是摘抄自贝贝集团的ppt，基本上是截图

1、node错误监控
1、初始化

node启动的时候，需要获取当前node一个初始id等信息。如果是集群化处理那么比如贝贝集团的情况ZooKeeper里面去获取对 应的业务 id 实现天网的初始化

2、错误捕获机制

Node 端使?用 process 对象监听 uncaughtException、unhandledRejection 事件， 捕获未处理理的 JS 异常和 Promise 异常



3、错误信息收集

捕获到错误对象后，使?了第三方库 stack- trace 进行解析，该库会将错误堆栈解析成 数 组 并获取堆栈调用链路路 中应用内文件的源码。

主要是用第三方库实现错误信息的一个可视化展示

2、weex错误监控
1、多端兼容

1）Weex 工程打包有两套 Webpack 配置，但业 务中统一引用的是 @weex/skynet 

2）针对 Web 端打包的时候使用 replace-loader 这个 loader 将包替换为 @fe-base/skynet



2、错误捕获

1）前端为客户端提供业务的模块名

 2）通过 hybrid 接口将模块名传给客户端 

3）客户端在捕获到 weex 的错误发生时上报



3、小程序监控
小程序监控我不截图了。写太多了。我直接总结

首先小程序的app启动的时候有一个错误捕获函数：onError，通过这个能捕获全局存在的错误

然后因为是C端，你需要获取当前启动的小程序的一个唯一id，除了appid之外就是和用户信息有关的id，作为记录，然后上报为本次的一个错误记录起始。

4、客户端监控
1、Android错误上报机制

使用系统提供的机制，实现 Thread.UncaughtExceptionHandler 接口，通过 uncaughtException 方法获取崩溃错误信 息，在应用初始化时替换掉默认的崩溃回调

为?方便便前端开发理理解：

 ? uncaughtException：可类比为前端的 window.onerror 

? Thread.UncaughtExceptionHandler: 回调函数



2、ios错误监控

使用系统提供的错误捕获机制，注册了了 Objective-C 异常和 POSIX signal 的处理理钩子，在发生崩溃的时候可以通过钩子函数记录崩溃的信息。 在下次 App 启动时将错误上报。



5、吐槽ghtException 方法获取崩溃错误信 息，在应用初始化时替换掉默认的崩溃回调

为?方便便前端开发理理解：

 ? uncaughtException：可类比为前端的 window.onerror 

? Thread.UncaughtExceptionHandler: 回调函数



2、ios错误监控

使用系统提供的错误捕获机制，注册了了 Objective-C 异常和 POSIX signal 的处理理钩子，在发生崩溃的时候可以通过钩子函数记录崩溃的信息。 在下次 App 启动时将错误上报。



5、吐槽