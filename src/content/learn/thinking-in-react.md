---
title: React 哲学
---

<Intro>

React 可以改变你对设计和应用开发的思维方式。当你使用 React 创建用户界面时，首先需要将界面拆分成称为组件的小块。接着，你将为每个组件描述不同的视觉状态。最后，你会将这些组件连接起来，以便数据能够在它们之间流动。在本教程中，我们将引导你思考如何用 React 构建一个可搜索的产品数据表。

</Intro>

## 从原型开始 {/*start-with-the-mockup*/}

假设你已经有一个 JSON API 和设计师提供的模型。

这个 JSON API 返回的数据大致是这样的：

```json
[
  { category: "Fruits", price: "$1", stocked: true, name: "Apple" },
  { category: "Fruits", price: "$1", stocked: true, name: "Dragonfruit" },
  { category: "Fruits", price: "$2", stocked: false, name: "Passionfruit" },
  { category: "Vegetables", price: "$2", stocked: true, name: "Spinach" },
  { category: "Vegetables", price: "$4", stocked: false, name: "Pumpkin" },
  { category: "Vegetables", price: "$1", stocked: true, name: "Peas" }
]
```

而这个原型则看起来像这样：

<img src="/images/docs/s_thinking-in-react_ui.png" width="300" style={{margin: '0 auto'}} />

仅需跟随下面的五步，即可使用 React 来实现 UI。

## 步骤 1：将 UI 拆分为组件层级 {/*step-1-break-the-ui-into-a-component-hierarchy*/}

首先，在原型图上为每个组件和子组件画框，并给它们命名。如果你与设计师一起工作，他们可能早已在其设计工具中对这些组件进行了命名。检查一下它们!

取决于你的使用背景，可以考虑通过不同的方式将设计分割为组件：

