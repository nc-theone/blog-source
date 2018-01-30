---
title: PWA入门指导
tags: pwa
---
PWA：一种webapp模型，中文译为`渐进式网页应用`；提供接近与Native的交互体验


## 简介

> 全称：Progressive Web Apps  
> 缩写：PWA  
> 中文：渐进式网页应用  
> 官方解释：[PWA](https://developers.google.com/web/progressive-web-apps/)  
> 参考：[D2-PWA带来急速离线web](https://files.alicdn.com/tpsservice/18ee3d30f31eefbab91915f39771d780.pdf)


## 目录

* PWA是什么 & 有什么特性
* PWA技术组成
* 做PWA需要重点关注的问题
* 有哪些PWA解决方案
* PWA现状 & 发展前景


## 是什么 & 有什么特性

* 一种app模型：只要满足下述几种特性的web app均可称之为PWA


| 特性 | 备注 |
| :--- | :--- |
| 可靠的(Reliable) | 在各种网络情况下，均可访问 |
| 迅速(Fast) | 用户使用时，能够提供平滑的交互体验 |
| 迷人的(Engaging) | 提供类native app的用户体验 |


* 与web app 以及 native app的对比


| Status | Web App | Native App | PWA |
| :--- | :--- | :--- | :--- |
| Online | 可用 | 可用 | 可用 |
| Offline | 不可用 | 离线功能可用 | 开机动画&离线可用 |


## PWA 技术组成

* [manifest.json](https://developer.mozilla.org/zh-CN/Add-ons/WebExtensions/manifest.json)
* [Service Worker](https://developer.mozilla.org/zh-CN/docs/Web/API/ServiceWorker)
* [Cache Storage](https://developer.mozilla.org/zh-CN/docs/Web/API/Cache)
* [Push](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
* [Notification](https://developer.mozilla.org/zh-CN/docs/Web/API/notification)


### manifest.json

* 描述文件，用来描述一个站点的基本信息、访问权限以及相关执行脚本，这里不做重点介绍


 
![image | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/e72d9d35-b1a7-443d-bdad-10c9ccfb4d9f.png "")

* 注意，其中的`gcm_sender_id`并不是一个标准属性，而是为了使用google的`web push`服务而指定的一个字段


### Service Worker

> Service worker是一个依托于浏览器的后台任务  
> Service worker是一个注册在指定源和路径下的事件驱动 Worker  
> Service worker运行在worker上下文，因此它不能访问DOM


```javascript
// 注册一个service-worker
navigator.serviceWorker.register('/service-worker.js');
```

```javascript
// service-worker.js
// self === ServiceWorkerGlobalScope
self.addEventListener('install', callback);
self.addEventListener('fetch', callback);
self.addEventListener('push', callback);
```

> Service Worker 所处的位置


![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/b106b962-e70c-48b8-a787-017ed3ef6d6d.png "")


> 主要功能


* 请求代理`self.addEventListener('fetch', callback);`


![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/b3127d85-321a-4d8e-b2bb-985d37a4831f.png "")


* 离线缓存`self.cachesopen(cacheName).then(callback)`


![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/cba6b84f-2841-472e-b059-4d0bb2f1b2bf.png "")


* 消息订阅 & 通知


```javascript
// 业务代码.js
navigator.serviceWorker.ready.then(function(registration) {
  // 使用push manager 订阅消息
  // 使用浏览器默认的推送服务
  registration.pushManager.getSubscription().then(function(subscription) {
	if (!subscription) {
	  registration.pushManager.subscribe({
		userVisibleOnly: true,
		applicationServerKey: '' // 无需指定 根据manifest.json中的gcm_sender_id
	  }).then(function(subscription) {
	  	console.info('已经完成订阅~');
		console.log(subscription);
		const subscription_id = subscription.endpoint.split('/gcm/send/')[1];
		//saveSubscriptionID(subscription);
		window.fetch('https://4a5ff7de.ngrok.io/pwa/save?id=' + subscription_id);
	  });
	}
  });
});

// service-worker.js
self.addEventListener('push', function(event) {
  self.registration.showNotification('提示', {
    body: '这里是提示信息'
  })
});
```

![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/b602d6c4-5ea5-421e-9745-95e25915e64f.png "")


* ....提供和客户端交互的各种api...


```javascript
// service-worker.js
self.clients.focus();
```

![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/c4f8ada4-0e1f-4f97-b491-a16f06ac3e59.png "")


* 难点：Service Worker 生命周期


![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/07a3668c-613a-489e-a9e7-01270ec12095.png "")


* 何时更新service-worker？
  * 同一个sw，浏览器会判断文件内容变化，来决定是否执行新的sw逻辑，执行了新的sw逻辑之后，可以重新安装
  * 重新安装的sw可以执行下面的代码立即生效

 
  ```javascript
  // 下面代码都在 ServiceWorkerGlobalScope 下执行
  self.skipWaiting();
  self.clients.claim();
  ```
 
* 如何终止，建议在业务逻辑代码中执行终止逻辑


```
navigator.serviceWorker.ready.then(function(registration) {
  registration.unregister();
});
```

---


* 坑点
  * 严格同域，不仅仅是同一个域名，而是某一个具体的目录，取决于`service-worker.js`注册时所在的目录

 
  ```javascript
  // 下述代码将sw注册在/dist/目录下
  // 则相同域名下的其他目录 不能使用该service-worker.js
  navigator.serviceWorker.register('/dist/service-worker.js');
  ```
 
  * 严格要求https，不过`127.0.0.1`以及`localhost`被chrome视为安全源，可以进行调试；此外`ngrok`也提供相应的https代理服务


### Cache Storage

| 存储方式 | 范围 | 使用方式 |
| :--- | :--- | :--- |
| `local storage` | 一个域下 | `key-value`的形式，key一般是字符串，即使是array或者Object也会转为字符串 |
| `session storage` | 一次会话(包含浏览器恢复以及刷新页面) | 和`local storage`大致相同 |
| `cache storage` | 一个域 | key/value为任意值，甚至可以是一个流对象，可以设置多个storage |


### Push & Notification

![undefined | center](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/d3272c5e-8ce5-4d07-a893-cc59fccc46b8.png "")


* 坑点：chorme自己的Web Push，依赖于自己的云推送服务`firebase`，我们距离它有`一墙之隔`
 
* 国内的[uc开放平台](http://open-uc.uc.cn/)也在做自己的推送，有兴趣可以了解一下


## 做PWA需要重点关注的问题
 
* 缓存：如何缓存 & 缓存如何更新
  * 图片？校验信息？
  * 如何处理异步接口的返回信息？

 
* 推送
  * 与google技术耦合性较强，大环境处于国内，google在墙外


## PWA现状 & 前景

* 国外：
  * 整站接入：[https://www.flipkart.com/](https://www.flipkart.com/)
  * AliExpress
  * Lazada
 
* 但是对大环境有要求，比如android占比相当高，原生chrome浏览器使用频率较高
* Safari 在预览版里面已经开始支持 ServiceWorker，但是很多API实现的并不标准
* UC 在自己的内核里面实现了基于自身消息推送的一套机制


---


* 国内一些站点也有接入
  * vuejs中文网：[https://cn.vuejs.org](https://cn.vuejs.org)
  * 打开控制台可以切换到`Application`tab，查看有那些站点使用

 
* [D2-PWA带来急速离线web](https://files.alicdn.com/tpsservice/18ee3d30f31eefbab91915f39771d780.pdf)


---


* 工具
  * [sw-precache](https://www.npmjs.com/package/sw-precache)：帮助生成通用的缓存代码，需要做业务定制，比如`异步请求的缓存处理`
  * [react-pwa](https://www.reactpwa.com/)：同构输出PWA


---


* 国内的发展前景有一定的局限性
  * 由于现在的推送依赖于google的firebase，墙内使用不是很方便
  * 此外国内的android操作系统属于深度定制，市面上流行的浏览器基本面目全非，对PWA的各项技术支持不尽相同，推广比较难
  * 希望：[uc-web-push](http://open-uc.uc.cn/document/develop/webpush-v3)

