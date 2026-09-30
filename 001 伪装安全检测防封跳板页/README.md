# 伪装安全检测防封跳板页 - 逆向分析报告

## 1. 项目概览

| 项目属性 | 详细描述 |
| :--- | :--- |
| **所属类型** | 流量防封跳板页 / Cloaking 障眼法落地页 / 广告引流过滤过渡页 |
| **核心文件** | `index.html` (约 22 KB) |
| **技术架构** | 原生 HTML5 + Vanilla CSS3 + 原生 JavaScript (ES6) |
| **第三方组件** | 百度统计 (`hm.baidu.com`) |
| **核心目标** | 规避平台安全审查与搜索引擎爬虫、阻断 PC 端审查人员、引导移动端真实用户流向目标落地页 |

---

## 2. 核心业务流程图

```mermaid
graph TD
    Start([用户/爬虫发起访问]) --> HeadCheck[HTML Head 爬虫拦截/反快照]
    HeadCheck --> Track[百度统计加载埋点]
    Track --> DeviceCheck{设备环境检测 isPC}
    
    %% PC 端分支
    DeviceCheck -->|判定为 PC 终端| PC_Block[显示 PC 访问受限 UI]
    PC_Block --> PC_Count[5秒倒计时]
    PC_Count --> PC_404[重定向至 /404.html]
    
    %% 移动端分支
    DeviceCheck -->|判定为 移动端| Mobile_Card[展示安全检测卡片 UI]
    Mobile_Card --> Step1[步骤 1: 设备真实性验证 (600ms)]
    Step1 --> Step2[步骤 2: 网络环境检测 (600ms)]
    Step2 --> Step3[步骤 3: 安全资源加载 (900ms)]
    Step3 --> Step4[步骤 4: 安全通道建立 (900ms)]
    
    Step4 --> AutoJump{检测完成}
    AutoJump -->|倒计时结束| Jump[跳转至目标落地页 targetUrl]
    Mobile_Card -->|手动点击立即跳转| Jump
```

---

## 3. 技术机制深度拆解

### 3.1 搜索引擎与网络爬虫规避 (SEO Cloaking)
在 `<head>` 部分配置了密集的 Meta 标签，旨在全面阻断搜索引擎收录与内容快照：
* **全网搜索引擎阻断**：`robots: noindex, nofollow, noarchive, nosnippet, noimageindex, nocache`
* **定向爬虫屏蔽**：单独对 `Googlebot`、`Baiduspider`、`Bingbot`、`Yandex`、`Sogou`（搜狗）、`360Spider`（360/好搜）指定了 `noindex, nofollow, noarchive`。
* **禁用缓存与转码**：配置 `Pragma: no-cache` 和 `Expires: 0` 防止运营商转码与代理服务器缓存。

### 3.2 设备指纹识别与流量分流 (Device Fingerprinting & Gating)
通过三维特征检测访问者的设备环境，严格区分移动端真实用户与 PC 端审查人员：
1. **User-Agent 关键词匹配**：
   * PC 端关键词：`windows`、`macintosh`、`linux`、`x11`
   * 移动端关键词：`mobile`、`android`、`iphone`、`ipad`、`ipod`
2. **触控支持检测**：`'ontouchstart' in window || navigator.maxTouchPoints > 0`
3. **屏幕尺寸判定**：`window.innerWidth <= 768`

**分流行为**：
* **PC 端**：隐藏安全卡片，显示 `#pcWarning` 提示“很遗憾不能展现真实的自己”，启动 5 秒倒计时后强制重定向至 `/404.html`。
* **移动端**：放行进入假安全检测主界面。

### 3.3 伪装安全检测交互逻辑 (Fake Verification UX)
利用用户对绿色盾牌、安全校验的信任心理，平滑过渡跳转延时：
* **模拟 4 项检测项目**：
  1. 设备真实性验证（耗时 600ms）
  2. 网络环境检测（耗时 600ms）
  3. 安全资源加载（耗时 900ms）
  4. 安全通道建立（耗时 900ms）
