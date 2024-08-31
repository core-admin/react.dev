---
title: '使用 Effect 同步'
---

<Intro>

一些组件需要与外部系统进行同步。例如，您可能希望根据 React state 控制一个非 React 组件，建立服务器连接，或者在组件出现在屏幕上时发送分析日志。*Effects* 允许您在渲染后执行一些代码，以便可以将组件与 React 之外的某些系统同步。

</Intro>

<YouWillLearn>

- 什么是 Effect
- Effect 和事件（event）的不同之处
- 如何在组件中声明 Effect
- 如何避免不必要的重新运行 Effect
- 为什么 Effect 在开发中运行两次，以及如何修复它们

</YouWillLearn>

## 什么是 Effect，它与事件（event）有何不同？ {/*what-are-effects-and-how-are-they-different-from-events*/}

在讨论 Effect 之前，我们需要了解 React 组件内部的两种逻辑类型：

- **渲染代码**（在 [描述 UI](/learn/describing-the-ui) 中介绍）位于组件的顶层。这里是您处理 props 和 state，最终返回你想在屏幕上看到的 JSX。[渲染代码必须是纯的。](/learn/keeping-components-pure) 就像数学公式一样，它应该只 _计算_ 结果，而不做其他事情。

- **事件处理程序**（在 [添加交互性](/learn/adding-interactivity) 中介绍）是嵌套在组件内部的函数，它们 *执行* 操作，而不仅仅是计算。事件处理程序可能会更新输入字段，提交 HTTP POST 请求以购买产品，或将用户导航到另一个屏幕。事件处理程序包含由特定用户操作（例如按钮点击或输入）引起的 ["副作用"](https://en.wikipedia.org/wiki/Side_effect_(computer_science))（它们改变程序的状态）。

有时，仅有这些还不够。考虑一个 `ChatRoom` 组件，它需要在屏幕上可见时连接到聊天服务器。连接到服务器并不是一个纯粹的计算（这是一个副作用），因此它不能在渲染期间发生。然而，并没有一个特定的事件（比如点击）导致 `ChatRoom` 被显示。

***Effect* 允许你指定由渲染本身，而不是特定事件引起的副作用。** 在聊天中发送消息是一个 *事件*，因为它是用户点击特定按钮直接引起的。然而，建立服务器连接是一个 *Effect*，因为它应该在组件出现时无论哪个交互都发生。Effect 在屏幕更新后的 [提交](/learn/render-and-commit) 结束时运行。这是将 React 组件与一些外部系统（如网络或第三方库）同步的好时机。

<Note>

这里以及后面的文本中，首字母大写的 "Effect" 指的是上面的 React 中是专有定义--即由渲染引起的副作用。为了指代更广泛的编程概念，也可以将其称为“副作用（side effect）”。

</Note>


## 你可能不需要 Effect {/*you-might-not-need-an-effect*/}

**不要急于向组件添加 Effects。** 请记住，Effects 通常用于“跳出”您的 React 代码，并与某些 *外部* 系统同步。这包括浏览器 API、第三方小部件、网络等等。如果您的 Effect 仅仅是根据其他状态调整某些状态，[您可能不需要一个 Effect。](/learn/you-might-not-need-an-effect)

## 如何编写 Effect {/*how-to-write-an-effect*/}

要编写一个 Effect，请遵循以下三个步骤：

1. **声明 Effect。** 默认情况下，您的 Effect 会在每次 [提交](/learn/render-and-commit) 后运行。
2. **指定 Effect 依赖项。** 大多数 Effect 应该只在 *需要时* 重新运行，而不是在每次渲染后。例如，淡入动画应该只在组件出现时触发。连接和断开服务器的操作应该只在组件出现和消失时，或切换聊天室时执行。您将学习如何通过指定 *依赖项* 来控制这一点。
3. **必要时添加清理（cleanup）函数。** 有时 Effect 需要指定如何停止、撤销或清理它们正在做的事情。例如，“连接”需要“断开”，“订阅”需要“取消订阅”，而“获取”需要“取消”或“忽略”。您将学习如何通过返回 *清理函数* 来做到这一点。

让我们详细看看每个步骤。

### 第一步：声明 Effect {/*step-1-declare-an-effect*/}

要在组件中声明一个 Effect，从 React 导入 [`useEffect` Hook](/reference/react/useEffect)：

```js
import { useEffect } from 'react';
```

然后，在组件的顶层调用它，并传入在每次渲染时都需要执行的代码：

```js {2-4}
function MyComponent() {
  useEffect(() => {
    // 每次渲染后都会执行此处的代码
  });
  return <div />;
}
```

每次组件渲染时，React 将首先更新屏幕 *然后* 运行 `useEffect` 中的代码。换句话说，**`useEffect` “延迟”了一段代码的运行，直到该渲染在屏幕上得到反映（`useEffect ` 会把这段代码放到屏幕更新渲染之后执行）。**

让我们看看如何使用 Effect 与外部系统进行同步。考虑一个 `<VideoPlayer>` React 组件。通过传递一个 `isPlaying` prop 来控制它是播放还是暂停：

```js
<VideoPlayer isPlaying={isPlaying} />;
```

自定义的 `VideoPlayer` 组件渲染了内置的 [`<video>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video) 标签：

```js
function VideoPlayer({ src, isPlaying }) {
  // TODO：使用 isPlaying 做一些事情
  return <video src={src} />;
}
```

然而，浏览器 `<video>` 标签没有 `isPlaying` 属性。控制它的唯一方法是手动调用 DOM 元素上的 [`play()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/play) 和 [`pause()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/pause) 方法。 **您需要将 `isPlaying` prop 的值与 `play()` 和 `pause()` 的调用进行同步，以决定当前视频 _是否_ 应该播放。**

我们需要首先 [获取一个引用](/learn/manipulating-the-dom-with-refs) 到 `<video>` DOM 节点。

你可能会尝试在渲染期间调用 `play()` 或 `pause()`，但这并不正确：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  if (isPlaying) {
    ref.current.play();  // 渲染期间不能调用 `play()`。 
  } else {
    ref.current.pause(); // 同样，调用 `pause()` 也不行。
  }

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  return (
    <>
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? '暂停' : '播放'}
      </button>
      <VideoPlayer
        isPlaying={isPlaying}
        src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

这段代码不正确的原因是它试图在渲染期间对 DOM 节点执行某些操作。在 React 中，[渲染应该是 JSX 的纯计算](/learn/keeping-components-pure)，不应包含诸如修改 DOM 之类的副作用。

此外，当 `VideoPlayer` 第一次被调用时，它的 DOM 还不存在！因为 React 在您返回 JSX 之前不知道要创建什么 DOM，所以尚不存在可以调用 `play()` 或 `pause()` 的 DOM 节点。

解决方案是 **用 `useEffect` 将副作用包裹起来，把它分离到渲染逻辑的计算过程之外：**

```js {6,12}
import { useEffect, useRef } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  });

  return <video ref={ref} src={src} loop playsInline />;
}
```

把调用 DOM 方法的操作封装在 Effect 中，你可以让 React 先更新屏幕，确定相关 DOM 创建好了以后然后再运行 Effect。

当您的 `VideoPlayer` 组件渲染（无论是第一次还是重新渲染）时，会发生几件事情。首先，React 将更新屏幕，确保 `<video>` 元素已经正确地出现在 DOM 中；然后 React 将运行您的 Effect。最后，您的 Effect 将根据 `isPlaying` 的值调用 `play()` 或 `pause()`。

试试按下几次播放和暂停操作，观察视频播放器的播放、暂停行为是如何与 `isPlaying` prop 同步的：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  });

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  return (
    <>
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? '暂停' : '播放'}
      </button>
      <VideoPlayer
        isPlaying={isPlaying}
        src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
      />
    </>
  );
}
```

