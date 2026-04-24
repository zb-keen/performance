# Skynet 1.5.6 代码分析

## 一、整体架构

### 1.1 项目定位
Skynet 是一个前端性能监控与错误上报 SDK，主要用于收集网页的性能指标、JavaScript 错误、API 请求耗时、资源加载情况等数据，并上报到指定服务端。

### 1.2 架构设计
采用模块化设计，通过自定义的模块加载器将各个功能模块打包在一起，主要包含以下核心模块：

```
skynet-1.5.6.js
├── 模块加载器 (主入口)
├── 配置模块 (452)
├── 功能开关模块 (900)
├── 事件总线模块 (28)
├── 数据上报模块 (502)
├── 错误捕获模块 (285)
├── 性能指标模块 (主初始化逻辑)
├── 资源监控模块 (758)
├── 网络检测模块 (501)
├── XHR 拦截模块 (564)
├── 环境信息模块 (883)
├── UserAgent 解析模块 (817)
├── UUID 生成模块 (922)
└── localStorage 工具模块 (534)
```

## 二、核心思想

### 2.1 非侵入式监控
通过拦截原生 API（如 `XMLHttpRequest`）、监听浏览器事件（如 `error`、`unhandledrejection`、`load`、`beforeunload`）来实现对页面行为的监控，无需修改业务代码。

### 2.2 按需上报
通过配置项控制是否上报性能数据、错误数据、API 耗时等，支持采样率设置，降低服务端压力。

### 2.3 可靠传输
优先使用 `navigator.sendBeacon` 进行数据上报，保证页面卸载时数据仍能可靠发送；降级使用普通 XHR。

### 2.4 数据缓存
使用 localStorage 存储客户端标识，实现用户追踪。

## 三、技术实现细节

### 3.1 模块系统
项目使用自定义的模块加载器，核心实现如下：

```javascript
!function(){
  var e = { /* 模块定义对象，key为模块ID */ };
  var t = {}; // 已加载模块缓存
  function r(n) { // 模块加载函数
    var o = t[n];
    if (void 0 !== o) return o.exports;
    var i = t[n] = {exports:{}};
    return e[n](i, i.exports, r), i.exports
  }
  // 入口逻辑
}();
```

### 3.2 配置模块 (452)
从 `window.__SKYNET` 或 `window.__SKYNET_APP` 获取配置，支持配置项包括：
- `appName`: 应用名称
- `appVersion`: 应用版本
- `dev`: 是否开发模式
- `sendError`: 是否上报错误
- `rate`: 采样率
- `checkNet`: 是否检测网络
- `uploadResTime`: 是否上报资源加载时间
- `uploadAPITime`: 是否上报 API 耗时

### 3.3 功能开关模块 (900)
根据配置项决定各个功能是否开启：
- `needSendPerformance`: 是否上报性能数据
- `needSendError`: 是否上报错误
- `needSendRes`: 是否上报资源时间
- `needSendAPI`: 是否上报 API 时间
- `needCheckPaint`: 是否检查绘制
- `needCheckNet`: 是否检测网络
- `startVisible`: 页面初始是否可见

### 3.4 错误捕获模块 (285)
- 监听 `window.error` 事件捕获 JS 运行时错误
- 监听 `window.unhandledrejection` 事件捕获 Promise 未处理异常
- 支持错误过滤（通过 `window.__SKYNET.errorFilter`）
- 收集错误的文件名、行号、列号、堆栈信息

### 3.5 XHR 拦截模块 (564)
重写 `XMLHttpRequest` 原型方法，拦截所有 XHR 请求：
- 记录请求开始时间
- 监听 `onreadystatechange`、`onload`、`ontimeout`、`onabort`、`onerror` 事件
- 计算请求耗时
- 上报 API 耗时和错误信息

### 3.6 性能指标收集
使用 `window.performance.timing` 和 `window.performance.getEntriesByType` 收集性能数据：
- 页面加载各阶段时间（unload、cache、tcp、dns、fb、domready、fullLoad、htmlLoad、pageRender）
- 资源加载时间
- 支持自定义性能指标（通过 `profilekeys` 事件）

### 3.7 数据上报模块 (502)
- 上报地址默认：`https://mtm.online-cmcc.cn/cmos/profile`
- 上报数据包含：系统信息、浏览器信息、页面信息、时间戳、UUID 等
- 支持批量上报和定时上报
- 页面卸载时通过 `beforeunload` 事件确保数据发送

### 3.8 环境信息收集 (883, 817)
- 解析 UserAgent 获取浏览器、操作系统、设备信息
- 识别多种移动端应用（微信、支付宝、10086APP 等）
- 获取页面 URL 和标题

### 3.9 UUID 生成 (922)
- 使用 localStorage 存储客户端标识 `cmosClientId`
- 结合时间戳和随机字符串生成唯一 UUID

### 3.10 资源监控 (758)
- 定期（5秒）检查资源加载情况
- 识别慢资源（超过配置阈值）
- 上报资源加载时间

### 3.11 网络检测 (501)
- 检测资源加载状态
- 通过 HEAD 请求检查资源可访问性

## 四、初始化流程

1. 读取配置
2. 检查功能开关
3. 初始化事件监听
4. 拦截 XHR
5. 等待页面加载或 `profilekeys` 事件
6. 收集性能数据并上报
7. 页面卸载时确保数据上报

## 五、关键特性

- 版本号：1.5.6
- 支持自定义事件上报（`window.__SKYNET.send`、`window.__SKYNET.sendAll`）
- 支持省份信息上报
- 支持 WebView 环境检测
- 支持错误过滤和状态码过滤

