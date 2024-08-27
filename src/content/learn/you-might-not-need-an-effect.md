---
title: '你可能不需要 Effect'
---

<Intro>

Effect 是 React 设计中的一种特殊机制，让你可以“跳出” React 的框架，将组件与一些外部系统（比如非 React 小部件、网络请求或浏览器 DOM）进行同步。如果你的操作不涉及外部系统（例如，当某些 props 或 state 变化时，你想更新组件的状态），那就不需要使用 Effect。去掉不必要的 Effect 可以让你的代码更清晰、运行更快，同时减少出错的概率。

</Intro>

<YouWillLearn>

* 如何识别并移除组件中不必要的 Effect
* 如何在没有 Effect 的情况下缓存耗时的计算
* 如何重置和调整组件状态而不使用 Effect
* 如何在事件处理函数之间共享逻辑
* 哪些逻辑需要放在事件处理函数中
* 如何通知父组件状态的变化

</YouWillLearn>

## 如何移除不必要的 Effect {/*how-to-remove-unnecessary-effects*/}

有两种常见情况，你不需要使用 Effect：

* **当你要转换数据以进行渲染时，不需要使用 Effect。** 举个例子，假设你想在显示列表之前对其进行过滤。你可能会想写一个 Effect，在列表变化时更新状态变量，但这效率不高。当你更新状态时，React 首先会调用组件函数来决定屏幕上应该显示什么，然后将这些变更[“提交”](/learn/render-and-commit)到 DOM，最后才会运行你的 Effect。如果 Effect 立即更新状态，这样会重新启动整个过程！为了避免不必要的渲染，最好在组件的顶层处理数据，这段代码在 props 或 state 变化时会自动重新运行。
* **当处理用户事件时，也不需要 Effect。** 例如，假设用户购买产品时你想发送一个 `/api/buy` POST 请求并显示通知。在购买按钮的点击事件处理函数中，你清楚地知道发生了什么。而当 Effect 执行时，你并不知道用户的具体操作（例如，点击了哪个按钮）。因此，通常在相应的事件处理函数中处理用户事件更为合适。