```css
button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

在这个例子中，您与 React state 同步的“外部系统”是浏览器媒体 API。您可以使用类似的方法将遗留的非 React 代码（如 jQuery 插件）包装成声明式 React 组件。

请注意，控制视频播放器在实践中要复杂得多。调用 `play()` 可能会失败，用户可能会使用内置的浏览器控件播放或暂停，等等。这个例子非常简化且不完整。

<Pitfall>

默认情况下，Effects 在 *每次* 渲染后运行。这就是为什么像这样的代码会 **产生无限循环/死循环：**

```js
const [count, setCount] = useState(0);
useEffect(() => {
  setCount(count + 1);
});
```

每次渲染结束都会执行 Effect；而更新 state 会触发重新渲染。但是新一轮渲染时又会再次执行 Effect，然后 Effect 再次更新 state……如此周而复始，从而陷入死循环。

Effect 通常应该使组件与 *外部* 系统保持同步。如果没有外部系统，您只想根据其他状态调整某些状态，[您可能不需要一个 Effect。](/learn/you-might-not-need-an-effect)

</Pitfall>

### 第二步：指定 Effect 依赖 {/*step-2-specify-the-effect-dependencies*/}

默认情况下，Effects 在 *每次* 渲染后运行。 **但更多时候，并不需要每次渲染的时候都执行 Effect**

- 有时这会拖慢运行速度。因为与外部系统的同步操作总是有一定时耗，在非必要时可能希望跳过它。例如，没有人会希望每次用键盘打字时都重新连接聊天服务器。
- 有时这会导致程序逻辑错误。例如，组件的淡入动画只需要在第一轮渲染出现时播放一次，而不是每次触发新一轮渲染后都播放。

为了演示这个问题，我们在前面的示例中加入一些 `console.log` 调用和一个更新父组件 state 的文本输入。请注意，输入时会导致 Effect 重新运行：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      console.log('调用 video.play()');
      ref.current.play();
    } else {
      console.log('调用 video.pause()');
      ref.current.pause();
    }
  });

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? '暂停' : '播放'}
      </button>
      <VideoPlayer
        isPlaying={isPlaying}
        src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
      />
    </>
  );
}
```

```css
input, button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

您可以告诉 React 通过将 *依赖项* 数组指定为 `useEffect` 调用的第二个参数来 **跳过不必要的重新运行的 Effect**。首先，在上面的示例中第 14 行添加一个空的 `[]` 数组：

```js {3}
  useEffect(() => {
    // ...
  }, []);
```

您应该会看到一个错误，提示 `React Hook useEffect has a missing dependency: 'isPlaying'`：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      console.log('调用 video.play()');
      ref.current.play();
    } else {
      console.log('调用 video.pause()');
      ref.current.pause();
    }
  }, []); // 这将产生错误

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? '暂停' : '播放'}
      </button>
      <VideoPlayer
        isPlaying={isPlaying}
        src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
      />
    </>
  );
}
```

```css
input, button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

问题在于，Effect 中的代码 *依赖于* `isPlaying` prop 来决定该做什么，但这个依赖没有被显式声明。要修复此问题，请将 `isPlaying` 添加到依赖数组中：

```js {2,7}
  useEffect(() => {
    if (isPlaying) { // isPlaying 在此处使用……
      // ...
    } else {
      // ...
    }
  }, [isPlaying]); // ……所以它必须在此处声明！
```

现在所有依赖项都已声明，所以没有错误了。将 `[isPlaying]` 作为依赖项添加到依赖数组中，告诉 React，如果 `isPlaying` 与上一渲染时相同，则应跳过本次要运行的 Effect。通过这个更改，输入时不会导致 Effect 重新运行，但按播放/暂停会：

<Sandpack>

```js
import { useState, useRef, useEffect } from 'react';

function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      console.log('调用 video.play()');
      ref.current.play();
    } else {
      console.log('调用 video.pause()');
      ref.current.pause();
    }
  }, [isPlaying]);

  return <video ref={ref} src={src} loop playsInline />;
}

