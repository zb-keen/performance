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
  !function(){
    var e = r(452), // 配置模块
        t = r(883), // 环境信息模块
        n = r(28),  // 事件总线模块
        o = r(502), // 数据上报模块
        i = r(285), // 错误捕获模块
        s = r(79),  // 性能指标采集模块
        a = r(758), // 资源监控模块
        u = r(922), // UUID 生成模块
        p = r(502), // 数据上报模块（引用）
        c = r(900), // 功能开关模块
        l = r(541), // 信息统计模块
        d = r(910), // 省份信息模块
        f = r(900), // 功能开关模块（引用）
        m = n.bindBrowserEvent; // 绑定浏览器事件方法
    // 初始化逻辑...
  }();
}();
```

### 3.2 配置模块 (452)
**实现**：从 `window.__SKYNET` 或 `window.__SKYNET_APP` 获取配置，若配置不存在则使用默认空对象。

```javascript
452:function(e){
  var t = {};
  -1 !== navigator.userAgent.indexOf(" sample/true") ? 
    window.__SKYNET_APP && (t = window.__SKYNET_APP) : 
    t = window.__SKYNET || {};
  e.exports = t
}
```

**配置项**：
- `appName`: 应用名称
- `appVersion`: 应用版本
- `dev`: 是否开发模式
- `sendError`: 是否上报错误
- `rate`: 采样率
- `checkNet`: 是否检测网络
- `uploadResTime`: 是否上报资源加载时间
- `uploadAPITime`: 是否上报 API 耗时
- `apiUrl`: API 地址匹配规则
- `xhrStatusFilter`: XHR 状态码过滤

### 3.3 功能开关模块 (900)
**实现**：根据配置项计算各个功能是否开启，支持采样率控制。

```javascript
900:function(e,t,r){
  var n = r(452), o = !1, i = !1, s = !1, a = !1;
  n.appName && !n.dev && (
    "hidden" !== document.visibilityState && (a = !0),
    o = !0,
    (void 0 === n.sendError || n.sendError) && (i = !0),
    void 0 !== n.rate && (o = 0 !== n.rate && (100 === n.rate || 100 * Math.random() >> 1 <= n.rate)),
    n.checkNet && (s = !0)
  );
  e.exports = {
    needSendPerformance: o,
    needSendError: i,
    needSendRes: function() { return n.uploadResTime },
    needSendAPI: function() { return n.uploadAPITime },
    needCheckPaint: function() { return void 0 === n._needCheckPaint || n._needCheckPaint },
    needCheckNet: s,
    startVisible: a
  }
}
```

### 3.4 事件总线模块 (28)
**实现**：提供事件的绑定、触发、解绑等功能，是模块间通信的核心。

```javascript
28:function(e,t,r){
  var n = r(452), o = {};
  function i(e,t,r){...}
  function s(e,t,r,n){...} // 绑定事件
  function a(e,t,r){...} // 触发事件
  function u(e,t,r){...} // 解绑事件
  function p(e,t){...} // 获取监听器
  function c(e,t,r,n){...} // 绑定一次性事件
  function l(e,t){...} // 清除所有事件
  function d(e,t,r){...} // 绑定浏览器事件（兼容 addEventListener/attachEvent）
  e.exports = {
    on: s,
    fire: a,
    off: u,
    getListeners: p,
    once: c,
    clear: l,
    bindBrowserEvent: d
  }
}
```

### 3.5 错误捕获模块 (285)
**实现**：监听 `window.error` 和 `window.unhandledrejection` 事件，捕获 JS 运行时错误和 Promise 未处理异常。

```javascript
285:function(e,t,r){
  var n = r(28), o = n.bindBrowserEvent, i = r(502), s = r(883),
      a = r(922), u = r(900), p = r(541), c = r(501),
      l = r(910), d = [], f = s();
  function m(e){
    var t, r, o = e.target || e.srcElement, s = "", u = o.src || o.href;
    if (o === window) {
      // 处理 JS 运行时错误和 Promise 异常
      var m = "", h = "";
      e.message || e.error ? (
        m = e.message || (e.error || {}).message || "",
        h = (e.error || {}).stack || ""
      ) : e.reason && e.reason.message ? (
        m = "Uncaught (in promise) " + e.reason.message,
        h = e.reason.stack
      ) : e.reason && (m = "Uncaught (in promise) " + e.reason, h = "");
      m = m.substring(0, 200);
      window.__SKYNET.errorFilter && window.__SKYNET.errorFilter(m) ? 
        s = "" : "" !== m.replace(/\s+/g, "") && (
          r = "9.2.0",
          s = "0" !== (t = f.clientVersion) && (t[0] < r[0] || t[2] < r[2] || t[4] < r[4]) && 
            -1 !== m.indexOf("Script error") ? "" : 
            JSON.stringify({
              line: e.lineno || 0,
              column: e.colno || 0,
              msg: m || "",
              fileName: e.filename || "",
              stack: h,
              pageName: f.page()
            })
        );
      s && (
        n.fire("JsError", s),
        p.addError(),
        i.send({...}) // 上报错误
      );
    } else if (u && /* 过滤掉特定资源 */) {
      // 处理资源加载错误
      i.sendBeaconWithInfo(o.tagName, "resource", (o.className || "") + " " + u);
      c.add(u);
      d.push(u);
      -1 === u.indexOf(".js") && -1 === u.indexOf(".css") || p.addErrorRes();
    }
  }
  e.exports = {
    errors: d,
    catchError: function() {
      u.needSendPerformance && u.needSendError && (
        o(window, "error", m, !0),
        o(window, "unhandledrejection", m, !0)
      );
    }
  }
}
```

### 3.6 XHR 拦截模块 (564)
**实现**：重写 `XMLHttpRequest` 构造函数和原型方法，拦截所有 XHR 请求，计算耗时并上报。

```javascript
564:function(e,t,r){
  var n = r(452), o = r(28), i = r(758), s = r(447),
      a = r(502).sendBeaconWithInfo, u = r(900), p = r(541);
  // 包装 XMLHttpRequest
  XMLHttpRequest = function() {
    this._xhr = new l.XHR; // l.XHR 是原生 XMLHttpRequest
    this._start = Date.now();
    this._url = "";
    this._method = "";
  };
  XMLHttpRequest.prototype.open = function(e, t, r, n, o) {
    this._url = t;
    this._method = e;
    this._xhr.open(e, t, r, n, o);
  };
  XMLHttpRequest.prototype.send = function(e) {
    var t = this, r = Date.now();
    // 检查是否需要上报 API 时间
    function f(e) {
      return u.needSendAPI() && (void 0 === o.apiUrl || o.apiUrl.some(function(t) {
        return -1 !== e.indexOf(t);
      }));
    }
    // 处理请求完成
    t._xhr.onreadystatechange = function(e) {
      4 === t._xhr.readyState && (
        200 <= t._xhr.status && t._xhr.status < 300 || 304 === t._xhr.status ? (
          p.addAPIDONE(),
          i.fire("apiTime", {
            time: Date.now() - t._start,
            url: t._url,
            netType: t._xhr.status
          })
        ) : n(e)
      );
      // 同步状态到包装对象
      t.readyState = t._xhr.readyState;
      t.status = t._xhr.status;
      t.onreadystatechange && t.onreadystatechange(e);
    };
    // 其他事件处理...
    this._xhr.send(e);
  };
  // 其他原型方法...
  e.exports = { XHR: l, getMaxTime: function() { return {} } };
}
```

### 3.7 数据上报模块 (502)
**实现**：负责将收集到的数据发送到服务端，支持 `sendBeacon` 和 XHR 两种方式。

```javascript
502:function(e,t,r){
  function n(e,t,r){...}
  var o, i, s = r(452), a = r(564),
      u = (s || {}).api || "https://mtm.online-cmcc.cn/cmos/profile",
      p = r(883), c = r(922), l = r(900),
      d = r(327), f = r(28), m = r(910),
      h = p(), g = [], w = [], v = [], x = 0,
      y = [], b = "hidden" !== document.visibilityState, _ = !1;
  
  function S(e) { y.length < 50 && y.push(e); }
  function O() { var e = y.shift(); e && T.post(e, O); }
  
  // 页面可见性变化时发送缓存数据
  l.startVisible || document.addEventListener("visibilitychange", function() {
    "hidden" !== document.visibilityState && (b = !0, _ || (_ = !0, O()));
  });
  
  var E = null, T = {
    post: function(e, t) {
      if (l.needSendPerformance && 0 !== e.length && !(x >= 3)) {
        if (l.startVisible || b) {
          var r = JSON.stringify(e), n = new (0, a.XHR);
          n.timeout = 3e3;
          n.open("POST", u);
          n.send(r);
          n.onreadystatechange = function() {
            4 === n.readyState && (
              n.status >= 200 && n.status < 300 || 304 === n.status || x++,
              t && t()
            );
          };
          // 尝试使用 WebView 日志
          try { ... } catch (e) {}
        } else S(e);
      }
    },
    sendBeacon: function(e) {
      l.needSendPerformance && (
        x >= 3 || 0 !== e.length && (
          l.startVisible || b ? 
            navigator.sendBeacon && (navigator.sendBeacon(u, JSON.stringify(e)) || x++) : 
            S(e)
        )
      );
    },
    // 其他发送方法...
  };
  
  // 监听 API 时间事件
  f.on("apiTime", function(e) {
    T.sendALLInfo("api", "network", {
      time: e.time,
      url: e.url,
      netType: e.netType
    }, "netTime");
  });
  
  e.exports = T;
}
```

### 3.8 环境信息模块 (883, 817)
**883 模块**：获取应用和设备信息
```javascript
883:function(e,t,r){
  var n = r(452), o = r(817);
  function i() { return document.title, (...); }
  (n = n || {}).__versions = "1.5.6";
  var s = o(navigator.userAgent),
      a = (/leadeon\/([^\/]+)/.exec(navigator.userAgent) || ["", "0"])[1];
  e.exports = function(e) {
    var t = n.noSendDevice;
    return {
      browser: t ? "" : ("0" === a ? "" : "webview-") + (s.browser || ""),
      browserVersion: t ? "" : s.bv || "0",
      os: t ? "" : s.os || "0",
      OSVersion: t ? "" : s.ov || "0",
      appName: n.appName || "unknown",
      appVersion: n.appVersion || "0",
      clientVersion: a,
      page: i
    };
  };
}
```

**817 模块**：解析 UserAgent
```javascript
817:function(e){
  var t = { browser: "", bv: "", os: "", ov: "", product: "" },
      r = { MicroMessenger: "WeChat" };
  e.exports = function(e) {
    try {
      // 复杂的 UserAgent 解析逻辑...
      return s || (s = { browser: "", bv: "", os: "", ov: "", product: "" }), s;
    } catch (e) { return t; }
  };
}
```

### 3.9 UUID 生成模块 (922)
**实现**：使用 localStorage 存储客户端标识，结合时间戳和随机字符串生成唯一 UUID。

```javascript
922:function(e,t,r){
  var n = r(452), o = r(534),
      i = (performance.timing.navigationStart || performance.timing.redirectStart || 
            performance.timing.fetchStart) + "-";
  function s() {
    for (var e = "", t = 4; t--;) 
      e += "01234456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"[63 * Math.random() >> 0];
    return e;
  }
  var a = o.getItem("cmosClientId");
  a || (a = s(), o.setItem("cmosClientId", a));
  i += s();
  e.exports = {
    getUUID: function() {
      var e = "";
      return n.getUID ? (e = n.getUID(), a + "-" + e + "-" + i) : a + "-" + i;
    }
  };
}
```

### 3.10 资源监控模块 (758)
**实现**：定期检查资源加载情况，识别慢资源并上报。

```javascript
758:function(e,t,r){
  var n = r(502), o = r(447), i = r(900),
      s = r(28), a = r(285),
      u = 1e3 * o.maxResource, p = {}, c = {};
  function l(e) { return a.errors.some(function(t) { return t === e; }) ? 0 : 1; }
  function d() {
    var e = [];
    try { e = window.performance.getEntriesByType("resource"); } catch (e) {}
    for (var t, r = [], o = [], s = 0, a = e.length; s < a; s++) {
      // 处理资源加载时间
      "link" !== e[s].initiatorType && "script" !== e[s].initiatorType && "img" !== e[s].initiatorType || 
        c[e[s].name] || (c[e[s].name] = !0, o.push({
          name: e[s].name,
          resType: (t = e[s].name, t.split("?")[0].split(".").pop() || e[s].initiatorType),
          time: e[s].duration >> 0,
          netType: l(e[s].name)
        }));
      // 识别慢资源
      "xmlhttprequest" !== e[s].initiatorType && "fetch" !== e[s].initiatorType && 
        "" !== e[s].nextHopProtocol && e[s].duration > u && !p[e[s].name] && 
        -1 === (e[s].name || "").indexOf("dcs.gif") && -1 === (e[s].name || "").indexOf("sa.gif") && 
        (p[e[s].name] = !0, r.push(e[s].name + " - " + (e[s].duration >> 0)));
    }
    // 上报资源加载时间和慢资源
    o.length && i.needSendRes() && n.sendBeaconWithInfo("resourse", "time", o, "resTime");
    r.length && i.needSendPerformance && i.needSendError && 
      n.sendBeaconWithInfo("resourse", "slow", { msg: r });
    setTimeout(d, 5e3); // 每 5 秒检查一次
  }
  s.bindBrowserEvent(window, "beforeunload", function() { d(); });
  var f = !1;
  e.exports = {
    init: function() { f || (f = !0, d()); }
  };
}
```

### 3.11 网络检测模块 (501)
**实现**：检测资源加载状态，通过 HEAD 请求检查资源可访问性。

```javascript
501:function(e,t,r){
  var n, o = r(900), i = r(541),
      s = r(28), a = r(502).sendBeaconWithInfo,
      u = r(564), p = {}, c = [], l = !1;
  function d() {
    var e = c.pop();
    if (e) {
      l = !0;
      var t = new u.XHR;
      t.open("GET", e);
      t.onerror = function() {
        a("netReason", "resource", "type: 0; code: " + t.status + "-error; url: " + e);
        l = !1;
        d();
      };
      t.ontimeout = function() {
        a("netReason", "resource", "type: 0; code: " + t.status + "-timeout; url: " + e);
        l = !1;
        d();
      };
      t.onload = function() {
        a("netReason", "resource", "type: 1; code: " + t.status + "; url: " + e);
        l = !1;
        d();
      };
      t.send();
    } else l = !1;
  }
  function f() {
    if (i.isonload()) {
      if (l) s.once(f);
      else {
        var e = p;
        p = {};
        c = [];
        Object.keys(e).forEach(function(t) {
          e[t] && e[t].length && c.push(e[t].pop());
        });
        d();
      }
    } else i.setLoadCallback(f);
  }
  e.exports = {
    add: function(e) {
      var t = e.split("//").pop().split("/")[0];
      p[t] || (p[t] = []);
      p[t].push(e);
      o.needCheckNet && (clearTimeout(n), n = setTimeout(f, 1e3));
    }
  };
}

