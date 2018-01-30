---
title: react单元测试
tags: react unit-test
---

> 这里主要记录react组件单元测试相关的知识 


## 总结一下

- 单测的目的是为了保障组件的可靠性和稳定性
	- 可靠性要求组件针对输入有固定的输出
	- 稳定性则在于，对项目进行重构的时候，保证不会影响现有的api调用方式返回的结果
- 单测覆盖率有一定的意义，但是其主要意义还是在于推动我们去反思代码的设计是否有问题，是否存在废代码，理清代码逻辑关系

## 单测行覆盖率的意义

- 分析未覆盖部分的代码，从而反推在前期测试设计是否充分，没有覆盖到的代码是否是测试设计的盲点，为什么没有考虑到？需求/设计不够清晰，测试设计的理解有误，工程方法应用后的造成的策略性放弃等等，之后进行补充测试用例设计。
- 检测出程序中的废代码，可以逆向反推在代码设计中思维混乱点，提醒设计/开发人员理清代码逻辑关系，提升代码质量。
- 代码覆盖率高不能说明代码质量高，但是反过来看，代码覆盖率低，代码质量不会高到哪里去，可以作为测试自我审视的重要工具之一。

## 好的组件

> 一个好的组件，需要具备下面几个特点

- api友好
	- 提供扁平化的使用方式
	- api类型校验，及时告知
- 文档清晰
	- 对应组件的api，有各种场景的demo
- 可测试
	- 便于借助`jest`测试

## 对组件哪些部分测试

> 在涉及`jsx`组件中，单测一般是比较耗时的，原因在于有一层解析机制，`babel-jest`需要对其进行解析、执行；
- 针对`view`部分进行测试，在编写组件的时候，如果能够将组件进行拆分，变为独立的几个view，那么在编写单元测试的时候会更有针对性，因为在模拟属性值以及响应事件的时候会更加明确
- 如果一个组件的props值太多，并且结构很复杂，那么是不利于测试用例编写的
- 如果一个组件的render方法比较庞大，也不利于单元测试编写
- react组件，我们貌似只能通过模拟props的值进行单测，state的值可以通过props的改变进行调整

## 组件代码组织

- 一个util中的依赖文件尽量不要太多，因为这样在做单侧的时候，会引入很多的变量，需要在测试用例中进行重置

