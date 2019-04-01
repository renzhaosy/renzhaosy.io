---
title: "[译]Tasks, microtasks, queues and schedules"
date: 2019-03-29 20:12:14
cover: https://image.cdn.renzhaosy.cn/random-img/1.jpg
tags:
  - Javascript
  - Tasks
runenable: true
---

# [译]Tasks, microtasks, queues and schedules

这一篇是对 Jake Archibald 大大的  [Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) 这篇文章部分的汉译。哈哈，第一次尝试对外文的翻译，渣渣英语。。。

下边开始正文。


实际上，如果你更新喜欢看视频，[Philip Roberts](https://twitter.com/philip_roberts) 在 JSConf 中就 event loop 进行了一次很棒的演讲。虽然没有包含微任务（microtasks），但是很好的介绍了其他部分。无论如何，言归正传。。。

先来一小段 Javascript:

```js
console.log("script start");

setTimeout(function() {
  console.log("setTimeout");
}, 0);

Promise.resolve()
  .then(function() {
    console.log("promise1");
  })
  .then(function() {
    console.log("promise2");
  });

console.log("script end");
```

log 日志会按照什么顺序出现？

### Try It

正确答案：`script start`, `script end`, `promise1`, `promise2`, `setTimeout`。但是各个浏览器的支持却显得很狂野。

Microsoft Edge、Firefox 40、iOS Safari 和桌面 Safari 8.0.8 中 `setTimeout` 会在 `promise1` 和 `promise2` 之前打印出来。这看起来很奇怪，因为 Firefox 39 and Safari 8.0.7 中打印结果是对的。

### 为什么会这样

要理解这一点，你需要理解事件循环（event loop）是怎样处理宏任务和微任务的。如果你是第一次听说，这可能会让你大吃一惊。深呼吸。。。

每一个线程都有自己的事件循环，所以每一个 web worker 也会有自己的事件循环，所以它可以独立执行。然而同一个 origin 上的所有窗口，因为它们相互之间可以同步通信，所以共享一个事件循环。事件循环一直运行，排队式地执行每一个宏任务。事件循环具有多个任务源，每个任务源内部确保了自己内部的执行顺序（比如：IndexedDB 规范定义了自己的执行顺序），而浏览器则决定在循环中的每一圈选择从哪个来源获取任务。

`宏任务`被安排好，这样浏览器可以从内部进入 JavaScript / DOM 领域，并确保这些操作按顺序发生。在任务之间，浏览器可以呈现更新。从鼠标点击到回调事件需要调度一个任务，比如解析 HTML,再比如上边的例子 `setTimeout`

`setTimeout` 等待一个给定的延迟时间，然后为他的回调（callback）安排一个新的宏任务。这就是`setTimeout` 在`script end`之后被打印的原因。因为打印`script end` 属于第一个宏任务的一部分，并且`setTimeout` 在单独的一个宏任务中被打印。

`微任务（Microtasks）` 通常被安排来做`当前执行脚本之后发生的事情`，比如：对一些动作（actions）做出反应,或者对某些事件异步化而不对完整的新的宏任务造成影响。微任务队列会在每个宏任务结束时处理，并且此时没有其他的 JavaScript 在执行。在微任务期间产生的任何其他微任务都将添加到队列的末尾，并进行处理。微任务包括：观察者模式的 callbacks、promise callbacks等。

一个 Promise 一旦完成，或者已经完成，它将为他的回调（callbacks）安排一个微任务。这个 Promise 完成之后的回调是异步的。所以执行`.then(yey, nay)` 会为完成的 Promise 在微任务队列中立即安排一个微任务。当前运行的 script 一定在微任务被处理之前结束，这就是`promise1`和`promise2`会在`script end`之后被打印的原因.而由于微任务处理永远发生在下一个宏任务之前，所以`promise1`和`promise2`在`setTimeout`之前被打印。

所以，一步一步：

```js
console.log("script start");

setTimeout(function() {
  console.log("setTimeout");
}, 0);

Promise.resolve()
  .then(function() {
    console.log("promise1");
  })
  .then(function() {
    console.log("promise2");
  });
console.log("script end");
```

### 不同浏览器有什么不同

有些浏览器打印结果是这样的：`script start`, `script end`, `setTimeout`, `promise1`, `promise2`。他们在`setTimeout`之后执行 promise 回调。这看起来像他们把执行 promise 回调当做新的宏任务的一部分而不是微任务。

这是可以原谅的，因为 promise 来自 ECMAScript 而不是 HTML。ECMAScript 有和微任务很相似的`jobs`的概念，但是除了 [vague mailing list discussions](https://esdiscuss.org/topic/the-initialization-steps-for-web-browsers#content-16)之外联系不是很明确。不管怎样,promise 是微任务队列的一部分这是普遍的共识，并且有充分的理由。

将 promise 视为宏任务会导致性能问题，因为和任务相关的回调事件可能不必延迟，例如渲染。这也可能会对交互的其他任务源造成不确认性，并且可能会中断交互的其他 APIs,但是稍后再进一步讨论。

这是一个关于使用微任务来实现 promise 的 Edge 浏览器的 ticket。WebKit Nightly 做的是正确的，所以我认为 Safari 最终会修复这个问题，并且这个问题似乎在 FireFox43 中得到了修复。

很有趣的是，Safari 和 Firefox 在这里都经历了一次回归，这已经被修复了。我想知道这是不是巧合。

### Level 1 bossfight

在写这篇文章之前，我也搞错了。这是 html 结构：

```html
<div class="outer">
  <div class="inner"></div>
</div>
```

按照接下来的 JS,如果点击`div.inner`会打印什么？

```js
// Let's get hold of those elements
var outer = document.querySelector(".outer");
var inner = document.querySelector(".inner");

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log("mutate");
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log("click");

  setTimeout(function() {
    console.log("timeout");
  }, 0);

  Promise.resolve().then(function() {
    console.log("promise");
  });

  outer.setAttribute("data-random", Math.random());
}

// …which we'll attach to both elements
inner.addEventListener("click", onClick);
outer.addEventListener("click", onClick);
```

在给出答案之前先跑一下代码。提示：日志不止出现一次

### Test it

点击里边的的方块以触发点击事件：

你的猜测不同吗？如果是这样，你可能仍然是对的。不幸的是，浏览器在这里并不完全一致：

![task1.png](https://image.cdn.renzhaosy.cn/post/task1.png)


### 谁是正确的？

调度点击事件是一个宏任务。Mutation observer 和 promise 回调 被视为微任务排列。`setTimeout` 回调被视为宏任务排列。下边是它怎么运行：

```js
// Let's get hold of those elements
var outer = document.querySelector(".outer");
var inner = document.querySelector(".inner");

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log("mutate");
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log("click");

  setTimeout(function() {
    console.log("timeout");
  }, 0);

  Promise.resolve().then(function() {
    console.log("promise");
  });

  outer.setAttribute("data-random", Math.random());
}

// …which we'll attach to both elements
inner.addEventListener("click", onClick);
outer.addEventListener("click", onClick);
```

所以 Chrome 是正确的。对我来说是新消息的一点是，微任务是在回调（只要没有其他的 JavaScript 在运行,就会触发回调）之后处理的，我认为这只限于宏任务结束。这条规则来自用于调用回调的 HTML 规范：

> If the stack of script settings objects is now empty, perform a microtask checkpoint — HTML: Cleaning up after a callback step 3

并且除非我们正在处理微任务队列，否则微任务检查会涉及遍历整个微任务队列。同样，ECMAScript 这样解释`jobs`:

> Execution of a Job can be initiated only when there is no running execution context and the execution context stack is empty… — ECMAScript: Jobs and Job Queues

> 只有在没有执行中的执行上下文并且执行上下文堆栈为空时，才能开始一个 Job 的执行。 - ECMAScript: Jobs and Job Queues

。。。尽管`can be` 在 HTML 上下文中变成了 `must be`

### Level 1 boss's angry older brother

使用上边相同的例子，如果我们执行下边的代码会发生什么

```js
inner.click();
```

这样会向之前一样调度事件，但是是使用 script 而不是真实点击。

### Try it

下边是不同浏览器的表现:

![task2.png](https://image.cdn.renzhaosy.cn/post/task2.png)

### 为什么会不同？

下边是发生的过程：

```js
// Let's get hold of those elements
var outer = document.querySelector(".outer");
var inner = document.querySelector(".inner");

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log("mutate");
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log("click");

  setTimeout(function() {
    console.log("timeout");
  }, 0);

  Promise.resolve().then(function() {
    console.log("promise");
  });

  outer.setAttribute("data-random", Math.random());
}

// …which we'll attach to both elements
inner.addEventListener("click", onClick);
outer.addEventListener("click", onClick);
```

正确的顺序是：`click`, `click`, `promise`, `mutate`, `promise`, `timeout`, `timeout`。

在每个监听回调被调用之后，遵循规范：

> If the stack of script settings objects is now empty, perform a microtask checkpoint

— HTML: Cleaning up after a callback step 3

前边的例子，表示微任务在监听回调（click监听）之间执行，但是`.click()`造成事件同步触发，所以调用`.click()`的 script 仍然在JS stack 中。而上边的规范保证了微任务不会打断正在执行中的JavaScript。意味着，我们不会在监听回调之间处理微任务队列，而是在所有的监听之后处理。


### You made it!

总结:

- 宏任务按顺序执行，并且浏览器可能在他们之间渲染
- 微任务按照顺序执行，并且按照下边规则被执行：
  - 在每个回调之后，只要没有其他 javascript 正在执行
  - 在每一个宏任务执行完之后

今天就到这里，感谢你的阅读！
完。
