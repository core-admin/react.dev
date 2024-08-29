---
title: '使用 ref 引用值'
---

<Intro>

当你希望组件“记住”某些信息，但又不想让这些信息 [触发新的渲染](/learn/render-and-commit) 时，你可以使用 ref 。

</Intro>

<YouWillLearn>

- 如何给你的组件添加 ref
- 如何更新 ref 的值
- ref 与 state 的区别
- 如何安全地使用 ref

</YouWillLearn>

## 给你的组件添加 ref {/*adding-a-ref-to-your-component*/}

你可以通过从 React 中导入 `useRef` Hook 来为你的组件添加 ref：

```js
import { useRef } from 'react';
```

在组件内部，调用 `useRef` Hook，并将你想引用的初始值作为唯一参数传入。例如，这里的 ref 引用的值是“0”：

```js
const ref = useRef(0);
```

`useRef` 返回一个这样的对象：

```js
{ 
  current: 0 // 你向 useRef 传入的值
}
```

<Illustration src="/images/docs/illustrations/i_ref.png" alt="An arrow with 'current' written on it stuffed into a pocket with 'ref' written on it." />

你可以通过 `ref.current` 属性访问这个 ref 的当前值。这个值是可变的，这意味着你可以对其进行读取和修改。可以把它看作是组件的一个秘密口袋，React 不会跟踪这个值。（这就是它成为 React 单向数据流的“逃生通道”的原因——详细内容见下文！）

这里，每次点击按钮时会使 `ref.current` 递增：

<Sandpack>

```js
import { useRef } from 'react';

export default function Counter() {
  let ref = useRef(0);

  function handleClick() {
    ref.current = ref.current + 1;
    alert('You clicked ' + ref.current + ' times!');
  }

  return (
    <button onClick={handleClick}>
      Click me!
    </button>
  );
}
```

</Sandpack>

这个 ref 指向一个数字，和[state](/learn/state-a-components-memory)一样，你也可以指向其他任何东西，比如字符串、对象，甚至是函数。与 state 不同，ref 是一个普通的 JavaScript 对象，拥有一个可以读取和修改的 `current` 属性。

N请注意 **每次增加时组件不会重新渲染。** 就像 state 一样，React 会在每次重新渲染之间保留 ref。然而，设置 state 会导致组件重新渲染，而改变 ref 则不会！

## 示例：构建一个秒表 {/*example-building-a-stopwatch*/}

你可以在一个组件中同时使用 refs 和 state。比如，我们可以做一个秒表，用户可以通过按下按钮来启动或停止。为了显示用户按下“开始”按钮后经过的时间，你需要记录按下“开始”按钮的时间和当前时间。 **这些信息用于渲染，因此你将其保存在 state 中：**

```js
const [startTime, setStartTime] = useState(null);
const [now, setNow] = useState(null);
```

