## ReactFiberUpdateQueue ##

- Update对象类型
  - priorityLevel 优先级
  - pertialState 部分状态
  - callback 回调函数
  - isReplace 是否被替换(TODO)
  - isForced 是否强制更新
  - isTopLevelUnmount 是否卸载优先级最高(TODO)
  - next 后一个Update对象

- UpdateQueue对象队列类型
  - first 第一个
  - last 最后一个
  - hasForceUpdate 是否强制更新
  - callbackList 回调对象队列
  - isProcessing 是否在处理中(Dev only)

- comparePriority 优先级比较函数, 小于0则a的优先级大于b, 等于则相同, 反之...
> 当a,b都为任务优先级*TaskPriority*或同步优先级*SynchronousPriority*时. 返回0, 也就是优先级相同
> 当a为优先级最高, b不是优先级最高时返回-255
> 反之返回255
> 一般情况返回a - b
> 优先级共有六种情况, 优先级从上之下依次降低
> - NoWork 当前没有其他更新任务
> - SynchronousPriority 为了一些受控的文本输出, 同步副作用
> - TaskPriority 在当前的任务时完成
> - HighPriority 为了完成一些需要快速反馈的更新
> - LowPriority 数据获取, 或者从store中更新数据
> - OffscreenPriority 在屏幕以外的一些视图更新

- createUpdateQueue 生成更新队列, 只是简单的返回了新对象
    虽然叫updateQueue, 但并不是一个数组, 他是一个对象, 他内部只记录了第一个Update对象和
    最后一个Update对象的地址引用, 而要形成链式的队列则依靠每个Update对象的next属性, next
    属性记录了下一个Update对象的地址引用
    ps. 这样的队列应该有一个专有名词, 不过我不是计算机专业的所以不知道
```javascript
  function createUpdateQueue(): UpdateQueue {
    const queue: UpdateQueue = {
      first: null,
      last: null,
      hasForceUpdate: false,
      callbackList: null,
    };
    if (__DEV__) {
      queue.isProcessing = false;
    }
    return queue;
  }
```
ps.React源码大量使用工厂模式构造新对象, 几乎不使用构造函数

- cloneUpdate 克隆Update对象
简单的复制, 并将next属性赋值为null
```javascript
function cloneUpdate(update: Update): Update {
  return {
    priorityLevel: update.priorityLevel,
    partialState: update.partialState,
    callback: update.callback,
    isReplace: update.isReplace,
    isForced: update.isForced,
    isTopLevelUnmount: update.isTopLevelUnmount,
    next: null,
  };
}
```

- insertUpdateIntoQueue 将Update对象插入队列中
整个函数接受四个参数
  - queue 被插入的队列
  - update 插入的对象
  - insertAfter 插入的对象前一个对象
  - insertBefore 插入对象的后一个对象
内部的操作就是将update赋值给insertAfter.next, insertBefore赋值给update.next
```javascript
function insertUpdateIntoQueue(
  queue: UpdateQueue,
  update: Update,
  insertAfter: Update | null,
  insertBefore: Update | null,
) {
  if (insertAfter !== null) {
    insertAfter.next = update;
  } else {
    // This is the first item in the queue.
    update.next = queue.first;
    queue.first = update;
  }

  if (insertBefore !== null) {
    update.next = insertBefore;
  } else {
    // This is the last item in the queue.
    queue.last = update;
  }
}
```

- findInsertionPosition 寻找插值的位置, 返回insertAfter
先和queue.last比较优先级, 若小于等于, 则queue.last = update, 优先判断这种特殊情况有利于避免循环
否则的话就要从头开始遍历, 直到找到优先级小于update的对象
```javascript
function findInsertionPosition(queue, update): Update | null {
  const priorityLevel = update.priorityLevel;
  let insertAfter = null;
  let insertBefore = null;
  if (
    queue.last !== null &&
    comparePriority(queue.last.priorityLevel, priorityLevel) <= 0
  ) {
    // Fast path for the common case where the update should be inserted at
    // the end of the queue.
    insertAfter = queue.last;
  } else {
    insertBefore = queue.first;
    while (
      insertBefore !== null &&
      comparePriority(insertBefore.priorityLevel, priorityLevel) <= 0
    ) {
      insertAfter = insertBefore;
      insertBefore = insertBefore.next;
    }
  }
  return insertAfter;
}
```

- ensureUpdateQueues 确保Fiber有更新队列, 没有的话就生成
并将fiber.updateQueue赋值给_queue1
若alternateFiber.updateQueue不等于_queue1, 则返回, 否则返回null
```javascript
function ensureUpdateQueues(fiber: Fiber) {
  // TODO
  // 不太明白fiber.alternate的作用
  const alternateFiber = fiber.alternate;

  let queue1 = fiber.updateQueue;
  if (queue1 === null) {
    queue1 = fiber.updateQueue = createUpdateQueue();
  }

  let queue2;
  if (alternateFiber !== null) {
    queue2 = alternateFiber.updateQueue;
    if (queue2 === null) {
      queue2 = alternateFiber.updateQueue = createUpdateQueue();
    }
  } else {
    queue2 = null;
  }

  _queue1 = queue1;
  // Return null if there is no alternate queue, or if its queue is the same.
  _queue2 = queue2 !== queue1 ? queue2 : null;
}
```

