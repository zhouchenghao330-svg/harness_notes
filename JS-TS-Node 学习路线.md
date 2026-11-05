# JS / TS / Node 学习路线

目标：不看视频，快速上手写代码

## 教材

**[javascript.info](https://javascript.info/)**  — 最好的 JS 入门文档，有中文版

## 只看这些（Part 1 精选）

### Day 1：基础
1. [An Introduction](https://javascript.info/intro) — 10 分钟扫完
2. [JavaScript Fundamentals](https://javascript.info/first-steps) — 只看 Variables、Data types、Functions、Arrow functions 四篇
3. [Objects: the basics](https://javascript.info/object) — 对象创建、属性访问、this 关键字

### Day 2：核心语法
4. [Data types](https://javascript.info/data-types) — 重点：Array methods（map/filter/reduce）、JSON、Destructuring
5. [Advanced working with functions](https://javascript.info/advanced-functions) — 闭包、箭头函数深入、rest/spread
6. [Modules](https://javascript.info/modules) — import / export

### Day 3：重中之重
7. **[Promises, async/await](https://javascript.info/async)** ⭐⭐⭐ — JS 和 Python 最大的差异点，看懂这章就懂了 JS 的灵魂

### 可以跳过
- Code quality
- Object properties configuration
- Prototypes, inheritance（了解有这个东西就行）
- Generators（跟 Python yield 类似）
- Error handling（try/catch 语法 10 分钟看完）

## TypeScript

看完 JS 基础后，看官方文档就够了：
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- 只看：Everyday Types、More on Functions、Object Types、Generics

## Node.js

不需要系统性学习，边用边查：
- 内置模块：fs（文件读写）、path（路径）、process（环境变量）
- npm：npm init、npm install、npx
- 框架：写 Web 用 Express/Fastify（≈ Flask），CLI 用 Commander（≈ Click）

## 后续方向（JS 学完后选一个）

学完 JS 语法后选一个方向：

1. **React（前端）** — 写网页交互，搭配 Vite 开发
2. **Express/Fastify（后端）** — 写 API 服务，≈ Flask
3. **Next.js（全栈）** — 一个项目搞定前后端，推荐 Python 背景的人入门
4. **CLI 工具** — 用 Node.js 内置模块，搭配 commander/zx

建议产出物：一个 LeetCode 刷题 CLI 工具（lc list / lc open / lc submit），把文件系统、子进程、HTTP 全过一遍。

JS 语法速通后回来继续讨论方向选择。

## 实战建议

最快的学法是：**用新语言重写你已经写过的东西**
- 把 LeetCode 100 里的题用 JS/TS 翻译一遍
- 把你 Python 项目里一个功能用 TS 重写
- 不要在教程里做练习题，动手写真实项目

---

## 笔记：Data types（数据类型）

来源：[zh.javascript.info/types](https://zh.javascript.info/types)

**8 种类型：7 原始 + 1 引用**

### number — 没有 int/float 之分
```javascript
let x = 42          // 整数
let y = 3.14        // 浮点数
let z = 1 / 0       // Infinity（不报错）
let w = "a" / 2     // NaN（不报错）
```
- Python 除零是 `ZeroDivisionError`，JS 给 `Infinity`
- 非法运算给 `NaN`，且 NaN 是粘性的（含 NaN 的运算结果都是 NaN）

### string — 三种引号
```javascript
"hello"       // 双引号
'hello'       // 单引号
`hello ${x}`  // 反引号 ≈ Python f-string
```
- 没有 char 类型，字符就是长度为 1 的字符串
- 模板字符串（反引号）最实用

### null vs undefined
```javascript
let a = null       // 主动赋值为"空"
let b              // undefined，声明但未赋值
```
- `null` = 你主动说没值，`undefined` = JS 默认忘赋值
- **写法上记住主动用 `null`，别用 `undefined`**
- Python 只有 `None`，JS 分两个

### typeof
```javascript
typeof null        // "object" ← 历史遗留 bug
typeof []          // "object" ← 数组也是 object
```

### 与 Python 对比

| 概念 | Python | JavaScript |
|------|--------|------------|
| 动态类型 | ✅ | ✅ |
| int/float | 分开 | 统一 number |
| 字符串插值 | f"{x}" | `` `${x}` `` |
| 空值 | None | null（主动）/ undefined（默认）|
| 除零 | 报错 | Infinity |
| 类型检查 | type(x) | typeof x |

---

## 笔记：Variables（变量）

来源：[zh.javascript.info/variables](https://zh.javascript.info/variables)

**let vs const vs var**

```javascript
let x = 1          // 可变变量，≈ Python 普通赋值
const y = 2        // 常量，不可重新赋值，≈ Python typing.Final
var z = 3          // 老式声明，别用
```

**const 不是值不可变，是引用不可变**
```javascript
const obj = { name: "foo" }
obj.name = "bar"   // ✅ 可以！属性可以改
obj = {}            // ❌ 报错，不能重新赋值
```

**命名规则：** 字母、数字、`$`、`_`，首字符不能是数字。没有 Python 的 `_` 特殊意义。

**保留字：** `let`、`class`、`return`、`function` 不能做变量名。

**命名习惯：** 驼峰命名 `myVariableName`（JS），不是蛇形 `my_variable_name`（Python）。

---

## 笔记：函数（Functions）

来源：[zh.javascript.info/function-basics](https://zh.javascript.info/function-basics)

### 函数声明

```javascript
function name(param1, param2) {
  // 函数体
  return 值;
}
```

- 无 `return` 或空 `return` → 返回 `undefined`（≈ Python 的 `None`）
- 参数缺省时**不报错**，值为 `undefined`（Python 是必填参数缺一就 TypeError）

### 默认值

```javascript
function showCount(count = "unknown") {
  console.log(count);
}
showCount();    // "unknown"
showCount(0);   // 0  （0 不是 undefined，所以传入 0）
showCount(null); // null （null 也不是 undefined，但 ?? 会拦截它）
```

后备写法学 `??` 操作符：
```javascript
count ?? "unknown"   // 只有 undefined/null 时用默认值
count || "unknown"   // 所有 falsy（0/""/false/null/undefined）都用默认值
```

### ⚠️ 核心坑：return 不能换行

```javascript
return
  x + y;   // ❌ 返回 undefined

// ✅ 正确
return (
  x + y
);
```

JS 自动在行末加分号（ASI），`return` 后面直接跟分号就结束了。

### ⚠️ 修改外部变量 vs 副作用

```javascript
let userName = 'John';
function showMessage() {
  userName = "Bob"; // 修改了外部变量（副作用）
}
```

函数内修改外部变量是**副作用**，和 Python 一样要小心。

### 命名约定

| 前缀 | 含义 | Python 对应 |
|------|------|-------------|
| `show…` | 显示 | `display_` |
| `get…` | 返回值 | `get_` |
| `calc…` | 计算 | `calc_` |
| `create…` | 创建 | `create_` |
| `check…` | 检查布尔 | `is_` / `has_` |

### 与 Python 对比

| 概念 | Python | JavaScript |
|------|--------|------------|
| 声明 | `def name():` | `function name()` |
| 缺参数 | 报 `TypeError` | 值为 `undefined`，不报错 |
| 返回值 | 无 return → `None` | 无 return → `undefined` |
| 三元 | `a if cond else b` | `cond ? a : b` |
| 函数作用域 | 闭包 | 闭包（更灵活） |
| `return` 换行 | 正常 | ❌ 不行（ASI 插分号） |
| 参数传递 | 值引用（可变对象可改） | 相同

---

## 笔记：箭头函数（Arrow Functions）

来源：[zh.javascript.info/arrow-functions-basics](https://zh.javascript.info/arrow-functions-basics)

### 三种形式

```javascript
// 单参数 → 可省略括号，自动 return
const double = n => n * 2;

// 多参数 → 括号不可省，自动 return
const sum = (a, b) => a + b;

// 无参数 → 括号必须保留，自动 return
const greet = () => console.log("Hi");
```

### 多行箭头函数

右边花括号 `{}` → **不自动 return**，要手动写 `return`：

```javascript
const sum = (a, b) => {
  let result = a + b;
  return result;  // 必须写
};
```

### 与 Python lambda 对比

| 概念 | Python lambda | JS 箭头函数 |
|------|---------------|-------------|
| 语法 | `lambda x: x*2` | `x => x * 2` |
| 单参数括号 | 必须 | 可省略 |
| 多行 | ❌ 不行 | ✅ `{ ... return }` |

### 总结

`=>` 右边是**表达式 → 自动 return**，是**花括号 → 手动 return**。

---

## 笔记：对象（Object）

来源：[zh.javascript.info/object](https://zh.javascript.info/object)

### 创建

```javascript
let user = {};              // 字面量（常用）
let user = new Object();    // 构造函数
```

### 属性操作

```javascript
let user = {
  name: "John",
  "likes birds": true,   // 多词键要加引号
  age: 30,                // 尾随逗号 ✅
};

// 访问
user.name             // 点号 — 简单键名
user["likes birds"]   // 方括号 — 多词键/变量/表达式

// 增删改
user.isAdmin = true;
delete user.age;
```

### 不存在的键 → undefined（不是报错！）

```javascript
console.log(user.xxx);       // undefined
console.log("xxx" in user);  // false — 用 in 确认
```

跟 Python 的 `KeyError` 不同，JS 默认不报错。

### 属性值简写

```javascript
let name = "John";
let age = 30;

let user = { name, age };
// 等价 { name: "John", age: 30 }
```

### 计算属性（可放表达式）

```javascript
let fruit = "apple";
let bag = {
  [fruit]: 5,                   // "apple": 5
  [fruit + "s"]: 10,            // "apples": 10
  [fruit.toUpperCase()]: 3,     // "APPLE": 3
  [1 + 2]: "three",             // "3": "three"
};
```

不加 `[]` 就是字面字符串键名，加 `[]` 就是表达式求值。

### 遍历：for..in

```javascript
for (let key in user) {
  console.log(key);       // 键名
  console.log(user[key]); // 值
}
```

**坑：整数键自动按数字升序排**，不想被排加 `+` 前缀。

### 存在性检查

```javascript
"age" in user   // true / false
```

比 `user.age !== undefined` 更可靠（属性值可能恰好是 undefined）。

### 与 Python dict 对比

| 概念 | Python dict | JS Object |
|------|-------------|-----------|
| 简单键名引号 | 必须 | 不用 |
| 访问方式 | `[]` 只有一种 | 点号 + `[]` 两种 |
| 不存在的键 | `KeyError` | `undefined`（不报错） |
| 遍历 | `for k in dict:` | `for (let k in obj)` |
| 存在检查 | `k in dict` | `"k" in obj` |
| 属性值简写 | 无 | `{ name }` |
| 计算键名 | `{k: v}` | `[expr]: v` |

---

## 笔记：数组方法（Array Methods）

来源：[zh.javascript.info/array-methods](https://zh.javascript.info/array-methods)

### 增删改

```javascript
arr.push("end");        // 尾插 ≈ list.append()
arr.pop();              // 尾删 ≈ list.pop()
arr.shift();            // 首删（慢）
arr.unshift("start");   // 首插（慢）
arr.splice(1, 2);       // 删：从索引1删2个
arr.splice(1, 0, "a");  // 插：从索引1插，不删
```

### ⭐ 三剑客（核心）

**map — 映射**：每个元素过一遍，返回等长新数组
```javascript
[1, 2, 3].map(n => n * 2)  // [2, 4, 6]
```
≈ Python `[n * 2 for n in [1, 2, 3]]`

**filter — 过滤**：保留符合条件的，返回新数组
```javascript
[1, -2, 3].filter(n => n > 0)  // [1, 3]
```
≈ Python `[n for n in [1, -2, 3] if n > 0]`

**reduce — 累积**：遍历数组，每次带上一步结果，最终返回**一个值**

```javascript
[1, 2, 3].reduce((acc, n) => acc + n, 0)
// 过程：acc=0 + n=1 → acc=1 + n=2 → acc=3 + n=3 → 返回 6
```
- `acc` = 累积器（上一轮的结果）
- `n` = 当前元素
- 最后那个 `0` = 初始值

| 步骤 | acc | n | 结果 |
|------|-----|---|------|
| 1 | 0（初始） | 1 | 1 |
| 2 | 1 | 2 | 3 |
| 3 | 3 | 3 | 6 ✅ |

### 查找

```javascript
items.find(item => item.id === targetId)   // 第一个匹配的元素
items.findIndex(item => item.id > 5)       // 第一个匹配的索引
items.some(item => item.active)            // 任一满足 → true/false
items.every(item => item.active)           // 全部满足 → true/false
items.includes(42)                         // 包含某值 → true/false
```

### 排序 & 转换

```javascript
arr.sort((a, b) => a - b);   // 数字升序（默认转字符串排，大坑！）
arr.reverse();                // 反转（改原数组）
arr.slice().sort()           // 先复制再排序，不改原数组

"hello-world".split("-")     // → ["hello", "world"]
["hello", "world"].join(" ") // → "hello world"
```

### 与 Python 对照

| 操作 | JavaScript | Python |
|------|-----------|--------|
| 映射 | `map(x => x*2)` | `[x*2 for x in lst]` |
| 过滤 | `filter(x => x > 2)` | `[x for x in lst if x > 2]` |
| 累加 | `reduce((a,b) => a+b, 0)` | `sum(lst)` |
| 遍历 | `forEach(x => ...)` | `for x in lst:` |
| 全部满足 | `every(x => x > 0)` | `all(x > 0 for x in lst)` |
| 任一满足 | `some(x => x > 0)` | `any(x > 0 for x in lst)` |
| 找第一个 | `find(x => x > 2)` | `next(x for x in lst if x > 2)` |
| 字符串拆分 | `"a-b".split("-")` | `"a-b".split("-")` |
| 数组合并 | `["a","b"].join("-")` | `"-".join(["a","b"])` |

### 常用模式

```javascript
// 提取字段
users.map(u => u.name)

// 数组 → 对象（按 id 索引）
users.reduce((obj, u) => { obj[u.id] = u; return obj; }, {})

// 去重
[...new Set(arr)]

// 扁平化
arr.flat(Infinity)

---

## 笔记：解构赋值（Destructuring）

来源：[zh.javascript.info/destructuring-assignment](https://zh.javascript.info/destructuring-assignment)

### 数组解构

```javascript
let [firstName, lastName] = ["John", "Smith"];
// firstName = "John", lastName = "Smith"

// 跳过
let [a, , c] = [1, 2, 3];  // a=1, c=3

// 交换变量
[a, b] = [b, a];

// 剩余
let [x, ...rest] = [1, 2, 3, 4];  // x=1, rest=[2,3,4]
```

≈ Python `a, b = arr`

### 对象解构（更常用）

```javascript
let user = { name: "John", age: 30 };

let { name, age } = user;
// name = "John", age = 30

// 重命名：取出 user.name 放进变量 userName
let { name: userName, age: userAge } = user;
// userName = "John", userAge = 30（不改原对象）

// 默认值
let { name, job = "unemployed" } = user;
// job = "unemployed"（user 里没有 job）
```

### ⭐ 智能函数参数（JS 代替关键字参数）

JS **没有** Python 的关键字参数（`f(a=1, b=2)`），所以用**对象解构**来模拟：

```javascript
// 不好：记顺序 + 塞 undefined
function showMenu(title, width, height, items) {}
showMenu("Menu", undefined, undefined, ["a"]);

// 好：传对象，不记顺序
function showMenu({ title = "Untitled", width = 200, height = 100, items = [] } = {}) {
  console.log(title, width, height, items);
}
showMenu({ items: ["a", "b"] });        // ✅ 只传 items
showMenu({ title: "Menu", height: 300 }); // ✅ 不记顺序
showMenu({});                            // ✅ 全默认
showMenu();                              // ✅ 全默认（= {} 兜底）
```

### 遍历键值对

```javascript
for (let [key, value] of Object.entries(user)) {
  console.log(key, value);
}
```
≈ Python `for k, v in user.items():`