### 3.12 主初始化流程
**实现**：串联各个模块，完成初始化和数据采集。

```javascript
!function(){
  var e = r(452), t = r(883), n = r(28),
      o = r(502), i = r(285), s = r(79),
      a = r(758), u = r(922), p = r(502),
      c = r(900), l = r(541), d = r(910),
      f = r(900), m = n.bindBrowserEvent;
  
  if (!(e = e || {}).init && f.needSendPerformance) {
    var h = function() {
      // 收集性能数据
      var e = (new Date).getTime(),
          r = w.unloadEventEnd - w.unloadEventStart,
          // 其他性能指标...
          f = t();
      return {
        event: "performance",
        // 性能数据...
        uuid: u.getUUID()
      };
    };
    
    var g = function(t, r) {
      if (!v) {
        v = !0;
        var n = r;
        n || (n = s()); // 获取 FP、TTI 等指标
        var i = h();
        i.fp = Math.round(n.fp);
        i.interactive = Math.round(n.tti);
        i.fsp = Math.round(n.fsp);
        // 其他指标...
        "unload" === t ? (i.sendType = "unload", o.sendBeacon(i)) : o.send(i);
      }
    };
    
    i.catchError(); // 初始化错误捕获
    
    var w, v = !1;
    w = window.performance && performance.timing ? performance.timing : {};
    
    // 监听页面加载完成
    m(window, "load", function() {
      l.setOnload(w.loadEventStart - (w.navigationStart || w.fetchStart) >> 0);
    });
    
    // 监听性能指标事件
    n.on("profilekeys", function(t) {
      var r = h(), n = null;
      r.fullLoad > 0 ? (
        a.init(), // 初始化资源监控
        r.fp = Math.round(t.fp),
        r.interactive = Math.round(t.tti),
        r.fsp = Math.round(t.fsp),
        // 其他指标...
        v = !0,
        o.send(r, !0)
      ) : (
        t.fp < t.fsp && (n = t),
        m(window, "load", function() {
          setTimeout(function() {
            a.init(), g("load", n);
          }, 100);
        })
      );
    });
    
    // 页面卸载时上报
    m(window, "beforeunload", function() {
      c.needSendPerformance && g("unload");
    });
    
    // 暴露 API
    e.send = function(e, t) {
      "string" != typeof e && (e = JSON.stringify(e));
      p.sendBeaconWithInfo("custom", t || "user", e);
    };
    e.sendAll = function(e, t) {
      "string" != typeof e && (e = JSON.stringify(e));
      p.sendBeaconWithInfoAll("custom", t || "user", e);
    };
    e.init = !0;
    e.fire = n.fire;
  }
}();
```

