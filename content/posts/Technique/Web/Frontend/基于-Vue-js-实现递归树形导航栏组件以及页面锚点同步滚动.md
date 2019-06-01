---
title: 基于 Vue.js 实现递归树形导航栏组件以及页面锚点同步滚动
date: 2018-12-22 16:36:45
tags:
  - vue
---

这篇博客的标题不是很直观 —— 事实上就是实现了类似百度百科（*以及本博客*）的导航栏效果 。可以这样描述核心功能：

- 首先，将正文内容按 H1 ~ H6 解析成递归的树形结构数据并映射到导航栏
- 其次，在用户点击导航栏时将正文平滑滚动到相应锚点
- 最后，当正文内容滚动时同步更新导航栏（高亮活动元素）

项目的全部源代码与效果图都在 [GitHub](https://github.com/queensferryme/treeview) 上，欢迎参观 。

<!--more-->

## 问题分析

初步分析发现存在一下技术上的问题需要解决：

- Vue 组件递归渲染：即一个组件的 template 中可以包含它自身 。由于树形导航栏的父节点与子节点在结构上是基本相似的，所以我使用递归渲染来创建导航栏；
- 将 DOM 中的 H1 ～ H6 元素依照原有的顺序解析成嵌套的树形结构数据 。这些数据会被直接反馈给导航栏组件以进行渲染；

这两个问题是初期稍加思索就能想到的主要难点 。在此次小项目的开发过程中还有许多零散的小问题，会在后文结合语境讲述。

## 递归渲染导航栏

分析问题后就开始着手写代码/doge 。首先规定具有如下递归结构的数据：

```json
nodes: [
  {
    $value: 'H1'，
    $children: [
      {
      	$value: 'H2',
        $children: [
          { ... }
        ]
      }
    ]
  }
]
```

每个节点都用一个对象来表示 。该对象又有两个属性：`$value` 与 `$children`；其中 `$value` 即该节点的值，`$children` 是一个数组，包含了其子节点 。这样我们就创建了一个递归的数据结构 —— 判断递归终止的条件也很简单，只要发现 `$children` 为空数组就可以停止递归了 。

有了数据以后，就可以开始设计组件了 。我的导航栏组件 `TreeView` 是这样设计的：

```html
<template>
  <div class="nodes">
    <div
      class="node"
      v-for="(node, index) in nodes"
      :key="index"
    >
      <div class="node-head">
        {{ node.$value }}
      </div>
      <tree-view
        class="node-content"
        :nodes="node.$children"
      >
      </tree-view>
    </div>
  </div>
</template>
```

后续为了实现活动元素高亮还有一些小改动，不过基本结构就是如此了 。我只需要一行 `<tree-view :nodes="nodes"></tree-view>` 就能通过递归树形数据创建一个完整的导航栏 。

当然，代码写到这我发现：实现活动元素高亮也有不小的麻烦 。为此我给每个节点分配了一个 `$path` 属性，该属性同样是一个数组，这个数组包含了一连串的下标 —— 每个下标都代表了该节点在其兄弟节点中的排行（rank）。例如，上述的 `H2` 节点的 `$path` 属性就是 `[0, 0, 0]`；其中第一个零代表了虚拟的根节点 。这样我就可以通过 `$path` 属性来对没一个节点唯一标识 。

除了 `nodes` 以外，我又为组建增加了一个 `active` 属性 。当且仅当某个节点的 `$path` 属性与 `active` 属性相同时其为当前活动元素 —— 这时我们就可以将该节点高亮显示了 。

最后，理所当然的 —— 导航栏至少要为用户提供点击跳转的功能 。这个功能其实相当简单 —— 只要在点击该节点时将 `active` 属性更新为该节点的 `$path` 属性即可；至于其他的页面滚动跳转就交由其他模块负责就行啦 。不过依然有一个坑点：**直接修改组件内部的属性是绝不可取的！**因为即使你修改了组件内部的属性，外部的数据值也不会同步更新 。所以我使用了 `.sync` 修饰符，当某个节点被点击事就触发 `update:active` 事件来做到组件内外的数据同步 。修改后的主要代码如下：

```html
<div
  class="node-head"
  :class="active.toString() === node.$path.toString() ? 'active' : ''"
  @click="$emit('update:active', node.$path)"
>
  {{ node.$value }}
</div>
<tree-view
  class="node-content"
  :active="acitve"
  :nodes="node.$children"
  @update:active="$emit('update:active', $event)"
>
</tree-view>
```

注意子节点同样属于组件内部，同样不能使用 `.sync` 来直接修改 `active` 属性 —— 而应该监听 `update:active` 属性并将事件继续冒泡传递给父级组件 。

## 由 DOM 解析 TOC 树

关于这部分，我想到了一种激动人心的解法 —— 只需要一次 map/reduce 就能解析出所需的 TOC 树 。这里我假设你已经明白了 map/reduce 函数的基本用法 。下面先贴出我的解法：

```javascript
Array.from(this.$refs['user-content']
     .querySelectorAll('h1, h2, h3, h4, h5, h6'))
     .map((item) => {
       item.$value = item.innerText;
       item.$children = [];
       return item;
     }).reduce((result, current, index, array) => {
       let parent = (() => {
         for (let i = index - 1; i >= 0; i--)
           if (array[i].tagName < current.tagName)
             return array[i];
           return current;
       })();
       if (current !== parent) {
         parent.$children.push(current);
         current.$path = parent.$path.concat(parent.$children.length - 1);
       }
       else {
         result.push(current);
         current.$path = [0].concat(result.length - 1);
       }
       return result;
     }, []);
```

记录一下几个比较难理解的点：

- 由于 `querySelectorAll` 方法返回的并非真正的数组对象，所以它没有 `map` 方法 。`Array.from` 方法能将一些类数组对象转化为真正的数组 。注意这个方法是 ES2015 中提出的 —— 这意味着你需要注意浏览器兼容性问题；

- 我的算法将该节点前的离它最近且比它级别更高的节点作为其父节点；如果这样的节点不存在，就以它自身为父节点 —— 在 reduce 过程中我就是这样初始化 `parent` 变量的 。有趣的是，这样的结构看起来很像一个并查集（Union），尤其是以自身为父节点的这个处理让后面的一些操作变得更简单直观了；

- 我利用了 javascript 语法中对象按引用传递的小特性 —— 这真的很有趣 。不妨先看一下下面的两个小例子：

  ```javascript
  var a = b = {};
  b.value = 'Hello World';
  console.log(a); // {value: "test"}
  ```

  ```javascript
  var a = [], b = [], c = [];
  a.push(b); b.push(c); c.push(1);
  console.log(a); // [[[1]]]
  ```

  利用这一特性，我们只需要将子节点 push 进父节点的 `$children` 数组 ，最后再将那些一级节点（父节点即为自身）按数组返回就能得到嵌套递归的树形结构数据了 。

于是乎，就成功的将 DOM 解析为 TOC 树了。

## 一些小问题

### 取消计算属性缓存

众所周知（x），Vue 的计算属性是有缓存功能的 。如果你想取消计算属性缓存，你可以这样写： 

```javascript
computed: {
    data: {
        cache: false,
        get () { ... },
        /* set () { ... } */
    }
}
```

这样 `data` 属性就不会被缓存 。

### 延迟更新 active

我们期望当页面滚动时，User-Content 组件能监听滚动事件并实时更新当前的 active 节点；我们也期望当用户点击导航栏时，页面平滑地滚动到对应锚点 —— 这就造成了一种矛盾：一旦用户点击导航栏，页面就开始滚动；一旦页面开始滚动，它就会触发滚动事件导致 active 属性被重置 。也就是说，页面还没来得及滚动到目标锚点就会被滚动事件的监听函数”拉回去“ 。

所以我采用了一点小技巧：延迟更行 active 属性 。当页面滚动停止时 active 属性才会更新 。这就能保证当用户点击导航栏时，页面平滑地滚动到对应锚点 。主要代码如下：

```javascript
this.$refs['user-content'].addEventListener('scroll', () => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
        this.$emit('update:active', this.current.$path);
    }, 100);
```

### v-html 与 Scoped CSS

一个平时不太容易注意到的点是 scoped CSS 中的样式无法被渲染到由 v-html 加载出的元素上；这是因为 vue-loader 在编译 Vue 组件时不会给 v-html 中的元素添加 `data-v-<hash>` 属性 。我在使用 marked 渲染 markdown 的时候发现了这个问题 。

对于新版本的 vue-loader，这个问题可以通过深度选择器（Deep Selectors）来给 v-html 中的元素添加样式 。例如：

```CSS
#user-content >>> pre { ... }
```

实际上它会被 vue-loader 解析为：

```css
#user-content[data-v-<hash>] pre { ... }
```

------

**参考文献：**

- [掘金：用 Vue.js 递归组件构建一个可折叠的树形菜单](https://juejin.im/entry/5a45e8a96fb9a04511717324)
- [Medium：Scoped styles with v-html](https://medium.com/@brockreece/scoped-styles-with-v-html-c0f6d2dc5d8e)