export default function App() {
  const [isPlaying, setIsPlaying] = useState(false);
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? '暂停' : '播放'}
      </button>
      <VideoPlayer
        isPlaying={isPlaying}
        src="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4"
      />
    </>
  );
}
```

```css
input, button { display: block; margin-bottom: 20px; }
video { width: 250px; }
```

</Sandpack>

依赖数组可以包含多个依赖项。只有当您指定的 *所有* 依赖项的值与它们在上一渲染时的值完全相同时，React 才会跳过重新运行 Effect。React 使用 [`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is) 来比较依赖项的值。有关详细信息，请参阅 [`useEffect` 参考](/reference/react/useEffect#reference)。

**请注意，不能随意选择依赖项。** 如果你指定的依赖项不能与 Effect 代码所期望的相匹配时，lint 将会报错，这将帮助你找到代码中的问题。如果你不希望某些代码重新运行，[那么你应当 *重新编辑 Effect 代码本身*，使其不需要该依赖项。](/learn/lifecycle-of-reactive-effects#what-to-do-when-you-dont-want-to-re-synchronize)

<Pitfall>

没有依赖数组作为第二个参数，与依赖数组为空数组 `[]` 的行为是不一致的：

```js {3,7,11}
useEffect(() => {
  // 这里的代码会在每次渲染后执行
});

useEffect(() => {
  // 这里的代码只会在组件挂载后执行
}, []);

useEffect(() => {
  // 这里的代码只会在每次渲染后，并且 a 或 b 的值与上次渲染不一致时执行
}, [a, b]);
```

我们将在下一步仔细看看“挂载（mount）”的含义。

</Pitfall>

<DeepDive>

#### 为什么依赖数组中可以省略 ref？ {/*why-was-the-ref-omitted-from-the-dependency-array*/}

下面的 Effect 同时使用 _ref_ 和 `isPlaying`，但只有 `isPlaying` 被声明为依赖项：

```js {9}
function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);
  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }, [isPlaying]);
```

这是因为 `ref` 对象具有 *稳定的标识：* React 保证 [每轮渲染中调用 `useRef` 所产生的引用对象时，获取到的对象引用总是相同的](/reference/react/useRef#returns)。即获取到的对象引用永远不会改变，因此单独包含它不会导致 Effect 重新运行。因此，是否包含它并不重要。包含它也是可以的：

```js {9}
function VideoPlayer({ src, isPlaying }) {
  const ref = useRef(null);
  useEffect(() => {
    if (isPlaying) {
      ref.current.play();
    } else {
      ref.current.pause();
    }
  }, [isPlaying, ref]);
```

由 `useState` 返回的 [`set` 函数](/reference/react/useState#setstate) 也是稳定的（引用不变），因此您通常会看到它们也被省略。如果在忽略某个依赖项时 linter 不会报错，那么这么做就是安全的。

但是，仅在 linter 可以“看到”对象稳定时，忽略稳定依赖项的规则才会起作用。例如，如果 `ref` 是从父组件传递的，您必须在依赖数组中指定它。这样做是合适的，因为您无法知道父组件是否始终传递相同的 ref，或者可能是有条件地传递几个 ref 之一。因此，您的 Effect _将_ 依赖于传递的那个 ref。

</DeepDive>

### 第三步：按需添加清理（cleanup）函数 {/*step-3-add-cleanup-if-needed*/}

考虑一个不同的示例。您正在编写一个 `ChatRoom` 组件，该组件出现时需要连接到聊天服务器。现在为你提供了 `createConnection` API，该 API 返回一个包含 `connect()` 和 `disconnect()` 方法的对象。考虑当组件展示给用户时，应该如何保持连接？

从编写 Effect 逻辑开始：

```js
useEffect(() => {
  const connection = createConnection();
  connection.connect();
});
```

每次重新渲染后连接到聊天室会很慢，因此可以添加依赖数组：

```js {4}
useEffect(() => {
  const connection = createConnection();
  connection.connect();
}, []);
```

**Effect 中的代码没有使用任何 props 或 state，因此您的依赖数组是 `[]`（空）。这告诉 React 只在组件“挂载”时运行此代码，即第一次出现在屏幕上时。**

试试运行下面的代码：

<Sandpack>

```js
import { useEffect } from 'react';
import { createConnection } from './chat.js';

export default function ChatRoom() {
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
  }, []);
  return <h1>欢迎来到聊天室！</h1>;
}
```

```js src/chat.js
export function createConnection() {
  // 真实的实现会将其连接到服务器，此处代码只是示例
  return {
    connect() {
      console.log('✅ 连接中……');
    },
    disconnect() {
      console.log('❌ 连接断开。');
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
```

</Sandpack>

这个 Effect 仅在挂载时运行，因此您可能期望在控制台中只打印一次 `"✅ Connecting..."`。**然而，如果您检查控制台，您会发现 `"✅ Connecting..."` 被打印了两次。这是为什么呢？**

想象一下 `ChatRoom` 组件是一个大规模的 App 中许多界面中的一部分。用户切换到含有 `ChatRoom` 组件的页面上时，该组件被挂载，并调用 `connection.connect()` 方法连接服务器。然后想象用户此时突然导航到另一个页面，例如，设置页面。这时，`ChatRoom` 组件被卸载了。最后，用户点击返回，`ChatRoom` 再次挂载。这将建立第二个连接——但第一个连接从未被销毁！随着用户在应用中导航，连接将不断堆积。

如果不进行大量的手动测试，这样的错误很容易被遗漏。为了帮助你快速发现它们，在开发环境中，React 会在初始挂载组件后，立即再挂载一次。

看到 `"✅ Connecting..."` 日志被打印两次，可以帮助您注意到真正的问题：在代码中，组件被卸载时没有关闭连接。

要解决此问题，可以在 Effect 中返回一个 *清理函数（cleanup）*：

```js {4-6}
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []);
```

每次重新执行 Effect 之前，React 都会调用清理函数；组件被卸载时，也会调用清理函数。让我们看看执行清理函数会做些什么：

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

export default function ChatRoom() {
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => connection.disconnect();
  }, []);
  return <h1>欢迎来到聊天室！</h1>;
}
```

```js src/chat.js
export function createConnection() {
  // 真实的实现会将其连接到服务器，此处代码只是示例
  return {
    connect() {
      console.log('✅ 连接中……');
    },
    disconnect() {
      console.log('❌ 连接断开。');
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
```

</Sandpack>

现在在开发模式下，控制台会打印三条记录：

1. `"✅ 连接中..."`
2. `"❌ 连接断开。"`
3. `"✅ 连接中..."`

**在开发环境下，出现这样的结果才是符合预期的。** 可以确保在 React 中离开和返回页面时不会导致代码运行出现问题。上面的代码中规定了挂载组件时连接服务器、卸载组件时断连服务器。所以断开、连接再重新连接是符合预期的行为。当为 Effect 正确实现清理函数时，无论 Effect 执行一次，还是执行、清理、再执行，用户都不会感受到明显的差异。所以，在开发环境下，出现额外的连接、断连时，这是 React 正在调试你的代码。这是很正常的现象，不要试图消除它！

**在生产环境下，`“✅ 连接中……”` 只会被打印一次。** 也就是说仅在开发环境下才会重复挂载组件，以帮助你找到需要清理的 Effect。您可以关闭 [严格模式](/reference/react/StrictMode) 以选择退出开发行为，但我们建议您保持开启。这让您发现许多像上面那样的错误。

## 如何处理在开发环境中 Effect 执行两次？ {/*how-to-handle-the-effect-firing-twice-in-development*/}

在开发环境中，React 有意重复挂载你的组件，以发现像上一个示例中的错误。**正确的态度是“如何修复 Effect 以便它在重复挂载后能正常工作”，而不是“如何只运行一次 Effect”。**

通常的解决办法是实现清理函数。清理函数应该停止或撤销 Effect 正在做的事情。简单来说，用户不应该感受到 Effect 只执行一次（如在生产环境中）和执行 _挂载 → 清理 → 挂载_ 过程（如在开发环境中）之间的差异。

下面提供一些常用的 Effect 应用模式。

<Pitfall>

#### 不要使用 refs 来防止触发 Effects {/*dont-use-refs-to-prevent-effects-from-firing*/}

在开发过程中，防止 Effects 触发两次的一个常见陷阱是使用 `ref` 来防止 Effect 运行多次。例如，你使用 `useRef` “修复”上面的错误：

```js {1,3-4}
  const connectionRef = useRef(null);
  useEffect(() => {
    // 🚩 这并不能修复这个错误！！！
    if (!connectionRef.current) {
      connectionRef.current = createConnection();
      connectionRef.current.connect();
    }
  }, []);
```

这使得你在开发过程中确实只看到了一次 `"✅ Connecting..."`，但其实并没有修复这个错误。

当用户导航离开时，连接仍然没有关闭，当他们返回时，将创建一个新连接。随着用户在应用中导航，连接将不断堆积，就像“修复”之前一样。

要修复错误，仅仅使 Effect 运行一次是不够的。Effect 需要在重新挂载后正常工作，这意味着需要像上面的解决方案一样清理连接。

请参阅下面的示例，了解如何处理常见模式。

</Pitfall>

### 控制非 React 组件 {/*controlling-non-react-widgets*/}

有时需要添加不是使用 React 编写的 UI 小部件。例如，假设您要在页面上添加一个地图组件。它有一个 `setZoomLevel()` 方法，您希望将缩放级别与 React 代码中的 `zoomLevel` 状态变量保持同步。Effect 看起来应该与下面类似：

```js
useEffect(() => {
  const map = mapRef.current;
  map.setZoomLevel(zoomLevel);
}, [zoomLevel]);
```

请注意，在这种情况下不需要清理。在开发中，React 将调用 Effect 两次，但这不是问题，这两次挂载时依赖项 `setZoomLevel` 都是相同的，所以即使执行两次 Effect，也不会造成任何影响。它可能会稍慢，但这没关系，因为在生产中不会进行不必要地重新挂载。

某些 API 可能不允许您连续两次调用它们。例如，内置的 [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement) 元素的 [`showModal`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal) 方法如果您调用两次会抛出异常。此时实现清理函数并使其关闭对话框：

```js {4}
useEffect(() => {
  const dialog = dialogRef.current;
  dialog.showModal();
  return () => dialog.close();
}, []);
```

在开发中，您的 Effect 将调用 `showModal()`，然后立即 `close()`，然后再次 `showModal()`。这与调用只一次 `showModal()` 的效果相同。也正如在生产环境中看到的那样。

### 订阅事件 {/*subscribing-to-events*/}

如果 Effect 订阅了某些事件，清理函数应该退订这些事件：

```js {6}
useEffect(() => {
  function handleScroll(e) {
    console.log(window.scrollX, window.scrollY);
  }
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

在开发中，您的 Effect 将调用 `addEventListener()`，然后立即 `removeEventListener()`，然后再次调用 `addEventListener()`，并使用相同的处理程序。因此，始终只有一个活动订阅。这具有与在生产中仅调用一次 `addEventListener()` 相同的用户可见行为。

### 触发动画 {/*triggering-animations*/}

如果 Effect 对某些内容加入了动画，清理函数应将动画重置：

```js {4-6}
useEffect(() => {
  const node = ref.current;
  node.style.opacity = 1; // 触发动画
  return () => {
    node.style.opacity = 0; // 重置为初始值
  };
}, []);
```

在开发环境中，透明度由 `1` 变为 `0`，再变为 `1`。这与在生产环境中，直接将其设置为 `1` 具有相同的感知效果，如果你使用支持过渡的第三方动画库，你的清理函数应将时间轴重置为其初始状态。

### 获取数据 {/*fetching-data*/}

如果 Effect 将会获取数据，清理函数应该要么 [中止该数据的获取操作](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) 或忽略其结果：

```js {2,6,13-15}
useEffect(() => {
  let ignore = false;

  async function startFetching() {
    const json = await fetchTodos(userId);
    if (!ignore) {
      setTodos(json);
    }
  }

  startFetching();

  return () => {
    ignore = true;
  };
}, [userId]);
```

我们无法撤消已经发生的网络请求，但是清理函数应当确保获取数据的过程以及获取到的结果不会继续影响程序运行。如果 `userId` 从 `'Alice'` 变为 `'Bob'`，那么请确保 `Alice` 响应数据被忽略，即使它在 `'Bob'` 之后到达。

**在开发环境中，浏览器调试工具的“网络”选项卡中会出现两个 fetch 请求。** 这是正常的。使用上述方法，第一个 Effect 将立即被清理，因此它的 `ignore` 变量将被设置为 `true`。因此，即使有额外的请求，由于 `if (!ignore)` 检查，它也不会影响程序状态。

**在生产环境中，只会有一个请求。** 如果开发中的第二个请求让您感到困扰，最好的方法是使用去重请求并在组件之间缓存响应的解决方案：

```js
function TodoList() {
  const todos = useSomeDataLibrary(`/api/user/${userId}/todos`);
  // ...
```

这不仅会改善开发体验，还会让您的应用程序感觉更快。例如，用户按下返回按钮时，不必等待某些数据再次加载，因为它将被缓存。您可以自己构建这样的缓存，也可以使用很多在 Effect 中手动加载数据的替代方法。

<DeepDive>

#### Effect 中有哪些好的数据获取替代方案？ {/*what-are-good-alternatives-to-data-fetching-in-effects*/}

在 Effects 中编写 `fetch` 请求是一种 [流行的数据获取方式](https://www.robinwieruch.de/react-hooks-fetch-data/)，特别是在客户端应用中。然而，这是一种非常手动的方法，并且有显著的缺点：

- **Effect 不能在服务端执行。** 这这意味着服务器最初传递的 HTML 不会包含任何数据。客户端的浏览器必须下载所有 JavaScript 脚本来渲染应用程序，然后才能加载数据——这并不高效。
- **直接在 Effect 中获取数据容易产生网络瀑布（network waterfall）。** 首先渲染了父组件，它会获取一些数据并进行渲染；然后渲染子组件，接着子组件开始获取它们的数据。如果网络速度不够快，这种方式比同时获取所有数据要慢得多。
- **直接在 Effect 中获取数据通常意味着无法预加载或缓存数据。** 例如，在组件卸载后然后再次挂载，那么它必须再次获取数据。
- **这不是很符合人机交互原则（不方便）。** 如果你不想出现像 [条件竞争](https://maxrozen.com/race-conditions-fetching-data-react-with-useeffect) 之类的问题，那么你需要编写更多的样板代码。

以上所列出来的缺点并不是 React 特有的。在任何框架或者库上的组件挂载过程中获取数据，都会遇到这些问题。与路由一样，要做好数据获取并非易事，因此我们推荐以下方法：

- **如果您使用 [框架](/learn/start-a-new-react-project#production-grade-react-frameworks)，请使用其内置的数据获取机制。** 现代 React 框架内置了高效的数据获取机制，避免了上述缺点。
- **否则，考虑使用或构建客户端缓存。** 流行的开源解决方案包括 [React Query](https://tanstack.com/query/latest)、[useSWR](https://swr.vercel.app/) 和 [React Router 6.4+](https://beta.reactrouter.com/en/main/start/overview)。您也可以构建自己的解决方案，在这种情况下，你可以在幕后使用 Effect，但是请注意添加用于删除重复请求、缓存响应和避免网络瀑布的逻辑（通过预加载数据或将数据需求提升到路由）。

如果这些方法都不适合你，你可以继续直接在 Effect 中获取数据。

</DeepDive>

### 发送分析报告 {/*sending-analytics*/}

考虑在访问页面时发送日志分析：

```js
useEffect(() => {
  logVisit(url); // 发送 POST 请求
}, [url]);
```

In development, `logVisit` will be called twice for every URL, so you might be tempted to try to fix that. **We recommend keeping this code as is.** Like with earlier examples, there is no *user-visible* behavior difference between running it once and running it twice. From a practical point of view, `logVisit` should not do anything in development because you don't want the logs from the development machines to skew the production metrics. Your component remounts every time you save its file, so it logs extra visits in development anyway.

**In production, there will be no duplicate visit logs.**

To debug the analytics events you're sending, you can deploy your app to a staging environment (which runs in production mode) or temporarily opt out of [Strict Mode](/reference/react/StrictMode) and its development-only remounting checks. You may also send analytics from the route change event handlers instead of Effects. For more precise analytics, [intersection observers](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) can help track which components are in the viewport and how long they remain visible.

### Not an Effect: Initializing the application {/*not-an-effect-initializing-the-application*/}

Some logic should only run once when the application starts. You can put it outside your components:

```js {2-3}
if (typeof window !== 'undefined') { // Check if we're running in the browser.
  checkAuthToken();
  loadDataFromLocalStorage();
}

function App() {
  // ...
}
```

This guarantees that such logic only runs once after the browser loads the page.

### Not an Effect: Buying a product {/*not-an-effect-buying-a-product*/}

Sometimes, even if you write a cleanup function, there's no way to prevent user-visible consequences of running the Effect twice. For example, maybe your Effect sends a POST request like buying a product:

```js {2-3}
useEffect(() => {
  // 🔴 Wrong: This Effect fires twice in development, exposing a problem in the code.
  fetch('/api/buy', { method: 'POST' });
}, []);
```

You wouldn't want to buy the product twice. However, this is also why you shouldn't put this logic in an Effect. What if the user goes to another page and then presses Back? Your Effect would run again. You don't want to buy the product when the user *visits* a page; you want to buy it when the user *clicks* the Buy button.

Buying is not caused by rendering; it's caused by a specific interaction. It should run only when the user presses the button. **Delete the Effect and move your `/api/buy` request into the Buy button event handler:**

```js {2-3}
  function handleClick() {
    // ✅ Buying is an event because it is caused by a particular interaction.
    fetch('/api/buy', { method: 'POST' });
  }
```

**This illustrates that if remounting breaks the logic of your application, this usually uncovers existing bugs.** From a user's perspective, visiting a page shouldn't be different from visiting it, clicking a link, then pressing Back to view the page again. React verifies that your components abide by this principle by remounting them once in development.

## Putting it all together {/*putting-it-all-together*/}

This playground can help you "get a feel" for how Effects work in practice.

This example uses [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout) to schedule a console log with the input text to appear three seconds after the Effect runs. The cleanup function cancels the pending timeout. Start by pressing "Mount the component":

<Sandpack>

```js
import { useState, useEffect } from 'react';

function Playground() {
  const [text, setText] = useState('a');

  useEffect(() => {
    function onTimeout() {
      console.log('⏰ ' + text);
    }

    console.log('🔵 Schedule "' + text + '" log');
    const timeoutId = setTimeout(onTimeout, 3000);

    return () => {
      console.log('🟡 Cancel "' + text + '" log');
      clearTimeout(timeoutId);
    };
  }, [text]);

  return (
    <>
      <label>
        What to log:{' '}
        <input
          value={text}
          onChange={e => setText(e.target.value)}
        />
      </label>
      <h1>{text}</h1>
    </>
  );
}

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Unmount' : 'Mount'} the component
      </button>
      {show && <hr />}
      {show && <Playground />}
    </>
  );
}
```

</Sandpack>

You will see three logs at first: `Schedule "a" log`, `Cancel "a" log`, and `Schedule "a" log` again. Three second later there will also be a log saying `a`. As you learned earlier, the extra schedule/cancel pair is because React remounts the component once in development to verify that you've implemented cleanup well.

Now edit the input to say `abc`. If you do it fast enough, you'll see `Schedule "ab" log` immediately followed by `Cancel "ab" log` and `Schedule "abc" log`. **React always cleans up the previous render's Effect before the next render's Effect.** This is why even if you type into the input fast, there is at most one timeout scheduled at a time. Edit the input a few times and watch the console to get a feel for how Effects get cleaned up.

Type something into the input and then immediately press "Unmount the component". Notice how unmounting cleans up the last render's Effect. Here, it clears the last timeout before it has a chance to fire.

Finally, edit the component above and comment out the cleanup function so that the timeouts don't get cancelled. Try typing `abcde` fast. What do you expect to happen in three seconds? Will `console.log(text)` inside the timeout print the *latest* `text` and produce five `abcde` logs? Give it a try to check your intuition!

Three seconds later, you should see a sequence of logs (`a`, `ab`, `abc`, `abcd`, and `abcde`) rather than five `abcde` logs. **Each Effect "captures" the `text` value from its corresponding render.**  It doesn't matter that the `text` state changed: an Effect from the render with `text = 'ab'` will always see `'ab'`. In other words, Effects from each render are isolated from each other. If you're curious how this works, you can read about [closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures).

<DeepDive>

#### Each render has its own Effects {/*each-render-has-its-own-effects*/}

You can think of `useEffect` as "attaching" a piece of behavior to the render output. Consider this Effect:

```js
export default function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return <h1>Welcome to {roomId}!</h1>;
}
```

Let's see what exactly happens as the user navigates around the app.

#### Initial render {/*initial-render*/}

The user visits `<ChatRoom roomId="general" />`. Let's [mentally substitute](/learn/state-as-a-snapshot#rendering-takes-a-snapshot-in-time) `roomId` with `'general'`:

```js
  // JSX for the first render (roomId = "general")
  return <h1>Welcome to general!</h1>;
```

**The Effect is *also* a part of the rendering output.** The first render's Effect becomes:

```js
  // Effect for the first render (roomId = "general")
  () => {
    const connection = createConnection('general');
    connection.connect();
    return () => connection.disconnect();
  },
  // Dependencies for the first render (roomId = "general")
  ['general']
```

React runs this Effect, which connects to the `'general'` chat room.

#### Re-render with same dependencies {/*re-render-with-same-dependencies*/}

Let's say `<ChatRoom roomId="general" />` re-renders. The JSX output is the same:

```js
  // JSX for the second render (roomId = "general")
  return <h1>Welcome to general!</h1>;
```

React sees that the rendering output has not changed, so it doesn't update the DOM.

The Effect from the second render looks like this:

```js
  // Effect for the second render (roomId = "general")
  () => {
    const connection = createConnection('general');
    connection.connect();
    return () => connection.disconnect();
  },
  // Dependencies for the second render (roomId = "general")
  ['general']
```

React compares `['general']` from the second render with `['general']` from the first render. **Because all dependencies are the same, React *ignores* the Effect from the second render.** It never gets called.

#### Re-render with different dependencies {/*re-render-with-different-dependencies*/}

Then, the user visits `<ChatRoom roomId="travel" />`. This time, the component returns different JSX:

```js
  // JSX for the third render (roomId = "travel")
  return <h1>Welcome to travel!</h1>;
```

React updates the DOM to change `"Welcome to general"` into `"Welcome to travel"`.

The Effect from the third render looks like this:

```js
  // Effect for the third render (roomId = "travel")
  () => {
    const connection = createConnection('travel');
    connection.connect();
    return () => connection.disconnect();
  },
  // Dependencies for the third render (roomId = "travel")
  ['travel']
```

React compares `['travel']` from the third render with `['general']` from the second render. One dependency is different: `Object.is('travel', 'general')` is `false`. The Effect can't be skipped.

**Before React can apply the Effect from the third render, it needs to clean up the last Effect that _did_ run.** The second render's Effect was skipped, so React needs to clean up the first render's Effect. If you scroll up to the first render, you'll see that its cleanup calls `disconnect()` on the connection that was created with `createConnection('general')`. This disconnects the app from the `'general'` chat room.

After that, React runs the third render's Effect. It connects to the `'travel'` chat room.

#### Unmount {/*unmount*/}

Finally, let's say the user navigates away, and the `ChatRoom` component unmounts. React runs the last Effect's cleanup function. The last Effect was from the third render. The third render's cleanup destroys the `createConnection('travel')` connection. So the app disconnects from the `'travel'` room.

#### Development-only behaviors {/*development-only-behaviors*/}

When [Strict Mode](/reference/react/StrictMode) is on, React remounts every component once after mount (state and DOM are preserved). This [helps you find Effects that need cleanup](#step-3-add-cleanup-if-needed) and exposes bugs like race conditions early. Additionally, React will remount the Effects whenever you save a file in development. Both of these behaviors are development-only.

</DeepDive>

<Recap>

- Unlike events, Effects are caused by rendering itself rather than a particular interaction.
- Effects let you synchronize a component with some external system (third-party API, network, etc).
- By default, Effects run after every render (including the initial one).
- React will skip the Effect if all of its dependencies have the same values as during the last render.
- You can't "choose" your dependencies. They are determined by the code inside the Effect.
- Empty dependency array (`[]`) corresponds to the component "mounting", i.e. being added to the screen.
- In Strict Mode, React mounts components twice (in development only!) to stress-test your Effects.
- If your Effect breaks because of remounting, you need to implement a cleanup function.
- React will call your cleanup function before the Effect runs next time, and during the unmount.

</Recap>

<Challenges>

#### Focus a field on mount {/*focus-a-field-on-mount*/}

In this example, the form renders a `<MyInput />` component.

Use the input's [`focus()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus) method to make `MyInput` automatically focus when it appears on the screen. There is already a commented out implementation, but it doesn't quite work. Figure out why it doesn't work, and fix it. (If you're familiar with the `autoFocus` attribute, pretend that it does not exist: we are reimplementing the same functionality from scratch.)

<Sandpack>

```js src/MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ value, onChange }) {
  const ref = useRef(null);

  // TODO: This doesn't quite work. Fix it.
  // ref.current.focus()    

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js src/App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [name, setName] = useState('Taylor');
  const [upper, setUpper] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your name:
            <MyInput
              value={name}
              onChange={e => setName(e.target.value)}
            />
          </label>
          <label>
            <input
              type="checkbox"
              checked={upper}
              onChange={e => setUpper(e.target.checked)}
            />
            Make it uppercase
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>


To verify that your solution works, press "Show form" and verify that the input receives focus (becomes highlighted and the cursor is placed inside). Press "Hide form" and "Show form" again. Verify the input is highlighted again.

`MyInput` should only focus _on mount_ rather than after every render. To verify that the behavior is right, press "Show form" and then repeatedly press the "Make it uppercase" checkbox. Clicking the checkbox should _not_ focus the input above it.

<Solution>

Calling `ref.current.focus()` during render is wrong because it is a *side effect*. Side effects should either be placed inside an event handler or be declared with `useEffect`. In this case, the side effect is _caused_ by the component appearing rather than by any specific interaction, so it makes sense to put it in an Effect.

To fix the mistake, wrap the `ref.current.focus()` call into an Effect declaration. Then, to ensure that this Effect runs only on mount rather than after every render, add the empty `[]` dependencies to it.

<Sandpack>

```js src/MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ value, onChange }) {
  const ref = useRef(null);

  useEffect(() => {
    ref.current.focus();
  }, []);

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js src/App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [name, setName] = useState('Taylor');
  const [upper, setUpper] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your name:
            <MyInput
              value={name}
              onChange={e => setName(e.target.value)}
            />
          </label>
          <label>
            <input
              type="checkbox"
              checked={upper}
              onChange={e => setUpper(e.target.checked)}
            />
            Make it uppercase
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

</Solution>

#### Focus a field conditionally {/*focus-a-field-conditionally*/}

This form renders two `<MyInput />` components.

Press "Show form" and notice that the second field automatically gets focused. This is because both of the `<MyInput />` components try to focus the field inside. When you call `focus()` for two input fields in a row, the last one always "wins".

Let's say you want to focus the first field. The first `MyInput` component now receives a boolean `shouldFocus` prop set to `true`. Change the logic so that `focus()` is only called if the `shouldFocus` prop received by `MyInput` is `true`.

<Sandpack>

```js src/MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ shouldFocus, value, onChange }) {
  const ref = useRef(null);

  // TODO: call focus() only if shouldFocus is true.
  useEffect(() => {
    ref.current.focus();
  }, []);

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js src/App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  const [upper, setUpper] = useState(false);
  const name = firstName + ' ' + lastName;
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your first name:
            <MyInput
              value={firstName}
              onChange={e => setFirstName(e.target.value)}
              shouldFocus={true}
            />
          </label>
          <label>
            Enter your last name:
            <MyInput
              value={lastName}
              onChange={e => setLastName(e.target.value)}
              shouldFocus={false}
            />
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

To verify your solution, press "Show form" and "Hide form" repeatedly. When the form appears, only the *first* input should get focused. This is because the parent component renders the first input with `shouldFocus={true}` and the second input with `shouldFocus={false}`. Also check that both inputs still work and you can type into both of them.

<Hint>

You can't declare an Effect conditionally, but your Effect can include conditional logic.

</Hint>

<Solution>

Put the conditional logic inside the Effect. You will need to specify `shouldFocus` as a dependency because you are using it inside the Effect. (This means that if some input's `shouldFocus` changes from `false` to `true`, it will focus after mount.)

<Sandpack>

```js src/MyInput.js active
import { useEffect, useRef } from 'react';

export default function MyInput({ shouldFocus, value, onChange }) {
  const ref = useRef(null);

  useEffect(() => {
    if (shouldFocus) {
      ref.current.focus();
    }
  }, [shouldFocus]);

  return (
    <input
      ref={ref}
      value={value}
      onChange={onChange}
    />
  );
}
```

```js src/App.js hidden
import { useState } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const [show, setShow] = useState(false);
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  const [upper, setUpper] = useState(false);
  const name = firstName + ' ' + lastName;
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} form</button>
      <br />
      <hr />
      {show && (
        <>
          <label>
            Enter your first name:
            <MyInput
              value={firstName}
              onChange={e => setFirstName(e.target.value)}
              shouldFocus={true}
            />
          </label>
          <label>
            Enter your last name:
            <MyInput
              value={lastName}
              onChange={e => setLastName(e.target.value)}
              shouldFocus={false}
            />
          </label>
          <p>Hello, <b>{upper ? name.toUpperCase() : name}</b></p>
        </>
      )}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

</Solution>

#### Fix an interval that fires twice {/*fix-an-interval-that-fires-twice*/}

This `Counter` component displays a counter that should increment every second. On mount, it calls [`setInterval`.](https://developer.mozilla.org/en-US/docs/Web/API/setInterval) This causes `onTick` to run every second. The `onTick` function increments the counter.

However, instead of incrementing once per second, it increments twice. Why is that? Find the cause of the bug and fix it.

<Hint>

Keep in mind that `setInterval` returns an interval ID, which you can pass to [`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval) to stop the interval.

</Hint>

<Sandpack>

```js src/Counter.js active
import { useState, useEffect } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    function onTick() {
      setCount(c => c + 1);
    }

    setInterval(onTick, 1000);
  }, []);

  return <h1>{count}</h1>;
}
```

```js src/App.js hidden
import { useState } from 'react';
import Counter from './Counter.js';

export default function Form() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} counter</button>
      <br />
      <hr />
      {show && <Counter />}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

<Solution>

When [Strict Mode](/reference/react/StrictMode) is on (like in the sandboxes on this site), React remounts each component once in development. This causes the interval to be set up twice, and this is why each second the counter increments twice.

However, React's behavior is not the *cause* of the bug: the bug already exists in the code. React's behavior makes the bug more noticeable. The real cause is that this Effect starts a process but doesn't provide a way to clean it up.

To fix this code, save the interval ID returned by `setInterval`, and implement a cleanup function with [`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval):

<Sandpack>

```js src/Counter.js active
import { useState, useEffect } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    function onTick() {
      setCount(c => c + 1);
    }

    const intervalId = setInterval(onTick, 1000);
    return () => clearInterval(intervalId);
  }, []);

  return <h1>{count}</h1>;
}
```

```js src/App.js hidden
import { useState } from 'react';
import Counter from './Counter.js';

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(s => !s)}>{show ? 'Hide' : 'Show'} counter</button>
      <br />
      <hr />
      {show && <Counter />}
    </>
  );
}
```

```css
label {
  display: block;
  margin-top: 20px;
  margin-bottom: 20px;
}

body {
  min-height: 150px;
}
```

</Sandpack>

In development, React will still remount your component once to verify that you've implemented cleanup well. So there will be a `setInterval` call, immediately followed by `clearInterval`, and `setInterval` again. In production, there will be only one `setInterval` call. The user-visible behavior in both cases is the same: the counter increments once per second.

</Solution>

#### Fix fetching inside an Effect {/*fix-fetching-inside-an-effect*/}

This component shows the biography for the selected person. It loads the biography by calling an asynchronous function `fetchBio(person)` on mount and whenever `person` changes. That asynchronous function returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) which eventually resolves to a string. When fetching is done, it calls `setBio` to display that string under the select box.

<Sandpack>

```js src/App.js
import { useState, useEffect } from 'react';
import { fetchBio } from './api.js';

export default function Page() {
  const [person, setPerson] = useState('Alice');
  const [bio, setBio] = useState(null);

  useEffect(() => {
    setBio(null);
    fetchBio(person).then(result => {
      setBio(result);
    });
  }, [person]);

  return (
    <>
      <select value={person} onChange={e => {
        setPerson(e.target.value);
      }}>
        <option value="Alice">Alice</option>
        <option value="Bob">Bob</option>
        <option value="Taylor">Taylor</option>
      </select>
      <hr />
      <p><i>{bio ?? 'Loading...'}</i></p>
    </>
  );
}
```

```js src/api.js hidden
export async function fetchBio(person) {
  const delay = person === 'Bob' ? 2000 : 200;
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('This is ' + person + '’s bio.');
    }, delay);
  })
}

