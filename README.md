# React 内部机制探秘 - React Component 和 Element（文末附彩蛋demo和源码）

这篇文章比较偏基础，但是对入门 React 内部机制和实现原理却至关重要。算是为以后深入解读的一个入门，如果您已经非常清楚:

> React Component Render => JSX => React.createElement => Virtual Dom 

的流程，可以直接略过此文。


## 谷歌工程师一个风骚的问题

在几个月前，谷歌的前端开发专家 Tyler McGinnis 在其个人 twitter 账号上发布了 [这样一条推文](https://twitter.com/tylermcginnis33/status/771087982858113024)，引发了对 React 组件的讨论。


![推文截图](http://upload-images.jianshu.io/upload_images/4363003-1d06300d86696e44.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


他抛出来的问题是 ：如上述代码，**React 组件 Icon 直接出现在代码中，到底算什么？**

提供的选项有：

- A. Component Declaration 组件声明
- B. Component Invocation 组件调用
- C. Component Instantiation 组件实例化
- D. Using a Component 单纯地使用组件

有趣的是，参与回答的开发者中：

- 有 15% 选择了 A 项；
- 有 8% 选择了 B 项；
- 有 45% 选择了 C 项；
- 有 32% 选择了 D 项；

对 React 开发经验丰富的前端工程师来说，这个问题其实很好理解。它的关键在于：**真正明白 React Element 和 React Components，以及 JSX 抽象层是如连通 React 的。**当然也需要明白一些浅显的 React 内部工作机制。

这篇文章，就带领大家研究一下这个 JSX 抽象层的奥秘和 **React Reconciliation** 过程。

## React 和 React Element 到底是什么？
让我们回到最初，思考一下最原始的问题，React 到底是什么？

简而言之，

>React is a library for building user interfaces.

React 是一个构建视图层的类库(框架...whatever...)。不管 React 本身如何复杂，不管其生态如何庞大，**构建视图**始终是他的核心。记住这个信息，我们即将进入今天的第一个概念 — **React Element**。

简单地说，React Element 描述了“你想”在屏幕上看到的事物。

抽象地说，React Element 元素是一个描述了 Dom Node 的对象。

请注意我的用词 — “**描述**”，因为 React Element 并不是你在屏幕上看见的真实事物。相反地，他是一个描述真实事物的集合。存在的就是合理的，我们看来看看 React Element 存在的意义，以及为什么会有这样一个概念：

- JavaScript 对象很轻量。用对象来作为 React Element，那么 React 可以轻松的创建或销毁这些元素，而不必去太担心操作成本；
- React 具有分析这些对象的能力，进一步，也具有分析虚拟 Dom 的能力。当改变出现时，（相比于真实 Dom）更新虚拟 Dom 的性能优势非常明显。


为了创建我们描述 Dom Node 的对象（或者 React Element），我们可以使用 React.createElement 方法：

    const element = React.createElement( 
      'div', 
      {id: 'login-btn'}, 
      'Login'
    )
    
这里 React.createElement 方法接受三个参数：

- 一个表述标签名称的字符串 (div, span, etc.)；
- 当前 React Element 需要具有的属性；
- 当前 React Element 要表达的内容，或者一个子元素。

上面 React.createElement 方法调用之后，会返回一个 javascript 对象：
    
    { 
      type: 'div', 
      props: { 
        children: 'Login', 
        id: 'login-btn' 
      } 
    }
    
接着当我们使用 ReactDOM.render 方法，这才渲染到真实 DOM 之上时，就会得到：

    <div id='login-btn'>Login</div>
    
而这个才是真实的 Dom 节点。

到目前为止，并没有什么很难理解的概念。


## React Element 深入和 React Component
这篇文章我们开篇就介绍了 React Element，而并不是像官网或者学习资料上来就介绍 React Component，我相信你理解了 React Element，理解 React Component 就是自然而然的事情了。


在真正开发时，我们并不直接使用 React.createElement，这样做简直太无聊了，每个组件都这样写一定会疯掉的。这时候就出现了 React Component，即 React 组件。

>A component is a function or a Class which optionally accepts input and returns a React element.

没错，组件就是一个函数或者一个 Class（当然 Class 也是 function），它根据输入参数，并最终返回一个 React Element，而不需要我们直接手写无聊的 React Element。

所以说，实际上我们使用了 React Component 来生成 React Element，这对于开发体验的提升无疑是巨大的。

这里剖出一个思考题：**所有 React Component 都需要返回  React Element 吗？**显然是不需要的，那么 return null; 的 React 组件有存在的意义吗，它能完成并实现哪些巧妙的设计和思想？（请关注作者，下篇文章将会专门进行分析、讲解）

### 从场景实例来看问题 

接下来，请看这样一段代码：

    function Button ({ onLogin }) { 
      return React.createElement( 
        'div', 
        {id: 'login-btn', onClick: onLogin}, 
        'Login' 
      )
    }
    
我们定义了一个 Button 组件，它接收 onLogin 参数，并返回一个 React Element。注意 onLogin 参数是一个函数，并最终像 id:'login-btn' 一样成为了这个 React Element 的属性。


直到目前，我们见到了一个 React Element type 为 HTML 标签(“span”, “div”, etc)的情况。事实上，我们也可以传递另一个 React Element ：

    const element = React.createElement(
      User, 
      {name: 'Lucas'},
      null 
    )
    
注意此时 React.createElement 第一个参数是另一个 React Element，这与 type 值为 HTML 标签的情况不尽相同，当 React 发现 type 值为一个 class 或者函数时，它就会先看这个 class 或函数会返回什么样的 Element，并为这个 Element 设置正确的属性。

React 会一直不断重复这个过程（有点类似递归），直到没有 “createElement 调用 type 值为 class 或者 function” 的情况。

我们结合代码再来体会一下：

    function Button ({ addFriend }) {
      return React.createElement(
        "button", 
        { onClick: addFriend }, 
        "Add Friend" 
      ) 
    } 
    function User({ name, addFriend }) { 
      return React.createElement(
        "div", 
        null,
        React.createElement( "p", null, name ),
        React.createElement(Button, { addFriend })
      ) 
    }
    
上面有两个组件：Button 和 User，User 描述的 Dom 是一个 div 标签，这个 div 内，又存在一个 p 标签，这个 p 标签展示了用户的 name；还存在一个 Button。

现在我们来看 User 和 Button 中，React.createElement 返回情况：

    function Button ({ addFriend }) { 
      return { 
        type: 'button', 
        props: { 
          onClick: addFriend, 
          children: 'Add Friend' 
        } 
      } 
    } 
    function User ({ name, addFriend }) { 
      return { 
        type: 'div', 
        props: { 
          children: [{ 
            type: 'p',
            props: { children: name } 
          }, 
          { 
           type: Button, 
           props: { addFriend } 
          }]
        }
      }
    }
    
你会发现，上面的输出中，我们发现了四种 type 值：

- "button";
- "div";
- "p";
- Button

当 React 发现 type 是 Button 时，它会查询这个 Button 组件会返回什么样的 React Element，并赋予正确的 props。 

直到最终，React 会得到完整的表述 Dom 树的对象。在我们的例子中，就是：

    {
      type: 'div', 
      props: {
        children: [{
          type: 'p',
          props: { children: 'Tyler McGinnis' }
        }, 
        { 
          type: 'button', 
          props: { 
            onClick: addFriend, 
            children: 'Add Friend'
          }
         }]
       } 
    }
    
React 处理这些逻辑的过程就叫做 **reconciliation**，那么“这个过程（reconciliation）在何时被触发呢？”

答案当然就是每次 setState 或 ReactDOM.render 调用时。以后的分析文章将会更加详细的说明。

好吧，再回到 Tyler McGinnis 那个风骚的问题上。


![风骚的问题](http://upload-images.jianshu.io/upload_images/4363003-e70d01b79376c276.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


此时我们具备回答这个问题的一切知识了吗？稍等等，我要引出 JSX 这个老朋友了。


## JSX 的角色

在 React Component 编写时，相信大家都在使用 JSX 来描述虚拟 Dom。当然，反过来说，React 其实也可以脱离 JSX 而存在。

文章开头部分，我提到 “不常被我们提起的 JSX 抽象层是如何联通 React 的？” 答案很简单，因为 JSX 总是被编译成为 React.createElement 而被调用。一般 Babel 为我们做了 JSX —> React.createElement 这件事情。

再看来先例：

    function Button ({ addFriend }) {
      return React.createElement(
        "button",
        { onClick: addFriend },
        "Add Friend" 
       )
    } 
    function User({ name, addFriend }) { 
      return React.createElement(
        "div",
        null,
        React.createElement( "p", null, name),
        React.createElement(Button, { addFriend })
      )
    }
        
对应我们总在写的 JSX 用法：

    function Button ({ addFriend }) { 
      return ( 
        <button onClick={addFriend}>Add Friend</button> 
      )
    }
    function User ({ name, addFriend }) {
      return ( 
        <div>
         <p>{name}</p>
         <Button addFriend={addFriend}/>
        </div>
      )
    }
    
就是一个编译产出的差别。

## 最终答案和文末彩蛋

那么，请你来回答“Icon 组件单独出现代表了什么？”

Icon 在 JSX 被编译之后，就有：

    React.createElement(Icon, null)
    

**你问我怎么知道这些编译结果的？**

或者

**你想知道你编写的 JSX 最终编译成了什么样子？**

我写了一个小工具，进行对 JSX 的实时编译，放在 [Github仓库中](https://github.com/HOUCe/JSXLiveCompiler)，它使用起来是这样子的：

平台一分为二，左边可以写 JSX，右边实时展现其编译结果：


![实时编译平台](http://upload-images.jianshu.io/upload_images/4363003-643351a64e8cf577.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以及：

![实时编译平台](http://upload-images.jianshu.io/upload_images/4363003-eead62eb98118091.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这个工具最核心的代码其实就是使用 babel 进行编译：
    
    let code = e.target.value;
    try {
        this.setState({
            output: window.Babel.transform(code, {presets: ['es2015', 'react']})
            .code,
            err: ''
        })
    }
    catch(err) {
        this.setState({err: err.message})
    }

感兴趣的读者可以去 GitHub 仓库参看源码。

## 总结
其实不管是 JSX 还是 React Element、React Component 这些概念，都是大家在开发中天天接触到的。有的开发者也许能上手做项目，但是并没有深入理解其中的概念，更无法真正掌握 React 核心思想。

这些内容其实比较基础，但同时又很关键，对于后续理解 React/Preact 源码至关重要。在这个基础上，我会更新更多更加深入的类 React 实现原理剖析，感兴趣的读者可以关注。




Happy Coding!

PS: 
作者[Github仓库](https://github.com/HOUCe) 和 [知乎问答链接](https://www.zhihu.com/people/lucas-hc/answers)
欢迎各种形式交流。