当然，Effect 在与外部系统[同步](/learn/synchronizing-with-effects#what-are-effects-and-how-are-they-different-from-events)时是必需的。比如，你可以写一个 Effect，使 jQuery 小部件与 React 状态保持一致。你还可以通过 Effect 来获取数据，例如，将搜索结果与当前的搜索查询同步。需要注意的是，现代[框架](/learn/start-a-new-react-project#production-grade-react-frameworks)通常提供比在组件中直接编写 Effect 更高效的数据获取机制。

为了帮助你更好地理解，让我们来看一些具体的常见例子！

### 基于 props 或 state 更新状态 {/*updating-state-based-on-props-or-state*/}

假设你有一个组件，里面有两个状态变量：`firstName` 和 `lastName`。你想通过把它们连接起来计算一个 `fullName`。更重要的是，你希望每当 `firstName` 或 `lastName` 发生变化时，`fullName` 也能自动更新。你最初可能的想法是添加一个 `fullName` 状态变量，并在 Effect 中更新它：

```js {5-9}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // 🔴 Avoid: redundant state and unnecessary Effect
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(firstName + ' ' + lastName);
  }, [firstName, lastName]);
  // ...
}
```

但这样做其实比必要的复杂得多，而且效率也不高：它会先用旧的 `fullName` 值进行一次完整的渲染，然后再立即用更新后的值重新渲染。解决方案是移除这个状态变量和 Effect：

```js {4-5}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  // ✅ Good: calculated during rendering
  const fullName = firstName + ' ' + lastName;
  // ...
}
```

**当某个值可以从现有的 props 或 state 中计算出来时，[不要把它作为一个 state。](/learn/choosing-the-state-structure#avoid-redundant-state) 相反，应该在渲染时进行计算。** 这样可以让你的代码更快（避免多余的“级联”更新），更简单（减少代码量），并且减少出错的机会（避免不同状态变量之间不一致的问题）。如果这对你来说是个新概念，[React 哲学](/learn/thinking-in-react#step-3-find-the-minimal-but-complete-representation-of-ui-state) 中解释了什么值应该作为 state。

### 缓存耗时计算 {/*caching-expensive-calculations*/}

这个组件通过接收的 `todos` 并根据 `filter` prop 进行过滤来计算 `visibleTodos`。你可能会想把结果存储在状态里，并通过 Effect 更新它：

```js {4-8}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // 🔴 Avoid: redundant state and unnecessary Effect
  const [visibleTodos, setVisibleTodos] = useState([]);
  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);

  // ...
}
```

但就像之前的例子，这种做法既不必要也低效。首先，移除状态和 Effect：

```js {3-4}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  // ✅ This is fine if getFilteredTodos() is not slow.
  const visibleTodos = getFilteredTodos(todos, filter);
  // ...
}
```

通常，这段代码是可以的！但是如果 `getFilteredTodos()` 的计算很慢，或者你的 `todos` 数量很多，那么当某个无关的状态变量比如 `newTodo` 发生变化时，你并不想重新执行 `getFilteredTodos()`。

你可以通过将这个计算放在 [`useMemo`](/reference/react/useMemo) Hook 中来缓存（或称为 [“记忆化”](https://en.wikipedia.org/wiki/Memoization)）一个耗时的计算：

```js {5-8}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  const visibleTodos = useMemo(() => {
    // ✅ Does not re-run unless todos or filter change
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
  // ...
}
```

或者，可以简化为一行：

```js {5-6}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  // ✅ Does not re-run getFilteredTodos() unless todos or filter change
  const visibleTodos = useMemo(() => getFilteredTodos(todos, filter), [todos, filter]);
  // ...
}
```

**这告诉 React，只有在 `todos` 或 `filter` 发生变化时，才会重新执行内部函数。** React 会在第一次渲染时记住 `getFilteredTodos()` 的返回值。在后续渲染中，它会检查 `todos` 或 `filter` 是否有所不同。如果没有变化，`useMemo` 将返回之前存储的结果；如果有变化，React 会再次调用内部函数并存储结果。

你在 [`useMemo`](/reference/react/useMemo) 中包装的函数是在渲染时执行的，所以这仅适用于 [纯函数](/learn/keeping-components-pure)场景。

<DeepDive>

#### 如何判断计算是否耗时（昂贵的）? {/*how-to-tell-if-a-calculation-is-expensive*/}

一般来说，除非你在创建或遍历成千上万的对象，否则计算通常不会很耗时。如果你想确认一下，可以在代码中添加一个控制台日志来测量执行时间：

```js {1,3}
console.time('filter array');
const visibleTodos = getFilteredTodos(todos, filter);
console.timeEnd('filter array');
```

执行你正在测试的交互（例如，输入框中输入）。然后在控制台中你会看到类似 `filter array: 0.15ms` 的日志。如果总的日志时间加起来显著（比如达到 `1ms` 或更多），那么把计算结果记忆（memoize）是有意义的。做一个实验，你可以将计算用 `useMemo` 包裹起来，看看这个交互的总日志时间是否有减少：

```js
console.time('filter array');
const visibleTodos = useMemo(() => {
  return getFilteredTodos(todos, filter); // Skipped if todos and filter haven't changed
}, [todos, filter]);
console.timeEnd('filter array');
```

`useMemo` 不会加快 *第一次* 渲染的速度。它只是帮助你在更新时跳过不必要的计算。

请注意，你的设备性能可能比用户的更好，因此最好通过人工限制工具来测试性能。例如，Chrome 提供了 [CPU 节流](https://developer.chrome.com/blog/new-in-devtools-61/#throttling) 的选项。

另外，要注意在开发环境中测量性能可能不够准确。（例如，当 [Strict Mode](/reference/react/StrictMode) 开启时，每个组件会渲染两次，而不是一次。）要获得最准确的时间，建议将你的应用构建为生产版本，并在与用户相似的设备上进行测试。

</DeepDive>

### 当 props 变化时重置所有 state {/*resetting-all-state-when-a-prop-changes*/}

这个 `ProfilePage` 组件接收一个 `userId` prop。页面上有一个评论输入框，你使用一个 `comment` 状态变量来保存它的值。但你发现了一个问题：当你从一个用户资料切换到另一个用户资料时，`comment` 状态并没有被重置。这样一来，就很容易在错误的用户资料上意外发送评论。为了解决这个问题，你希望在 `userId` 变化时清除 `comment` 状态变量：

```js {4-7}
export default function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');

  // 🔴 Avoid: Resetting state on prop change in an Effect
  useEffect(() => {
    setComment('');
  }, [userId]);
  // ...
}
```

这样做的效率不高，因为 `ProfilePage` 和它的子组件会先用旧值渲染，然后再重新渲染。而且这也很复杂，因为你需要在 `ProfilePage` 中的 *每一个* 有状态的组件里都这样做。如果评论界面是嵌套的，你也需要清空嵌套的评论状态。

相反，你可以通过给组件一个显式的 key 来告诉 React，它们原则上是**不同**的个人资料组件。将组件拆分为两个，并从外部组件传递一个 `key` 属性到内部组件：

```js {5,11-12}
export default function ProfilePage({ userId }) {
  return (
    <Profile
      userId={userId}
      key={userId}
    />
  );
}

function Profile({ userId }) {
  // ✅ This and any other state below will reset on key change automatically
  const [comment, setComment] = useState('');
  // ...
}
```

通常，当在相同的位置渲染相同的组件时，React 会保留状态。**通过将 `userId` 作为 `key` 传递给 `Profile` 组件，你是在要求 React 将两个不同 `userId` 的 `Profile` 组件视为两个不同的组件，从而不共享任何状态。** 每当这个 key（你设置为 `userId`）发生变化时，React 会重新创建 DOM，并 [重置](/learn/preserving-and-resetting-state#option-2-resetting-state-with-a-key) `Profile` 组件的 state 及其所有子组件。这样，当在用户资料之间切换时，`comment` 字段就会自动清空。

需要注意的是，在这个例子中，只有外部的 `ProfilePage` 组件被导出，并对项目中的其他文件可见。渲染 `ProfilePage` 的组件不需要将 key 传递给它：它们将 `userId` 作为常规 prop 传递。`ProfilePage` 将其作为 `key` 传递给内部 `Profile` 组件的做法属于实现细节。

### 当 prop 变化时调整某些 state {/*adjusting-some-state-when-a-prop-changes*/}

有时，你可能希望在 prop 变化时重置或调整 state 的一部分，而不是全部 state。

这个 `List` 组件接收一个 `items` 列表作为 prop，并在 `selection` 状态变量中保存所选项目。你希望在 `items` prop 接收到不同数组时将 `selection` 重置为 `null`：

```js {5-8}
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // 🔴 避免：当 prop 变化时，在 Effect 中调整 state
  useEffect(() => {
    setSelection(null);
  }, [items]);
  // ...
}
```

这样做也不理想。每次 `items` 变化时，`List` 和它的子组件会先用过时的 `selection` 值渲染。然后 React 会更新 DOM 并运行 Effects，最后 `setSelection(null)` 的调用又会导致 `List` 和它的子组件重新渲染，重新开始这个过程。

首先，删除 Effect。取而代之的是在渲染期间直接调整 state：

```js {5-11}
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // 好一些：在渲染期间调整 state
  const [prevItems, setPrevItems] = useState(items);
  if (items !== prevItems) {
    setPrevItems(items);
    setSelection(null);
  }
  // ...
}
```

像这样 [存储来自前一次渲染的信息](/reference/react/useState#storing-information-from-previous-renders) 可能会让人难以理解，但比在 Effect 中更新相同状态要好。在上面的例子中，`setSelection` 是在渲染期间直接调用的。React 会在退出 `return` 语句后 *立即* 重新渲染 `List`。此时，React 尚未渲染 `List` 的子组件或更新 DOM，因此这使得 `List` 的子组件跳过渲染过时的 `selection` 值。

当你在渲染期间更新组件时，React 会丢弃返回的 JSX，并立即尝试重新渲染。为了避免非常缓慢的级联重试，React 只允许你在渲染期间更新 *同一* 组件的状态。如果你在渲染期间更新另一个组件的状态，你会看到错误。像 `items !== prevItems` 这样的条件是必要的，以避免循环。你可以这样调整 state，但任何其他副作用（如更改 DOM 或设置的延时）应该保留在事件处理程序或 Effects 中，以 [保持组件的纯粹性。](/learn/keeping-components-pure)

**虽然这种模式比 Effect 更高效，但大多数组件也不应该需要这种做法。** 无论你怎么做，基于 props 或其他状态调整状态都会使数据流更难以理解和调试。始终检查是否可以 [通过 key 重置所有状态](#resetting-all-state-when-a-prop-changes) 或 [在渲染时计算所有内容](#updating-state-based-on-props-or-state)。例如，与其存储（和重置）所选的 *项目*，不如存储所选的 *项目 ID*：

```js {3-5}
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selectedId, setSelectedId] = useState(null);
  // ✅ 好的：在渲染期间计算所需内容
  const selection = items.find(item => item.id === selectedId) ?? null;
  // ...
}
```

这样就完全不需要“调整”状态了。如果所选 ID 的项目在列表中，它将保持选中状态。如果没有，渲染期间计算的 `selection` 将为 `null`，因为没有找到匹配的项目。这种行为有所不同，但可以说更好，因为大多数对 `items` 的更改仍可以保持选中状态。

### 在事件处理函数中共享逻辑 {/*sharing-logic-between-event-handlers*/}

假设你有一个产品页面，上面有两个按钮（购买和付款），都可以让用户购买该产品。你希望每当用户将产品放入购物车时显示一个通知。在两个按钮的点击处理程序中调用 `showNotification()` 感觉重复，因此你可能会想把这个逻辑放在一个 Effect 中：

```js {2-7}
function ProductPage({ product, addToCart }) {
  // 🔴 避免：在 Effect 中处理属于事件特定的逻辑
  useEffect(() => {
    if (product.isInCart) {
      showNotification(`Added ${product.name} to the shopping cart!`);
    }
  }, [product]);

  function handleBuyClick() {
    addToCart(product);
  }

  function handleCheckoutClick() {
    addToCart(product);
    navigateTo('/checkout');
  }
  // ...
}
```

这个 Effect 是不必要的。它也很可能导致错误。例如，假设你的应用在页面重新加载之前“记住”了购物车中的产品。如果你把一个产品添加到购物车中并刷新页面，通知会再次出现。每当你刷新该产品页面时，它都会继续出现。这是因为 `product.isInCart` 在页面加载时已经是 `true`，因此上面的 Effect 会调用 `showNotification()`。

**当你不确定某段代码应该放在 Effect 还是事件处理程序中时，问问自己 *为什么* 要执行这些代码。Effect 只用来执行那些显示给用户时组件 需要执行 的代码。** 在这个例子中，通知应该在用户 *按下了按钮* 后出现，而不是因为页面显示出来时！删除 Effect，并将共享逻辑放入一个被两个事件处理程序调用的函数中：

```js {2-6,9,13}
function ProductPage({ product, addToCart }) {
  // ✅ 好的：事件特定的逻辑在事件处理函数中处理
  function buyProduct() {
    addToCart(product);
    showNotification(`Added ${product.name} to the shopping cart!`);
  }

  function handleBuyClick() {
    buyProduct();
  }

  function handleCheckoutClick() {
    buyProduct();
    navigateTo('/checkout');
  }
  // ...
}
```

这样既删除了不必要的 Effect，又修复了错误。

### 发送 POST 请求 {/*sending-a-post-request*/}

这个 `Form` 组件会发送两个 POST 请求。 它在页面加载之际会发送一个分析请求。当你填写表格并点击提交按钮时，它会向 `/api/register` 接口发送一个 POST 请求：

```js {5-8,10-16}
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');

  // ✅ 好的 ：这个逻辑应该在组件显示时运行
  useEffect(() => {
    post('/analytics/event', { eventName: 'visit_form' });
  }, []);

  // 🔴 避免：在 Effect 中处理属于事件特定的逻辑
  const [jsonToSubmit, setJsonToSubmit] = useState(null);
  useEffect(() => {
    if (jsonToSubmit !== null) {
      post('/api/register', jsonToSubmit);
    }
  }, [jsonToSubmit]);

  function handleSubmit(e) {
    e.preventDefault();
    setJsonToSubmit({ firstName, lastName });
  }
  // ...
}
```

让我们用之前的标准来分析一下。

分析的 POST 请求应该保留在 Effect 中。这是**因为**发送分析请求是表单显示时就需要执行的。（在开发环境中会触发两次，但 [请参见这里](/learn/synchronizing-with-effects#sending-analytics) 了解如何处理这个情况。）

但是，向 `/api/register` 发送的 POST 请求并不是因为表单被**显示** 而触发的。你只希望在一个特定的时刻发送这个请求：用户按下按钮时。这个请求只应该在**这个特定的交互**中发生。你应该删除第二个 Effect，并将这个 POST 请求放到事件处理程序中：

```js {12-13}
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');

  // ✅ Good ：这个逻辑应该在组件显示时运行
  useEffect(() => {
    post('/analytics/event', { eventName: 'visit_form' });
  }, []);

  function handleSubmit(e) {
    e.preventDefault();
    // ✅ Good: 事件特定的逻辑在事件处理函数中处理
    post('/api/register', { firstName, lastName });
  }
  // ...
}
```

在决定将某些逻辑放在事件处理程序还是 Effect 中时，主要需要考虑的问题是从用户的角度来看，这是什么类型的逻辑。如果这个逻辑是由特定的用户交互引起的，应该将其放在事件处理程序中；如果是因为用户 _看到_ 组件而引发的，应该将其放在 Effect 中。

### 链式计算 {/*chains-of-computations*/}

有时你可能会想要链式调用 Effects，每个 Effect 都基于某些 state 来调整其他的 state：

```js {7-29}
function Game() {
  const [card, setCard] = useState(null);
  // 金卡数
  const [goldCardCount, setGoldCardCount] = useState(0);
  // 圆卡
  const [round, setRound] = useState(1);
  // 游戏结束
  const [isGameOver, setIsGameOver] = useState(false);

  // 🔴 避免: 链接多个 Effect 仅仅为了相互触发调整 state
  useEffect(() => {
    // 如果是金卡
    if (card !== null && card.gold) {
      setGoldCardCount(c => c + 1);
    }
  }, [card]);

  useEffect(() => {
    if (goldCardCount > 3) {
      setRound(r => r + 1)
      setGoldCardCount(0);
    }
  }, [goldCardCount]);

  useEffect(() => {
    if (round > 5) {
      setIsGameOver(true);
    }
  }, [round]);

  useEffect(() => {
    alert('Good game!');
  }, [isGameOver]);

  function handlePlaceCard(nextCard) {
    if (isGameOver) {
      throw Error('Game already ended.');
    } else {
      setCard(nextCard);
    }
  }

  // ...
```

这段代码有两个主要问题。

第一个问题是效率低下：组件（及其子组件）必须在每个 `set` 调用之间重新渲染。在最坏的情况下（`setCard` → 渲染 → `setGoldCardCount` → 渲染 → `setRound` → 渲染 → `setIsGameOver` → 渲染），会出现多次不必要的重新渲染。

第二个问题是，即使它不是慢的，随着代码的演变，你可能会遇到“链式”不再符合新需求的情况。比如说，假设你想添加一种方式来逐步查看游戏的历史操作。你会通过更新每个状态变量为过去的值来实现。但是将 `card` 状态设置为之前的的某个值会再次触发 Effect 链，改变你展示的数据。这种代码通常比较僵硬和脆弱。

在这种情况下，最好在渲染期间计算可以计算的内容，并在事件处理程序中调整 state：

```js {6-7,14-26}
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);

  // ✅ 尽可能在渲染期间进行计算
  const isGameOver = round > 5;

  function handlePlaceCard(nextCard) {
    if (isGameOver) {
      throw Error('Game already ended.');
    }

    // ✅ 在事件处理函数中计算剩下的所有 state
    setCard(nextCard);
    if (nextCard.gold) {
      if (goldCardCount <= 3) {
        setGoldCardCount(goldCardCount + 1);
      } else {
        setGoldCardCount(0);
        setRound(round + 1);
        if (round === 5) {
          alert('Good game!');
        }
      }
    }
  }

  // ...
```

这样做效率高得多。而且，如果你实现了查看游戏历史的功能，现在可以将每个状态变量设置为过去的某个操作，而不会触发调整其他值的 Effect 链。如果你需要在多个事件处理程序之间重用逻辑，可以 [提取成一个函数](#sharing-logic-between-event-handlers) 并从中调用。

请记住，在事件处理程序中，[state 的行为类似快照](/learn/state-as-a-snapshot)。 例如，即使你调用 `setRound(round + 1)`，`round` 变量仍会反映用户点击按钮时的值。如果你需要使用下一个值进行计算，请手动定义它，例如 `const nextRound = round + 1`。

在某些情况下，你 *不能* 直接在事件处理程序中计算下一个状态。例如，想象一个包含多个下拉框的表单，其中下一个下拉框的选项依赖于前一个下拉框的选定值。在这种情况下，链式 Effect 是合适的，因为你是在与网络同步。

### 初始化应用 {/*initializing-the-application*/}

有些逻辑只需要在应用加载时执行一次。

你可能会想将其放入顶层组件的 Effect 中：

```js {2-6}
function App() {
  // 🔴 避免：把只需要执行一次的逻辑放在 Effect 中
  useEffect(() => {
    loadDataFromLocalStorage();
    checkAuthToken();
  }, []);
  // ...
}
```

然而， 你会很快发现它在[开发环境会执行两次。](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development) 这可能会导致一些问题——例如，它可能使身份验证令牌失效，因为该函数不是为被调用两次而设计的。一般来说，当组件重新挂载时应该具有一致性。这包括你的顶层 `App` 组件。

尽管在生产环境中它可能不会被重新挂载，但在所有组件中遵循相同的约束使得代码更易于移动和重用。如果某些逻辑必须在 *每次应用加载时* 运行，而不是 *每次组件挂载时*，可以添加一个顶层变量来跟踪它是否已经执行：

```js {1,5-6,10}
let didInit = false;

function App() {
  useEffect(() => {
    if (!didInit) {
      didInit = true;
      // ✅ 只在每次应用加载时执行一次
      loadDataFromLocalStorage();
      checkAuthToken();
    }
  }, []);
  // ...
}
```

你也可以在模块初始化和应用渲染之前执行它：

```js {1,5}
if (typeof window !== 'undefined') { // 检测我们是否在浏览器环境
   // ✅ 只在每次应用加载时执行一次
  checkAuthToken();
  loadDataFromLocalStorage();
}

function App() {
  // ...
}
```

顶层的代码在组件被导入时运行一次——即使它最终没有被渲染。为了避免在导入任意组件时降低性能或产生意外行为，不要过度使用这种模式。将应用范围的初始化逻辑保留在像 `App.js` 这样的根组件模块中，或者放在应用的入口点中。

### 通知父组件有关 state 变化的信息 {/*notifying-parent-components-about-state-changes*/}

假设你正在编写一个 `Toggle` 组件，内部有一个 `isOn` 状态，可以是 `true` 或 `false`。有几种不同的方式来切换它（例如，点击或拖动）。你希望在 `Toggle` 内部状态变化时通知父组件，因此你暴露一个 `onChange` 事件，并在 Effect 中调用它：

```js {4-7}
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  // 🔴 避免：onChange 处理函数执行的时间太晚了
  useEffect(() => {
    onChange(isOn);
  }, [isOn, onChange])

  function handleClick() {
    setIsOn(!isOn);
  }

  function handleDragEnd(e) {
    if (isCloserToRightEdge(e)) {
      setIsOn(true);
    } else {
      setIsOn(false);
    }
  }

  // ...
}
```

这并不理想。`Toggle` 首先更新其状态，然后 React 更新屏幕。接着 React 运行 Effect，调用从父组件传递过来的 `onChange` 函数。此时父组件会更新自己的状态，开始另一次渲染过程。最好在一次渲染中完成所有操作。

删除 Effect，而是在同一个事件处理程序中更新 *两个* 组件的 state：

```js {5-7,11,16,18}
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  function updateToggle(nextIsOn) {
    // ✅ 好的: 在触发它们的事件中执行所有更新
    setIsOn(nextIsOn);
    onChange(nextIsOn); // 父组件收到onChange的调用，会更新父组件的state，此处虽然会setState两次，但是只会触发一次render（react会合并更新）
  }

  function handleClick() {
    updateToggle(!isOn);
  }

  function handleDragEnd(e) {
    if (isCloserToRightEdge(e)) {
      updateToggle(true);
    } else {
      updateToggle(false);
    }
  }

  // ...
}
```

通过这种方式，`Toggle` 组件和它的父组件会在事件期间更新它们的状态。React [批量更新/处理](/learn/queueing-a-series-of-state-updates) 来自不同组件的更新，因此只会有一次渲染过程。

你也可以完全移除状态，而是从父组件接收 `isOn`：

```js {1,2}
// ✅ 也很好：该组件完全由它的父组件控制
function Toggle({ isOn, onChange }) {
  function handleClick() {
    onChange(!isOn);
  }

  function handleDragEnd(e) {
    if (isCloserToRightEdge(e)) {
      onChange(true);
    } else {
      onChange(false);
    }
  }

  // ...
}
```

[“提升状态”](/learn/sharing-state-between-components) 让父组件通过切换自身的 state 来完全控制 `Toggle`。这意味着父组件需要包含更多的逻辑，但总体上需要关心的状态会更少。每当你试图保持两个不同的 state 变量之间的同步时，尝试提升状态而不是保持状态同步！

### 将数据传递给父组件 {/*passing-data-to-the-parent*/}

这个 `Child` 组件会获取一些数据，并在一个 Effect 中将其传递给 `Parent` 组件：

```js {9-14}
function Parent() {
  const [data, setData] = useState(null);
  // ...
  return <Child onFetched={setData} />;
}

function Child({ onFetched }) {
  const data = useSomeAPI();
  // 🔴 避免：在 Effect 中传递数据给父组件
  useEffect(() => {
    if (data) {
      onFetched(data);
    }
  }, [onFetched, data]);
  // ...
}
```

在 React 中，数据是从父组件流向子组件的。当你在屏幕上看到问题时，你可以通过一路追踪组件树来寻找错误信息是从哪个组件传递下来的，找到哪个组件传递了错误的 props 或 state。当子组件在 Effects 中更新父组件的 state 时，数据流就变得难以追踪。既然子组件和父组件都需要相同的数据，那么可以让父组件来获取这些数据，然后将其 *传递给* 子组件：

```js {4-5}
function Parent() {
  const data = useSomeAPI();
  // ...
  // ✅ 非常好：向子组件传递数据
  return <Child data={data} />;
}

function Child({ data }) {
  // ...
}
```

这样做更简单，并且保持了数据流的可预测性：数据从父组件流向子组件。

### 订阅外部 store {/*subscribing-to-an-external-store*/}

有时，组件可能需要订阅一些来自 React state 之外的数据。这些数据可能来自第三方库或浏览器的内置 API。由于这些数据可能在 React 无法感知的情况下发变化，你需要在你的组件中手动订阅它们。通常会使用 Effect 来完成这一点，例如：

```js {2-17}
function useOnlineStatus() {
  // 不理想：在 Effect 中手动订阅 store
  const [isOnline, setIsOnline] = useState(true);
  useEffect(() => {
    function updateState() {
      setIsOnline(navigator.onLine);
    }

    updateState();

    window.addEventListener('online', updateState);
    window.addEventListener('offline', updateState);
    return () => {
      window.removeEventListener('online', updateState);
      window.removeEventListener('offline', updateState);
    };
  }, []);
  return isOnline;
}

function ChatIndicator() {
  const isOnline = useOnlineStatus();
  // ...
}
```

在这个例子中，组件订阅了一个外部 store（这里是浏览器的 `navigator.onLine` API）。由于这个 API 在服务端上不存在（因此不能用于初始的 HTML），因此初始状态被设为 `true`。每当浏览器 store 中的值发生变化时，组件就会更新其状态。

虽然常用 Effects 来实现这一点，但 React 提供了一个专门用于订阅外部数据源的 Hook，推荐使用。你可以删除 Effect，并用 [`useSyncExternalStore`](/reference/react/useSyncExternalStore) 来将其替换：

```js {11-16}
function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  // ✅ 非常好：用内置的 Hook 订阅外部 store
  return useSyncExternalStore(
    subscribe, // 只要传递的是同一个函数，React 不会重新订阅
    () => navigator.onLine, // 如何在客户端获取值
    () => true // 如何在服务端获取值
  );
}

function ChatIndicator() {
  const isOnline = useOnlineStatus();
  // ...
}
```

与手动使用 Effect 将可变数据同步到 React state 相比，这种方法能减少错误。这种方法比手动将可变数据同步到 React 状态的 Effect 更少出错。通常，你会写一个像 `useOnlineStatus()` 这样的自定义 Hook，这样就不需要在各个组件中重复写这些代码。[了解更多关于从 React 组件订阅外部数据源的信息.](/reference/react/useSyncExternalStore)

### 获取数据 {/*fetching-data*/}

许多应用使用 Effect 来发起数据获取请求。像这样在 Effect 中写一个数据获取请求是相当常见的：

```js {5-10}
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const [page, setPage] = useState(1);

  useEffect(() => {
    // 🔴 避免：没有清除逻辑的获取数据
    fetchResults(query, page).then(json => {
      setResults(json);
    });
  }, [query, page]);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}
```

你 *不需要* 将这个获取操作放到事件处理函数中。

这看起来可能与之前需要把逻辑放到事件处理函数的例子相矛盾！但是，请考虑一下，获取数据的主要原因并不是 *输入事件*。搜索框的内容通常是从 URL 中预填充的，用户可能在不触碰输入框的情况下进行前后导航。

`page` 和 `query` 的来源并不重要。只要这个组件可见，你希望保持 `results` [与网络数据同步](/learn/synchronizing-with-effects)，以适应当前的 `page` 和 `query`。这就是为什么这部分代码是一个 Effect。

然而，上面的代码存在一个问题。想象一下，你快速输入 `"hello"`。此时 `query` 将会从 `"h"` 变为 `"he"`、`"hel"`、`"hell"` 和 `"hello"`。这会触发多个独立的获取请求，但无法保证响应到达的顺序。例如，`"hell"` 的响应可能在 `"hello"` 的响应 *之后* 到达。由于它最后调用 `setResults()`，你将显示错误的搜索结果。这种情况被称为 ["竞态条件"](https://en.wikipedia.org/wiki/Race_condition)：两个不同的请求“竞速”并以不同于预期的顺序到达。

**为了修复这个问题，你需要添加一个 [清理函数](/learn/synchronizing-with-effects#fetching-data) 来忽略过时的响应（较早的返回结果）：**

```js {5,7,9,11-13}
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const [page, setPage] = useState(1);
  useEffect(() => {
    let ignore = false;
    fetchResults(query, page).then(json => {
      if (!ignore) {
        setResults(json);
      }
    });
    return () => {
      ignore = true;
    };
  }, [query, page]);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}
```

这样可以确保当你的 Effect 获取数据时，除了最后请求的响应外，其他所有响应都将被忽略。

处理竞态条件并不是实现数据获取的唯一挑战。你还需要考虑如何缓存响应（让用户点击返回时能够立刻看到之前的内容）、如何在服务器上获取数据（让初始服务器渲染的 HTML 包含获取的内容，而不是加载指示器），以及如何避免网络瀑布（让子组件能够在不等待所有父组件的情况下获取数据）。

**这些问题适用于任何 UI 库，而不仅仅是 React。解决这些问题并不简单，这也是为什么现代 [框架](/learn/start-a-new-react-project#production-grade-react-frameworks) 提供了比在 Effects 中获取数据更高效的内置数据获取机制的原因。**

如果你不使用框架（也不想自己构建一个），但希望使从 Effects 中获取数据更方便，可以考虑将获取逻辑提取到自定义 Hook 中，如下面的示例所示：

```js {4}
function SearchResults({ query }) {
  const [page, setPage] = useState(1);
  const params = new URLSearchParams({ query, page });
  const results = useData(`/api/search?${params}`);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}

function useData(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    let ignore = false;
    fetch(url)
      .then(response => response.json())
      .then(json => {
        if (!ignore) {
          setData(json);
        }
      });
    return () => {
      ignore = true;
    };
  }, [url]);
  return data;
}
```

你可能还想添加一些错误处理的逻辑，并跟踪内容是否正在加载。你可以自己构建这样的 Hook，或者使用 React 生态系统中已有的许多解决方案之一。**虽然这样做的效率不如使用框架的内置数据获取机制，但将数据获取逻辑移动到自定义 Hook 中将让你更容易采用高效的数据获取策略。**

总的来说，每当你不得不写 Effects 时，请注意是否可以将某个功能提取到一个更具声明性的自定义 Hook 中和专门化的 API 的自定义 Hook 中，例如上面的 `useData`。你的组件中原始的 `useEffect` 调用越少，维护应用程序就会越容易。

<Recap>

- 如果你可以在渲染时计算某些内容，就不需要 Effect。
- 要缓存开销较大的计算，使用 `useMemo` 而不是 `useEffect`。
- 想要重置整个组件树的 state，请传入不同的 `key`。
- 想要在 prop 变化时重置某些特定的 state，请在渲染期间处理。
- 组件 *显示* 时就需要执行的代码应该放在 Effect 中，否则应该放在事件处理函数中。
- 如果你需要更新多个组件的 state，最好在单个事件处理函数中处理。
- 当你尝试在不同组件中同步 state 变量时，请考虑状态提升。
- 你可以使用 Effect 获取数据，但你需要实现清除逻辑以避免竞态条件。

</Recap>

<Challenges>

#### 在不使用 Effects 的情况下转换数据 {/*transform-data-without-effects*/}

下面的 `TodoList` 组件展示了一个待办事项列表。当“只显示未完成的事项”复选框被勾选时，已完成的待办事项不会显示在列表中。无论哪些待办事项可见，页脚都会显示尚未完成的待办事项的计数。

通过移除不必要的 state 和 Effect 来简化这个组件。

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { initialTodos, createTodo } from './todos.js';

export default function TodoList() {
  const [todos, setTodos] = useState(initialTodos);
  const [showActive, setShowActive] = useState(false);
  const [activeTodos, setActiveTodos] = useState([]);
  const [visibleTodos, setVisibleTodos] = useState([]);
  const [footer, setFooter] = useState(null);

  useEffect(() => {
    setActiveTodos(todos.filter(todo => !todo.completed));
  }, [todos]);

  useEffect(() => {
    setVisibleTodos(showActive ? activeTodos : todos);
  }, [showActive, todos, activeTodos]);

  useEffect(() => {
    setFooter(
      <footer>
        {activeTodos.length} todos left
      </footer>
    );
  }, [activeTodos]);

  return (
    <>
      <label>
        <input
          type="checkbox"
          checked={showActive}
          onChange={e => setShowActive(e.target.checked)}
        />
        只显示未完成的事项
      </label>
      <NewTodo onAdd={newTodo => setTodos([...todos, newTodo])} />
      <ul>
        {visibleTodos.map(todo => (
          <li key={todo.id}>
            {todo.completed ? <s>{todo.text}</s> : todo.text}
          </li>
        ))}
      </ul>
      {footer}
    </>
  );
}

function NewTodo({ onAdd }) {
  const [text, setText] = useState('');

  function handleAddClick() {
    setText('');
    onAdd(createTodo(text));
  }

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAddClick}>
        Add
      </button>
    </>
  );
}
```

```js src/todos.js
let nextId = 0;

export function createTodo(text, completed = false) {
  return {
    id: nextId++,
    text,
    completed
  };
}

export const initialTodos = [
  createTodo('Get apples', true),
  createTodo('Get oranges', true),
  createTodo('Get carrots'),
];
```

```css
label { display: block; }
input { margin-top: 10px; }
```

</Sandpack>

<Hint>

如果你可以在渲染期间计算出某些值，那么就不需要使用 state 或 Effect 来更新它。

</Hint>

<Solution>

这个例子中只有两个必要的 state 变量：`todos` 列表和表示复选框是否被选中的 `showActive` state 变量。所有其他 state 变量都是 [冗余的](/learn/choosing-the-state-structure#avoid-redundant-state)，可以在渲染期间计算。这包括你可以直接搬到 JSX 中的 `footer`。

最终你得到的结果应该是这样的：

<Sandpack>

```js
import { useState } from 'react';
import { initialTodos, createTodo } from './todos.js';

export default function TodoList() {
  const [todos, setTodos] = useState(initialTodos);
  const [showActive, setShowActive] = useState(false);
  const activeTodos = todos.filter(todo => !todo.completed);
  const visibleTodos = showActive ? activeTodos : todos;

  return (
    <>
      <label>
        <input
          type="checkbox"
          checked={showActive}
          onChange={e => setShowActive(e.target.checked)}
        />
        只显示未完成的事项
      </label>
      <NewTodo onAdd={newTodo => setTodos([...todos, newTodo])} />
      <ul>
        {visibleTodos.map(todo => (
          <li key={todo.id}>
            {todo.completed ? <s>{todo.text}</s> : todo.text}
          </li>
        ))}
      </ul>
      <footer>
        {activeTodos.length} todos left
      </footer>
    </>
  );
}

function NewTodo({ onAdd }) {
  const [text, setText] = useState('');

  function handleAddClick() {
    setText('');
    onAdd(createTodo(text));
  }

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAddClick}>
        Add
      </button>
    </>
  );
}
```

```js src/todos.js
let nextId = 0;

export function createTodo(text, completed = false) {
  return {
    id: nextId++,
    text,
    completed
  };
}

export const initialTodos = [
  createTodo('Get apples', true),
  createTodo('Get oranges', true),
  createTodo('Get carrots'),
];
```

```css
label { display: block; }
input { margin-top: 10px; }
```

</Sandpack>

</Solution>

#### 在不使用 Effects 的情况下缓存计算 {/*cache-a-calculation-without-effects*/}

在这个例子中，过滤待办事项的逻辑被提取到一个名为 `getVisibleTodos()` 的单独函数中。这个函数内部有一个 `console.log()` 调用，可以帮助你注意到它何时被调用。切换“只显示未完成的事项”这会导致 `getVisibleTodos()` 重新执行。这是预期中的，因为可见的待办事项会因你切换显示的待办事项而改变。

你的任务是删除 `TodoList` 组件中重新计算 `visibleTodos` 列表的 Effect。但是，你需要确保在输入框中输入时，`getVisibleTodos()` *不会* 重新执行（因此不打印任何日志）。

<Hint>

一个解决方案是添加一个 `useMemo` 调用来缓存可见的待办事项。还有另一种不那么明显的解决方案。

</Hint>

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { initialTodos, createTodo, getVisibleTodos } from './todos.js';

export default function TodoList() {
  const [todos, setTodos] = useState(initialTodos);
  const [showActive, setShowActive] = useState(false);
  const [text, setText] = useState('');
  const [visibleTodos, setVisibleTodos] = useState([]);

  useEffect(() => {
    setVisibleTodos(getVisibleTodos(todos, showActive));
  }, [todos, showActive]);

  function handleAddClick() {
    setText('');
    setTodos([...todos, createTodo(text)]);
  }

  return (
    <>
      <label>
        <input
          type="checkbox"
          checked={showActive}
          onChange={e => setShowActive(e.target.checked)}
        />
        Show only active todos
      </label>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAddClick}>
        Add
      </button>
      <ul>
        {visibleTodos.map(todo => (
          <li key={todo.id}>
            {todo.completed ? <s>{todo.text}</s> : todo.text}
          </li>
        ))}
      </ul>
    </>
  );
}
```

```js src/todos.js
let nextId = 0;
let calls = 0;

export function getVisibleTodos(todos, showActive) {
  console.log(`getVisibleTodos() was called ${++calls} times`);
  const activeTodos = todos.filter(todo => !todo.completed);
  const visibleTodos = showActive ? activeTodos : todos;
  return visibleTodos;
}

export function createTodo(text, completed = false) {
  return {
    id: nextId++,
    text,
    completed
  };
}

export const initialTodos = [
  createTodo('Get apples', true),
  createTodo('Get oranges', true),
  createTodo('Get carrots'),
];
```

```css
label { display: block; }
input { margin-top: 10px; }
```

</Sandpack>

<Solution>

移除 state 和 Effect，而是添加一个 `useMemo` 来缓存调用 `getVisibleTodos()` 的结果：

<Sandpack>

```js
import { useState, useMemo } from 'react';
import { initialTodos, createTodo, getVisibleTodos } from './todos.js';

export default function TodoList() {
  const [todos, setTodos] = useState(initialTodos);
  const [showActive, setShowActive] = useState(false);
  const [text, setText] = useState('');
  const visibleTodos = useMemo(
    () => getVisibleTodos(todos, showActive),
    [todos, showActive]
  );

  function handleAddClick() {
    setText('');
    setTodos([...todos, createTodo(text)]);
  }

  return (
    <>
      <label>
        <input
          type="checkbox"
          checked={showActive}
          onChange={e => setShowActive(e.target.checked)}
        />
        Show only active todos
      </label>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAddClick}>
        Add
      </button>
      <ul>
        {visibleTodos.map(todo => (
          <li key={todo.id}>
            {todo.completed ? <s>{todo.text}</s> : todo.text}
          </li>
        ))}
      </ul>
    </>
  );
}
```

```js src/todos.js
let nextId = 0;
let calls = 0;

export function getVisibleTodos(todos, showActive) {
  console.log(`getVisibleTodos() was called ${++calls} times`);
  const activeTodos = todos.filter(todo => !todo.completed);
  const visibleTodos = showActive ? activeTodos : todos;
  return visibleTodos;
}

export function createTodo(text, completed = false) {
  return {
    id: nextId++,
    text,
    completed
  };
}

export const initialTodos = [
  createTodo('Get apples', true),
  createTodo('Get oranges', true),
  createTodo('Get carrots'),
];
```

```css
label { display: block; }
input { margin-top: 10px; }
```

</Sandpack>

通过这些改动，`getVisibleTodos()` 只会在 `todos` 或 `showActive` 更改时被调用。输入框中的输入只会更改 `text` 变量的值，因此不会触发对 `getVisibleTodos()` 的调用。

还有另一种不需要 `useMemo` 的解决方案。由于 `text` 状态变量不可能影响待办事项列表，你可以将 `NewTodo` 表单提取到一个单独的组件中，并将 `text` 状态变量放到其中：

<Sandpack>

```js
import { useState, useMemo } from 'react';
import { initialTodos, createTodo, getVisibleTodos } from './todos.js';

export default function TodoList() {
  const [todos, setTodos] = useState(initialTodos);
  const [showActive, setShowActive] = useState(false);
  const visibleTodos = getVisibleTodos(todos, showActive);

  return (
    <>
      <label>
        <input
          type="checkbox"
          checked={showActive}
          onChange={e => setShowActive(e.target.checked)}
        />
        Show only active todos
      </label>
      <NewTodo onAdd={newTodo => setTodos([...todos, newTodo])} />
      <ul>
        {visibleTodos.map(todo => (
          <li key={todo.id}>
            {todo.completed ? <s>{todo.text}</s> : todo.text}
          </li>
        ))}
      </ul>
    </>
  );
}

function NewTodo({ onAdd }) {
  const [text, setText] = useState('');

  function handleAddClick() {
    setText('');
    onAdd(createTodo(text));
  }

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAddClick}>
        Add
      </button>
    </>
  );
}
```

```js src/todos.js
let nextId = 0;
let calls = 0;

export function getVisibleTodos(todos, showActive) {
  console.log(`getVisibleTodos() was called ${++calls} times`);
  const activeTodos = todos.filter(todo => !todo.completed);
  const visibleTodos = showActive ? activeTodos : todos;
  return visibleTodos;
}

export function createTodo(text, completed = false) {
  return {
    id: nextId++,
    text,
    completed
  };
}

export const initialTodos = [
  createTodo('Get apples', true),
  createTodo('Get oranges', true),
  createTodo('Get carrots'),
];
```

```css
label { display: block; }
input { margin-top: 10px; }
```

</Sandpack>

这种方法同样满足要求。当你在输入框中输入时，只有 `text` 变量更新。由于 `text` 变量在子组件 `NewTodo` 中，父组件 `TodoList` 将不会重新渲染。这就是为什么在输入时 `getVisibleTodos()` 不会被调用的原因。（如果 `TodoList` 因其他原因重新渲染，它仍会被调用。）

</Solution>

#### 不用 Effect 重置 state {/*reset-state-without-effects*/}

`EditContact` 组件的 props 接收一个结构为 `{ id, name, email }` 的联系人对象作为 `savedContact` 属性。尝试编辑名称和邮箱的输入框。当你点击保存时，表单上方的联系人按钮会更新为你编辑后的名称；而点击重置则会丢弃表单中的任何未保存更改。可以多试试这个界面，熟悉一下它的操作。

当你通过顶部的按钮选择一个联系人时，表单会重置以显示该联系人的信息。这是通过 `EditContact.js` 中的一个 Effect 实现的。现在，去掉这个 Effect，找到另一种方法，在 `savedContact.id` 更改时重置表单。

<Sandpack>

```js src/App.js hidden
import { useState } from 'react';
import ContactList from './ContactList.js';
import EditContact from './EditContact.js';

export default function ContactManager() {
  const [
    contacts,
    setContacts
  ] = useState(initialContacts);
  const [
    selectedId,
    setSelectedId
  ] = useState(0);
  const selectedContact = contacts.find(c =>
    c.id === selectedId
  );

  function handleSave(updatedData) {
    const nextContacts = contacts.map(c => {
      if (c.id === updatedData.id) {
        return updatedData;
      } else {
        return c;
      }
    });
    setContacts(nextContacts);
  }

  return (
    <div>
      <ContactList
        contacts={contacts}
        selectedId={selectedId}
        onSelect={id => setSelectedId(id)}
      />
      <hr />
      <EditContact
        savedContact={selectedContact}
        onSave={handleSave}
      />
    </div>
  )
}

const initialContacts = [
  { id: 0, name: 'Taylor', email: 'taylor@mail.com' },
  { id: 1, name: 'Alice', email: 'alice@mail.com' },
  { id: 2, name: 'Bob', email: 'bob@mail.com' }
];
```

```js src/ContactList.js hidden
export default function ContactList({
  contacts,
  selectedId,
  onSelect
}) {
  return (
    <section>
      <ul>
        {contacts.map(contact =>
          <li key={contact.id}>
            <button onClick={() => {
              onSelect(contact.id);
            }}>
              {contact.id === selectedId ?
                <b>{contact.name}</b> :
                contact.name
              }
            </button>
          </li>
        )}
      </ul>
    </section>
  );
}
```

```js src/EditContact.js active
import { useState, useEffect } from 'react';

export default function EditContact({ savedContact, onSave }) {
  const [name, setName] = useState(savedContact.name);
  const [email, setEmail] = useState(savedContact.email);

  useEffect(() => {
    setName(savedContact.name);
    setEmail(savedContact.email);
  }, [savedContact]);

  return (
    <section>
      <label>
        Name:{' '}
        <input
          type="text"
          value={name}
          onChange={e => setName(e.target.value)}
        />
      </label>
      <label>
        Email:{' '}
        <input
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
        />
      </label>
      <button onClick={() => {
        const updatedData = {
          id: savedContact.id,
          name: name,
          email: email
        };
        onSave(updatedData);
      }}>
        Save
      </button>
      <button onClick={() => {
        setName(savedContact.name);
        setEmail(savedContact.email);
      }}>
        Reset
      </button>
    </section>
  );
}
```

```css
ul, li {
  list-style: none;
  margin: 0;
  padding: 0;
}
li { display: inline-block; }
li button {
  padding: 10px;
}
label {
  display: block;
  margin: 10px 0;
}
button {
  margin-right: 10px;
  margin-bottom: 10px;
}
```

</Sandpack>

<Hint>

如果有办法告诉 React，当 `savedContact.id` 变化时，`EditContact` 表单实际上是一个 _不同联系人的表单_，并且不应保留之前的状态，那就太好了。你还记得有什么方法可以做到这一点吗？

</Hint>

<Solution>

将 `EditContact` 组件拆分为两个部分。把所有表单状态移到内部的 `EditForm` 组件中。导出外部的 `EditContact` 组件，并让它把 `savedContact.id` 作为 `key` 传递给内部的 `EditForm` 组件。这样，每当你选择不同的联系人时，内部的 `EditForm` 组件都会重置所有表单状态并重新生成 DOM。

<Sandpack>

```js src/App.js hidden
import { useState } from 'react';
import ContactList from './ContactList.js';
import EditContact from './EditContact.js';

export default function ContactManager() {
  const [
    contacts,
    setContacts
  ] = useState(initialContacts);
  const [
    selectedId,
    setSelectedId
  ] = useState(0);
  const selectedContact = contacts.find(c =>
    c.id === selectedId
  );

  function handleSave(updatedData) {
    const nextContacts = contacts.map(c => {
      if (c.id === updatedData.id) {
        return updatedData;
      } else {
        return c;
      }
    });
    setContacts(nextContacts);
  }

  return (
    <div>
      <ContactList
        contacts={contacts}
        selectedId={selectedId}
        onSelect={id => setSelectedId(id)}
      />
      <hr />
      <EditContact
        savedContact={selectedContact}
        onSave={handleSave}
      />
    </div>
  )
}

const initialContacts = [
  { id: 0, name: 'Taylor', email: 'taylor@mail.com' },
  { id: 1, name: 'Alice', email: 'alice@mail.com' },
  { id: 2, name: 'Bob', email: 'bob@mail.com' }
];
```

```js src/ContactList.js hidden
export default function ContactList({
  contacts,
  selectedId,
  onSelect
}) {
  return (
    <section>
      <ul>
        {contacts.map(contact =>
          <li key={contact.id}>
            <button onClick={() => {
              onSelect(contact.id);
            }}>
              {contact.id === selectedId ?
                <b>{contact.name}</b> :
                contact.name
              }
            </button>
          </li>
        )}
      </ul>
    </section>
  );
}
```

```js src/EditContact.js active
import { useState } from 'react';

export default function EditContact(props) {
  return (
    <EditForm
      {...props}
      key={props.savedContact.id}
    />
  );
}

function EditForm({ savedContact, onSave }) {
  const [name, setName] = useState(savedContact.name);
  const [email, setEmail] = useState(savedContact.email);

  return (
    <section>
      <label>
        Name:{' '}
        <input
          type="text"
          value={name}
          onChange={e => setName(e.target.value)}
        />
      </label>
      <label>
        Email:{' '}
        <input
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
        />
      </label>
      <button onClick={() => {
        const updatedData = {
          id: savedContact.id,
          name: name,
          email: email
        };
        onSave(updatedData);
      }}>
        Save
      </button>
      <button onClick={() => {
        setName(savedContact.name);
        setEmail(savedContact.email);
      }}>
        Reset
      </button>
    </section>
  );
}
```

```css
ul, li {
  list-style: none;
  margin: 0;
  padding: 0;
}
li { display: inline-block; }
li button {
  padding: 10px;
}
label {
  display: block;
  margin: 10px 0;
}
button {
  margin-right: 10px;
  margin-bottom: 10px;
}
```

</Sandpack>

</Solution>

#### 不用 Effect 提交表单 {/*submit-a-form-without-effects*/}

这个 `Form` 组件可以让你向朋友发送消息。当你提交表单时，`showForm` state 会被设置为 `false`。这会触发一个 Effect，调用 `sendMessage(message)` 来发送消息（你可以在控制台看到发送的内容）。消息发送后，你会看到一个“谢谢”的对话框，里面有一个“打开聊天”的按钮，可以让你返回到表单。

你的应用的用户发送的消息太多了。为了让聊天变得稍微困难一些，你决定先显示“谢谢”的提示语，而不是表单。将 `showForm` 状态变量的初始值设置为 `false`，而不是 `true`。一旦你这样做，控制台就会显示发送了一个空消息。这说明逻辑上有问题！

这个问题的根本原因是什么？你该如何解决？

<Hint>

是 _因为_ 用户看到了 “谢谢” 提示语，才应该发送消息吗？还是其他什么原因？

</Hint>

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function Form() {
  const [showForm, setShowForm] = useState(true);
  const [message, setMessage] = useState('');

  useEffect(() => {
    if (!showForm) {
      sendMessage(message);
    }
  }, [showForm, message]);

  function handleSubmit(e) {
    e.preventDefault();
    setShowForm(false);
  }

  if (!showForm) {
    return (
      <>
        <h1>Thanks for using our services!</h1>
        <button onClick={() => {
          setMessage('');
          setShowForm(true);
        }}>
          Open chat
        </button>
      </>
    );
  }

  return (
    <form onSubmit={handleSubmit}>
      <textarea
        placeholder="Message"
        value={message}
        onChange={e => setMessage(e.target.value)}
      />
      <button type="submit" disabled={message === ''}>
        Send
      </button>
    </form>
  );
}

function sendMessage(message) {
  console.log('Sending message: ' + message);
}
```

```css
label, textarea { margin-bottom: 10px; display: block; }
```

</Sandpack>

<Solution>

`showForm` 状态变量决定显示表单还是“谢谢”的对话框。然而，你并不是因为“谢谢”的对话框被 _显示_ 而发送消息。你想要的是在用户 _提交表单_ 的时候发送消息。删除那个误导的 Effect，将 `sendMessage` 的调用放到 `handleSubmit` 事件处理程序内部：

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function Form() {
  const [showForm, setShowForm] = useState(true);
  const [message, setMessage] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    setShowForm(false);
    sendMessage(message);
  }

  if (!showForm) {
    return (
      <>
        <h1>Thanks for using our services!</h1>
        <button onClick={() => {
          setMessage('');
          setShowForm(true);
        }}>
          Open chat
        </button>
      </>
    );
  }

  return (
    <form onSubmit={handleSubmit}>
      <textarea
        placeholder="Message"
        value={message}
        onChange={e => setMessage(e.target.value)}
      />
      <button type="submit" disabled={message === ''}>
        Send
      </button>
    </form>
  );
}

function sendMessage(message) {
  console.log('Sending message: ' + message);
}
```

```css
label, textarea { margin-bottom: 10px; display: block; }
```

</Sandpack>

`showForm` 状态变量决定显示表单还是“谢谢”的对话框。然而，你并不是因为“谢谢”的对话框被 _显示_ 而发送消息。你想要的是在用户 _提交表单_ 的时候发送消息。删除那个误导的 Effect，将 `sendMessage` 的调用放到 `handleSubmit` 事件处理程序内部：

</Solution>

</Challenges>