```

</Sandpack>


There is a bug in this code. Start by selecting "Alice". Then select "Bob" and then immediately after that select "Taylor". If you do this fast enough, you will notice that bug: Taylor is selected, but the paragraph below says "This is Bob's bio."

Why does this happen? Fix the bug inside this Effect.

<Hint>

If an Effect fetches something asynchronously, it usually needs cleanup.

</Hint>

<Solution>

To trigger the bug, things need to happen in this order:

- Selecting `'Bob'` triggers `fetchBio('Bob')`
- Selecting `'Taylor'` triggers `fetchBio('Taylor')`
- **Fetching `'Taylor'` completes *before* fetching `'Bob'`**
- The Effect from the `'Taylor'` render calls `setBio('This is Taylor’s bio')`
- Fetching `'Bob'` completes
- The Effect from the `'Bob'` render calls `setBio('This is Bob’s bio')`

This is why you see Bob's bio even though Taylor is selected. Bugs like this are called [race conditions](https://en.wikipedia.org/wiki/Race_condition) because two asynchronous operations are "racing" with each other, and they might arrive in an unexpected order.

To fix this race condition, add a cleanup function:

<Sandpack>

```js src/App.js
import { useState, useEffect } from 'react';
import { fetchBio } from './api.js';

export default function Page() {
  const [person, setPerson] = useState('Alice');
  const [bio, setBio] = useState(null);
  useEffect(() => {
    let ignore = false;
    setBio(null);
    fetchBio(person).then(result => {
      if (!ignore) {
        setBio(result);
      }
    });
    return () => {
      ignore = true;
    }
  }, [person]);

  return (
    <>
      <select value={person} onChange={e => {
        setPerson(e.target.value);
      }}>
        <option value="Alice">Alice</option>
        <option value="Bob">Bob</option>
        <option value="Taylor">Taylor</option>
      </select>
      <hr />
      <p><i>{bio ?? 'Loading...'}</i></p>
    </>
  );
}
```

```js src/api.js hidden
export async function fetchBio(person) {
  const delay = person === 'Bob' ? 2000 : 200;
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('This is ' + person + '’s bio.');
    }, delay);
  })
}

