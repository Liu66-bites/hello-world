# 前端八股文

## 一、JavaScript

### 1. 数据类型有哪些？如何判断？
- 基本类型：`undefined`、`null`、`boolean`、`number`、`string`、`symbol`、`bigint`；引用类型：`object`（含数组、函数等）。
- `typeof`：判断基本类型，`typeof null === 'object'` 是历史遗留 bug。
- `instanceof`：沿原型链判断；`Array.isArray()` 判断数组；最通用：`Object.prototype.toString.call(x)`。

### 2. `var`、`let`、`const` 区别？
- `var`：函数作用域，存在变量提升，可重复声明，全局声明会挂到 `window` 上。
- `let` / `const`：块级作用域，存在暂时性死区，不可重复声明；`const` 声明后不能重新赋值（对象内部属性仍可改）。

### 3. 闭包是什么？
- 函数能够访问其定义时所在词法作用域中的变量，即使函数在作用域外执行。
- 用途：数据私有化、函数柯里化、防抖节流、模块模式。
- 注意：不当使用会导致变量无法释放，引起内存泄漏。

### 4. 原型和原型链？
- 每个函数有 `prototype`，每个对象有 `__proto__` 指向其构造函数的 `prototype`。
- 访问属性时沿 `__proto__` 逐级向上查找，直到 `Object.prototype`（其 `__proto__` 为 `null`）。

### 5. `this` 指向规则？
- 默认绑定：独立调用指向全局对象（严格模式下是 `undefined`）。
- 隐式绑定：`obj.fn()` 指向 `obj`。
- 显式绑定：`call` / `apply` / `bind`。
- `new` 绑定：指向新创建的对象。优先级：new > 显式 > 隐式 > 默认。
- 箭头函数没有自己的 `this`，继承外层作用域。

### 6. `new` 做了什么？
1. 创建一个空对象；2. 把对象的 `__proto__` 指向构造函数的 `prototype`；3. 以该对象为 `this` 执行构造函数；4. 若构造函数返回对象则用它，否则返回新对象。

### 7. 事件循环（Event Loop）？
- JS 单线程，同步任务在调用栈执行，异步任务回调进入任务队列。
- 每执行完一个宏任务，就清空所有微任务，再进行渲染，然后取下一个宏任务。
- 宏任务：`script`、`setTimeout`、`setInterval`、I/O、`MessageChannel`；微任务：`Promise.then`、`queueMicrotask`、`MutationObserver`。

```js
console.log(1);
setTimeout(() => console.log(2));
Promise.resolve().then(() => console.log(3));
console.log(4);
// 输出：1 4 3 2
```

### 8. Promise 与 async / await？
- Promise 三种状态：pending、fulfilled、rejected，状态一旦改变不可逆。
- 静态方法：`all`（全部成功）、`allSettled`（全部完成）、`race`（第一个完成）、`any`（第一个成功）。
- `async/await` 是 Promise + 生成器的语法糖，`await` 后面的代码相当于放进 `then` 中（微任务）。

### 9. 深拷贝怎么实现？
- `structuredClone(obj)`（现代浏览器原生支持，可处理循环引用）。
- `JSON.parse(JSON.stringify(obj))`：会丢失函数、`undefined`、Symbol，不能处理循环引用和 Date。
- 手写：递归 + `WeakMap` 记录已拷贝对象处理循环引用。

### 10. 防抖和节流？
- 防抖：事件触发后 n 秒内不再触发才执行（搜索框输入）。
- 节流：n 秒内只执行一次（滚动、resize）。

```js
function debounce(fn, delay) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

function throttle(fn, interval) {
  let last = 0;
  return function (...args) {
    const now = Date.now();
    if (now - last >= interval) {
      last = now;
      fn.apply(this, args);
    }
  };
}
```

### 11. `==` 和 `===`？
- `===` 严格相等，不做类型转换；`==` 会做隐式转换（`null == undefined` 为 true，`NaN` 不等于任何值）。推荐使用 `===`。