当用户按下“开始”时，你可以使用 [`setInterval`](https://developer.mozilla.org/docs/Web/API/setInterval) 每 10 毫秒更新一次时间：

<Sandpack>

```js
import { useState } from 'react';

export default function Stopwatch() {
  const [startTime, setStartTime] = useState(null);
  const [now, setNow] = useState(null);

  function handleStart() {
    // Start counting.
    setStartTime(Date.now());
    setNow(Date.now());

    setInterval(() => {
      // Update the current time every 10ms.
      setNow(Date.now());
    }, 10);
  }

  let secondsPassed = 0;
  if (startTime != null && now != null) {
    secondsPassed = (now - startTime) / 1000;
  }

  return (
    <>
      <h1>Time passed: {secondsPassed.toFixed(3)}</h1>
      <button onClick={handleStart}>
        Start
      </button>
    </>
  );
}
```

</Sandpack>

当用户按下“停止”按钮时，你需要取消当前的计时器，以便停止更新 `now` 状态变量。你可以通过调用 [`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval) 来实现，但你需要提供在用户按下“开始”时 `setInterval` 返回的计时器 ID。你需要把这个计时器 ID 存储在某个地方。 **由于计时器 ID 不用于渲染，你可以将其保存在 ref 中：**

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Stopwatch() {
  const [startTime, setStartTime] = useState(null);
  const [now, setNow] = useState(null);
  const intervalRef = useRef(null);

  function handleStart() {
    setStartTime(Date.now());
    setNow(Date.now());

    clearInterval(intervalRef.current);
    intervalRef.current = setInterval(() => {
      setNow(Date.now());
    }, 10);
  }

  function handleStop() {
    clearInterval(intervalRef.current);
  }

  let secondsPassed = 0;
  if (startTime != null && now != null) {
    secondsPassed = (now - startTime) / 1000;
  }

  return (
    <>
      <h1>Time passed: {secondsPassed.toFixed(3)}</h1>
      <button onClick={handleStart}>
        Start
      </button>
      <button onClick={handleStop}>
        Stop
      </button>
    </>
  );
}
```

</Sandpack>

当某些信息用于渲染时，应该将其保存在 state 中。当某些信息仅在事件处理程序中需要，并且改变它不需要重新渲染时，使用 ref 可能会更有效。

## ref 和 state 的不同之处 {/*differences-between-refs-and-state*/}

也许你会觉得 refs 比 state “宽松”——例如，你可以直接修改 refs，而不必每次都使用状态设置函数。但在大多数情况下，使用 state 是更好的选择。Refs 是一个不常用的“逃生通道”。以下是 state 和 refs 的比较：

| refs                                                                                  | state                                                                                                                     |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `useRef(initialValue)` 返回 `{ current: initialValue }`                                | `useState(initialValue)` 返回当前状态值和一个状态设置函数 (`[value, setValue]`)                                                 |
| 更改时不会触发组件重新渲染。                                                                | 更改时会触发组件重新渲染。                                                                                                    |
| 可变的——你可以在渲染过程中修改和更新 `current` 的值。                                         | “不可变的”——你必须使用状态设置函数来修改状态变量，从而排队进行重新渲染。                                                              |
| 你不应该在渲染期间读取（或写入）`current` 值。                                               | 你可以随时读取 state。然而，每次渲染都有自己的[state快照](/learn/state-as-a-snapshot)，这个快照不会改变。                             |

这是一个使用 state 实现的计数器按钮：

<Sandpack>

```js
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      You clicked {count} times
    </button>
  );
}
```

</Sandpack>

因为 `count` 值是可显示的，所以使用 state 是合适的。当使用 `setCount()` 设置计数器的值时，React 会重新渲染组件，屏幕会更新以反映新的计数。

如果你尝试用 ref 来实现这一点，React 将永远不会重新渲染组件，因此你永远看不到计数的变化！看看这个按钮 **不会更新其文本**：

<Sandpack>

```js
import { useRef } from 'react';

export default function Counter() {
  let countRef = useRef(0);

  function handleClick() {
    // 这样并未重新渲染组件！
    countRef.current = countRef.current + 1;
  }

  return (
    <button onClick={handleClick}>
      你点击了 {countRef.current} 次
    </button>
  );
}
```

</Sandpack>

这就是为什么在渲染期间读取 `ref.current` 会导致不可靠的代码。如果需要这样做，请使用 state。

<DeepDive>

#### useRef 内部是如何运行的？ {/*how-does-use-ref-work-inside*/}

尽管 `useState` 和 `useRef` 都是 React 提供的，但原则上 `useRef` 可以在 `useState` 之上实现。你可以想象，在 React 内部，`useRef` 的实现类似于这样：

```js
// React 内部
function useRef(initialValue) {
  const [ref, unused] = useState({ current: initialValue });
  return ref;
}
```

在第一次渲染时，`useRef` 返回 `{ current: initialValue }`。这个对象由 React 存储，因此在下一次渲染时将返回相同的对象。请注意，在这个例子中state 设置函数未被使用。这是因为 `useRef` 总是需要返回相同的对象，所以不需要状态设置器！

React 提供了一个内置版本的 `useRef`，因为它在实践中非常常见。你可以将其视为没有设置函数的常规 state 变量。如果你熟悉面向对象编程，ref 可能会让你想起实例字段——但不是使用 `this.something`，而是 `somethingRef.current`。

</DeepDive>

## 何时使用 ref {/*when-to-use-refs*/}

通常，当你的组件需要“跳出” React 并与外部 API 通信时（通常是某个不会影响组件外观的浏览器 API），你会选择使用 refs。这些情况相对少见，包括：

- 存储 [timeout IDs](https://developer.mozilla.org/docs/Web/API/setTimeout)。
- 存储和操作 [DOM 元素](https://developer.mozilla.org/docs/Web/API/Element)，我们将在[下一页](/learn/manipulating-the-dom-with-refs)中讨论。
- 存储不需要被用来计算 JSX 的其他对象。

如果你的组件需要存储某个值，但它不影响渲染逻辑，使用 refs 是合适的选择。

## ref 的最佳实践 {/*best-practices-for-refs*/}

遵循以下原则可以使你的组件更具可预测性：

- **将 refs 视为逃生通道。** 当你与外部系统或浏览器 API 一起工作时，ref 是有用的。如果你的应用程序逻辑和数据流大部分依赖于 ref，你可能需要重新考虑你的方法。
- **不要在渲染期间读取或写入 `ref.current`。** 如果在渲染期间需要某些信息，请使用[state](/learn/state-a-components-memory)。因为 React 不知道 `ref.current` 何时发生变化，即使在渲染时读取它也会使组件的行为难以预测。（唯一的例外是像 `if (!ref.current) ref.current = new Thing()` 这样的代码，它只在第一次渲染期间设置 ref 一次。）

React state 的限制不适用于 ref。例如，state 就像 [每次渲染的快照](/learn/state-as-a-snapshot)，并且[不会同步更新](/learn/queueing-a-series-of-state-updates)。但是，当你修改 ref 的当前值时，它会立即改变：

```js
ref.current = 5;
console.log(ref.current); // 5
```

这是因为 **ref 本身是一个普通的 JavaScript 对象，** 因此它的行为就像一个普通对象。

在处理 ref 时，你也不需要担心[避免变更](/learn/updating-objects-in-state)。只要你修改的对象不用于渲染，React 不在乎你对 ref 或其内容做什么。

## ref 和 DOM {/*refs-and-the-dom*/}

你可以将 ref 指向任何值。然而，最常见的用法是访问 DOM 元素。例如，当你想以编程方式聚焦一个输入框时，这会很方便。当你在 JSX 中将 ref 传递给 `ref` 属性，例如 `<div ref={myRef}>`，React 会将相应的 DOM 元素放入 `myRef.current`。一旦元素从 DOM 中移除，React 会将 `myRef.current` 更新为 `null`。你可以在[使用 ref 操作 DOM](/learn/manipulating-the-dom-with-refs)中了解更多信息。

<Recap>

- ref 是一个逃生通道，用于保留不用于渲染的值。你不会经常需要它们。
- ref 是一个普通的 JavaScript 对象，具有一个名为 `current` 的单一属性，你可以读取或设置。
- 你可以通过调用 `useRef` Hook 来让 React 给你一个 ref。
- 与 state 类似，ref 允许你在组件的重新渲染之间保留信息。
- 与 state 不同，设置 ref 的 `current` 值不会触发重新渲染。
- 不要在渲染期间读取或写入 `ref.current`。这会使你的组件难以预测。

</Recap>



<Challenges>

#### 修复一个损坏的聊天输入框 {/*fix-a-broken-chat-input*/}

输入一条消息并点击“发送”。你会发现，在你看到“已发送！”提示框之前，有三秒的延迟。在这段时间里，你会看到一个“撤销”按钮。点击它。这个“撤销”按钮应该阻止“已发送！”消息出现。它通过调用 [`clearTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/clearTimeout) 来实现，使用的是在 `handleSend` 中保存的超时 ID。但是，即使在单击“撤消”后，“已发送！”消息仍然出现。找出原因并修复它。

<Hint>

像 `let timeoutID` 这样的常规变量在重新渲染之间不会“存活”，因为每次渲染都会从头开始运行你的组件（并初始化其变量）。你应该将超时 ID 存储在其他地方吗？

</Hint>

<Sandpack>

```js
import { useState } from 'react';

export default function Chat() {
  const [text, setText] = useState('');
  const [isSending, setIsSending] = useState(false);
  let timeoutID = null;

  function handleSend() {
    setIsSending(true);
    timeoutID = setTimeout(() => {
      alert('已发送！');
      setIsSending(false);
    }, 3000);
  }

  function handleUndo() {
    setIsSending(false);
    clearTimeout(timeoutID);
  }

  return (
    <>
      <input
        disabled={isSending}
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <button
        disabled={isSending}
        onClick={handleSend}>
        {isSending ? '发送中……' : '发送'}
      </button>
      {isSending &&
        <button onClick={handleUndo}>
          撤销
        </button>
      }
    </>
  );
}
```

</Sandpack>

<Solution>

每当你的组件重新渲染（例如，当你设置 state 时），所有局部变量都会从头开始初始化。这就是为什么你不能将超时 ID 保存在本地变量 `timeoutID` 中，然后希望另一个事件处理程序在将来“看到”它。相反，你应该将其存储在 ref 中，React 将在渲染之间保存它。

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Chat() {
  const [text, setText] = useState('');
  const [isSending, setIsSending] = useState(false);
  const timeoutRef = useRef(null);

  function handleSend() {
    setIsSending(true);
    timeoutRef.current = setTimeout(() => {
      alert('Sent!');
      setIsSending(false);
    }, 3000);
  }

  function handleUndo() {
    setIsSending(false);
    clearTimeout(timeoutRef.current);
  }

  return (
    <>
      <input
        disabled={isSending}
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <button
        disabled={isSending}
        onClick={handleSend}>
        {isSending ? 'Sending...' : 'Send'}
      </button>
      {isSending &&
        <button onClick={handleUndo}>
          Undo
        </button>
      }
    </>
  );
}
```

</Sandpack>

</Solution>


#### 修复无法重新渲染的组件 {/*fix-a-component-failing-to-re-render*/}

这个按钮应该在显示“开”和“关”之间切换。然而，它总是显示“关”。这段代码有什么问题？请修复它。

<Sandpack>

```js
import { useRef } from 'react';

export default function Toggle() {
  const isOnRef = useRef(false);

  return (
    <button onClick={() => {
      isOnRef.current = !isOnRef.current;
    }}>
      {isOnRef.current ? 'On' : 'Off'}
    </button>
  );
}
```

</Sandpack>

<Solution>

在这个例子中，ref 的当前值用于计算渲染输出：`{isOnRef.current ? 'On' : 'Off'}`。这表明这些信息不应该在 ref 中，而应该放在 state 中。要修复它，删除 ref 并改用 state：

<Sandpack>

```js
import { useState } from 'react';

export default function Toggle() {
  const [isOn, setIsOn] = useState(false);

  return (
    <button onClick={() => {
      setIsOn(!isOn);
    }}>
      {isOn ? 'On' : 'Off'}
    </button>
  );
}
```

</Sandpack>

</Solution>

#### 修复防抖 {/*fix-debouncing*/}

在这个例子中，所有按钮点击处理程序都是[“防抖”](https://redd.one/blog/debounce-vs-throttle)。要了解这意味着什么，请按下一个按钮。注意消息延迟一秒后才出现。如果你在等待消息的同时按下按钮，计时器将重置。因此，如果你不停快速点击同一个按钮，消息将在你停止点击后一秒 *才* 出现。防抖可以让你将一些动作推迟到用户“停止动作”之后。

这个例子可以工作，但并不是完全按照预期工作。按钮之间不是独立的。要看到这个问题，点击一个按钮，然后立即点击另一个按钮。你会期待在延迟后看到两个按钮的消息。但只有最后一个按钮的消息显示出来。第一个按钮的消息被丢失了。

为什么按钮之间会相互干扰？找出并修复这个问题。

<Hint>

最后一个超时 ID 变量在所有 `DebouncedButton` 组件之间共享。这就是为什么点击一个按钮会重置另一个按钮的超时。你能为每个按钮存储一个单独的超时 ID 吗？

</Hint>

<Sandpack>

```js
let timeoutID;

function DebouncedButton({ onClick, children }) {
  return (
    <button onClick={() => {
      clearTimeout(timeoutID);
      timeoutID = setTimeout(() => {
        onClick();
      }, 1000);
    }}>
      {children}
    </button>
  );
}

export default function Dashboard() {
  return (
    <>
      <DebouncedButton
        onClick={() => alert('宇宙飞船已发射！')}
      >
        发射宇宙飞船
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('汤煮好了！')}
      >
        煮点儿汤
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('摇篮曲唱完了！')}
      >
        唱首摇篮曲
      </DebouncedButton>
    </>
  )
}
```

```css
button { display: block; margin: 10px; }
```

</Sandpack>

<Solution>

像 `timeoutID` 这样的变量在所有组件之间共享。这就是为什么点击第二个按钮会重置第一个按钮的待定超时。要修复这个问题，你可以将超时保存在一个 ref 中。每个按钮将获得自己的 ref，因此它们不会相互冲突。注意快速点击两个按钮会显示两个消息。

<Sandpack>

```js
import { useRef } from 'react';

function DebouncedButton({ onClick, children }) {
  const timeoutRef = useRef(null);
  return (
    <button onClick={() => {
      clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => {
        onClick();
      }, 1000);
    }}>
      {children}
    </button>
  );
}

export default function Dashboard() {
  return (
    <>
      <DebouncedButton
        onClick={() => alert('Spaceship launched!')}
      >
        Launch the spaceship
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('Soup boiled!')}
      >
        Boil the soup
      </DebouncedButton>
      <DebouncedButton
        onClick={() => alert('Lullaby sung!')}
      >
        Sing a lullaby
      </DebouncedButton>
    </>
  )
}
```

```css
button { display: block; margin: 10px; }
```

</Sandpack>

</Solution>

#### Read the latest state {/*read-the-latest-state*/}

In this example, after you press "Send", there is a small delay before the message is shown. Type "hello", press Send, and then quickly edit the input again. Despite your edits, the alert would still show "hello" (which was the value of state [at the time](/learn/state-as-a-snapshot#state-over-time) the button was clicked).

Usually, this behavior is what you want in an app. However, there may be occasional cases where you want some asynchronous code to read the *latest* version of some state. Can you think of a way to make the alert show the *current* input text rather than what it was at the time of the click?

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Chat() {
  const [text, setText] = useState('');

  function handleSend() {
    setTimeout(() => {
      alert('Sending: ' + text);
    }, 3000);
  }

  return (
    <>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />
      <button
        onClick={handleSend}>
        Send
      </button>
    </>
  );
}
```

</Sandpack>

<Solution>

State works [like a snapshot](/learn/state-as-a-snapshot), so you can't read the latest state from an asynchronous operation like a timeout. However, you can keep the latest input text in a ref. A ref is mutable, so you can read the `current` property at any time. Since the current text is also used for rendering, in this example, you will need *both* a state variable (for rendering), *and* a ref (to read it in the timeout). You will need to update the current ref value manually.

<Sandpack>

```js
import { useState, useRef } from 'react';

export default function Chat() {
  const [text, setText] = useState('');
  const textRef = useRef(text);

  function handleChange(e) {
    setText(e.target.value);
    textRef.current = e.target.value;
  }

  function handleSend() {
    setTimeout(() => {
      alert('Sending: ' + textRef.current);
    }, 3000);
  }

  return (
    <>
      <input
        value={text}
        onChange={handleChange}
      />
      <button
        onClick={handleSend}>
        Send
      </button>
    </>
  );
}
```

</Sandpack>

</Solution>

</Challenges>