```

</Sandpack>

Each render's Effect has its own `ignore` variable. Initially, the `ignore` variable is set to `false`. However, if an Effect gets cleaned up (such as when you select a different person), its `ignore` variable becomes `true`. So now it doesn't matter in which order the requests complete. Only the last person's Effect will have `ignore` set to `false`, so it will call `setBio(result)`. Past Effects have been cleaned up, so the `if (!ignore)` check will prevent them from calling `setBio`:

- Selecting `'Bob'` triggers `fetchBio('Bob')`
- Selecting `'Taylor'` triggers `fetchBio('Taylor')` **and cleans up the previous (Bob's) Effect**
- Fetching `'Taylor'` completes *before* fetching `'Bob'`
- The Effect from the `'Taylor'` render calls `setBio('This is Taylor’s bio')`
- Fetching `'Bob'` completes
- The Effect from the `'Bob'` render **does not do anything because its `ignore` flag was set to `true`**

In addition to ignoring the result of an outdated API call, you can also use [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) to cancel the requests that are no longer needed. However, by itself this is not enough to protect against race conditions. More asynchronous steps could be chained after the fetch, so using an explicit flag like `ignore` is the most reliable way to fix this type of problem.

</Solution>

</Challenges>

```
如何处理开发中 Effect 两次触发的问题？