## 四、模块交互与数据流向

### 4.1 模块依赖关系
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  配置模块 (452)  │────▶│  功能开关模块 (900)  │────▶│  其他所有模块  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
          ▲                         ▲                         │
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    │
                           ┌─────────────────┐
                           │  事件总线模块 (28)  │
                           └─────────────────┘
                                    │
                                    ▼
                           ┌─────────────────┐
                           │  数据上报模块 (502)  │
                           └─────────────────┘
                                    │
                                    ▼
                           ┌─────────────────┐
                           │  服务端 API  │
                           └─────────────────┘
```

### 4.2 数据流向

1. **数据采集**：
   - 错误捕获模块 (285) → 捕获 JS 错误和资源加载错误
   - XHR 拦截模块 (564) → 收集 API 请求耗时和错误
   - 性能指标模块 → 收集页面加载性能数据
   - 资源监控模块 (758) → 收集资源加载时间和慢资源
   - 网络检测模块 (501) → 检测资源可访问性

2. **数据传递**：
   - 通过事件总线模块 (28) 传递数据（如 `apiTime` 事件）
   - 直接调用数据上报模块 (502) 的方法

3. **数据处理**：
   - 环境信息模块 (883, 817) → 提供设备和应用信息
   - UUID 生成模块 (922) → 提供唯一标识
   - 信息统计模块 (541) → 统计错误和 API 调用次数

4. **数据上报**：
   - 数据上报模块 (502) → 负责将数据发送到服务端
   - 支持批量上报和定时上报
   - 优先使用 `navigator.sendBeacon`，降级使用 XHR

### 4.3 核心交互流程

1. **初始化阶段**：
   - 读取配置（452）
   - 计算功能开关（900）
   - 初始化错误捕获（285）
   - 拦截 XHR（564）

2. **运行时阶段**：
   - 捕获错误和异常（285）
   - 收集 API 耗时（564）
   - 定期检查资源加载（758）
   - 检测网络状态（501）

3. **上报阶段**：
   - 页面加载完成或收到 `profilekeys` 事件 → 上报性能数据
   - 错误发生时 → 实时上报错误
   - 页面卸载时 → 确保所有数据都被上报

## 五、初始化流程

1. **读取配置**：从 `window.__SKYNET` 或 `window.__SKYNET_APP` 获取配置
2. **检查功能开关**：根据配置计算各功能是否开启，支持采样率控制
3. **初始化错误捕获**：监听 `window.error` 和 `window.unhandledrejection` 事件
4. **拦截 XHR**：重写 `XMLHttpRequest` 构造函数和原型方法
5. **监听页面加载**：绑定 `load` 事件，记录页面加载完成时间
6. **监听性能指标**：绑定 `profilekeys` 事件，收集 FP、TTI 等指标
7. **初始化资源监控**：定期检查资源加载情况
8. **页面卸载处理**：绑定 `beforeunload` 事件，确保数据上报
9. **暴露 API**：提供 `window.__SKYNET.send` 和 `window.__SKYNET.sendAll` 方法

## 六、关键特性

- **版本号**：1.5.6
- **非侵入式**：通过事件监听和 API 拦截实现监控，无需修改业务代码
- **按需上报**：支持配置项控制上报内容和采样率
- **可靠传输**：优先使用 `navigator.sendBeacon`，保证页面卸载时数据可靠发送
- **环境适配**：支持多种浏览器和移动端应用环境
- **自定义事件**：支持通过 `window.__SKYNET.send` 上报自定义事件
- **错误过滤**：支持通过 `window.__SKYNET.errorFilter` 过滤错误
- **状态码过滤**：支持通过 `xhrStatusFilter` 过滤 XHR 状态码
- **省份信息**：支持上报省份信息
- **WebView 检测**：支持检测 WebView 环境并使用 WebView 日志

## 七、技术亮点

1. **模块化设计**：使用自定义模块加载器，代码结构清晰
2. **事件驱动**：通过事件总线实现模块间通信，解耦模块依赖
3. **性能优化**：
   - 使用 `sendBeacon` 减少对页面性能的影响
   - 批量上报和定时上报减少网络请求
   - 采样率控制降低服务端压力
4. **兼容性处理**：
   - 兼容不同浏览器的事件绑定方式
   - 降级处理（如 `sendBeacon` 不可用时使用 XHR）
5. **数据安全**：
   - 错误信息脱敏（如截取错误消息长度）
   - 过滤敏感资源（如二维码图片）
6. **可扩展性**：
   - 暴露自定义事件上报 API
   - 支持配置项自定义行为

## 八、使用场景

1. **前端性能监控**：收集页面加载性能指标，识别性能瓶颈
2. **错误监控**：实时捕获 JS 错误和资源加载错误，快速定位问题
3. **API 监控**：监控 API 请求耗时和错误，优化接口性能
4. **用户体验分析**：通过 FP、TTI 等指标评估用户体验
5. **环境统计**：收集设备、浏览器、操作系统信息，为兼容性测试提供依据

## 九、代码优化建议

1. **代码可读性**：
   - 变量命名过于简洁（如 e, t, r），建议使用更具描述性的变量名
   - 部分逻辑过于复杂，建议拆分为更小的函数

2. **性能优化**：
   - 资源监控模块每 5 秒执行一次，可能在复杂页面上造成性能开销
   - 建议增加节流或防抖机制

3. **错误处理**：
   - 部分 try-catch 块为空，建议添加适当的错误处理

4. **代码可维护性**：
   - 模块间依赖关系较复杂，建议添加模块间依赖文档
   - 配置项较多，建议添加配置项说明文档

5. **安全性**：
   - 直接使用 `navigator.userAgent` 可能存在安全风险
   - 建议使用更安全的方式检测浏览器和设备信息

