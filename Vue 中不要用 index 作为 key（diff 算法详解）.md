# Vue 中不要用 index 作为 key（diff 算法详解）

## 示例

以这样一个列表为例

```html
<ul>
    <Li>1</Li>
    <Li>2</Li>
</ul>
```

那么它的`vnode`也就是虚拟dom节点大概是这样的。

```json
{
    tag: 'ul',
    children: [
        {tag: 'li',children: [{vnode: {text: '1'}}]},
        {tag: 'li',children: [{vnode: {text: '2'}}]},
    ]
}
```

假设更新以后，我们把子节点的顺序调换一下：

```diff
{
  tag: 'ul',
  children: [
+   { tag: 'li', children: [ { vnode: { text: '2' }}]  },
+   { tag: 'li', children: [ { vnode: { text: '1' }}]  },
  ]
}
```

很显然，这里的`children`部分是我们本文`diff`算法要讲的的重点。

首先响应式数据更新后，触发了渲染`vnode`，而`vm._update`就会带着新的`vnode`走去触发

`vm._render()`其实生成的就是`vnode`，而`vm._update`就会带着新的`vnode`去走触发`__path__`过程。

我们直接进入`ul`这个`vnode`的`path`过程

# 对比新旧节点是否是相同类型的节点

## 1.不是相同节点

`isSameNode`为`false`的话，直接销毁旧的`vnode`,渲染新的`vnode`。这也解释了为什么`diff`是同层对比。

## 2.是相同节点，要尽可能的做节点的复用（都是`ul`）

会调用`src/core/vdom/path.js`下的`pathVNode`方法。

### 如果新的 vnode 是文字 vnode

就直接调用浏览器的`dom api`把节点的直接替换掉文字内容就好。

### 如果新的 vnode 不是文字 vnode

那么就要开始对子节点`children`进行对比了。（可以对比`ul`中的`li`子元素）

### 如果有新的 children 而没有旧 children

说明是新增children，直接 `addVnodes` 添加新子节点。

### 如果有旧 children 而没有新的 children 

说明是删除 childer，直接`removeVnodes`删除旧的字节的

### 如果新旧 children 都存在（都存在`li子节点列表`，进入）

那么就是我们`diff算法`想要考察的最核心的点了，也就是新旧节点的`diff`过程。

就可打开源码仓库里大致看下这个函数，接下来我们逐步讲解。