在开发环境中，React 有意重复挂载你的组件，以发现像上一个示例中的错误。**正确的态度是“如何修复 Effect 以便它在重复挂载后能正常工作”，而不是“如何只运行一次 Effect”。**

通常的解决办法是实现清理函数。清理函数应该停止或撤销 Effect 正在做的事情。简单来说，用户不应该感受到 Effect 只执行一次（如在生产环境中）和执行 _挂载 → 清理 → 挂载_ 过程（如在开发环境中）之间的差异。

下面提供一些常用的 Effect 应用模式。

不要使用 refs 来防止触发 Effects

在开发过程中，防止 Effects 触发两次的一个常见陷阱是使用 `ref` 来防止 Effect 运行多次。例如，你使用 `useRef` “修复”上面的错误：

这使得你在开发过程中确实只看到了一次 `"✅ Connecting..."`，但其实并没有修复这个错误。

当用户导航离开时，连接仍然没有关闭，当他们返回时，将创建一个新连接。随着用户在应用中导航，连接将不断堆积，就像“修复”之前一样。

要修复错误，仅仅使 Effect 运行一次是不够的。Effect 需要在重新挂载后正常工作，这意味着需要像上面的解决方案一样清理连接。

请参阅下面的示例，了解如何处理常见模式。

控制非 React 小部件