- insertUpdate 将update对象插入fiber
> *work-in-progress* 队列是*current*队列的一个子集.
我们需要将新的update对象同时插入这个两个队列
然而， 有一种可能情况是update将要插入的位置是不相同的
为了解决这个问题, 我们需要克隆一份update对象插入另一个队列中
并且两个队列的保持相同的持久层数据结构, 因为update插入*work-in-progress*总是在队列的前面
当两个插入的位置相同时， 我们不需要克隆update
如果update被克隆了， 则返回克隆的update, 否则返回null

```javascript
function insertUpdate(fiber: Fiber, update: Update): Update | null {
  // We'll have at least one and at most two distinct update queues.
  ensureUpdateQueues(fiber);
  const queue1 = _queue1;
  const queue2 = _queue2;

  // Warn if an update is scheduled from inside an updater function.
  if (__DEV__) {
    if (queue1.isProcessing || (queue2 !== null && queue2.isProcessing)) {
      warning(
        false,
        'An update (setState, replaceState, or forceUpdate) was scheduled ' +
          'from inside an update function. Update functions should be pure, ' +
          'with zero side-effects. Consider using componentDidUpdate or a ' +
          'callback.',
      );
    }
  }

  // Find the insertion position in the first queue.
  const insertAfter1 = findInsertionPosition(queue1, update);
  const insertBefore1 = insertAfter1 !== null
    ? insertAfter1.next
    : queue1.first;

  if (queue2 === null) {
    // If there's no alternate queue, there's nothing else to do but insert.
    insertUpdateIntoQueue(queue1, update, insertAfter1, insertBefore1);
    return null;
  }

  // If there is an alternate queue, find the insertion position.
  const insertAfter2 = findInsertionPosition(queue2, update);
  const insertBefore2 = insertAfter2 !== null
    ? insertAfter2.next
    : queue2.first;

  // Now we can insert into the first queue. This must come after finding both
  // insertion positions because it mutates the list.
  insertUpdateIntoQueue(queue1, update, insertAfter1, insertBefore1);

  // See if the insertion positions are equal. Be careful to only compare
  // non-null values.
  if (
    (insertBefore1 === insertBefore2 && insertBefore1 !== null) ||
    (insertAfter1 === insertAfter2 && insertAfter1 !== null)
  ) {
    // The insertion positions are the same, so when we inserted into the first
    // queue, it also inserted into the alternate. All we need to do is update
    // the alternate queue's `first` and `last` pointers, in case they
    // have changed.
    if (insertAfter2 === null) {
      queue2.first = update;
    }
    if (insertBefore2 === null) {
      queue2.last = null;
    }
    return null;
  } else {
    // The insertion positions are different, so we need to clone the update and
    // insert the clone into the alternate queue.
    const update2 = cloneUpdate(update);
    insertUpdateIntoQueue(queue2, update2, insertAfter2, insertBefore2);
    return update2;
  }
}
```
- addUpdate
已知partialState, callback, priotiryLevel, 向fiber插入Update

```javascript
function addUpdate(
  fiber: Fiber,
  partialState: PartialState<any, any> | null,
  callback: mixed,
  priorityLevel: PriorityLevel,
): void {
  const update = {
    priorityLevel,
    partialState,
    callback,
    isReplace: false,
    isForced: false,
    isTopLevelUnmount: false,
    next: null,
  };
  insertUpdate(fiber, update);
}
```

- addReplaceUpdate
插入ReplaceUpdate, 与addUpdate函数不同的是, 第二个参数是一个完整的state, 而不是partialState
```javascript
function addReplaceUpdate(
  fiber: Fiber,
  state: any | null,
  callback: Callback | null,
  priorityLevel: PriorityLevel,
): void {
  const update = {
    priorityLevel,
    partialState: state,
    callback,
    isReplace: true,
    isForced: false,
    isTopLevelUnmount: false,
    next: null,
  };
  insertUpdate(fiber, update);
}
```

- addForceUpdate
插入forceUpdate
```js
function addForceUpdate(
  fiber: Fiber,
  callback: Callback | null,
  priorityLevel: PriorityLevel,
): void {
  const update = {
    priorityLevel,
    partialState: null,
    callback,
    isReplace: false,
    isForced: true,
    isTopLevelUnmount: false,
    next: null,
  };
  insertUpdate(fiber, update);
}
```

- getUpdatepriority
获取fiber的updateQueue优先级
如`updateQueue === null`,则优先级最高
如该fiber不是ClassComponent, 也不是HostRoot, 则优先级最高
不然的话返回第一个Update的优先级
```js
function getUpdatePriority(fiber: Fiber): PriorityLevel {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    return NoWork;
  }
  if (fiber.tag !== ClassComponent && fiber.tag !== HostRoot) {
    return NoWork;
  }
  return updateQueue.first !== null ? updateQueue.first.priorityLevel : NoWork;
}
```