* **程序设计**——使用相同的思维方式来判断是否应该创建新的函数或对象。一种方法是 [单一职责原则](https://en.wikipedia.org/wiki/Single_responsibility_principle)，也就是说，一个组件理想上只应该负责一件事情。如果它变得复杂，就需要拆分成更小的子组件。
* **CSS**——考虑你会为哪些内容创建类选择器。（不过，组件的粒度通常会稍微大一些。）
* **设计**——思考你将如何组织布局的层级。

如果你的 JSON 数据结构设计合理，通常能很自然地映射到用户界面的组件结构。这是因为 UI 和数据模型往往有相同的信息架构——即相似的结构。将你的用户界面分解成组件，每个组件对应数据模型中的一部分。

以下展示了五个组件:

<FullWidth>

<CodeDiagram flip>

  <img src="/images/docs/s_thinking-in-react_ui_outline.png" width="500" style={{margin: '0 auto'}} />

1. `FilterableProductTable`（灰色）是整个应用的容器。
2. `SearchBar`（蓝色）负责接收用户输入。
3. `ProductTable`（淡紫色）根据用户输入显示和过滤产品列表。
4. `ProductCategoryRow`（绿色）为每个产品类别显示标题。
5. `ProductRow`（黄色）显示每个产品的详细信息。

</CodeDiagram>

</FullWidth>

如果你查看 `ProductTable`（淡紫色），会发现表头（包含 "Name" 和 "Price" 标签）并不是一个独立的组件。这主要取决于你的个人偏好，你可以选择任意一种方式。在这个例子中，它是 `ProductTable` 的一部分，因为它在 `ProductTable` 的列表中。然而，如果表头变得复杂（例如，添加排序功能），你可以将其拆分成一个新的 `ProductTableHeader` 组件。

现在你已经在原型中辨别了组件，并将它们转化为了层级结构。在原型中，组件可以展示在其它组件之中，在层级结构中如同其孩子一般：

* `FilterableProductTable`
    * `SearchBar`
    * `ProductTable`
        * `ProductCategoryRow`
        * `ProductRow`

## 步骤二：使用 React 构建一个静态版本 {/*step-2-build-a-static-version-in-react*/}

现在你有了组件层级，是时候实现你的应用了。最直接的办法是根据你的数据模型，构建一个不带任何交互的 UI 渲染代码版本，通常先构建静态版本再添加交互功能会更容易。构建静态版本需要大量的代码而不需思考，但添加交互功能则需要深入思考而不需要太多代码。

构建应用程序的静态版本来渲染你的数据模型，你需要构建 [组件](/learn/your-first-component)，重用其他组件并通过 [props](/learn/passing-props-to-a-component) 传递数据。Props 是从父组件向子组件传递数据的方式。（如果你熟悉 [状态](/learn/state-a-components-memory) 的概念，不要在静态版本中使用 state 进行构建。state 只用于处理交互性的数据，即随时间变化的数据。因为这是静态版本的应用，所以不需要使用 state。）

你可以选择“自上而下”构建，从层级较高的组件（如 `FilterableProductTable`）开始，或者“自下而上”构建，从层级较低的组件（如 `ProductRow`）开始。在简单的例子中，通常自上而下更容易，而在较大的项目中，自下而上更有效。

<Sandpack>

```jsx src/App.js
function ProductCategoryRow({ category }) {
  return (
    <tr>
      <th colSpan="2">
        {category}
      </th>
    </tr>
  );
}

function ProductRow({ product }) {
  const name = product.stocked ? product.name :
    <span style={{ color: 'red' }}>
      {product.name}
    </span>;

  return (
    <tr>
      <td>{name}</td>
      <td>{product.price}</td>
    </tr>
  );
}

function ProductTable({ products }) {
  const rows = [];
  let lastCategory = null;

  products.forEach((product) => {
    if (product.category !== lastCategory) {
      rows.push(
        <ProductCategoryRow
          category={product.category}
          key={product.category} />
      );
    }
    rows.push(
      <ProductRow
        product={product}
        key={product.name} />
    );
    lastCategory = product.category;
  });

  return (
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Price</th>
        </tr>
      </thead>
      <tbody>{rows}</tbody>
    </table>
  );
}

function SearchBar() {
  return (
    <form>
      <input type="text" placeholder="Search..." />
      <label>
        <input type="checkbox" />
        {' '}
        Only show products in stock
      </label>
    </form>
  );
}

function FilterableProductTable({ products }) {
  return (
    <div>
      <SearchBar />
      <ProductTable products={products} />
    </div>
  );
}

const PRODUCTS = [
  {category: "Fruits", price: "$1", stocked: true, name: "Apple"},
  {category: "Fruits", price: "$1", stocked: true, name: "Dragonfruit"},
  {category: "Fruits", price: "$2", stocked: false, name: "Passionfruit"},
  {category: "Vegetables", price: "$2", stocked: true, name: "Spinach"},
  {category: "Vegetables", price: "$4", stocked: false, name: "Pumpkin"},
  {category: "Vegetables", price: "$1", stocked: true, name: "Peas"}
];

export default function App() {
  return <FilterableProductTable products={PRODUCTS} />;
}
```

```css
body {
  padding: 5px
}
label {
  display: block;
  margin-top: 5px;
  margin-bottom: 5px;
}
th {
  padding-top: 10px;
}
td {
  padding: 2px;
  padding-right: 40px;
}
```

</Sandpack>

（如果这段代码让你感到困惑，建议先浏览 [快速入门](/learn/)！）

完成组件的构建后，你将拥有一个可重用组件的库来渲染你的数据模型。由于这是一个静态应用，组件只会返回 JSX。层级顶部的组件（`FilterableProductTable`）将作为 prop 接收你的数据模型。这种方式被称为 _单向数据流_，因为数据从顶层组件向下流动到树形结构底部的组件。

<Pitfall>

在这部分中，你不需要使用任何 state，这是下一步的内容！

</Pitfall>

## 步骤 3：找到 UI 状态的最小完整表示 {/*step-3-find-the-minimal-but-complete-representation-of-ui-state*/}

为了使 UI 可交互，你需要用户更改潜在的数据结构。你将可以使用 *state* 进行实现。

可以将 state 理解为应用需要记住的最小可变数据集合。构建 state 时，最重要的原则是要遵循 [DRY（不要重复自己）](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)。你需要找出应用所需状态的最小表示，并按需计算其他数据。例如，如果你在制作一个购物清单，你可将他们在 state 中存储为数组。如果你想显示清单中的商品数量，不必单独存储这个数量，而是可以通过读取数组的长度来获取。

现在考虑示例应用程序中的每一条数据:

1. 原始产品列表
2. 用户输入的搜索文本
3. 复选框的值
4. 过滤后的产品列表

其中哪些是 state 呢？标记出那些不是的:

* 是否有数据是 **保持不变** 的？如果是，那它就不是 state。
* 是否有数据是 **通过 props 从父组件传入** 的？如果是，那它也不是 state。
* 你 **能否根据现有的状态或 props 计算出这个数据**？如果可以，那它 *肯定* 不是 state！

剩下的可能是 state。

让我们逐一分析一下：

1. 原始产品列表 **是通过 props 传入的，因此它不是 state。**
2. 搜索文本似乎应该是 state，因为它会随时间变化，且无法从其他信息计算得出。
3. 复选框的值似乎是 state，因为它同样会变化，且无法计算。
4. 过滤后的产品列表 **不是 state，因为它可以通过** 原始产品列表与搜索文本及复选框值进行过滤来计算。

这就意味着只有搜索文本和复选框的值是 state！非常好！

<DeepDive>

#### Props vs State {/*props-vs-state*/}

在 React 中有两种类型的 "模型" 数据：props 和状态。它们有着显著的不同：

* [**Props 就像是你传递给函数的参数**](/learn/passing-props-to-a-component)。它们允许父组件向子组件传递数据，举个例子，例如，`Form` 可以将 `color` props 传递给 `Button`。
* [**状态则像是组件的内存**](/learn/state-a-components-memory)。它允许组件跟踪某些信息，并根据用户的交互进行更改。例如，`Button` 可以跟踪 `isHovered` state。

尽管 props 和 state 不同，但它们是相辅相成的。父组件通常会将一些信息放在 state 中（以便能进行更改），然后 *将这些信息传递给* 子组件作为 props。如果你在第一次阅读时对这两者的区别感到模糊也没关系，理解这个概念需要一些时间和练习！

</DeepDive>

## 验证 state 应该被放置在哪里 {/*step-4-identify-where-your-state-should-live*/}

在识别出应用的最小状态数据后，你需要找出哪个组件负责修改这些状态，或者说 *拥有* 这些状态。请记住：React 使用单向数据流，从父组件向子组件传递数据。有时候，哪个组件应该拥有哪个状态并不总是一目了然。如果你对这个概念不太熟悉，可能会有些困难，但你可以通过以下步骤来理清思路！

对于应用中的每个状态：

1. 确定 *每个* 根据该状态渲染内容的组件。
2. 找到它们最近的共同父组件——在层次结构中位于它们之上的组件。
3. 决定状态应该存放在哪里：
    1. 通常，你可以将状态直接放入它们的共同父组件中。
    2. 你也可以将状态放入共同父组件之上的某个组件中。
    3. 如果找不到合适的组件来拥有该状态，可以创建一个新组件专门用于保存状态，并将其放在共同父组件之上的某个层次中。

在上一步中，你发现了两个状态：搜索输入文本和复选框的值。在这个示例中，它们总是一起出现，因此将它们放在同一位置是合适的。

现在为这个 state 贯彻我们的策略：

1. **验证使用 state 的组件：**
    * `ProductTable` 需要基于 state (搜索文本和复选框值) 过滤产品列表。
    * `SearchBar` 需要展示 state (搜索文本和复选框值)。
2. **找到它们的共同父组件：** 这两个组件共有的第一个父组件是 `FilterableProductTable`。
3. **决定 state 放置的地方**：我们将在 `FilterableProductTable` 中保存过滤文本和复选框的状态值。

因此，状态值将存放在 `FilterableProductTable` 中。

接下来，使用 [`useState()` Hook](/reference/react/useState) 向组件添加状态。Hooks 是特殊的函数，允许你“连接”到 React。在 `FilterableProductTable` 的顶部添加两个状态变量，并指定它们的初始状态：

```js
function FilterableProductTable({ products }) {
  const [filterText, setFilterText] = useState('');
  const [inStockOnly, setInStockOnly] = useState(false);  
```

然后，`filterText` 和 `inStockOnly` 作为 props 传递至 `ProductTable` 和 `SearchBar`：

```js
<div>
  <SearchBar 
    filterText={filterText} 
    inStockOnly={inStockOnly} />
  <ProductTable 
    products={products}
    filterText={filterText}
    inStockOnly={inStockOnly} />
</div>
```

你可以开始观察你的应用是如何运行的。将 `useState('')` 中的 `filterText` 初始值修改为 `useState('fruit')`，在下面的沙盒代码中，你将看到搜索输入文本和表格都更新了：

<Sandpack>

```jsx src/App.js
import { useState } from 'react';

function FilterableProductTable({ products }) {
  const [filterText, setFilterText] = useState('');
  const [inStockOnly, setInStockOnly] = useState(false);

  return (
    <div>
      <SearchBar 
        filterText={filterText} 
        inStockOnly={inStockOnly} />
      <ProductTable 
        products={products}
        filterText={filterText}
        inStockOnly={inStockOnly} />
    </div>
  );
}

function ProductCategoryRow({ category }) {
  return (
    <tr>
      <th colSpan="2">
        {category}
      </th>
    </tr>
  );
}

function ProductRow({ product }) {
  const name = product.stocked ? product.name :
    <span style={{ color: 'red' }}>
      {product.name}
    </span>;

  return (
    <tr>
      <td>{name}</td>
      <td>{product.price}</td>
    </tr>
  );
}

function ProductTable({ products, filterText, inStockOnly }) {
  const rows = [];
  let lastCategory = null;

  products.forEach((product) => {
    if (
      product.name.toLowerCase().indexOf(
        filterText.toLowerCase()
      ) === -1
    ) {
      return;
    }
    if (inStockOnly && !product.stocked) {
      return;
    }
    if (product.category !== lastCategory) {
      rows.push(
        <ProductCategoryRow
          category={product.category}
          key={product.category} />
      );
    }
    rows.push(
      <ProductRow
        product={product}
        key={product.name} />
    );
    lastCategory = product.category;
  });

  return (
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Price</th>
        </tr>
      </thead>
      <tbody>{rows}</tbody>
    </table>
  );
}

function SearchBar({ filterText, inStockOnly }) {
  return (
    <form>
      <input 
        type="text" 
        value={filterText} 
        placeholder="Search..."/>
      <label>
        <input 
          type="checkbox" 
          checked={inStockOnly} />
        {' '}
        Only show products in stock
      </label>
    </form>
  );
}

const PRODUCTS = [
  {category: "Fruits", price: "$1", stocked: true, name: "Apple"},
  {category: "Fruits", price: "$1", stocked: true, name: "Dragonfruit"},
  {category: "Fruits", price: "$2", stocked: false, name: "Passionfruit"},
  {category: "Vegetables", price: "$2", stocked: true, name: "Spinach"},
  {category: "Vegetables", price: "$4", stocked: false, name: "Pumpkin"},
  {category: "Vegetables", price: "$1", stocked: true, name: "Peas"}
];

export default function App() {
  return <FilterableProductTable products={PRODUCTS} />;
}
```

```css
body {
  padding: 5px
}
label {
  display: block;
  margin-top: 5px;
  margin-bottom: 5px;
}
th {
  padding-top: 5px;
}
td {
  padding: 2px;
}
```

</Sandpack>

注意，编辑表单目前还没有效果。沙盒中的控制台会显示一个错误，解释了原因：

<ConsoleBlock level="error">

You provided a \`value\` prop to a form field without an \`onChange\` handler. This will render a read-only field.

</ConsoleBlock>

在上面的沙盒中，`ProductTable` 和 `SearchBar` 组件通过读取 `filterText` 和 `inStockOnly` 属性来渲染表格、输入框和复选框。例如，下面是 `SearchBar` 如何填充输入框的值的示例：

```js {1,6}
function SearchBar({ filterText, inStockOnly }) {
  return (
    <form>
      <input 
        type="text" 
        value={filterText} 
        placeholder="Search..."/>
```

另外，你还没有添加任何代码来响应用户的输入等操作。这将是你最后需要完成的步骤。


## 步骤五：添加反向数据流 {/*step-5-add-inverse-data-flow*/}

目前，您的应用已能够在 props 和 state 沿层次结构向下流动的情况下正常渲染。但是，要根据用户的输入更改 state，您需要支持数据向相反方向流动：深层结构的表单组件需要在 `FilterableProductTable` 中更新 state。

React 明确了这种数据流动，但这比双向数据绑定需要更多的代码。如果您在上面的示例中尝试输入或勾选复选框，您会发现 React 会忽略您的输入。这是故意设计的。通过使用 `<input value={filterText} />`，您将输入框的 `value` 属性设置为始终等于从 `FilterableProductTable` 传入的 `filterText` 状态。因为 `filterText` 状态从未被更新，所以输入框的值始终保持不变。

您希望实现的是，每当用户更改表单输入时，状态能够更新以反映这些变化。状态由 `FilterableProductTable` 所拥有，因此只有它能够调用 `setFilterText` 和 `setInStockOnly`。为了让 `SearchBar` 更新 `FilterableProductTable` 的状态，您需要将这些函数传递给 `SearchBar` 组件：

```js {2,3,10,11}
function FilterableProductTable({ products }) {
  const [filterText, setFilterText] = useState('');
  const [inStockOnly, setInStockOnly] = useState(false);

  return (
    <div>
      <SearchBar 
        filterText={filterText} 
        inStockOnly={inStockOnly}
        onFilterTextChange={setFilterText}
        onInStockOnlyChange={setInStockOnly} />
```

在 `SearchBar` 中，您将添加 `onChange` 事件处理程序，并通过它们来更新父组件的状态：

```js {4,5,13,19}
function SearchBar({
  filterText,
  inStockOnly,
  onFilterTextChange,
  onInStockOnlyChange
}) {
  return (
    <form>
      <input
        type="text"
        value={filterText}
        placeholder="Search..."
        onChange={(e) => onFilterTextChange(e.target.value)}
      />
      <label>
        <input
          type="checkbox"
          checked={inStockOnly}
          onChange={(e) => onInStockOnlyChange(e.target.checked)}
```

现在，应用程序就完全正常工作了！

<Sandpack>

```jsx src/App.js
import { useState } from 'react';

function FilterableProductTable({ products }) {
  const [filterText, setFilterText] = useState('');
  const [inStockOnly, setInStockOnly] = useState(false);

  return (
    <div>
      <SearchBar 
        filterText={filterText} 
        inStockOnly={inStockOnly} 
        onFilterTextChange={setFilterText} 
        onInStockOnlyChange={setInStockOnly} />
      <ProductTable 
        products={products} 
        filterText={filterText}
        inStockOnly={inStockOnly} />
    </div>
  );
}

function ProductCategoryRow({ category }) {
  return (
    <tr>
      <th colSpan="2">
        {category}
      </th>
    </tr>
  );
}

function ProductRow({ product }) {
  const name = product.stocked ? product.name :
    <span style={{ color: 'red' }}>
      {product.name}
    </span>;

  return (
    <tr>
      <td>{name}</td>
      <td>{product.price}</td>
    </tr>
  );
}

function ProductTable({ products, filterText, inStockOnly }) {
  const rows = [];
  let lastCategory = null;

  products.forEach((product) => {
    if (
      product.name.toLowerCase().indexOf(
        filterText.toLowerCase()
      ) === -1
    ) {
      return;
    }
    if (inStockOnly && !product.stocked) {
      return;
    }
    if (product.category !== lastCategory) {
      rows.push(
        <ProductCategoryRow
          category={product.category}
          key={product.category} />
      );
    }
    rows.push(
      <ProductRow
        product={product}
        key={product.name} />
    );
    lastCategory = product.category;
  });

  return (
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Price</th>
        </tr>
      </thead>
      <tbody>{rows}</tbody>
    </table>
  );
}

function SearchBar({
  filterText,
  inStockOnly,
  onFilterTextChange,
  onInStockOnlyChange
}) {
  return (
    <form>
      <input 
        type="text" 
        value={filterText} placeholder="Search..." 
        onChange={(e) => onFilterTextChange(e.target.value)} />
      <label>
        <input 
          type="checkbox" 
          checked={inStockOnly} 
          onChange={(e) => onInStockOnlyChange(e.target.checked)} />
        {' '}
        Only show products in stock
      </label>
    </form>
  );
}

const PRODUCTS = [
  {category: "Fruits", price: "$1", stocked: true, name: "Apple"},
  {category: "Fruits", price: "$1", stocked: true, name: "Dragonfruit"},
  {category: "Fruits", price: "$2", stocked: false, name: "Passionfruit"},
  {category: "Vegetables", price: "$2", stocked: true, name: "Spinach"},
  {category: "Vegetables", price: "$4", stocked: false, name: "Pumpkin"},
  {category: "Vegetables", price: "$1", stocked: true, name: "Peas"}
];

export default function App() {
  return <FilterableProductTable products={PRODUCTS} />;
}
```

```css
body {
  padding: 5px
}
label {
  display: block;
  margin-top: 5px;
  margin-bottom: 5px;
}
th {
  padding: 4px;
}
td {
  padding: 2px;
}
```

</Sandpack>

您可以在 [添加交互性](/learn/adding-interactivity) 部分了解如何处理事件和更新状态。

## 下一节，我该做什么？ {/*where-to-go-from-here*/}

这只是一个简短的介绍，帮助您理解如何使用 React 构建组件和应用程序。您可以立即 [开始一个 React 项目](/learn/installation)，或 [深入了解本教程中使用的所有语法](/learn/describing-the-ui)。