有时需要添加不是使用 React 编写的 UI 小部件。例如，假设您要在页面上添加一个地图组件。它有一个 `setZoomLevel()` 方法，您希望将缩放级别与 React 代码中的 `zoomLevel` 状态变量保持同步。Effect 看起来应该与下面类似：

请注意，在这种情况下不需要清理。在开发中，React 将调用 Effect 两次，但这不是问题，这两次挂载时依赖项 `setZoomLevel` 都是相同的，所以即使执行两次 Effect，也不会造成任何影响。它可能会稍慢，但这没关系，因为在生产中不会进行不必要地重新挂载。

某些 API 可能不允许您连续两次调用它们。例如，内置的 [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement) 元素的 [`showModal`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal) 方法如果您调用两次会抛出异常。此时实现清理函数并使其关闭对话框：

在开发中，您的 Effect 将调用 `showModal()`，然后立即 `close()`，然后再次 `showModal()`。这与调用只一次 `showModal()` 的效果相同。也正如在生产环境中看到的那样。

订阅事件

如果 Effect 订阅了某些事件，清理函数应该退订这些事件：

在开发中，您的 Effect 将调用 `addEventListener()`，然后立即 `removeEventListener()`，然后再次调用 `addEventListener()`，并使用相同的处理程序。因此，始终只有一个活动订阅。这具有与在生产中仅调用一次 `addEventListener()` 相同的用户可见行为。

触发动画

如果 Effect 对某些内容加入了动画，清理函数应将动画重置：

在开发环境中，透明度由 `1` 变为 `0`，再变为 `1`。这与在生产环境中，直接将其设置为 `1` 具有相同的感知效果，如果你使用支持过渡的第三方动画库，你的清理函数应将时间轴重置为其初始状态。

如果 Effect 将会获取数据，清理函数应该要么 [中止该数据的获取操作](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) 或忽略其结果：

我们无法撤消已经发生的网络请求，但是清理函数应当确保获取数据的过程以及获取到的结果不会继续影响程序运行。如果 `userId` 从 `'Alice'` 变为 `'Bob'`，那么请确保 `Alice` 响应数据被忽略，即使它在 `'Bob'` 之后到达。

**在开发环境中，浏览器调试工具的“网络”选项卡中会出现两个 fetch 请求。** 这是正常的。使用上述方法，第一个 Effect 将立即被清理，因此它的 `ignore` 变量将被设置为 `true`。因此，即使有额外的请求，由于 `if (!ignore)` 检查，它也不会影响程序状态。

**在生产环境中，只会有一个请求。** 如果开发中的第二个请求让您感到困扰，最好的方法是使用去重请求并在组件之间缓存响应的解决方案：

这不仅会改善开发体验，还会让您的应用程序感觉更快。例如，用户按下返回按钮时，不必等待某些数据再次加载，因为它将被缓存。您可以自己构建这样的缓存，也可以使用很多在 Effect 中手动加载数据的替代方法。

在 Effects 中获取数据的好替代方案是什么？