### 12. ES Module 和 CommonJS 区别？
- CommonJS：`require` / `module.exports`，运行时同步加载，输出值的拷贝，Node.js 传统使用。
- ESM：`import` / `export`，编译时静态分析（支持 Tree Shaking），输出值的引用（live binding），浏览器和现代 Node 原生支持。

## 二、浏览器

### 1. 浏览器渲染流程？
- 解析 HTML 生成 DOM 树 → 解析 CSS 生成 CSSOM → 合并成渲染树 → 布局（Layout）→ 绘制（Paint）→ 合成（Composite）。
- JS 会阻塞 HTML 解析（可用 `defer` / `async`）；CSS 不阻塞 DOM 解析但会阻塞渲染和后续 JS 执行。

### 2. 重排和重绘？
- 重排（回流）：几何属性变化（宽高、位置、增删 DOM），需要重新布局，开销大。
- 重绘：外观变化（颜色、背景），不影响布局。重排一定触发重绘。
- 优化：批量修改样式、使用 `transform` / `opacity` 做动画（只触发合成）、避免频繁读取 `offsetWidth` 等布局属性。

### 3. `defer` 和 `async`？
- 都异步下载脚本。`async` 下载完立即执行，不保证顺序；`defer` 等 HTML 解析完后、`DOMContentLoaded` 前按顺序执行。

### 4. 浏览器缓存？
- 强缓存：`Cache-Control: max-age`（优先）、`Expires`，命中时不发请求（200 from cache）。
- 协商缓存：`Last-Modified` / `If-Modified-Since`、`ETag` / `If-None-Match`（优先），命中返回 304。

### 5. 本地存储对比？
- Cookie：4KB，每次请求自动携带，可设过期时间、`HttpOnly`、`Secure`、`SameSite`。
- localStorage：约 5MB，永久保存，同源共享。
- sessionStorage：约 5MB，仅当前标签页会话有效。
- IndexedDB：大容量、异步、支持事务的数据库。

### 6. XSS 和 CSRF？
- XSS（跨站脚本）：注入恶意脚本 → 对输出转义、CSP 内容安全策略、Cookie 设置 `HttpOnly`。
- CSRF（跨站请求伪造）：利用用户登录态伪造请求 → CSRF Token、`SameSite` Cookie、校验 `Origin` / `Referer`。

### 7. 事件冒泡、捕获与事件委托？
- 事件流：捕获阶段 → 目标阶段 → 冒泡阶段。`addEventListener` 第三个参数 `true` 表示在捕获阶段触发。
- 事件委托：把事件绑定在父元素上，利用冒泡通过 `e.target` 处理子元素事件，减少监听器数量，支持动态元素。

## 三、CSS

### 1. 盒模型？
- 标准盒模型：`width` = content；IE 盒模型（`box-sizing: border-box`）：`width` = content + padding + border。

### 2. BFC 是什么？
- 块级格式化上下文，一个独立的渲染区域。触发：`overflow` 非 visible、`display: flow-root / flex / inline-block`、`float`、`position: absolute / fixed`。
- 作用：清除浮动、防止外边距合并、防止元素被浮动元素覆盖。

### 3. 水平垂直居中的方法？
- Flex：`display: flex; justify-content: center; align-items: center;`
- Grid：`display: grid; place-items: center;`
- 绝对定位：`position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);`

### 4. 选择器优先级？
- `!important` > 内联样式 > ID > 类 / 属性 / 伪类 > 元素 / 伪元素 > 通配符。

## 四、Vue

### 1. Vue 2 和 Vue 3 响应式原理区别？
- Vue 2：`Object.defineProperty` 劫持 getter / setter，无法监听新增 / 删除属性和数组下标变化（需要 `$set`）。
- Vue 3：`Proxy` 代理整个对象，可以监听新增、删除、数组变化，按需递归，性能更好。
- 依赖收集：读取时 `track` 收集副作用，修改时 `trigger` 触发更新。

