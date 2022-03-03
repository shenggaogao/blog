# 浏览器相关

## 1 新开一个 tab 页签，浏览器会打开几个进程？

- **浏览器进程**。主要负责界面显示、用户交互、子进程管理，同时提供存储等功能。
- **渲染进程**。核心任务是将 HTML、CSS 和 JavaScript 转换为用户可以与之交互的网页，排版引擎 Blink 和 JavaScript 引擎 V8 都是运行在该进程中，默认情况下，Chrome 会为每个 Tab 标签创建一个渲染进程。出于安全考虑，渲染进程都是运行在沙箱模式下。
- **GPU 进程**。其实，Chrome 刚开始发布的时候是没有 GPU 进程的。而 GPU 的使用初衷是为了实现 3D CSS 的效果，只是随后网页、Chrome 的 UI 界面都选择采用 GPU 来绘制，这使得 GPU 成为浏览器普遍的需求。最后，Chrome 在其多进程架构上也引入了 GPU 进程。
- **网络进程**。主要负责页面的网络资源加载，之前是作为一个模块运行在浏览器进程里面的，直至最近才独立出来，成为一个单独的进程。
- **插件进程**。主要是负责插件的运行，因插件易崩溃，所以需要通过插件进程来隔离，以保证插件进程崩溃不会对浏览器和页面造成影响

## 2 浏览器多进程模型带来的一些问题？

- **更高的资源占用**。因为每个进程都会包含公共基础结构的副本（如 JavaScript 运行环境），这就意味着浏览器会消耗更多的内存资源。
- **更复杂的体系架构**。浏览器各模块之间耦合性高、扩展性差等问题，会导致现在的架构已经很难适应新的需求了。

## 3 从浏览器里输入 url 到页面展示，这中间发生了什么？

### 导航

**用户输入**

1. 用户在地址栏按下回车，检查输入（关键字 or 符合 URL 规则），组装完整 URL；
2. 回车前，当前页面执行 onbeforeunload 事件；
3. 浏览器进入加载状态。

**URL 请求**

1. 浏览器进程通过 IPC 把 URL 请求发送至网络进程；
2. 查找资源缓存（有效期内）；
3. DNS 解析（查询 DNS 缓存）；
4. 进入 TCP 队列（单个域名 TCP 连接数量限制）；
5. 创建 TCP 连接（三次握手）；
6. HTTPS 建立 TLS 连接（client hello, server hello, pre-master key 生成『对话密钥』）；
7. 发送 HTTP 请求（请求行[方法、URL、协议]、请求头 Cookie 等、请求体 POST）；
8. 接受请求（响应行[协议、状态码、状态消息]、响应头、响应体等）；
   - 状态码 301 / 302，根据响应头中的 Location 重定向；
   - 状态码 200，根据响应头中的 Content-Type 决定如何响应（下载文件、加载资源、渲染 HTML）。

### 准备渲染进程

1. 根据是否同一站点（相同的协议和根域名），决定是否复用渲染进程。

**提交文档**

1. 浏览器进程接受到网路进程的响应头数据，向渲染进程发送『提交文档』消息；
2. 渲染进程收到『提交文档』消息后，与网络进程建立传输数据『管道』；
3. 传输完成后，渲染进程返回『确认提交』消息给浏览器进程；
4. 浏览器接受『确认提交』消息后，移除旧文档、更新界面、地址栏，导航历史状态等；
5. 此时标识浏览器加载状态的小圆圈，从此前 URL 网络请求时的逆时针选择，即将变成顺时针旋转（进入渲染阶段）。

### 渲染

**渲染流水线**

构建 DOM 树

1. 输入：HTML 文档；
2. 处理：HTML 解析器解析；
3. 输出：DOM 数据解构。

**样式计算**

1. 输入：CSS 文本；
2. 处理：属性值标准化，每个节点具体样式（继承、层叠）；
3. 输出：styleSheets(CSSOM)。

**布局(DOM 树中元素的计划位置)**

1. DOM & CSSOM 合并成渲染树；
2. 布局树（DOM 树中的可见元素）；
3. 布局计算。

**分层**

1. 特定节点生成专用图层，生成一棵图层树（层叠上下文、Clip，类似 PhotoShop 里的图层）；
2. 拥有层叠上下文属性（明确定位属性、透明属性、CSS 滤镜、z-index 等）的元素会创建单独图层；
3. 没有图层的 DOM 节点属于父节点图层；
4. 需要剪裁的地方也会创建图层。

**绘制指令**

1. 输入：图层树；
2. 渲染引擎对图层树中每个图层进行绘制；
3. 拆分成绘制指令，生成绘制列表，提交到合成线程；
4. 输出：绘制列表。

**分块**

1. 合成线程会将较大、较长的图层（一屏显示不完，大部分不在视口内）划分为图块（tile, 256*256, 512*512）。

**光栅化（栅格化）**

1. 在光栅化线程池中，将视口附近的图块优先生成位图（栅格化执行该操作）；
2. 快速栅格化：GPU 加速，生成位图（GPU 进程）。

**合成绘制**

1. 绘制图块命令——DrawQuad，提交给浏览器进程；
2. 浏览器进程的 viz 组件，根据 DrawQuad 命令，绘制在屏幕上。

相关概念

重排

1. 更新了元素的几何属性（如宽、高、边距）；
2. 触发重新布局，解析之后的一系列子阶段；
3. 更新完成的渲染流水线，开销最大。

重绘

1. 更新元素的绘制属性（元素的颜色、背景色、边框等）；
2. 布局阶段不会执行（无几何位置变换），直接进入绘制阶段。

合成

1. 直接进入合成阶段（例如 CSS 的 transform 动画）；
2. 直接执行合成阶段，开销最小。

## 4 如何减少重排重绘

1. 使用 class 操作样式，而不是频繁操作 style
2. 避免使用 table 布局
3. 批量 dom 操作，例如 createDocumentFragment，或者使用框架，例如 React
4. Debounce window resize 事件
5. 对 dom 属性的读写要分离
6. will-change: transform 做优化