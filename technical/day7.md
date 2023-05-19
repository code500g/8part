# 如果100个请求，你怎么用Promise去控制并发？

我们可以使用一个队列来维护这100个请求，然后设定一个最大并发数，比如10，也就是说同时只能有10个请求在运行。每次当一个请求结束后，就从队列中取出一个新的请求开始执行，这样就可以保证同时运行的请求数量不超过10。

在JavaScript中，可以通过 async/await 和 Promise.all 来实现这个逻辑：

场景一：任务列表是固定的，并且并发数也是固定的
```js
//方案一
const maxConcurrent = 10;  // 最大并发数
const allTasks = [...];  // 所有的请求任务，每个任务是一个返回 Promise 的函数

async function handleTasks() {
  let tasks = allTasks.slice(0, maxConcurrent);  // 取出前10个任务开始执行
  allTasks = allTasks.slice(maxConcurrent);  // 剩下的任务

  while (tasks.length) {
    let task = tasks.shift();  // 取出一个任务
    await task().then(() => {
      if (allTasks.length) {  // 如果还有剩下的任务，就添加到队列中
        tasks.push(allTasks.shift());
      }
    });
  }
}

handleTasks();

```
方案一更简单直观，适用于并发数固定，任务列表不会在运行过程中变化的情况。



场景二：需要处理的任务数量很大，或者需要在运行过程中动态地控制并发的数量
```js
//方案二：
const maxConcurrent = 10;  // 最大并发数
const allTasks = new Set();  // 所有的请求任务，每个任务是一个返回 Promise 的函数

async function handleTasks() {
  let promises = Array.from(allTasks).slice(0, maxConcurrent);  // 取出前10个任务开始执行
  for (let promise of promises) {
    allTasks.delete(promise);
  }

  while (promises.length) {
    let finishedPromise = await Promise.race(promises);  // 等待最先结束的请求
    promises = promises.filter(p => p !== finishedPromise);  // 从运行中的任务列表中删除已经结束的任务

    if (allTasks.size) {  // 如果还有剩下的任务，就添加到队列中
      let newPromise = allTasks.values().next().value;
      allTasks.delete(newPromise);
      promises.push(newPromise);
    }
  }
}

handleTasks();

```
方案二通过使用 Set 来管理任务列表，我们可以在运行过程中动态地添加或删除任务，并且通过使用 Promise.race，我们可以更精确地控制并发的数量。