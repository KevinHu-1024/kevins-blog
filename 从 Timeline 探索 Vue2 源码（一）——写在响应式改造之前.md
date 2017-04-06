# 从 Timeline 探索 Vue2 源码（一）——写在响应式改造之前

## 前言

React、Angular、Vue可以说是国内比较流行的三种 Web 框架

![trends](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/trends.png)

> 来自[谷歌指数](https://trends.google.com/trends/explore?cat=31&date=today%2012-m&geo=CN&q=React,%2Fm%2F0j45p7w,Angular2,Vue)

其中 Vue 作为后起之秀，以其易上手、低侵入等特点，受到了开发者们的青睐。社区中对 Vue 源码进行剖析的文章也是不少，比如 @xiaofuzi [200行代码实现精简版 Vue](https://github.com/xiaofuzi/deep-in-vue/blob/master/src/the-super-tiny-vue.js)、前同事 @youngwind 的 [Vue 早期源码探索](https://github.com/youngwind/blog/issues/84)等文章，让笔者也是获益良多。于是按捺不住探索的欲望，也开始的读源码的过程。

开始阅读的时候，我当然是**懵逼的**，完全不知道从哪里下手，硬着头皮读完了初始化函数，脑图记了一大片，却依然对整个框架没有整体概念，而面对更加复杂的后续源码，实在是没有驱动力继续读下去了。后来参考了少峰的《 Vue 早期源码探索》系列中新颖的源码阅读方式，我也试了一下，不过读来之后又是只知其味，不得其法，数据绑定更新的原理是知道了，但是依然对整个框架的运行过程云里雾里，继续看下去依然几千个 commit，又包含若干 breakchange ，so 又烂尾了...

后来笔者注意到调试工具里面的 Timeline 工具，这个工具一般是用来分析前端性能的，我之前也用它来调试过一些奇怪的 bug（比如 Vue 2.1.17和2.1.18版本对动画处理的不同），在阅读源码工作中，**Timeline 能够图形化的显示调用栈**，你能够很清晰的阅读在特定场景中，整个框架是如何运行的。在知道整体运行框架之后，再去阅读某个小模块的源码才能做到有的放矢，也能知道**模块与模块之间的关系**，所谓“场景驱动阅读”。你还能调整场景，看在**另一个场景中，调用栈是不是改变了，为何而变，涉及到了什么知识点**，目前看来是一种可取的阅读源码的方案。

下面我们来实践一下。

## 环境说明：

为了统一读者的运行环境，下面列出本文所用的 Vue 版本及构建方式：

* Vue 版本：`v2.2.4`

* 构建：`vue-cli`

* 初始化：`$ vue init simple learn-vue-source`

* 工具：

  * **Chrome DevTool** 用来查看函数调用栈及断点调试，为了保证时间线的纯净，减少浏览器插件脚本对时间线造成“污染”，请使用隐私模式
  * **WebStorm** 用来在打包前的代码中搜索及跳转模块
  * **lambda-view** 调试工具中的Source 面板没有语法高亮，用它来实现更好的源码阅读体验

## 场景一

```html
<div id="app">
  {{ message }}
</div>
```

```javascript
const app = new Vue({
      el: '#app',
      data: {
      	message: 'Hello, Vue!'
      },
    });
    console.log(app);
```

这个就是官方起步分档中的例子，下面我们在 Timeline 中看一下，这个应用是怎么跑起来的。

首先设置你的 Timeline 如下，这样方便你通过截图来判断，程序开始时间（当然你也可以通过下面的资源占用情况来判断）

![setup](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/setup.png)

刷新页面，等一会我们就能看见生成好的 Timeline 了。

![overview](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/overview.png)

前面一部分有几个匿名函数执行，通过我们的 HTML 我们可以知道，这里是 vue.js 释放的过程，即做一些环境判断、一些预处理、最后把 Vue 挂载到 window 的过程，最后红框内是生成 Timeline 的过程，这两个部分我们就不深究了。

ParseHTML 和 EvaluateScript是浏览器自身的行为，解析 HTML 和 JS，重点关注中间的 Vue 的运行过程，放大中间部分，能够看到中间这大概20ms的部分就是 Vue 干活的时间了。

![process](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/process.png)

图中绿色的部分是 vue.js 运行时的调用栈，所谓调用栈通俗理解（我就不放学院派的定义了）就是函数调用的顺序，函数都是从顶层向下调用，调用到最下面之后，相邻的同级别的函数执行，继续从上向下调用，类似于下图的方式：

![callstack](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/callstack.png)

### 构造函数

明白了调用栈，我们就看一下我们应用的启动过程吧！从我们的代码上来看，我们先是用`new Vue(xxx)`生成的一个 Vue 的实例，毫无疑问会调用 Vue 的构造函数，在 Timeline 上点击 Vue$3 ，在下面的 Summary 面板上通过点击代码行，我们能够跳转到 Source 面板查看源码。

![newvue](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/newvue.png)

这就是我们的构造函数的真面目:

![constructor](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/constructor.png)

（至于为什么是Vue$3，在这里我还不太明白，可能是不同编译 target 导致的不同吧（从编译后的源码看，runtime 版本的是$2），不过从log出的实例和 `window.Vue`上来看，Vue$3确实是我们的实例。）

![$instance](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/$instance.png)

### 初始化函数 _init
从函数上我们看到，构造函数调用了实例上面的_init方法，这时实例还没有创建，哪里来的_init方法呢？一定是沿着原型链找到了实例公共方法上面去了，即调用的是Vue.proptotype._init()。

沿着这个线索，我们在Source窗口中command+F搜索（windows用户使用xxx+F）.init，于是我们在3661行找到了它的初始定义：

![search-init](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/search-init.png)

我们发现，Vue.proptotype._init()是定义在一个initMixin函数中的，这个函数又是从哪里运行的呢？继续搜索initMixin：

![init-mixin](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/initmixin.png)

在构造函数下方，我们看到了他的身影，顺便我们还看到了`stateMixin`、`eventsMixin`、`lifecycleMixin`、`renderMixin`这几个函数调用，从命名上面看，他们分别初始化了**状态相关**、**事件相关**、**生命周期相关**、**渲染相关**的东西。他们都发生在匿名函数执行时，在我们使用Vue类时，他们已经初始化完成了，所以我们先往后面看，待需要的时候回头来看匿名函数都做了什么。

我们继续来看Timeline：

![init-all](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/init-all.png)

从Timeline上我们看到Vue._init一共做了这么几件事情：
`mergeOptions`、`initRender`、`initState`，然后就是一个长长的 `Vue$3.$mount` 直到视图渲染完成。

在Timeline点击Vue._init，然后在下面Summy面板中点击源码位置，进入Source面板：

![enter-init](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/enter-init.png)

![init-in-source](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/init-in-source.png)

前面的if判断似乎是，判断实例是否是一个组件，如果不是组件的话（是根实例），就执行mergeOptions。（在Source中搜索_isComponent，确实搜到了`createComponentInstanceForVnode`方法，与创建实例有关。从注释上看似乎是由于merge操作缓慢，而组件实例又没有必要做这步操作，所以有了这有么一个判断）

### 选项合并与格式化 mergeOptions

我们打上断点看看mergeOptions做了些什么：

![break-merge](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/break-merge.png)

mergeOptions传入了三个参数：Vue构造器的options（包括Vue的默认option，全局中使用Vue.config/mixin等等设置的选项）、我们 `new Vue` 时传入的options、当前vm实例。

通过点击源码位置，我们找到了`mergeOptions`的函数定义(编译之后的)，通过在各个函数（`checkComponents`, `normalizeProps`, `normalizeDirectives`）上打断点，我们大概理清楚了`mergeOptions`是做什么的：

```javascript
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
function mergeOptions (
  parent, // Vue的默认option，全局中使用Vue.config/mixin等等设置的选项
  child, // 我们new Vue时传入的options
  vm // 当前vm实例
) {
  {
    // 检测我们输入的options.component中是否有Vue的保留关键字
    // 如组件不能命名成slot，component，也不能命名成html和svg中已有的标签名
    // 我们的这个场景是根元素，所以这个部分就不跑了
    checkComponents(child);
  }
  // 这个函数是对我们自己传入的options.props进行格式化，用来支持数组和对象另种props形式
  // 经过他的处理，props都变成了对象形式，同时对prop的类型做了处理
  // 需要注意的是
  // 1. 数组声明的方式下，prop的类型声明在这里被统一为null（编译后的第1161行）
  // 2. 数组声明方式下，每个prop声明必须是字符串类型的，否则会报警告
  // 3. 数组/对象声明方式下，每个prop命名在这里都被转换成了驼峰命名风格
  // 4. 转换结果：
  //    ['one-prop'] => { oneProp: { type: null } }
  //    { one: Number, two: { type: Number, default: 1, .... } }
  //    => { one: { type: Number }, two: { type: Number, default: 1, .... } }
  // 当前场景中并没有用到，所以跳过
  normalizeProps(child);

  // 格式化用户在实例上自定义的指令，即 options.directive，支持对象方式定义，和函数方式定义
  // 如果是直接用函数的方式定义的话，会在这里被转换成对象形式
  // 转换结果：
  // directives: { direc: function() {} }
  // =>
  // directives: { direc: { bind: function() {}, update: function() {} }}
  // 当前场景中并没有用到，所以跳过
  normalizeDirectives(child);
  var extendsFrom = child.extends;

  // 处理在options中使用extends写法，用以支持声明式扩展/继承另一个组件，而不必使用Vue.extend
  // 自然，被扩展的组件也需要mergeOptions
  // 当前场景中并没有用到，所以跳过
  if (extendsFrom) {
    parent = typeof extendsFrom === 'function'
      ? mergeOptions(parent, extendsFrom.options, vm)
      : mergeOptions(parent, extendsFrom, vm);
  }

  // 处理使用mixin的情况，方便做细粒度的组件复用
  // 当前场景中并没有用到，所以跳过
  if (child.mixins) {
    for (var i = 0, l = child.mixins.length; i < l; i++) {
      var mixin = child.mixins[i];
      if (mixin.prototype instanceof Vue$3) {
        mixin = mixin.options;
      }
      parent = mergeOptions(parent, mixin, vm);
    }
  }
  var options = {};
  var key;
  for (key in parent) {
    mergeField(key);
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key);
    }
  }
  function mergeField (key) {
    var strat = strats[key] || defaultStrat;
    options[key] = strat(parent[key], child[key], vm, key);
  }
  return options
}
```

> 我们刚才一连跳过了5个函数，跳过是因为我们既没有子组件、也没扩展构造器、也没使用混入、也没自定义指令及props，这也就解答了我们在Timeline上为何只看到了mergeFields一个函数执行的原因，Timeline忠实地为我们记录了一切。**这也是使用Timeline查看源码的好处：跳过在当前场景无用的函数，专注于对整个框架运行的理解，同时也能减少读源码的压力。**

### 策略模式与闭包

接下来就是看看这唯一执行的 mergeField做了什么吧：

这里面有个变量 `strat` 让人比较疑惑：其实这里运用了一个**策略模式**，就是我们打算合并的 key 不同，是有不同的合并策略的（举例说一下有哪些不同，比如说某些是 child 覆盖 parent，有些则是 parent 覆盖 child），这里的 `strat` 其实就是 stratgy 的意思，比如合并 `component` 属性时，根据 `key` 使用合并 component 时的策略。然后让策略执行，传入 `mergeOptions` 传入的参数。

还记得 mergeOption 传入了什么参数吗？

mergeOptions传入了三个参数：`parent`:Vue构造器的options、`child`:我们new Vue时传入的options || {}、`vm`:当前vm实例。

通过 mergeFiled，我们的 options 变成了如下结构：

merge之前：

![merge-before](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/merge-before.png)

之后：

![merge-res](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/merge-res.png)

然后问题来了：我们的 `message` 哪去了？

这就是策略搞的鬼，合并`data`字段的处理策略在*994行*，在我们的场景下，代码走了*1028行*的分支，策略执行完直接返回了一个`mergedInstanceDataFn`函数，我们的`message`在策略执行的函数的闭包中被保存了下来，`option.data`现在是一个函数，它将在后续处理中被调用。

![data-func](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/data-func.png)

为了观察到后面的调用过程，我们在这个返回的函数中打一个断点（`1030`行）。

### 向实例上挂载 render

根据 Timeline，`mergeoptions` 运行之后，接下来就是 `initRender` 部分了：

这里Vue实例挂载了两个createElement函数，`vm._c` 和 `vm.$createElement`，其中按照Vue的编程代码风格，带`_`都是内部方法，`$`都是外部方法。

```javascript
  ...
  // 用来支持slot
  vm.$slots = resolveSlots(vm.$options._renderChildren, renderContext);
  // 用来支持带作用域的slot
  vm.$scopedSlots = emptyObject;
  // 把createElement函数绑定在vm自身上
  // 参数列表: tag, data, children, normalizationType, alwaysNormalize
  // 内部使用vm._c时，由于没有外来的干扰，不进行normalization过程，以提高性能，所以最后一个参数是false
  vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
  // 当外部使用vm.$createElement时，则必须要对开发者输入的内容进行normalization过程，所以随后一个参数为true
  vm.$createElement = function (a, b, c, d) { return createElement(vm, a, b, c, d, true); };
```

### beforeCreate 生命周期触发

`initRender` 后便运行了 `callHook` 方法，来触发 `beforeCreate` 生命周期，如果 `vm.$options` 中包含了相应字段（如$options.mounted），则将函数this改成当前vm并运行之。

此时，数据观测(data observer) 和 event/watcher 事件都还没有进行。

### 谜一样的 callHook

在 `callHook` 方法中，从代码上看似乎生命周期函数支持数组形式：

```javascript
  ...
  var handlers = vm.$options[hook];
  if (handlers) {
    for (var i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm);
      } catch (e) {
        handleError(e, vm, (hook + " hook"));
      }
    }
  }
  ...
```

于是用下面的代码验证之：

```javascript
const app = new Vue({
	el: '#app',
  mounted: [
  	function() {
    	console.log(1);
    },
    function() {
    	console.log(2);
    },
    () => {console.log(3)}
  ]
})

// -> 1,2,3
```

这时又意识到，源码中的写法应该是**只支持**数组形式，这与我们平时的使用方式就不太一样了，我们平时使用基本都是直接一个函数，这时`.length`应该是0，函数不执行，那么Vue又使用了什么来把我传入函数放入数组的呢？

通过打断点来确认，最后发现还是 `mergeFields` 中的策略捣的鬼，在 merge 生命周期函数时，统一在这里被转换为数组形式。代码在 `1048` 行，`mergeHook` 函数中。实现如下：

```javascript
/**
 * Hooks and props are merged as arrays.
 */
function mergeHook (
  parentVal, // 以合并beforeCreate为例，这里是父级vm的befoCreate
  childVal  // 这里是当前vm的befoCreate
) {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  // 如果存在父子级关系，就把父子级的生命周期函数数组合并
  // 问题是：数组中的函数执行顺序，和触发机制，是由根vm统一按顺序触发吗？还是由各个子vm分别触发？
}
```
### 山雨欲来：initState 拦截$options.data的存取

![init-state-time](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/init-state-time.png)

根据 Timeline 我们继续往下走，到了`initState`，`initState` 传入了当前的vm作为参数，我们先看一下当前vm的样子：

![before-init-state](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/before-init-state.png)

在当前场景下，`initState` 执行了 `initData`，这里 Vue 对数据进行了响应式改造，这里就接近 Vue 进行数据绑定的核心部分了。

接下来把断点打在`2648`行，准备进入 `initData`。

#### 函数形式的 $options.data

在 `initData` 中，我们发现了 `$options.data` 是函数的情况（2704-2706），这个 `$options.data` 就是在 `mergeOptions` 函数中被返回的函数。

![init-data](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/init-data.png)

这个函数做了一次数据合并，问题是：为什么要做数据合并？为何要延迟到现在才合并？

![merge-data](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/merge-data.png)

合并之后的数据就变成了对象，返回出来。

#### 拦截对$options.data中键值的存取

![proxy](https://github.com/KevinHu-1024/kevins-blog/raw/draft/imgs/vue-source-1/proxy.png)

这里拦截对$options.data中键值的访问，全部映射到`vm._data[对应键值上]`。

proxy的实现如下：

```javascript
function proxy (target, sourceKey, key) {
  // 配置Object.defineProperty的第三参数，把访问都映射到_data[key]
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  };
  // 配置Object.defineProperty的第三参数，把写入都映射到_data[key]
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

Vue 通过这里完成了内外数据存取的分离，当我们在操作`$options.data`时，Vue内部则在操作`_data`。举个例子：
当我访问`vm.message`的时候返回`vm._data.message`，当我设置`vm.message`的时候，返回`vm._data.message`，完成了内部属性`_data.message`和外部属性`data.message`的连接。

接下来就是重头戏：[**对$options.data进行观察（observe）**](https://github.com/KevinHu-1024/kevins-blog/issues/5)，请待下期分解！
