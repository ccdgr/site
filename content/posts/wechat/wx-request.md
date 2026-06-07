---
date: '2022-03-17T09:42:13+08:00'
draft: false
title: '微信小程序 wx.request 封装与全局挂载'
tags: ["WeChat", "JavaScript"]
description: "封装 wx.request 统一处理 token、异常、loading，挂载到 app 全局替代原生请求"
toc: true
---

> 小程序项目里到处散落着 `wx.request`，token 过期处理、loading 状态、错误提示各有各的写法。统一封装一次，全局挂载，所有页面直接用 `app.request`。

---

## 封装目标

- 自动携带 token（从 storage 读取）
- token 过期自动跳转登录
- 可选 loading 遮罩
- 统一错误提示
- Promise 风格调用
- 挂载到 `app` 全局可用

---

## 完整封装

在项目根目录创建 `utils/request.js`：

```javascript
const BASE_URL = 'https://api.example.com';

/**
 * 统一请求封装
 * @param {Object} options - 请求配置
 * @param {string} options.url - 接口路径（相对路径，自动拼接 BASE_URL）
 * @param {string} [options.method='GET'] - 请求方法
 * @param {Object} [options.data] - 请求参数
 * @param {Object} [options.header] - 自定义 header
 * @param {boolean} [options.showLoading=true] - 是否显示 loading
 * @param {string} [options.loadingText='加载中...'] - loading 文案
 * @returns {Promise}
 */
function request(options = {}) {
  const {
    url,
    method = 'GET',
    data = {},
    header = {},
    showLoading = true,
    loadingText = '加载中...',
  } = options;

  // 1. loading
  if (showLoading) {
    wx.showLoading({ title: loadingText, mask: true });
  }

  // 2. 组装 header — 自动带 token
  const token = wx.getStorageSync('token');
  const headers = {
    'Content-Type': 'application/json',
    ...header,
  };
  if (token) {
    headers['Authorization'] = `Bearer ${token}`;
  }

  // 3. 发起请求
  return new Promise((resolve, reject) => {
    wx.request({
      url: BASE_URL + url,
      method,
      data,
      header: headers,
      success(res) {
        if (showLoading) wx.hideLoading();

        const { statusCode, data } = res;

        // HTTP 成功
        if (statusCode === 200) {
          // 业务码判断（按你们后端约定调整）
          if (data.code === 0) {
            resolve(data.data);
          } else if (data.code === 401) {
            // token 过期
            wx.removeStorageSync('token');
            wx.showToast({ title: '登录已过期', icon: 'none' });
            wx.reLaunch({ url: '/pages/login/index' });
            reject(data);
          } else {
            wx.showToast({ title: data.msg || '请求失败', icon: 'none' });
            reject(data);
          }
        } else if (statusCode === 401) {
          wx.removeStorageSync('token');
          wx.showToast({ title: '登录已过期', icon: 'none' });
          wx.reLaunch({ url: '/pages/login/index' });
          reject(res);
        } else {
          wx.showToast({ title: `网络错误 ${statusCode}`, icon: 'none' });
          reject(res);
        }
      },
      fail(err) {
        if (showLoading) wx.hideLoading();
        wx.showToast({ title: '网络异常，请检查网络', icon: 'none' });
        reject(err);
      },
    });
  });
}

/**
 * 快捷方法
 */
function get(url, data, options = {}) {
  return request({ ...options, url, method: 'GET', data });
}

function post(url, data, options = {}) {
  return request({ ...options, url, method: 'POST', data });
}

function put(url, data, options = {}) {
  return request({ ...options, url, method: 'PUT', data });
}

function del(url, data, options = {}) {
  return request({ ...options, url, method: 'DELETE', data });
}

// 导出快捷方法 + 原始方法
module.exports = {
  request,
  get,
  post,
  put,
  del,
};
```

---

## 挂载到全局

在 `app.js` 中引入并挂载，这样所有页面通过 `getApp().request()` 即可使用，无需每个页面单独 `require`。

```javascript
// app.js
const http = require('./utils/request');

App({
  onLaunch() {
    // 检查登录态等...
  },

  // 挂载请求方法
  request: http.request,
  get: http.get,
  post: http.post,
  put: http.put,
  del: http.del,
});
```

---

## 页面中使用

```javascript
// pages/user/index.js
const app = getApp();

Page({
  data: {
    userInfo: null,
  },

  onLoad() {
    this.fetchUserInfo();
  },

  async fetchUserInfo() {
    try {
      const userInfo = await app.get('/api/user/profile');
      this.setData({ userInfo });
    } catch (err) {
      console.log('获取用户信息失败', err);
    }
  },

  async updateNickname(nickname) {
    try {
      // 不显示 loading
      await app.post('/api/user/update', { nickname }, { showLoading: false });
      wx.showToast({ title: '修改成功' });
    } catch (err) {
      // 已在封装层统一提示，这里只做额外处理
    }
  },

  async uploadFile(filePath) {
    wx.showLoading({ title: '上传中...' });
    try {
      const result = await app.post('/api/upload', { filePath });
      return result;
    } finally {
      wx.hideLoading();
    }
  },
});
```

---

## 按需扩展

### 请求拦截器

```javascript
function request(options) {
  // 前置拦截 — 比如加时间戳防缓存
  if (options.data) {
    options.data._t = Date.now();
  }

  // ... 原有逻辑 ...

  // 后置拦截 — 比如打日志
  const originalSuccess = options.__onSuccess;
  Object.assign(options, {
    __onSuccess: (data) => {
      console.log(`[request] ${options.url} ->`, data);
      originalSuccess && originalSuccess(data);
    },
  });
}
```

### 并发请求

```javascript
const [user, orders] = await Promise.all([
  app.get('/api/user/profile'),
  app.get('/api/order/list', { page: 1 }),
]);
```

### 静默请求（不显示 loading 和 toast）

```javascript
async function silentGet(url, data) {
  try {
    return await app.request({
      url,
      method: 'GET',
      data,
      showLoading: false,
    });
  } catch {
    return null; // 静默失败
  }
}
```

---

## 注意事项

1. **token 存储 key** 按实际项目约定调整，`wx.getStorageSync('token')` 中的 `'token'` 可能是 `'access_token'` 等
2. **业务码** `data.code === 0` 和 `401` 需要和后端约定一致
3. **登录页路径** `/pages/login/index` 替换为实际路径
4. **BASE_URL** 生产/开发环境可用小程序环境变量区分
5. 文件上传 `wx.uploadFile` 需要单独封装，不支持 `wx.request` 的 header 透传方式
