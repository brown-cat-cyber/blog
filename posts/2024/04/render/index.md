# 从为什么不要把 React hooks 写在条件语句中说起

## 引子

对 React 的初学者来说，除去 useEffect 这个大坑，React 还有不少看起来有些诡异的规则，比如

1. 不要把 React hooks 写在条件语句中
2. React 会保留相同位置相同类型的组件的状态
3. 为什么数组遍历需要 key 属性

刚开始学习的时候，我只是盲目地遵从着 eslint 的报错提示，心里觉得很生硬；等熟悉 React 后，我才明白了它们的原因。接下来我将尝试分别解释这三条规则的必要性，而到最后，大家将会明白它们都是因 React 的渲染机制而起。解释过程中我并不会引入 fiber 以及更底层的 React 技术细节概念，因为一是我希望文章总是能被更多人阅读，尽量保持简洁，非必要不抬高门槛。二是我觉得底层实现细节和这些规则实际上并没有直接关系，引入它们反而会制造噪音。对于想要了解更多（乃至从头实现一个 React）的朋友，我在文章结尾给出了一些我搜集到的拓展阅读。

## React 的渲染过程

首先让我们先描述一下 React 的渲染过程。
假设我们有这样一个 Counter 组件，（[摘自官方教程](https://React.dev/blog/2023/03/16/introducing-React-dev#counter-number)）

```javascript
export default function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return <button onClick={handleClick}>You pressed me {count} times</button>;
}
```

当这个*组件函数*(`function Counter`)第一次执行，即 React 挂载 Counter 组件时，*组件函数*第一次执行`useState`函数，执行后 React 给相对应的组件注册一个`useState`hook，并初始化 state；useState 最后返回当前 state 的值，以及一个对 state 进行更改的 `setState` 函数。函数组件在最后返回的 jsx 将 `setState` 绑定到 button 的点击事件中。当用户点击 button 时，setState 触发 rerender，React 再一次执行了这个*组件函数*，执行过程中再一次运行`useState`函数，更新 state，并返回最新的值，React 将其填充到 jsx 中，最后更新视图。
整个过程最重要的其实就一点：**每次用 `setState` 更改状态的时候，React 都会重新执行整个组件函数**。

## 为什么不要在条件判断中使用 hook

记住这点后，那首先来思考第一条规则——为什么不要在条件判断中使用 hook？
要回答这个问题，我们首先要明白，hook 的作用是什么？hook 首先是函数，回顾对 React 渲染过程的描述，我们在**执行组件函数**的过程中，**调用了`useState`这个 hook 函数**；随后在 rerender 的过程中, React **再次执行**了组件函数，**并再次调用 `useState`**拿到了更新后的状态。
所以关键在于，每一次触发 rerender，我们都会重新执行一遍组件函数，**组件函数的执行结果也就是 rerender 的结果**。
那如果从函数的执行角度来讲，前一次和后一次函数的执行结果分别是什么呢？React 看到了什么？
假设我们有这样一个表单组件([摘自 React 官方教程](https://React.dev/learn/choosing-the-state-structure#avoid-redundant-state))

```javascript
import {`useState`} from "React";

export default function Form() {
  const [firstName, setFirstName] = useState("");
  const [lastName, setLastName] = useState("");

  const fullName = firstName + " " + lastName;

  function handleFirstNameChange(e) {
    setFirstName(e.target.value);
  }

  function handleLastNameChange(e) {
    setLastName(e.target.value);
  }

  return (
    <>
      <h2>Let’s check you in</h2>
      <label>
        First name: <input value={firstName} onChange={handleFirstNameChange} />
      </label>
      <label>
        Last name: <input value={lastName} onChange={handleLastNameChange} />
      </label>
      <p>
        Your ticket will be issued to: <b>{fullName}</b>
      </p>
    </>
  );
}
```

要注意，**“firstName”是对`useState`返回的状态值的命名，而不是对状态本身命名，“firstName”并没有作为参数参与到对`useState`的调用当中**。所以从 React 的视角来看，组件函数执行的结果是这样的。

```javascript
import {`useState`} from 'React';

export default function Form() {
  useState('');
  useState('');
  ...

  return (
    <>
    ...
    </>
  );
}

```

从这个视角来看，**能区分这两个 state 的唯一办法，就是他们对应的 useState 的*调用顺序*或者说*书写顺序*。**
那如果在条件语句中使用 useState hook，会出现什么情况？
假设原始组件是这样：

```javascript
import {`useState`} from 'React';

export default function Form() {
  if (Math.random() > 0.5) {
    const [firstName, setFirstName] = useState('');
  }
  if (Math.random() > 0.5) {
    const [lastName, setLastName] = useState('');
  }
  ...

  return (
    <>
    ...
    </>
  );
}

```

那执行结果可能是这样

```javascript
import {`useState`} from 'React';

export default function Form() {
  useState('');
  ...

  return (
    <>
    ...
    </>
  );
}

```

也有可能是这样

```javascript
import {`useState`} from 'React';

export default function Form() {
  useState('');
  ...

  return (
    <>
    ...
    </>
  );
}

```

从这个视角看，在条件判断中使用 hook 带来的问题就呼之欲出了：请问调用这唯一的`useState`获取的是代表"firstName"的那个状态，还是代表"lastName"的那个状态？我们没有办法判断，我们只看到了一个 useState。
不仅仅有这个限制，[官方文档](https://React.dev/reference/rules/rules-of-hooks)还列出了诸如不要在循环、嵌套函数、try/catch 代码块中使用 hooks 的规则，这些规则被归纳为“仅在顶层调用 hooks”。它们的原因都是类似的：**在多次函数执行中，区分 hooks 的唯一办法就是它们的调用顺序，因此要避免一切*有可能*打乱顺序的行为**。
这里提一下，我在写这篇文章的时候查询了一些资料，其中很多都有一个大概这样的总结“因为 React 用一个链表（自制 React 则多用数组）来储存 hooks 的状态，所以必须要保证它的调用顺序与链表/数组中的排序一致”。这个说法不能说错，但我觉得可能过于聚焦于技术细节了。**问题不是 React 用什么数据结构去储存 hooks，问题在于，只要 React 每当状态变更就重新执行一遍组件函数，只要每次执行函数都会重新调用一遍 hooks，那在没有 key、id 等标识符的情况下，React 就只能凭借在函数中的调用顺序去辨认不同的 hooks。**
这里提到了 key ，这也是另外两个问题的关键。

## 为什么 React 会保留相同位置相同类型的组件的状态

为什么 React 会保留相同位置相同类型的组件的状态？让我们看看另外一个例子（摘自[官方文档](https://React.dev/learn/preserving-and-resetting-state)），继续观察组件函数的执行结果，但这次关注返回的 jsx 部分。

```javascript

export default function App() {
  const [isFancy, setIsFancy] = useState(false);
  return (
    <>
      {isFancy ? (
        <Counter isFancy={true} />
      ) : (
        <Counter isFancy={false} />
      )}
      ...
    </>
  );
}

function Counter({ isFancy }) {
  ...
  return (
    <div className={isFancy && "fancy"}>
    ...
    </div>
  );
}
```

App 函数的执行结果可以简化为这样（isFancy 为 true），

```javascript
export default function App() {
  useState(false);
  return (
    <>
      <Counter isFancy={true} />
      ...
    </>
  );
}

function Counter({ isFancy }) {
  ...
  return (
    <div className={isFancy && "fancy"}>
    ...
    </div>
  );
}
```

或这样（isFancy 为 false）

```javascript
export default function App() {
  useState(false);
  return (
    <>
      <Counter isFancy={false} />
      ...
    </>
  );
}

function Counter({ isFancy }) {
  ...
  return (
    <div className={isFancy && "fancy"}>
    ...
    </div>
  );
}
```

当 React 看到返回的这两个 jsx 时，它同样没法判断这还是不是同一个 counter。但这个时候 React 多了另一个信息——它们的*组件名*是相同的。**出于性能考虑，React 会默认这还是同一个组件，这样就可以仅更新这个组件的属性而非重新挂载这个组件。**（在这个案例中，当 React 计算以及提交虚拟 dom 时，浏览器仅需更新 div 的 class，而非删除掉这个 div 后再重新创建并添加一个 div）
那如何让 React 认识到这并不是同一个组件呢？这就是 key 属性的作用，它作为组件的唯一标识，类似于数据库中的 id。有了它，React 就不用再借助位置、名称类型来判断组件的同一性了，所以我们可以[通过设置不同的 key 来重制掉同一类型同一位置组件的状态](https://react.dev/learn/preserving-and-resetting-state#option-2-resetting-state-with-a-key)。

## 为什么 React 会要求开发者给 jsx 中的数组项加上 key 属性

有了前面的铺垫，我们就可以很顺利地解释第三个规则——为什么 React 会要求开发者给 jsx 中的数组项加上 key 属性？
因为相比于其它固定的代码，用来 map 的数组在 jsx 中是一个非常不稳定的结构，它随时有可能受 state 和 prop 的影响而增减数组项，因此数组项的顺序（index）在数组中并不是一个稳定的标识。这个时候，**React 为了避免重复挂载数组项，就必须要有一个唯一而稳定的标识去区分它们，将它们与上一次快照中的数组项一一对应。**（所以为什么不能把 index 作为 key 呢？因为这样你等于没有告诉 React 任何新的信息啊，React 不需要你告诉它 index，它看得出来……）

## 总结

其实这三个问题最后都可以归结为“React 如何比较重复执行组件函数的不同结果”，如果没有标识符 key，React 就只能根据顺序去识别不同的 hooks 和组件。无论 React 框架底层细节是如何实现的，只要 React 遵循*每次渲染时都执行一遍组件函数来生成 UI 结果，而非把它当成一种初始化模板*的做法，那就肯定会出现这些问题。
