---
title: '使用 ref 操作 DOM'
---

<Intro>

React 会自动更新 [DOM](https://developer.mozilla.org/docs/Web/API/Document_Object_Model/Introduction) 来匹配你的渲染输出，因此通常情况下，你的组件不需要直接操作 DOM。然而，有时候你可能需要访问 React 管理的 DOM 元素，比如为了聚焦某个节点、滚动到它，或者测量它的大小和位置。由于 React 中没有内置的方法来实现这些功能，你需要使用一个 *ref* 来引用该 DOM 节点。

</Intro>

<YouWillLearn>

- 如何使用 `ref` 属性来访问 React 管理的 DOM 节点
- `ref` JSX 属性与 `useRef` Hook 的关系
- 如何访问其他组件的 DOM 节点
- 在哪些情况下可以安全地修改 React 管理的 DOM

</YouWillLearn>

## 获取节点的 ref {/*getting-a-ref-to-the-node*/}

要访问 React 管理的 DOM 节点，首先需要导入 `useRef` Hook：

```js
import { useRef } from 'react';
```

然后在你的组件内使用它声明一个 ref：

```js
const myRef = useRef(null);
```

最后，将这个 ref 作为 `ref` 属性传递给你想要获取 DOM 节点的 JSX 标签：

```js
<div ref={myRef}>
```

`useRef` Hook 返回一个包含单一属性 `current` 的对象。最开始，`myRef.current` 将是 `null`。当 React 为这个 `<div>` 创建 DOM 节点时，它会将该节点的引用放入 `myRef.current` 中。这样，你就可以在你的 [事件处理器](/learn/responding-to-events) 中访问这个 DOM 节点，并使用定义在其上的内置 [浏览器 API](https://developer.mozilla.org/docs/Web/API/Element)。

```js
// 你可以使用任意浏览器 API，例如：
myRef.current.scrollIntoView();
```

### 示例: 使文本输入框获得焦点 {/*example-focusing-a-text-input*/}

在本例中，单击按钮将使输入框获得焦点：

<Sandpack>

```js
import { useRef } from 'react';

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

</Sandpack>

实现步骤如下：

1. 使用 `useRef` Hook 声明 `inputRef`。
2. 将其作为 `<input ref={inputRef}>` 传递。这告诉 React **将这个 `<input>` 的 DOM 节点放入 `inputRef.current` 中。**
3. 在 `handleClick` 函数中，从 `inputRef.current` 读取输入的 DOM 节点，并调用 [`focus()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus) 方法：`inputRef.current.focus()`。
4. 将 `handleClick` 事件处理器传递给 `<button>` 的 `onClick`。

虽然 DOM 操作是 refs 最常见的用例，但 `useRef` Hook 也可以用于存储 React 之外的其他东西，比如定时器 ID。与状态类似，refs 在渲染之间保持不变。Refs 就像状态变量，当你设置它们时不会触发重新渲染。可以在 [通过 Refs 引用值](https://reactjs.org/docs/refs-and-the-dom.html) 中了解更多关于 refs 的内容。

### 示例: 滚动至一个元素 {/*example-scrolling-to-an-element*/}

你可以在一个组件中使用多个 refs。在这个示例中，有一个包含三张图片的轮播图。每个按钮通过在相应 DOM 节点上调用浏览器的 [`scrollIntoView()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView) 方法来居中一张图片：

<Sandpack>

```js
import { useRef } from 'react';

export default function CatFriends() {
  const firstCatRef = useRef(null);
  const secondCatRef = useRef(null);
  const thirdCatRef = useRef(null);

  function handleScrollToFirstCat() {
    firstCatRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  function handleScrollToSecondCat() {
    secondCatRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  function handleScrollToThirdCat() {
    thirdCatRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
      inline: 'center'
    });
  }

  return (
    <>
      <nav>
        <button onClick={handleScrollToFirstCat}>
          Tom
        </button>
        <button onClick={handleScrollToSecondCat}>
          Maru
        </button>
        <button onClick={handleScrollToThirdCat}>
          Jellylorum
        </button>
      </nav>
      <div>
        <ul>
          <li>
            <img
              src="https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg"
              alt="Tom"
              width="200"
              height="200"
              ref={firstCatRef}
            />
          </li>
          <li>
            <img
              src="https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg"
              alt="Maru"
              width="200"
              height="200"
              ref={secondCatRef}
            />
          </li>
          <li>
            <img
              src="https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg"
              alt="Jellylorum"
              width="200"
              height="200"
              ref={thirdCatRef}
            />
          </li>
        </ul>
      </div>
    </>
  );
}
```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}
```

</Sandpack>

<DeepDive>

#### 如何使用 ref 回调管理 refs 列表 {/*how-to-manage-a-list-of-refs-using-a-ref-callback*/}

在上面的例子中，ref 的数量是预先确定的。但有时候，你可能需要为列表中的每一项都绑定 ref ，而你又不知道会有多少项。像下面这样做 **是行不通的**：

```js
<ul>
  {items.map((item) => {
    // 行不通！
    const ref = useRef(null);
    return <li ref={ref} />;
  })}
</ul>
```

这是因为 **Hooks 只能在组件的顶层调用。** 你不能在循环中、条件中或 `map()` 调用中调用 `useRef`。

一种可能的解决方法是获取其父元素的单个 ref，然后使用 DOM 操作方法如 [`querySelectorAll`](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelectorAll) 从中“查找”单个子节点。然而，这种做法比较脆弱，如果你的 DOM 结构发生变化，就会出问题。

另一种解决方案是 **将一个函数传递给 `ref` 属性。** 这称为 [`ref` 回调](https://reactjs.org/docs/forwarding-refs.html#ref-callback)。当需要设置 ref 时，React 会调用你的 ref 回调并传入 DOM 节点，而在清除时则传入 `null`。这样你就可以维护自己的数组或 [Map](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Map)，并通过索引或某种 ID 访问任何 ref。

这个示例展示了如何使用这种方法滚动到长列表中的任意节点：

<Sandpack>

```js
import { useRef, useState } from "react";

export default function CatFriends() {
  const itemsRef = useRef(null);
  const [catList, setCatList] = useState(setupCatList);

  function scrollToCat(cat) {
    const map = getMap();
    const node = map.get(cat);
    node.scrollIntoView({
      behavior: "smooth",
      block: "nearest",
      inline: "center",
    });
  }

  function getMap() {
    if (!itemsRef.current) {
      // Initialize the Map on first usage.
      itemsRef.current = new Map();
    }
    return itemsRef.current;
  }

  return (
    <>
      <nav>
        <button onClick={() => scrollToCat(catList[0])}>Tom</button>
        <button onClick={() => scrollToCat(catList[5])}>Maru</button>
        <button onClick={() => scrollToCat(catList[9])}>Jellylorum</button>
      </nav>
      <div>
        <ul>
          {catList.map((cat) => (
            <li
              key={cat}
              ref={(node) => {
                const map = getMap();
                if (node) {
                  map.set(cat, node);
                } else {
                  map.delete(cat);
                }
              }}
            >
              <img src={cat} />
            </li>
          ))}
        </ul>
      </div>
    </>
  );
}

function setupCatList() {
  const catList = [];
  for (let i = 0; i < 10; i++) {
    catList.push("https://loremflickr.com/320/240/cat?lock=" + i);
  }

  return catList;
}

```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}
```

```json package.json hidden
{
  "dependencies": {
    "react": "canary",
    "react-dom": "canary",
    "react-scripts": "^5.0.0"
  }
}
```

</Sandpack>

在这个例子中，`itemsRef` 保存的不是单个 DOM 节点，而是保存了包含列表项 ID 和 DOM 节点的 [Map](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Map)。([Ref 可以保存任何值！](/learn/referencing-values-with-refs)) 每个列表项上的 [`ref` 回调](https://reactjs.org/docs/forwarding-refs.html#ref-callback) 负责更新 Map：

```js
<li
  key={cat.id}
  ref={node => {
    const map = getMap();
    if (node) {
      // Add to the Map
      map.set(cat, node);
    } else {
      // Remove from the Map
      map.delete(cat);
    }
  }}
>
```

这使你可以之后从 Map 读取单个 DOM 节点。

<Canary>

这个示例展示了使用 `ref` 回调清理函数管理 Map 的另一种方法。

```js
<li
  key={cat.id}
  ref={node => {
    const map = getMap();
    // Add to the Map
    map.set(cat, node);

    return () => {
      // Remove from the Map
      map.delete(cat);
    };
  }}
>
```

</Canary>

</DeepDive>

## 访问另一个组件的 DOM 节点 {/*accessing-another-components-dom-nodes*/}

当你在一个内置组件上放置 ref，例如 `<input />`，React 会将该 ref 的 `current` 属性设置为相应的 DOM 节点（例如浏览器中的实际 `<input />`）。

但是，如果你尝试在 **你自己的** 组件上放置 ref，例如 `<MyInput />`，默认情况下你将得到 `null`。以下是一个演示这一点的示例。注意点击按钮 **并不会** 聚焦输入框：

<Sandpack>

```js
import { useRef } from 'react';

function MyInput(props) {
  return <input {...props} />;
}

export default function MyForm() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

</Sandpack>

为了帮助你注意到这个问题，React 还在控制台打印了一个错误：

<ConsoleBlock level="error">

Warning: Function components cannot be given refs. Attempts to access this ref will fail. Did you mean to use React.forwardRef()?

</ConsoleBlock>

<ConsoleBlock level="error">

警告：函数组件不能被赋予 refs。尝试访问这个 ref 将失败。你是想使用 React.forwardRef() 吗？

</ConsoleBlock>

发生这种情况是因为默认情况下 React 不允许一个组件访问其他组件的 DOM 节点。即使是它自己的子组件！这是有意为之的。Refs 是一种逃生机制，应当谨慎使用。手动操作 _另一个_ 组件的 DOM 节点会使你的代码变得更加脆弱。

相反，想要暴露其 DOM 节点的组件必须 **选择** 这种行为。一个组件可以指定它将 ref "转发" 到其子组件之一。以下是 `MyInput` 如何使用 `forwardRef` API：

```js
const MyInput = forwardRef((props, ref) => {
  return <input {...props} ref={ref} />;
});
```

它是这样工作的:

1. `<MyInput ref={inputRef} />` 告诉 React 将相应的 DOM 节点放入 `inputRef.current`。然而，是否选择这种行为取决于 `MyInput` 组件——默认情况下，它不会这样做。
2. `MyInput` 组件使用 `forwardRef` 声明。**这使得它能够接收来自上面的 `inputRef` 作为第二个 `ref` 参数，该参数在 `props` 之后声明。**
3. `MyInput` 自身将接收到的 `ref` 传递给它内部的 `<input>`。

现在，点击按钮聚焦输入框的操作有效了：

<Sandpack>

```js
import { forwardRef, useRef } from 'react';

const MyInput = forwardRef((props, ref) => {
  return <input {...props} ref={ref} />;
});

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        聚焦输入框
      </button>
    </>
  );
}
```

</Sandpack>

在设计系统中，将低级组件（如按钮、输入框等）的 ref 转发给其 DOM 节点是一种常见模式。另一方面，高级组件如表单、列表或页面部分通常不会暴露其 DOM 节点，以避免意外依赖于 DOM 结构。

<DeepDive>

#### 使用命令句柄暴露一部分 API {/*exposing-a-subset-of-the-api-with-an-imperative-handle*/}

在上述示例中，`MyInput` 暴露了原始的 DOM 元素 input。这让父组件可以对其调用 `focus()`。然而，这也让父组件能够做其他事情——例如，改变其 CSS 样式。在不常见的情况下，你可能想限制暴露的功能。你可以通过 `useImperativeHandle` 来做到这一点：

<Sandpack>

```js
import {
  forwardRef, 
  useRef, 
  useImperativeHandle
} from 'react';

const MyInput = forwardRef((props, ref) => {
  const realInputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    // 只暴露 focus，没有别的
    focus() {
      realInputRef.current.focus();
    },
  }));
  return <input {...props} ref={realInputRef} />;
});

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        聚焦输入框
      </button>
    </>
  );
}
```

</Sandpack>

在这里，`MyInput` 内部的 `realInputRef` 保存了实际的 input DOM 节点。然而，`useImperativeHandle` 指示 React 将你自己指定的对象作为父组件的 ref 值。因此，`Form` 组件内部的 `inputRef.current` 只会有 `focus` 方法。在这种情况下，ref "句柄" 不是 DOM 节点，而是你在 `useImperativeHandle` 调用中创建的自定义对象。

</DeepDive>

## React 何时添加 refs {/*when-react-attaches-the-refs*/}

在 React 中，每次更新分为 [两个阶段](/learn/render-and-commit#step-3-react-commits-changes-to-the-dom)：

* 在 **渲染** 阶段，React 调用你的组件以确定屏幕上应该显示什么。
* 在 **提交** 阶段，React 将更改应用到 DOM。

通常，你 [不希望](/learn/referencing-values-with-refs#best-practices-for-refs) 在渲染期间访问 refs。对于保存 DOM 节点的 refs 也是如此。在第一次渲染期间，DOM 节点尚未创建，因此 `ref.current` 将为 `null`。而在更新的渲染期间，DOM 节点尚未更新。因此，读取它们还为时已早。

React 在提交阶段设置 `ref.current`。在更新 DOM 之前，React 将受影响的 `ref.current` 值设置为 `null`。在更新 DOM 之后，React 会立即将它们设置为相应的 DOM 节点。

**通常，你会在事件处理器中访问 refs。** 如果你想对一个 ref 执行某些操作，但没有特定事件来执行它，你可能需要使用 Effect。我们将在接下来的页面中讨论 Effect。

<DeepDive>

#### 用 flushSync 同步更新 state {/*flushing-state-updates-synchronously-with-flush-sync*/}

考虑这样的代码，它添加一个新的待办事项并将屏幕滚动到列表的最后一个子项。注意，由于某种原因，请注意，出于某种原因，它总是滚动到最后一个添加 *之前* 的待办事项：

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function TodoList() {
  const listRef = useRef(null);
  const [text, setText] = useState('');
  const [todos, setTodos] = useState(
    initialTodos
  );

  function handleAdd() {
    const newTodo = { id: nextId++, text: text };
    setText('');
    setTodos([ ...todos, newTodo]);
    listRef.current.lastChild.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest'
    });
  }

  return (
    <>
      <button onClick={handleAdd}>
        Add
      </button>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <ul ref={listRef}>
        {todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </>
  );
}

let nextId = 0;
let initialTodos = [];
for (let i = 0; i < 20; i++) {
  initialTodos.push({
    id: nextId++,
    text: '待办 #' + (i + 1)
  });
}
```

</Sandpack>

问题出在这两行代码上：

```js
setTodos([ ...todos, newTodo]);
listRef.current.lastChild.scrollIntoView();
```

在 React 中， [state 更新是排队进行的](/learn/queueing-a-series-of-state-updates)。通常，这正是你想要的。然而，在这里这造成了问题，因为 `setTodos` 不会立即更新 DOM。因此，当你将列表滚动到最后一个元素时，待办事项尚未添加。这就是为什么滚动总是“滞后”一个项目的原因。

要解决这个问题，你可以强制 React 同步更新（"flush"）DOM。为此，导入 `flushSync` 从 `react-dom` 并 **将 state 更新包裹在 `flushSync` 调用中**：

```js
flushSync(() => {
  setTodos([ ...todos, newTodo]);
});
listRef.current.lastChild.scrollIntoView();
```

这将指示 React 在执行包裹在 `flushSync` 中的代码后立即同步更新 DOM。因此，当你尝试滚动到它时，最后的待办事项将已存在于 DOM 中：

<Sandpack>

```js
import { useState, useRef } from 'react';
import { flushSync } from 'react-dom';

export default function TodoList() {
  const listRef = useRef(null);
  const [text, setText] = useState('');
  const [todos, setTodos] = useState(
    initialTodos
  );

  function handleAdd() {
    const newTodo = { id: nextId++, text: text };
    flushSync(() => {
      setText('');
      setTodos([ ...todos, newTodo]);      
    });
    listRef.current.lastChild.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest'
    });
  }

  return (
    <>
      <button onClick={handleAdd}>
        Add
      </button>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <ul ref={listRef}>
        {todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </>
  );
}

let nextId = 0;
let initialTodos = [];
for (let i = 0; i < 20; i++) {
  initialTodos.push({
    id: nextId++,
    text: '待办 #' + (i + 1)
  });
}
```

</Sandpack>

</DeepDive>

## 使用 refs 操作 DOM 的最佳实践 {/*best-practices-for-dom-manipulation-with-refs*/}

Refs 是一种逃生机制。你应该只在你必须“跳出 React”的时候使用它们。常见的例子包括管理焦点、滚动位置，或调用 React 未暴露的浏览器 API。

如果你坚持进行非破坏性操作，如聚焦和滚动，你不应该遇到任何问题。然而，如果你尝试 **手动修改** DOM，你可能会与 React 的更改发生冲突。

为说明这个问题，这个例子包括一条欢迎消息和两个按钮。第一个按钮使用 [条件渲染](/learn/conditional-rendering) 和 [state](/learn/state-a-components-memory) 切换其存在状态，就像你通常在 React 中做的那样。第二个按钮使用 [`remove()` DOM API](https://developer.mozilla.org/en-US/docs/Web/API/Element/remove) 强制将其从 DOM 中移除，脱离 React 的控制。

尝试多次按下“使用 setState 切换”。消息应该消失并再次出现。然后按下“从 DOM 中移除”。这将强制移除它。最后，按下“使用 setState 切换”：

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Counter() {
  const [show, setShow] = useState(true);
  const ref = useRef(null);

  return (
    <div>
      <button
        onClick={() => {
          setShow(!show);
        }}>
        通过 setState 切换
      </button>
      <button
        onClick={() => {
          ref.current.remove();
        }}>
        从 DOM 中删除
      </button>
      {show && <p ref={ref}>Hello world</p>}
    </div>
  );
}
```

```css
p,
button {
  display: block;
  margin: 10px;
}
```

</Sandpack>

在你手动移除 DOM 元素后，尝试使用 `setState` 再次显示它将导致崩溃。这是因为你已经改变了 DOM，而 React 不知道如何继续正确管理它。

**避免更改由 React 管理的 DOM 节点。** 修改、向元素添加子节点或从 React 管理的元素中移除子节点可能会导致不一致的视觉效果或像上述那样的崩溃。

然而，这并不意味着你绝对不能这样做。它需要谨慎。**你可以安全地修改 React _没有理由_ 更新的 部分 DOM。** 例如，如果某个 `<div>` 在 JSX 中始终为空，React 就没有理由去触碰其子节点列表。因此，在那里手动添加或移除元素是安全的。

<Recap>

- Refs 是一个通用概念，但大多数情况下你会使用它们来保存 DOM 元素。
- 你通过传递 `<div ref={myRef}>` 指示 React 将 DOM 节点放入 `myRef.current` 中。
- 通常，你会使用 refs 进行非破坏性操作，如聚焦、滚动或测量 DOM 元素。
- 组件默认不会暴露其 DOM 节点。你可以通过使用 `forwardRef` 并将第二个 `ref` 参数传递给特定节点来选择暴露一个 DOM 节点。
- 避免更改由 React 管理的 DOM 节点。
- 如果你确实修改由 React 管理的 DOM 节点，请修改 React 没有理由更新的部分。

</Recap>



<Challenges>

#### 播放和暂停视频 {/*play-and-pause-the-video*/}

在这个示例中，按钮切换 state 变量以在播放和暂停状态之间切换。然而，为了真正播放或暂停视频，仅切换状态是不够的。你还需要在 `<video>` 的 DOM 元素上调用 [`play()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/play) 和 [`pause()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/pause)。给它添加一个 ref，并使按钮正常工作。

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function VideoPlayer() {
  const [isPlaying, setIsPlaying] = useState(false);

  function handleClick() {
    const nextIsPlaying = !isPlaying;
    setIsPlaying(nextIsPlaying);
  }

  return (
    <>
      <button onClick={handleClick}>
        {isPlaying ? '展厅' : '播放'}
      </button>
      <video width="250">
        <source
          src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
          type="video/mp4"
        />
      </video>
    </>
  )
}
```

```css
button { display: block; margin-bottom: 20px; }
```

</Sandpack>

作为额外挑战，保持“播放”按钮与视频是否正在播放同步，即使用户右键单击视频并使用内置的浏览器媒体控件播放它。你可能需要监听视频的 `onPlay` 和 `onPause` 事件来实现这一点。

<Solution>

声明一个 ref，并将其放在 `<video>` 元素上。然后根据下一个状态调用 `ref.current.play()` 和 `ref.current.pause()`。

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function VideoPlayer() {
  const [isPlaying, setIsPlaying] = useState(false);
  const ref = useRef(null);

  function handleClick() {
    const nextIsPlaying = !isPlaying;
    setIsPlaying(nextIsPlaying);

    if (nextIsPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }

  return (
    <>
      <button onClick={handleClick}>
        {isPlaying ? '展厅' : '播放'}
      </button>
      <video
        width="250"
        ref={ref}
        onPlay={() => setIsPlaying(true)}
        onPause={() => setIsPlaying(false)}
      >
        <source
          src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
          type="video/mp4"
        />
      </video>
    </>
  )
}
```

```css
button { display: block; margin-bottom: 20px; }
```

</Sandpack>

为了处理内置的浏览器控件，你可以在 `<video>` 元素上添加 `onPlay` 和 `onPause` 处理器，并从中调用 `setIsPlaying`。这样，如果用户使用浏览器控件播放视频，状态将相应地调整。

</Solution>

#### 使搜索域获得焦点 {/*focus-the-search-field*/}

使得点击“搜索”按钮时将焦点放入输入框。

<Sandpack>

```js
export default function Page() {
  return (
    <>
      <nav>
        <button>Search</button>
      </nav>
      <input
        placeholder="找什么呢？"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

<Solution>

给输入框添加一个 ref，并在 DOM 节点上调用 `focus()` 以使其聚焦：

<Sandpack>

```js
import { useRef } from 'react';

export default function Page() {
  const inputRef = useRef(null);
  return (
    <>
      <nav>
        <button onClick={() => {
          inputRef.current.focus();
        }}>
          Search
        </button>
      </nav>
      <input
        ref={inputRef}
        placeholder="找什么呢？"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

</Solution>

#### 滚动图像轮播 {/*scrolling-an-image-carousel*/}

这个图像轮播有一个“下一张”按钮，可以切换当前的图像。点击时使画廊水平滚动到当前活动的图像。你将需要在当前图像的 DOM 节点上调用 [`scrollIntoView()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView)：

```js
node.scrollIntoView({
  behavior: 'smooth',
  block: 'nearest',
  inline: 'center'
});
```

<Hint>

对于这个练习，你不需要为每张图片都有一个 ref。拥有一个指向当前活动图像或列表本身的 ref 就足够了。使用 `flushSync` 确保 DOM 在滚动之前更新。

</Hint>

<Sandpack>

```js
import { useState } from 'react';

export default function CatFriends() {
  const [index, setIndex] = useState(0);
  return (
    <>
      <nav>
        <button onClick={() => {
          if (index < catList.length - 1) {
            setIndex(index + 1);
          } else {
            setIndex(0);
          }
        }}>
          Next
        </button>
      </nav>
      <div>
        <ul>
          {catList.map((cat, i) => (
            <li key={cat.id}>
              <img
                className={
                  index === i ?
                    'active' :
                    ''
                }
                width={200}
                height={200}
                src={cat.imageUrl}
                alt={'Cat #' + cat.id}
              />
            </li>
          ))}
        </ul>
      </div>
    </>
  );
}

const catList = [];
for (let i = 0; i < 10; i++) {
  catList.push({
    id: i,
    imageUrl: 'https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg'
  });
}

```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}

img {
  padding: 10px;
  margin: -10px;
  transition: background 0.2s linear;
}

.active {
  background: rgba(0, 100, 150, 0.4);
}
```

</Sandpack>

<Solution>

你可以声明一个 `selectedRef`，然后根据条件将其仅传递给当前图像：

```js
<li ref={index === i ? selectedRef : null}>
```

当 `index === i`，意味着该图像是选中的，`<li>` 将接收到 `selectedRef`。React 会确保 `selectedRef.current` 始终指向正确的 DOM 节点。

请注意，`flushSync` 调用是必要的，以强制 React 在滚动之前更新 DOM。否则，`selectedRef.current` 将始终指向上一个选中的项目。

<Sandpack>

```js
import { useRef, useState } from 'react';
import { flushSync } from 'react-dom';

export default function CatFriends() {
  const selectedRef = useRef(null);
  const [index, setIndex] = useState(0);

  return (
    <>
      <nav>
        <button onClick={() => {
          flushSync(() => {
            if (index < catList.length - 1) {
              setIndex(index + 1);
            } else {
              setIndex(0);
            }
          });
          selectedRef.current.scrollIntoView({
            behavior: 'smooth',
            block: 'nearest',
            inline: 'center'
          });            
        }}>
          Next
        </button>
      </nav>
      <div>
        <ul>
          {catList.map((cat, i) => (
            <li
              key={cat.id}
              ref={index === i ?
                selectedRef :
                null
              }
            >
              <img
                className={
                  index === i ?
                    'active'
                    : ''
                }
                width={200}
                height={200}
                src={cat.imageUrl}
                alt={'Cat #' + cat.id}
              />
            </li>
          ))}
        </ul>
      </div>
    </>
  );
}

const catList = [];
for (let i = 0; i < 10; i++) {
  catList.push({
    id: i,
    imageUrl: 'https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg'
  });
}

```

```css
div {
  width: 100%;
  overflow: hidden;
}

nav {
  text-align: center;
}

button {
  margin: .25rem;
}

ul,
li {
  list-style: none;
  white-space: nowrap;
}

li {
  display: inline;
  padding: 0.5rem;
}

img {
  padding: 10px;
  margin: -10px;
  transition: background 0.2s linear;
}

.active {
  background: rgba(0, 100, 150, 0.4);
}
```

</Sandpack>

</Solution>

#### 使分开的组件中的搜索域获得焦点 {/*focus-the-search-field-with-separate-components*/}

使得点击“搜索”按钮时将焦点放入输入框。请注意，每个组件都在单独的文件中定义，不应移出。你将如何将它们连接在一起？

<Hint>

你需要使用 `forwardRef` 来选择暴露一个来自你自己组件（如 `SearchInput`）的 DOM 节点。

</Hint>

<Sandpack>

```js src/App.js
import SearchButton from './SearchButton.js';
import SearchInput from './SearchInput.js';

export default function Page() {
  return (
    <>
      <nav>
        <SearchButton />
      </nav>
      <SearchInput />
    </>
  );
}
```

```js src/SearchButton.js
export default function SearchButton() {
  return (
    <button>
      搜索
    </button>
  );
}
```

```js src/SearchInput.js
export default function SearchInput() {
  return (
    <input
      placeholder="找什么呢？"
    />
  );
}
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

<Solution>

你需要向 `SearchButton` 添加一个 `onClick` 属性，并使 `SearchButton` 将其传递给浏览器的 `<button>`。你还将向 `<SearchInput>` 传递一个 ref，该 ref 将转发给实际的 `<input>` 并填充它。最后，在点击处理程序中，你将调用 `focus` 在存储在该 ref 中的 DOM 节点上。

<Sandpack>

```js src/App.js
import { useRef } from 'react';
import SearchButton from './SearchButton.js';
import SearchInput from './SearchInput.js';

export default function Page() {
  const inputRef = useRef(null);
  return (
    <>
      <nav>
        <SearchButton onClick={() => {
          inputRef.current.focus();
        }} />
      </nav>
      <SearchInput ref={inputRef} />
    </>
  );
}
```

```js src/SearchButton.js
export default function SearchButton({ onClick }) {
  return (
    <button onClick={onClick}>
      搜索
    </button>
  );
}
```

```js src/SearchInput.js
import { forwardRef } from 'react';

export default forwardRef(
  function SearchInput(props, ref) {
    return (
      <input
        ref={ref}
        placeholder="找什么呢？"
      />
    );
  }
);
```

```css
button { display: block; margin-bottom: 10px; }
```

</Sandpack>

</Solution>

</Challenges>