* **视觉动态反馈**：加载时呈现 CSS 环形旋转动画（`animation: spin`），完成后变为绿色对勾图标（`✓`），并伴随进度条从 `0%` 平滑递增至 `100%`。
* **跳转控制**：
  * **自动跳转**：检测跑满 100% 后延迟 300ms 触发 `autoJump()`，跳转到 `config.targetUrl`。
  * **手动点击**：底部大按钮随时可触发 `immediateJump()` 立即跳转。

### 3.4 前端反逆向与防审查 (Anti-Debugging & Source Protection)
* **禁止文本复制与框选**：全局 CSS 设定 `-webkit-user-select: none; user-select: none;`，并拦截 `selectstart` 事件。
* **禁止右键菜单**：监听并拦截 `contextmenu` 事件。
* **拦截调试与查看源码快捷键**：
  * `F12`（打开开发者工具）
  * `Ctrl + U`（查看网页源码）
  * `Ctrl + Shift + I`（审查元素）
  * `Ctrl + Shift + J`（控制台）
  * `Ctrl + S`（保存网页为本地文件）

### 3.5 防嵌套与防跳出保护 (Anti-Frame & Navigation Lock)
* **防 Iframe 沙盒嵌套（Framebusting）**：
  ```javascript
  if (window.top !== window.self) {
      window.top.location = window.self.location;
  }
  ```
  防止安全扫描器或第三方审核平台将该页面置于 iframe 容器中进行沙箱监控。
* **拦截意外关闭/回退**：监听 `beforeunload` 事件，在检测流程未完成前阻止用户取消跳转。

### 3.6 流量统计与监控
* 页面中注入了百度统计代码：
  ```html
  <script>
  var _hmt = _hmt || [];
  (function() {
    var hm = document.createElement("script");
    hm.src = "https://hm.baidu.com/hm.js?64c3f33887624bbbb02fa196fa48b147";
    var s = document.getElementsByTagName("script")[0]; 
    s.parentNode.insertBefore(hm, s);
  })();
  </script>
  ```
  用于追踪推广流量来源、进站转化率及移动端占比。

---

## 4. 攻防对抗与安全性评估

### 💡 优势特征
1. **纯静态零依赖**：加载极快（<50ms 首屏渲染），移动端体验流畅，不依赖任何第三方 UI 库。
2. **信任感营造到位**：拟物化的安全图标、进度条与动态步骤反馈，大幅降低跳出率并减少用户反感。
3. **多层响应式适配**：针对标准手机屏幕及 iPhone SE 等小屏均做了断点微调（360px / 480px）。

### ⚠️ 脆弱点与被破解风险
1. **纯前端分流（无后端验证）**：
   * 审核工具或爬虫只需在 Headless 浏览器（如 Puppeteer / Playwright）中伪造 User-Agent 和 Touch 属性，即可轻松绕过 PC 端拦截。
   * 通过查看网页源码或抓包代理（Charles/Fiddler/Mitmproxy），可直接提取到 JS 变量中的 `config.targetUrl`。
2. **反调试机制较初级**：
   * 提前开启 DevTools 后再输入 URL 访问，按键拦截逻辑完全失效。
   * 禁用浏览器 JavaScript 后，所有前端反右键与防选择限制全部失效。
3. **特征指纹过于明显**：
   * 固定的 class 命名（如 `.security-card`、`.check-item`）、固定文本（“设备真实性验证”）及百度统计 Token，极易被平台自动化反欺诈算法录入特征库识别。

---

## 5. 架构优化与改进建议

1. **目标地址动态获取**：将 `config.targetUrl` 改为向后端接口发起带有时效性 Token/签名参数的异步请求（Fetch API），由后端返回动态跳转 URL，避免明文暴露目标地址。
2. **服务端前置分流（推荐）**：在 Nginx / CDN 边缘计算（Edge Workers）层根据 IP 地理位置、ASN、User-Agent 直接完成 PC/移动端分流，提高对抗平台审核的防御强度。
3. **JS 代码混淆**：使用 Terser + JavaScript Obfuscator 对核心控制流、字符串常量及按键拦截逻辑进行高强度混淆加密。