[updateChildren](https://github.com/vuejs/vue/blob/bd6cea0973247e2a8e1d4a2250614c0bf44f0b26/src/core/vdom/patch.js#L404-L474)

通过

```js
// 旧首节点
let oldStartIdx = 0
// 新首节点
let newStartIdx = 0
// 旧尾节点
let oldEndIdx = oldCh.length - 1
// 新尾节点
let newEndIdx = newCh.length -1
```

这些变量分别指向 `旧节点的首尾`、`新节点的首尾`。

根据这些指针，在一个`while`循环中不停的对新旧节点俩端进行对比，然后把俩端的指针向不断内部收缩，直到没有节点可以对比。

在讲对比过程之前，要讲一个比较重要的函数：`sameVnode`

```javascript
function sameVnode(a,b){
    return{
        a.key === b.key&&(
        	a.tag === b.tag &&
        	a.isComment === b.isComment &&
        	isDef(a.data) === isDef(b.data) &&
            sameInput(a,b)
        )
    }
}
```

他是用来判断节点是否可用的关键函数，可以看到，判断是否是 `sameVnode`，传递给节点的`key`是关键。

然后我们接着进入`diff`过程，每一轮都是同样的对比，其中某一环命中了，就递归进入`pathVnode`针对单个`vnode`进行的过程（如果这个`vnode`又有`children`，那么还会来到这个`diff children`的过程）:

1. 旧首节点和新首节点用`sameNode`对比。
2. 旧尾节点和新尾节点用`sameNode`对比。
3. 旧首节点和新尾节点用`sameNode`对比。
4. 旧尾节点和新首节点用`sameNode`对比。
5. 如果以上逻辑都匹配不到，再把所有旧子节点的`key`做一个映射到旧节点下标的`key -> index`表，然后用新`vnode`的`key`去找出在接节点中可以复用的位置。

然后不停的把匹配到的指针向内部收缩，直到新旧节点有一端的指针相遇（说明这个端的节点都被patch过了）。

在指针相遇以后，还有俩种比较特殊的情况：

1. 有新节点需要加入。如果更新完以后，`oldStartIdx > oldEndIdx`，说明旧节点都被`patch`完了，但是有可能还有新的节点没有被处理到。接着会去判断是否有新增子节点。
2. 有旧节点需要删除。如果新节点先`patch`完了，那么此时会走`newStartIdx > newEndIdx`的逻辑，那么就会去删除多余的旧子节点。

##　为什么不要以index作为key?

### 节点reverse 场景

假设我们有这样的一段代码：

```vue
<div id="app">
    <ul>
        <item :key="index" v-for="(num, index) in nums" :num="num" :class="`item${num}`"></item>
    </ul>
    <button @click="change">改变</button>
</div>
<script src="./vue.js"></script>
<script>
var vm = new Vue({
    	name: 'parent',
    	el: '#app',
    	data: {
            nums: [1,2,3]
        },
    	methods: {
            change(){
                this.nums.reverse();
            }
        },
    	components: {
            item: {
                props: ['num'],
                template: `<div>{{num}}</div>`,
                name: 'child'
            }
        }
    })
</script>
```

其实是一个很简单的列表组件，渲染出`1 2 3`三个数字。我们先以`index`作为`key`,来跟踪一下它的更新。

我们接下来只关注`item`列表节点的更新，在首次渲染的时候，我们虚拟节点列表`oldChildren`粗略表示是这样的：

```json
[
    {
        tag: 'item',
        key: 0,
        props:{ num: 1 },
    },
    {
        tag: 'item',
        key: 1,
        props: { num: 2 },
    },
    {
        tag: "item",
        key: 2,
        props: { num: 3 },
    }
]
```

在我们点击按钮时候，会对数组做`reverse`的操作。那我们此时生产的`newChildren`列表是这样的：

```diff
[
  {
    tag: "item",
    key: 0,
    props: {
+     num: 3
    }
  },
  {
    tag: "item",
    key: 1,
    props: {
+     num: 2
    }
  },
  {
    tag: "item",
    key: 2,
    props: {
+     num: 1
    }
  }
];

```

发现什么问题没有？key的顺序没变，传入的值完全变了。这会导致一个什么问题？

本来按照最合理的逻辑来说，`旧的第一个vnode` 是应该直接完全复用 `新的第三个vnode`的，因为它们本来就应该是同一个vnode，自然所有的属性都是相同的。

但是在进行子节点的 `diff` 过程中，会在 `旧首节点和新首节点用`sameNode`对比。` 这一步命中逻辑，因为现在`新旧两次首部节点` 的 `key` 都是 `0`了，

然后把旧的节点中的第一个 `vnode` 和 新的节点中的第一个 `vnode` 进行 `patchVnode` 操作。

这会发生什么呢？我可以大致给你列一下： 首先，正如我之前的文章[props的更新如何触发重渲染？](https://juejin.im/post/5e854a32518825736c5b807f#heading-8)里所说，在进行 `patchVnode` 的时候，会去检查 `props` 有没有变更，如果有的话，会通过 `_props.num = 3` 这样的逻辑去更新这个响应式的值，触发 `dep.notify`，触发子组件视图的重新渲染等一套很重的逻辑。

然后，还会额外的触发以下几个钩子，假设我们的组件上定义了一些dom的属性或者类名、样式、指令，那么都会被全量的更新。

1. updateAttrs
2. updateClass
3. updateDOMListeners
4. updateDOMProps
5. updateStyle
6. updateDirectives

而这些所有重量级的操作（虚拟dom发明的其中一个目的不就是为了减少真实dom的操作么？），都可以通过直接复用 `第三个vnode` 来避免，是因为我们偷懒写了 `index` 作为 `key`，而导致所有的优化失效了。

### 节点删除场景

另外，除了会导致性能损耗以外，在`删除子节点`的场景下还会造成更严重的错误，

可以看[sea_ljf](https://juejin.im/user/584a7760128fe1006c7b17f9)同学提供的这个[demo](https://codesandbox.io/s/ancient-moon-427u7)。

假设我们有这样的一段代码：

```vue
<body>
  <div id="app">
    <ul>
      <li v-for="(value, index) in arr" :key="index">
        <test />
      </li>
    </ul>
    <button @click="handleDelete">delete</button>
  </div>
  </div>
</body>
<script>
  new Vue({
    name: "App",
    el: '#app',
    data() {
      return {
        arr: [1, 2, 3]
      };
    },
    methods: {
      handleDelete() {
        this.arr.splice(0, 1);
      }
    },
    components: {
      test: {
        template: "<li>{{Math.random()}}</li>"
      }
    }
  })
</script>

```

那么一开始的 `vnode列表`是：

```js
[
  {
    tag: "li",
    key: 0,
    // 这里其实子组件对应的是第一个 假设子组件的text是1
  },
  {
    tag: "li",
    key: 1,
    // 这里其实子组件对应的是第二个 假设子组件的text是2
  },
  {
    tag: "li",
    key: 2,
    // 这里其实子组件对应的是第三个 假设子组件的text是3
  }
];
```

有一个细节需要注意，正如我上一篇文章中所提到的[为什么说 Vue 的响应式更新比 React 快？](https://juejin.im/post/5e854a32518825736c5b807f#heading-7)，Vue 对于组件的 `diff` 是不关心子组件内部实现的，它只会看你在模板上声明的传递给子组件的一些属性是否有更新。

也就是和v-for平级的那部分，回顾一下判断 `sameNode` 的时候，只会判断`key`、 `tag`、`是否有data的存在（不关心内部具体的值）`、`是否是注释节点`、`是否是相同的input type`，来判断是否可以复用这个节点。

```vue
<li v-for="(value, index) in arr" :key="index"> // 这里声明的属性
  <test />
</li>

```

有了这些前置知识以后，我们来看看，点击删除子元素后，`vnode 列表` 变成什么样了。

```json
[
  // 第一个被删了
  {
    tag: "li",
    key: 0,
    // 这里其实上一轮子组件对应的是第二个 假设子组件的text是2
  },
  {
    tag: "li",
    key: 1,
    // 这里其实子组件对应的是第三个 假设子组件的text是3
  },
];

```

虽然在注释里我们自己清楚的知道，第一个 `vnode` 被删除了，但是对于 Vue 来说，它是感知不到子组件里面到底是什么样的实现（它不会深入子组件去对比文本内容），那么这时候 Vue 会怎么 `patch` 呢？

由于对应的 `key`使用了 `index`导致的错乱，它会把

1. `原来的第一个节点text: 1`直接复用。
2. `原来的第二个节点text: 2`直接复用。
3. 然后发现新节点里少了一个，直接把多出来的第三个节点`text: 3` 丢掉。

至此为止，我们本应该把 `text: 1`节点删掉，然后`text: 2`、`text: 3` 节点复用，就变成了错误的把 `text: 3` 节点给删掉了。

## 为什么不要用随机数作为key？

```vue
<item
  :key="Math.random()"
  v-for="(num, index) in nums"
  :num="num"
  :class="`item${num}`"
/>

```

其实我听过一种说法，既然官方要求一个 `唯一的key`，是不是可以用 `Math.random()` 作为 `key` 来偷懒？这是一个很鸡贼的想法，看看会发生什么吧。

首先 `oldVnode` 是这样的：

```js
[
  {
    tag: "item",
    key: 0.6330715699108844,
    props: {
      num: 1
    }
  },
  {
    tag: "item",
    key: 0.25104533240710514,
    props: {
      num: 2
    }
  },
  {
    tag: "item",
    key: 0.4114769152411637,
    props: {
      num: 3
    }
  }
];

```

更新以后是：

```diff
[
  {
    tag: "item",
+   key: 0.11046018699748683,
    props: {
+     num: 3
    }
  },
  {
    tag: "item",
+   key: 0.8549799545696619,
    props: {
+     num: 2
    }
  },
  {
    tag: "item",
+   key: 0.18674467938937478,
    props: {
+     num: 1
    }
  }
];


```

可以看到，`key` 变成了完全全新的 3 个随机数。

上面说到，`diff` 子节点的首尾对比如果都没有命中，就会进入 `key` 的详细对比过程，简单来说，就是利用旧节点的 `key -> index` 的关系建立一个 `map` 映射表，然后用新节点的 `key` 去匹配，如果没找到的话，就会调用 `createElm` 方法 **重新建立** 一个新节点。

具体代码在这：

```js
// 建立旧节点的 key -> index 映射表
oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);

// 去映射表里找可以复用的 index
idxInOld = findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
// 一定是找不到的，因为新节点的 key 是随机生成的。
if (isUndef(idxInOld)) {
  // 完全通过 vnode 新建一个真实的子节点
  createElm();
}

```

也就是说，咱们的这个更新过程可以这样描述： `123` -> 前面重新创建三个子组件 -> `321123`  -> 删除、销毁后面三个子组件 -> `321`。

发现问题了吧？这是毁灭性的灾难，创建新的组件和销毁组件的成本你们晓得的伐……本来仅仅是对组件移动位置就可以完成的更新，被我们毁成这样了。

## 总结

经过这样的一段旅行，`diff` 这个庞大的过程就结束了。

我们收获了什么？

1. 用组件唯一的 `id`（一般由后端返回）作为它的 `key`，实在没有的情况下，可以在获取到列表的时候通过某种规则为它们创建一个 `key`，并保证这个 `key` 在组件整个生命周期中都保持稳定。
2. 如果你的列表顺序会改变，别用 `index` 作为 `key`，和没写基本上没区别，因为不管你数组的顺序怎么颠倒，index 都是 `0, 1, 2` 这样排列，导致 Vue 会复用错误的旧子节点，做很多额外的工作。列表顺序不变也尽量别用，可能会误导新人。
3. 千万别用随机数作为 `key`，不然旧节点会被全部删掉，新节点重新创建，你的老板会被你气死。

## 反驳标题之前，请先看这段

这篇文章发布以后，很多小伙伴提出了自己的建议和优化。但是也有很多人在评论区说，既然 `index` 只是在某些特定的场景下会出问题，那 `列表顺序保持不变` 的情况下还是可以接着用。这样做有什么问题呢？

1. 团队代码规范，假设这样一个场景吧，你这边代码里全部写的 `:key="index"`，有一个新人入职了跟着写，结果他的场景是删除和乱序的，这种情况你一个个讲原理指正？这就是统一代码规范和最佳实践的作用啊。`eslint` 甚至也专门有一个 `rule` 叫做 `react/no-array-index-key`，为什么要有这些约束和规范？如果社区总结了最佳实践，为什么一定要去打破它？这都是值得思考的。 就像 `==` 操作符，为什么要禁止？就是因为隐式转换会出很多问题，你说你熟背隐式转换所有原理，你能保证团队所有小伙伴都熟背？何苦有更简单的 `===` 操作符可以用。
2. 说开发效率的问题，`index` 作为 `key` 我在上面已经提到了好几种会出问题的情况了，还是坚持要用，就因为简单。那么 `TypeScript` 也没有火起来的必要吗？它需要多写很多代码，“效率” 很低，为什么它火了？不是因为用 `JavaScript` 就一定会出现类型错误，而是因为用了 `TypeScript` 可以更好的保证你代码的稳定性。正如用了 `id` 作为key，可以比 `index` 更好的保证稳定性，更何况用 `id` 也不费事啊。完全都不像 `TypeScript` 带来的额外的语法成本。
3. 所谓的列表顺序稳定，这个稳定你真的能保证吗？除了你前端写死的永远不变的一个列表，就假设你的列表没有在头部`新增一项`（导致节点全部依次错误复用），在任意位置 `删除一项`（有时导致错误删除）等这些会导致 `patch` 过程出现问题的操作。 就举个很简单的例子，你的“静态”列表的顺序是`[1, 2, 3]`，数据库里突然加入了一条新数据`0`，那么你认为的不会变的列表的就变成了`[0, 1, 2, 3]`。然后，`1` 节点就错误的和 `0`节点进行 `patchVnode`， `2` 节点就错误的和 `1` 节点进行 `patch`、导致原本只需要把新增的`0`节点插入到头部，然后分别对 `1 -> 1`、`2 -> 2`、`3 -> 3` 进行 `patchVnode`即可（基本没有变化），变成了毁灭的`全量更新`。（如果子组件是个很重的组件呢？它的每一项都会经历完整的 `vm._update(vm._render())`）过程，因为 `props` 变了。

