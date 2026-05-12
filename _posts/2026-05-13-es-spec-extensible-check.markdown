---
layout: post
title: "从一次 quickjs PR 看 ES 规范：赋值时，extensible 检查发生在哪一步？"
date: 2026-05-13 09:00:00 +0800
categories: programming
tags: [javascript, ecmascript, quickjs]
---

> 最近 quickjs-ng 的一个 [PR](https://github.com/quickjs-ng/quickjs/commit/a7f0b8a4c294e15c69e67dae55b2ad2e2e3e6f76) 修复了一个属性赋值相关的 bug。排查过程中涉及到 ES 规范里 [`[[Set]]`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinary-object-internal-methods-and-internal-slots-set-p-v-receiver) 的一条关键分支——**"修改已有属性"和"创建新属性"走的是两条不同的路径**。本文借这个 PR 的由头，聊聊规范里 extensible 检查到底发生在什么时候。

---

## 一个常见的失败场景

先看一段简单的代码：

```js
"use strict";

var obj = {};
Object.preventExtensions(obj);

obj.x = 1;  // TypeError
```

对象被 `preventExtensions` 之后就不能再添加新属性了。规范是如何定义这个行为的呢？我们顺着 ES 规范的内部流程走一遍，就知道**"不可扩展"这个检查到底发生在哪一步**了。

## 规范里的执行流程

当我们写 `obj.x = 1` 时，规范里会依次进入这些"过程"（你可以把它们理解为引擎内部的函数调用链）：

```
PutValue(lref, 1)
  ↓
baseObj.[[Set]]("x", 1, baseObj)
  ↓
OrdinarySet(baseObj, "x", 1, baseObj)
  ↓
  // baseObj 上没有 "x"（ownDesc = undefined）
  ↓
OrdinarySetWithOwnDescriptor(baseObj, "x", 1, baseObj, undefined)
  ↓
  // ownDesc is undefined，查原型链
  ↓
Object.prototype.[[Set]]("x", 1, baseObj)
  ↓
OrdinarySet(Object.prototype, "x", 1, baseObj)
  ↓
OrdinarySetWithOwnDescriptor(Object.prototype, "x", 1, baseObj, undefined)
  ↓
  // Object.prototype 上也没有 "x"（ownDesc = undefined）
  // 原型链到头（parent is null）
  // ownDesc 被设为默认值 { [[Value]]: undefined, [[Writable]]: true, ... }
  ↓
  // IsDataDescriptor(ownDesc) is true
  // ownDesc.[[Writable]] is true
  // Receiver（baseObj）是 Object
  ↓
existingDescriptor = Receiver.[[GetOwnProperty]]("x")  // undefined
  ↓
  // Receiver 上无此属性 → 走"创建新属性"分支
  ↓
CreateDataProperty(baseObj, "x", 1)
  ↓
baseObj.[[DefineOwnProperty]]("x", { [[Value]]: 1 })
  ↓
OrdinaryDefineOwnProperty(baseObj, "x", desc)
  ↓
  // 先查属性是否存在 → current = undefined
  // 再查是否 extensible → false
  ↓
ValidateAndApplyPropertyDescriptor(baseObj, "x", false, desc, undefined)
  ↓
  // current is undefined，extensible is false
  // 规范：如果 !extensible 且 current 为 undefined，return false
  ↓
return false
  ↓
一路返回到 PutValue
  ↓
如果赋值语句处于严格模式 → throw TypeError
```

关键一步在 [`ValidateAndApplyPropertyDescriptor`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-validateandapplypropertydescriptor)。这个抽象操作在**创建新属性**时会检查对象是否 extensible：如果不是，且属性还不存在，就直接返回 `false`。这个 `false` 一路传回 `PutValue`，最终根据是否严格模式决定是否抛错。

**简单说：给不可扩展的对象添加新属性，规范在 ValidateAndApplyPropertyDescriptor 这一步拦住了。**

---

## 那"修改已有属性"呢？

现在来看 PR 相关的场景。它要表达的核心问题是：**如果我不是创建新属性，而是修改一个已经存在的属性，还会检查 extensible 吗？**

我们把 PR 里的测试改写成一个更常见的版本：

```js
var parent = { x: 1 };
var child = Object.create(parent);
child.x = 0;                      // child 上已有自己的属性 x
Object.preventExtensions(child);  // 让 child 不可扩展

child.x = 2;                      // 修改已有的 x，能成功吗？
console.log(child.x);             // ?
```

结果是：**能成功**，输出 `2`。

奇怪，`child` 明明不可扩展，为什么赋值还能成功？我们再走一遍规范流程，看看这次走的是哪条分支。

## 修改已有属性的规范流程

`child.x = 2` 进入规范后，同样走到 [`OrdinarySetWithOwnDescriptor`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarysetwithowndescriptor)。但这一次和前面的关键区别是——**receiver（也就是 `child`）上已经有属性 `x` 了**。

```
PutValue(lref, 2)
  ↓
child.[[Set]]("x", 2, child)
  ↓
OrdinarySet(child, "x", 2, child)
  ↓
OrdinarySetWithOwnDescriptor(child, "x", 2, child, ownDesc)
  ↓
  // IsDataDescriptor(ownDesc) is true
  // ownDesc.[[Writable]] is true
  // Receiver is Object
  ↓
existingDescriptor = Receiver.[[GetOwnProperty]]("x")
  ↓
  // existingDescriptor is NOT undefined！child 上已有 x
  ↓
  // existingDescriptor.[[Writable]] is true
  ↓
valueDesc = PropertyDescriptor { [[Value]]: 2 }
  ↓
Receiver.[[DefineOwnProperty]]("x", valueDesc)
  ↓
OrdinaryDefineOwnProperty(child, "x", valueDesc)
  ↓
ValidateAndApplyPropertyDescriptor(child, "x", false, valueDesc, current)
  ↓
  // current 不是 undefined（属性已存在），跳过 extensible 检查
  // 只检查 writable → true，允许修改
  // 更新 [[Value]] 为 2
  ↓
return true
  ↓
一路返回到 PutValue
  ↓
succeeded is true，赋值成功
```

注意这里 [`ValidateAndApplyPropertyDescriptor`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-validateandapplypropertydescriptor) 的第 3 个参数 `extensible` 仍然是 `false`（因为 `preventExtensions`），但规范在这个分支的伪代码里写的是：

> If **current is undefined**, then ... if extensible is false, return false.

也就是说，**只有当属性不存在时，才看 extensible。属性已经存在的话，extensible 直接被忽略，只检查 `writable`**。

所以 `child` 虽然不可扩展，但因为 `x` 本来就存在，而且可写，规范允许直接修改它的值。返回 `true`，赋值成功。

---

## 关键的顺序问题：为什么 quickjs 会出错

我们稍微认真看一遍 `ValidateAndApplyPropertyDescriptor`

`ValidateAndApplyPropertyDescriptor` 的 `current` 参数是由它的调用方——[`OrdinaryDefineOwnProperty`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarydefineownproperty)——通过 `O.[[GetOwnProperty]](P)` 查询后传入的：

```
OrdinaryDefineOwnProperty(O, P, Desc):
  1. Let current be ? O.[[GetOwnProperty]](P).
  2. Let extensible be ? IsExtensible(O).
  3. Return ValidateAndApplyPropertyDescriptor(O, P, extensible, Desc, current).
```

而 `ValidateAndApplyPropertyDescriptor` 收到 `current` 后，关键步骤：

```
1. Assert: extensible is true or false.
2. If current is undefined, then
   a. If extensible is false, return false.
   b. ...
   c. ...
   d. Else,
      i. Create an own data property named propertyKey ...
   e. Return true.
3. Assert: current is a fully populated Property Descriptor.
4. ...
5. ...
6. Return true.
```

注意步骤 1 和 2 的顺序：

1. **步骤 1（OrdinaryDefineOwnProperty 中）**：先查询属性是否存在，得到 `current`
2. **步骤 2（ValidateAndApplyPropertyDescriptor 中）**：只有在 `current is undefined`（属性不存在）时，才去检查 `extensible is false`
3. **步骤 3**：如果属性存在，直接断言它是个完整的属性描述符，跳过 extensible 检查

这个顺序非常关键：**先判断"有没有"，再决定"查不查 extensible"**。

而 quickjs 在这个 PR 之前的实现，顺序正好搞反了。它在入口处就先检查了 extensible，发现对象不可扩展就直接拦截，完全没去判断属性是否已经存在。于是遇到下面这种情况：

```js
var proto = { x: 1 };
var receiver = { x: 0 };
Object.preventExtensions(receiver);

receiver.x = 2;  // 规范：应该成功（已有属性可写）
```

quickjs 错误地先把 "`receiver` 不可扩展" 当成了拒绝理由，抛出了 `TypeError`（或在 `Reflect.set` 场景下返回 `false`）。规范的正确行为是：虽然 `receiver` 不可扩展，但 `x` 本来就存在且可写，赋值应当成功。

这个 PR 本质上就是**把检查的先后顺序调回到规范定义的样子**。

---

## 小结

| 场景 | 属性是否存在 | 规范走的分支 | 是否检查 extensible |
|------|-----------|-----------|------------------|
| `obj.x = 1`（obj 不可扩展，无此属性） | 不存在 | [`CreateDataProperty`](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-createdataproperty) → [`ValidateAndApplyPropertyDescriptor`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-validateandapplypropertydescriptor) | **是**，不存在时检查 |
| `child.x = 2`（child 不可扩展，但已有 x） | 存在 | [`[[DefineOwnProperty]]`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinary-object-internal-methods-and-internal-slots-defineownproperty-p-desc) → [`ValidateAndApplyPropertyDescriptor`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-validateandapplypropertydescriptor) | **否**，只检查 writable |

ES 规范对每个操作都定义了精确的流程。`extensible` 限制的只是**新增属性**，不是**修改已有属性**。更准确地说：规范在 [`ValidateAndApplyPropertyDescriptor`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-validateandapplypropertydescriptor) 中**先判断属性是否存在，只在不存在的情况下才检查 extensible**。把这个顺序搞反了，引擎就会错误地拦截合法的赋值操作。