- addTopLevelUpdate
插入一个卸载等级最高的update?
并且需要看partialState的element是否为`null`
若为 null 则将这个update对象放到更新队列最后

```js
function addTopLevelUpdate(
  fiber: Fiber,
  partialState: PartialState<any, any>,
  callback: Callback | null,
  priorityLevel: PriorityLevel,
): void {
  const isTopLevelUnmount = partialState.element === null;

  const update = {
    priorityLevel,
    partialState,
    callback,
    isReplace: false,
    isForced: false,
    isTopLevelUnmount,
    next: null,
  };
  const update2 = insertUpdate(fiber, update);

  if (isTopLevelUnmount) {
    // TODO: Redesign the top-level mount/update/unmount API to avoid this
    // special case.
    const queue1 = _queue1;
    const queue2 = _queue2;

    // Drop all updates that are lower-priority, so that the tree is not
    // remounted. We need to do this for both queues.
    if (queue1 !== null && update.next !== null) {
      update.next = null;
      queue1.last = update;
    }
    if (queue2 !== null && update2 !== null && update2.next !== null) {
      update2.next = null;
      queue2.last = update;
    }
  }
}
```

- getStateFromUpdate
setState接受参数两种形式, 对象与函数
```js
function getStateFromUpdate(update, instance, prevState, props) {
  const partialState = update.partialState;
  if (typeof partialState === 'function') {
    const updateFn = partialState;
    return updateFn.call(instance, prevState, props);
  } else {
    return partialState;
  }
}
```

- beginUpdateQueue
UpdateQueue中的partialState合并至State中
```js
function beginUpdateQueue(
  current: Fiber | null,
  workInProgress: Fiber,
  queue: UpdateQueue,
  instance: any,
  prevState: any,
  props: any,
  priorityLevel: PriorityLevel,
): any {
  if (current !== null && current.updateQueue === queue) {
    // We need to create a work-in-progress queue, by cloning the current queue.
    const currentQueue = queue;
    queue = workInProgress.updateQueue = {
      first: currentQueue.first,
      last: currentQueue.last,
      // These fields are no longer valid because they were already committed.
      // Reset them.
      callbackList: null,
      hasForceUpdate: false,
    };
  }

  if (__DEV__) {
    // Set this flag so we can warn if setState is called inside the update
    // function of another setState.
    queue.isProcessing = true;
  }

  // Calculate these using the the existing values as a base.
  let callbackList = queue.callbackList;
  let hasForceUpdate = queue.hasForceUpdate;

  // Applies updates with matching priority to the previous state to create
  // a new state object.
  let state = prevState;
  let dontMutatePrevState = true;
  let update = queue.first;
  while (
    update !== null &&
    comparePriority(update.priorityLevel, priorityLevel) <= 0
  ) {
    // Remove each update from the queue right before it is processed. That way
    // if setState is called from inside an updater function, the new update
    // will be inserted in the correct position.
    queue.first = update.next;
    if (queue.first === null) {
      queue.last = null;
    }

    let partialState;
    if (update.isReplace) {
      state = getStateFromUpdate(update, instance, state, props);
      dontMutatePrevState = true;
    } else {
      partialState = getStateFromUpdate(update, instance, state, props);
      if (partialState) {
        if (dontMutatePrevState) {
          state = Object.assign({}, state, partialState);
        } else {
          state = Object.assign(state, partialState);
        }
        dontMutatePrevState = false;
      }
    }
    if (update.isForced) {
      hasForceUpdate = true;
    }
    // Second condition ignores top-level unmount callbacks if they are not the
    // last update in the queue, since a subsequent update will cause a remount.
    if (
      update.callback !== null &&
      !(update.isTopLevelUnmount && update.next !== null)
    ) {
      callbackList = callbackList !== null ? callbackList : [];
      callbackList.push(update.callback);
      workInProgress.effectTag |= CallbackEffect;
    }
    update = update.next;
  }

  queue.callbackList = callbackList;
  queue.hasForceUpdate = hasForceUpdate;

  if (queue.first === null && callbackList === null && !hasForceUpdate) {
    // The queue is empty and there are no callbacks. We can reset it.
    workInProgress.updateQueue = null;
  }

  if (__DEV__) {
    // No longer processing.
    queue.isProcessing = false;
  }

  return state;
}
```

- commitCallbacks
调用callbacks
```js
function commitCallbacks(
  finishedWork: Fiber,
  queue: UpdateQueue,
  context: mixed,
) {
  const callbackList = queue.callbackList;
  if (callbackList === null) {
    return;
  }

  // Set the list to null to make sure they don't get called more than once.
  queue.callbackList = null;

  for (let i = 0; i < callbackList.length; i++) {
    const callback = callbackList[i];
    invariant(
      typeof callback === 'function',
      'Invalid argument passed as callback. Expected a function. Instead ' +
        'received: %s',
      callback,
    );
    callback.call(context);
  }
}
```
