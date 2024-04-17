
# 为什么React的数组项需要key，为什么React会保留相同位置的相同组件的状态，为什么不能把hook放在条件语句中
首先可以先描述react的渲染过程
当组件函数第一次执行，即组件（MyButton）挂载时，函数第一次执行useState函数，执行后react在给相对应的组件（fiber）注册一个state hook，并初始化state；useState最后返回当前state的值，以及一个对state进行更改的setState函数。假设函数组件在返回的jsx对setState进行绑定，例如绑定到button的点击事件中，每点一次state + 1。当用户点击button时，react首先更新state，然后再一次执行这个组件函数（function MyButton），执行过程中再一次运行useState函数，拿到最新的值，填充到jsx中，最后更新视图。
整个过程最重要的就是一点：每次用setState更改状态的时候，react都会重新执行整个组件函数。
那首先来思考第一个问题——为什么无法在条件判断中使用hook。
要回答这个问题，我们首先要明白，hook的作用是什么？hook首先是函数，回顾对react渲染过程的描述，我们在**执行组件函数**的过程中，执行了useState这个hook函数，初始化了一个状态，并拿到了状态的初始值；随后setState触发 状态更新和rerender，**再次执行**了这个组件函数，拿到了更新后的状态。