```javascript

// import Login from '@ali/lib-login';
// import Mtop from '@ali/lib-mtop';
// import HttpUrl from '@ali/lib-httpurl';
// import windvane from '@ali/lib-windvane';
import { aliapp, os } from 'amfe-env';
import fetch from './mtop/index.js';
// 这里在做打包的时候 希望可以拿掉 因为要做兼容

const Login = window.lib.login;
const Mtop = window.lib.mtop;
const windvane = window.lib.windvane;
const Promise = window.Promise;

export default {
  isLocal: () => {
    const hostname = location.hostname;
    // 手机调试的时候 要判断 是否全是数字的IP地址
    return /127\.0\.0\.1/.test(hostname)
      || /localhost/.test(hostname)
      || /^(\d+\.){1,}\d+$/.test(hostname);
  },
  /**
   * 从页面url中获取指定参数
   * @param {string}
   */
  getUrlParam: (name) => {
    const match = RegExp('[?&]' + name + '=([^&]*)').exec(window.location.search);
    return match && decodeURIComponent(match[1].replace(/\+/g, ' '));
  },
  /**
   * 替换日期字符串 - -> .
   */
  dateStrFormatter: str => {
    return str.substring(0, 10).replace(/-/g, '.');
  },
  /**
   * 将日期字符串转换为日期对象 兼容mobile&pc
   * safari(ios)下不支持 yyyy-MM-DD 的格式 https://stackoverflow.com/questions/4310953/invalid-date-in-safari
   * 因此要替换一下 统一替换为 yy/MM/DD 的格式
   * @param {string} str 日期字符串
   * @return {object} Date instance
   */
  convertDateStrToObj: (str = '', isStamp = false) => {
    let [date, time] = str.split(' ');

    // 自动拼接时间字符串
    !time && (time = '00:00:00');

    const newDate = new Date(`${date.replace(/-/g, '/')} ${time}`);
    if (newDate instanceof Date) {
      return isStamp ? newDate.getTime() : newDate;
    }

    return null;
  }
};

/*
 * 呼起猫客的登录框
 */
export function openTMLogin(suc, err) {
  if (!windvane || !windvane.call) {
    return;
  }
  // 天猫未登录 则强制登录
  // 回调地址参考这里
  // http://groups.demo.taobao.net/hybrid/api/doc/Ali.html#commonCallback
  windvane.call('TMWVApplication', 'forceLogin', {}, suc, err);
}

/*
 * 是否为天猫客户端
 */
export function isTmall() {
  return aliapp.appname === 'TM';
}

/*
 * 是否为淘宝客户端
 */
export function isTaoBao() {
  return aliapp.appname === 'TB';
}

/*
 * 获取浏览器内核信息
 */
function getChromeKernel() {
  const ua = navigator.userAgent;
  const chromeInfo = ua.match(/Chrome\/([\d\.\_]+)/i);
  return chromeInfo ? chromeInfo['0'] : '';
}

/**
 * 获取客户端信息
 * @return {string} iPhone_TM_6.0.0
 */
export function getAppInfo() {
  let osName = os.name;
  let appName = aliapp.appname;
  let appVersion = aliapp.version;
  if (!osName || !appName || !appVersion) {
    return '';
  }
  const appInfo = osName + '_' + appName + '_' + appVersion;
  const kernelInfo = getChromeKernel();

  return kernelInfo ? `${appInfo}_${kernelInfo}` : appInfo;
}

/**
 * 上报客户端信息
 * @param {string} str 除了客户端之外的上报信息
 */
export function sendAppInfo(str) {
  const appInfo = getAppInfo();
  if (!appInfo) {
    return;
  }
  const msg = str ? `${appInfo}_${str}` : appInfo;
  window.__WPO && window.__WPO.custom && window.__WPO.custom('count', msg);
}

export function retCode (iName, flag, consume, msg) {
  window.__WPO && window.__WPO.retCode(iName, flag, consume, msg);
};

export function goldlog (k1, k2 = '', k3 = '', k4) {
  window.goldlog && window.goldlog.record(k1, k2, k3, k4);
}

/**
 * 从mtop返回结果中提取文案信息
 * @param {string} str mtop结果：SUCCESS:调用成功
 */
export function getMtopRetMsg(str) {
  var sep = '::';
  var msgs = str.split(sep);
  return msgs.length > 1 ? msgs[msgs.length - 1] : msgs[0];
}

/**
 *
 * @param {string} version1 版本号1
 * @param {string} version2 版本号2
 */
export function compareVersion(version1, version2) {
  const arrV1 = version1.split('.');
  const arrV2 = version2.split('.');
  const len = Math.max(arrV1.length, arrV2.length);
  for (let i = 0; i < len; i++) {
    if (arrV1[i] === undefined) {
      return false;
    } else if (arrV2[i] === undefined) {
      return true;
    } else if ((arrV1[i] - 0) < (arrV2[i] - 0)) {
      return false;
    } else if ((arrV1[i] - 0) > (arrV2[i] - 0)) {
      return true;
    }
  }
  return true;
}

/**
 * 跳转到详情页面
 * @param {string} itemId 商品id
 * @param {object} extParam 详情链接后面需要拼接的参数
 */
export function goToDetail(itemId, extParam) {
  const host = '//h5.m.taobao.com/awp/core/detail.htm?id=';
  let extPath = '';

  if (extParam) {
    extPath += `&${transObjToUrl(extParam)}`;
  }
  console.info('详情页跳转地址');
  console.log(host + itemId + extPath);

  openWindow(`https:${host}${itemId}${extPath}`);
}

function isObject(param) {
  return Object.prototype.toString.call(param) === '[object Object]';
}

/**
 * 将对象转化为url可拼接的字符串
 * @param {object} obj map结构
 */
export function transObjToUrl(obj) {
  const result = [];
  if (isObject(obj)) {
    for (const key in obj) {
      if (Object.prototype.hasOwnProperty.call(obj, key)
        && typeof obj[key] !== 'undefined'
      ) {
        result.push(`${encodeURIComponent(key)}=${encodeURIComponent(obj[key])}`);
      }
    }
  }
  return result.join('&');
}

/**
 * 打开新页面 优先级 windvane -> location
 * @param {string} url 需要呼起的页面url
 */
export function openWindow(url) {
  windvane.call('WVNative', 'openWindow', {
    url,
  }, () => {
    console.log('success to open new window');
  }, (e) => {
    console.log(e);
    console.log('fail to open new window');
    window.location.href = url;
  });
}

/**
 * 获取客户端名称
 */
export function getAppName() {
  let appName = '';

  if (aliapp.appname) {
    appName = aliapp.appname.toUpperCase();
  }

  return appName;
}

/**
 * 获取店铺信息
 * mtop.taobao.macao.shopactivity.shop.queryshopinfo
 */
export function getShopInfo(sellerId) {
  return new Promise((resolve, reject) => {
    fetch({
      api: 'SHOP_INFO',
      data: { sellerId },
      suc: (resJson) => { resolve(resJson) },
      err: (...args) => { reject(args); }
    });
  });
}

```

- 上述代码中的`window.lib`在单测运行时，并不是很好模拟

### 总结一下

- 一个组件应该包含`container.jsx`、`index.jsx`以及`components`几个部分
	- `index.jsx`是对外的入口，在里面可以做一些高阶组件的封装
	- `container.jsx`里面包含了一些异步请求，数据处理等操作
	- `components`部分，可以理解为PureComponent，只负责逻辑渲染部分
	- 这样划分的好处是，可以便于对每个部分进行独立的单元测试


---

- 反思：什么样的组件库适合做单测？

## 测试框架 & 类库

- 测试框架：[jest][jest]
- 测试类库：[enzyme][enzyme]
- 常用的前端js测试覆盖率框架： [istanbul](https://github.com/gotwarlost/istanbul)

### 使用过程中遇到的坑
- `react`、`react-test-renderer`以及`enzyme-adapter`的版本号要一致

## 测试用例

> 区分测试用例类型

- ui测试：主要用来检测类名等是否有添加上去
	- 比如针对不同的`props`字段，渲染的dom节点上面的类名不同
- 功能测试
	- utils的输入输出
	- react组件节点
	- 异步数据返回结果，返回结果是mock的

- 好的测试用例在项目进行重构的时候尤其重要，如果对项目进行重构，保证api不变，那么以往的测试用例跑一边就可以覆盖大部分需求，
- 测试用例是在逐步完善的，开源或者是项目使用都是为了增加模块的稳定性


[jest]: http://facebook.github.io/jest/zh-Hans/
[enzyme]: http://airbnb.io/enzyme/