### 2. Composition API 相比 Options API 的优势？
- 逻辑按功能组织而不是按选项拆分，便于复用（composables 取代 mixin），TypeScript 支持更好。

### 3. `computed` 和 `watch` 区别？
- `computed`：有缓存，依赖不变不会重新计算，用于派生数据。
- `watch`：监听数据变化执行副作用（异步请求等），无缓存。

### 4. Vue 生命周期（Vue 3）？
- `setup` → `onBeforeMount` → `onMounted` → `onBeforeUpdate` → `onUpdated` → `onBeforeUnmount` → `onUnmounted`。

### 5. `v-if` 和 `v-show`？
- `v-if` 真正销毁 / 创建元素，切换开销大；`v-show` 只切换 `display`，初始渲染开销大。频繁切换用 `v-show`。

### 6. `key` 的作用？
- 帮助 diff 算法识别节点，实现高效复用。不要用 index 作为 key（列表增删时会导致错误复用和状态错乱）。

### 7. `nextTick` 原理？
- Vue 异步批量更新 DOM，`nextTick` 把回调放在 DOM 更新之后执行，优先使用 Promise 微任务实现。

## 五、React

### 1. 虚拟 DOM 和 diff 算法？
- 虚拟 DOM 是描述真实 DOM 的 JS 对象，通过对比新旧虚拟 DOM 最小化真实 DOM 操作。
- diff 策略：只做同层比较；类型不同直接替换；列表通过 `key` 识别节点。复杂度从 O(n³) 降到 O(n)。

### 2. Fiber 架构？
- 把渲染工作拆成可中断的小单元（Fiber 节点链表），利用时间切片避免长任务阻塞主线程，支持优先级调度（并发渲染）。

### 3. 常用 Hooks？
- `useState`、`useEffect`（副作用，依赖数组控制执行时机，返回清理函数）、`useRef`、`useMemo`（缓存值）、`useCallback`（缓存函数）、`useContext`、`useReducer`。
- 规则：只在函数组件顶层调用，不能在条件、循环中调用（Hooks 依赖调用顺序存储在链表中）。
- React 19 新增 `use`、`useActionState`、`useOptimistic`；React Compiler 可自动做记忆化优化。

### 4. `setState` 是同步还是异步？
- React 18 起所有更新默认自动批处理，表现为"异步"；需要立即同步更新可用 `flushSync`。

### 5. 受控组件和非受控组件？
- 受控：表单值由 state 管理，通过 `onChange` 更新。
- 非受控：表单值由 DOM 自身管理，通过 `ref` 读取。

### 6. React 性能优化手段？
- `React.memo`、`useMemo`、`useCallback` 避免不必要渲染；列表虚拟滚动；`React.lazy` + `Suspense` 代码分割；合理拆分组件与状态下沉。

## 六、工程化

### 1. Webpack 和 Vite 区别？
- Webpack：先打包整个应用再启动开发服务器，项目大时启动慢；生态成熟、配置灵活。
- Vite：开发环境基于浏览器原生 ESM 按需编译，启动极快；生产环境使用 Rollup（新版本迁移到 Rolldown）打包。

### 2. Loader 和 Plugin 区别？
- Loader：文件转换器（如把 TS / SCSS 转为 JS / CSS）。
- Plugin：在构建生命周期的钩子中扩展功能（如生成 HTML、压缩代码、提取 CSS）。

### 3. 前端性能优化有哪些？
- 网络：CDN、HTTP/2、Gzip / Brotli 压缩、缓存策略、资源预加载（preload / prefetch）。
- 资源：代码分割、Tree Shaking、图片压缩与懒加载、使用 WebP / AVIF。
- 渲染：减少重排重绘、虚拟列表、防抖节流、Web Worker 处理耗时计算。
- 指标：LCP、INP、CLS（Core Web Vitals）。