在 Effects 中编写 `fetch` 请求是一种 [流行的数据获取方式](https://www.robinwieruch.de/react-hooks-fetch-data/)，特别是在客户端应用中。然而，这是一种非常手动的方法，并且有显著的缺点：

- **Effect 不能在服务端执行。** 这这意味着服务器最初传递的 HTML 不会包含任何数据。客户端的浏览器必须下载所有 JavaScript 脚本来渲染应用程序，然后才能加载数据——这并不高效。
- **直接在 Effect 中获取数据容易产生网络瀑布（network waterfall）。** 首先渲染了父组件，它会获取一些数据并进行渲染；然后渲染子组件，接着子组件开始获取它们的数据。如果网络速度不够快，这种方式比同时获取所有数据要慢得多。
- **直接在 Effect 中获取数据通常意味着无法预加载或缓存数据。** 例如，在组件卸载后然后再次挂载，那么它必须再次获取数据。
- **这不是很符合人机交互原则（不方便）。** 如果你不想出现像 [条件竞争](https://maxrozen.com/race-conditions-fetching-data-react-with-useeffect) 之类的问题，那么你需要编写更多的样板代码。

以上所列出来的缺点并不是 React 特有的。在任何框架或者库上的组件挂载过程中获取数据，都会遇到这些问题。与路由一样，要做好数据获取并非易事，因此我们推荐以下方法：

- **如果您使用 [框架](/learn/start-a-new-react-project#production-grade-react-frameworks)，请使用其内置的数据获取机制。** 现代 React 框架内置了高效的数据获取机制，避免了上述缺点。
- **否则，考虑使用或构建客户端缓存。** 流行的开源解决方案包括 [React Query](https://tanstack.com/query/latest)、[useSWR](https://swr.vercel.app/) 和 [React Router 6.4+](https://beta.reactrouter.com/en/main/start/overview)。您也可以构建自己的解决方案，在这种情况下，你可以在幕后使用 Effect，但是请注意添加用于删除重复请求、缓存响应和避免网络瀑布的逻辑（通过预加载数据或将数据需求提升到路由）。

如果这两种方法都不适合您，您仍然可以继续在 Effects 中直接获取数据。

发送分析

考虑这段代码，它在页面访问时发送分析事件：

在开发中，每个 URL 的 `logVisit` 将被调用两次，因此您可能会想尝试修复它。**我们建议保持此代码不变。** 就像之前的示例一样，运行一次和运行两次之间没有 *用户可见* 行为差异。从实际的角度来看，`logVisit` 在开发中不应做任何事情，因为您不希望开发机器上的日志影响生产指标。每当您保存其文件时，组件都会重新挂载，因此在开发中无论如何都会记录额外的访问。

**在生产中，不会有重复的访问日志。**

要调试您发送的分析事件，您可以将应用部署到一个运行在生产模式下的暂存环境，或暂时选择退出 [严格模式](/reference/react/StrictMode) 及其仅限于开发的重新挂载检查。您还可以从路由更改事件处理程序而不是 Effects 发送分析。对于更精确的分析，[交叉观察者](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) 可以帮助跟踪哪些组件在视口中，以及它们保持可见的时间。

不是一个 Effect：初始化应用程序

某些逻辑应该只在应用程序启动时运行一次。您可以将其放在组件外部：

这保证了这样的逻辑只在浏览器加载页面后运行一次。

不是一个 Effect：购买产品

有时，即使您编写了清理函数，也无法防止运行 Effect 两次的用户可见后果。例如，也许您的 Effect 发送一个 POST 请求来购买产品：

您不希望购买该产品两次。然而，这也是您不应该将此逻辑放在 Effect 中的原因。假设用户转到另一页面，然后按下返回？您的 Effect 将再次运行。您不想在用户 *访问* 页面时购买产品；您希望在用户 *点击* 购买按钮时购买。

购买不是由渲染引起的；而是由特定的交互引起的。它应该仅在用户按下按钮时运行。**删除 Effect 并将您的 `/api/buy` 请求移动到购买按钮事件处理程序中：**

**这说明如果重新挂载破坏了您的应用逻辑，通常会暴露出现有的错误。** 从用户的角度来看，访问页面不应与访问页面、点击链接然后按返回查看页面之间有所不同。React 通过在开发中重新挂载组件来验证您的组件是否遵循这一原则。

将所有内容整合在一起

这个游乐场可以帮助您“感受” Effects 在实践中的工作方式。

这个示例使用 [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout) 来安排在 Effect 运行三秒后显示输入文本的控制台日志。清理函数取消挂起的超时。首先按下“挂载组件”：

您会看到三条日志：`Schedule "a" log`、`Cancel "a" log` 和 `Schedule "a" log` 再次。三秒后，还会有一条日志显示 `a`。正如您之前所学，额外的调度/取消对是因为 React 在开发中重新挂载组件以验证您是否正确实现了清理。

现在编辑输入以说 `abc`。如果您足够快，您会看到 `Schedule "ab" log` 紧接着 `Cancel "ab" log` 和 `Schedule "abc" log`。**React 总是清理上一个渲染的 Effect，然后再执行下一个渲染的 Effect。** 这就是为什么即使您快速输入，最多也只会调度一个超时。多次编辑输入并观察控制台，以感受 Effects 是如何被清理的。

在输入中键入一些内容，然后立即按下“卸载组件”。注意，卸载会清理最后一个渲染的 Effect。在这里，它在超时有机会触发之前清理了最后的超时。

最后，编辑上面的组件，并注释掉清理函数，以便超时不被取消。尝试快速输入 `abcde`。您预期在三秒后会发生什么？`console.log(text)` 是否会打印 *最新* 的 `text` 并产生五个 `abcde` 日志？试试看以检查您的直觉！

三秒后，您应该看到一系列日志（`a`、`ab`、`abc`、`abcd` 和 `abcde`），而不是五个 `abcde` 日志。**每个 Effect “捕获” 其对应渲染的 `text` 值。** 不管 `text` 状态如何变化：来自渲染的 Effect 中的 `text = 'ab'` 将始终看到 `'ab'`。换句话说，每个渲染的 Effects 彼此隔离。如果您想了解这是如何运作的，您可以阅读 [闭包](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)。

每次渲染都有自己的 Effects

您可以将 `useEffect` 视为“将”一段行为附加到渲染输出。考虑这个 Effect：

我们来看看当用户在应用中导航时会发生什么。

**Effect 也是渲染输出的一部分。** 第一次渲染的 Effect 变成：

React 运行此 Effect，连接到 `'general'` 聊天室。

使用相同的依赖项重新渲染

假设 `<ChatRoom roomId="general" />` 重新渲染。JSX 输出是相同的：

React 看到渲染输出没有改变，所以它不会更新 DOM。

第二次渲染的 Effect 看起来是这样的：

React 将第二次渲染的 `['general']` 与第一次渲染的 `['general']` 进行比较。**因为所有依赖项都是相同的，React *忽略* 第二次渲染的 Effect。** 它从未被调用。

使用不同的依赖项重新渲染

然后，用户访问 `<ChatRoom roomId="travel" />`。这次，组件返回不同的 JSX：

React 更新 DOM，将 `"Welcome to general"` 改为 `"Welcome to travel"`。

第三次渲染的 Effect 看起来是这样的：

React 将第三次渲染的 `['travel']` 与第二次渲染的 `['general']` 进行比较。一个依赖项不同：`Object.is('travel', 'general')` 为 `false`。Effect 无法被跳过。

**在 React 应用第三次渲染的 Effect 之前，它需要清理上一个 Effect。** 第二次渲染的 Effect 被跳过，因此 React 需要清理第一次渲染的 Effect。如果您向上滚动到第一次渲染，您会看到它的清理调用在使用 `createConnection('general')` 创建的连接上调用 `disconnect()`。这将应用程序从 `'general'` 聊天室断开。

之后，React 运行第三次渲染的 Effect。它连接到 `'travel'` 聊天室。

卸载

最后，假设用户导航离开，`ChatRoom` 组件卸载。React 运行最后一次 Effect 的清理函数。最后一次 Effect 来自第三次渲染。第三次渲染的清理销毁了 `createConnection('travel')` 连接。因此，应用程序断开了与 `'travel'` 房间的连接。

仅限开发的行为

当 [严格模式](/reference/react/StrictMode) 打开时，React 在挂载后重新挂载每个组件一次（状态和 DOM 被保留）。这 [帮助您发现需要清理的 Effects](#step-3-add-cleanup-if-needed) 并及早暴露出错误，例如竞争条件。此外，React 会在您开发时保存文件时重新挂载 Effects。所有这些行为都是仅限开发的。

- 与事件不同，Effects 是由渲染本身引起的，而不是由特定交互引起的。
- Effects 允许您将组件与某些外部系统（第三方 API、网络等）同步。
- 默认情况下，Effects 在每次渲染后运行（包括初始渲染）。
- 如果所有依赖项的值与上次渲染时相同，React 将跳过 Effect。
- 您无法“选择”您的依赖项。它们由 Effect 内的代码决定。
- 空依赖数组 (`[]`) 对应于组件“挂载”，即添加到屏幕上。
- 在严格模式下，React 在开发中将组件挂载两次，以压力测试您的 Effects。
- 如果您的 Effect 因重新挂载而中断，您需要实现一个清理函数。
- React 会在 Effect 下次运行之前以及在卸载时调用您的清理函数。

在挂载时聚焦字段

在这个示例中，表单渲染了一个 `<MyInput />` 组件。

使用输入的 [`focus()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus) 方法使 `MyInput` 在出现在屏幕上时自动获得焦点。已经有一个注释掉的实现，但它并没有完全工作。找出它为什么不起作用，并修复它。（如果您熟悉 `autoFocus` 属性，请假装它不存在：我们正在从头实现相同的功能。）

要验证您的解决方案是否有效，请按“显示表单”，并验证输入是否获得焦点（变为高亮并且光标位于内部）。按“隐藏表单”再按一次“显示表单”。验证输入再次被高亮。

`MyInput` 应该只在 _挂载_ 时获得焦点，而不是在每次渲染后。要验证行为是否正确，按“显示表单”，然后反复按“将其变为大写”复选框。点击复选框不应使上面的输入获得焦点。

在渲染期间调用 `ref.current.focus()` 是错误的，因为它是一个 *副作用*。副作用应该放在事件处理程序内，或者用 `useEffect` 声明。在这种情况下，副作用是 _由于_ 组件的出现引起的，而不是由任何特定的交互引起的，因此将其放在 Effect 中是有意义的。

要修复错误，将 `ref.current.focus()` 调用包装在 Effect 声明中。然后，为确保该 Effect 仅在挂载时运行，而不是在每次渲染后运行，请向其添加空的 `[]` 依赖项。
```