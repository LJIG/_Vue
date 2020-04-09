# Vue 中slot和slot-scope的原理

在Vue2.6中

1. `slot`和`slot-scope`在组件内部被统一整合成了`函数`
2. 他们的渲染作用域都是`子组件`
3. 并且都能通过`this.$slotScopes`

社区异步流程管理库：`vue-promised`，它的用法是这样的：

```vue
<Promised :promise="usersPromise">
    <template v-slot:pending>
		<p> Loading... </p>
    </template>
    <template v-slot="data">
		<ul>
            <li v-for="user in data">{{ user.name }}</li>
        </ul>
    </template>
    <template v-slot:rejected="error">
		<p> Error: {{ error.message }} </p>
    </template>
</Promised>
```

可以看到，我们只要把一个用来处理请求的异步`promise`传递给组件，他就会自动帮组我们去完成这个`promise`,并且响应式的对外抛出`pending`，`rejected`，和异步执行成功后的数据`data`。

这可以大大简化我们的异步体验，原本我们要手动执行这个`promise`，手动管理状态处理错误等等...

而这一切强大的功能都得益于Vue提供的`slot-scope`功能，他在封装的灵活性上甚至有点接近于`Hook`，组件甚至可以完全不关心`UI`渲染，只帮组父组件管理一些`状态`。

## 类比 React

----

如果你有`react`的开发经验，其实这就类比`React`中的`renderProps`去理解就好了。

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import PropTypes from 'prop-types'

// 这是一个对外提供鼠标位置的 render props 组件
class Mouse extends React.Component {
  state = { x: 0, y: 0 }

  handleMouseMove = (event) => {
    this.setState({
      x: event.clientX,
      y: event.clientY
    })
  }

  render() {
    return (
      <div style={{ height: '100%' }} onMouseMove={this.handleMouseMove}>
        // 这里把 children 当做函数执行，来对外提供子组件内部的 state
        {this.props.children(this.state)}
      </div>
    )
  }
}

class App extends React.Component {
  render() {
    return (
      <div style={{ height: '100%' }}>
        // 这里就很像 Vue 的 作用域插槽
        <Mouse>
         ({ x, y }) => (
           // render prop 给了我们所需要的 state 来渲染我们想要的
           <h1>The mouse position is ({x}, {y})</h1>
         )
        </Mouse>
      </div>
    )
  }
})

ReactDOM.render(<App/>, document.getElementById('app'))

```

## 原理解析

----

### 初始化

对于这样的一个例子来说

```vue
<test>
    <template v-slot:bar>
		<span>Hello</span>
    </template>
    <template v-slot:foo="prop">
    	<span>{{ prop.msg }}</span>
    </template>
</test>
```

这段模板会被编译成这样

```javascript
with(this){
    return _c("test",{
        scopedSlots: _u{[
        	{
        		key: "bar",
        		fn: function(){
        			return [_c('span',[_v("Hello")])]
            	}
            },
            {
                key: "foo",
                fn: function(prop){
                    return [_c("span",[_v(_s(prop.msg))])]
                }
            }
        ]}
    })
}
```

然后经过初始化时的一系列处理(resolveScopedSlots,normalizeScopedSlots) test组件实例`this.$slotScopes`就可以访问这俩个`foo`、`bar`函数。(如果未命名的话，key会是`default`。)

进入`test`组件的内部，假设他是这样定义的：

```vue
<div>
  <slot name="bar"></slot>
  <slot name="foo" v-bind="{ msg }"></slot>
</div>
<script>
  new Vue({
    name: "test",
    data() {
      return {
        msg: "World",
      };
    },
    mounted() {
      // 一秒后更新
      setTimeout(() => {
        this.msg = "Changed";
      }, 1000);
    },
  });
</script>

```

那么`template`就会被编译成这样的函数：

```jsvascript
with(this){
	return _c("div",[_t("bar"),_t("foo",null,null,{msg})],2)
}
```

已经有那么些端倪了，接下来就研究一下`_t`函数的实现，就可以接近真相了。

`_t`也就是`renderSlot`的别名，简化后的实现是这样的：

```javascript
export function renderSlot(
	name: string,
    fallback: ?Array<VNode>,
    props: ?Object,
    bindObject: ?Object
): ?Array<VNode> {
    // 通过 name 拿到函数
    const scopedSlotFn = this.$scopedSlots[name]
    let nodes
    // scoped slot
    if(scopedSlotFn){
        props = props || {}
        // 执行函数返回 vnode
        nodes = scopedSlotFn(props) || fallback
    }
	return nodes;
}
```

其实很简单，

如果是`普通插槽`，就直接调用函数生产`vnode`。

如果是`作用域插槽`，就直接带着`props`也就是`{ msg }`去调用函数生成`vnode`。

在上面的 `test` 组件中，1s后我们通过`this.msg="Changed"`;触发响应式更新，此时编译后的`render`函数：

```javascript
with(this){
    return _c("div",[_t("bar"),_t("foo",null,null,{msg})],2)
}
```

重新执行，此时的`msg`已经是更新后的`Changed`了，自然也就实现了更新。

一种特殊情况是，在父组件的作用域里也使用了响应式的属性并更新，比如这样：

```xml
<test>
  <template v-slot:bar>
    <span>Hello</span>
  </template>
  <template v-slot:foo="prop">
    <span>{{prop.msg}}</span>
  </template>
</test>
<script>
  new Vue({
    name: "App",
    el: "#app",
    mounted() {
      setTimeout(() => {
        this.msgInParent = "Changed";
      }, 1000);
    },
    data() {
      return {
        msgInParent: "msgInParent",
      };
    },
    components: {
      test: {
        name: "test",
        data() {
          return {
            msg: "World",
          };
        },
        template: `
          <div>
            <slot name="bar"></slot>
            <slot name="foo" v-bind="{ msg }"></slot>
          </div>
        `,
      },
    },
  });
</script>

```

其实，是因为执行 `_t` 函数时，全局的组件渲染上下文是 `子组件`，那么依赖收集自然也就是收集到 `子组件`的依赖了。所以在 `msgInParent` 更新后，其实是直接去触发子组件的重新渲染的，对比 2.5 的版本，这是一个优化。

那么还有一些额外的情况，比如说 `template` 上有 `v-if`、 `v-for` 这种情况，举个例子来说：

```xml
<test>
  <template v-slot:bar v-if="show">
    <span>Hello</span>
  </template>
</test>

```

```javascript
function render() {
  with(this) {
    return _c('test', {
      scopedSlots: _u([(show) ? {
        key: "bar",
        fn: function () {
          return [_c('span', [_v("Hello")])]
        },
        proxy: true
      } : null], null, true)
    })
  }
}

```

注意这里的 `_u` 内部直接是一个三元表达式，读取 `_u` 是发生在父组件的 `_render` 中，那么此时子组件是收集不到这个 `show` 的依赖的，所以说 `show` 的更新只会触发父组件的更新，那这种情况下子组件是怎么重新执行 `$scopedSlot` 函数并重渲染的呢？

我们已经有了一定的前置知识：[Vue的更新粒度](https://juejin.im/post/5e854a32518825736c5b807f#heading-3)，知道 `Vue` 的组件不是`递归更新`的，但是 `slotScopes` 的函数执行是发生在子组件内的，父组件在更新的时候一定是有某种方式去通知子组件也进行更新。

其实这个过程就发生在父组件的重渲染的 `patchVnode`中，到了 `test` 组件的 `patch` 过程，进入了 `updateChildComponent` 这个函数后，会去检查它的 `slot` 是否是`稳定`的，显然 `v-if` 控制的 `slot` 是非常不稳定的。

```javascript
  const newScopedSlots = parentVnode.data.scopedSlots
  const oldScopedSlots = vm.$scopedSlots
  const hasDynamicScopedSlot = !!(
    (newScopedSlots && !newScopedSlots.$stable) ||
    (oldScopedSlots !== emptyObject && !oldScopedSlots.$stable) ||
    (newScopedSlots && vm.$scopedSlots.$key !== newScopedSlots.$key)
  )

  // Any static slot children from the parent may have changed during parent's
  // update. Dynamic scoped slots may also have changed. In such cases, a forced
  // update is necessary to ensure correctness.
  const needsForceUpdate = !!hasDynamicScopedSlot
  
  if (needsForceUpdate) {
    // 这里的 vm 对应 test 也就是子组件的实例，相当于触发了子组件强制渲染。
    vm.$forceUpdate()
  }

```

这里有一些优化措施，并不是说只要有 `slotScope` 就会去触发子组件强制更新。

有如下三种情况会强制触发子组件更新：

1. `scopedSlots` 上的 `$stable` 属性为 `false`

一路追寻这个逻辑，最终发现这个 `$stable` 是 `_u` 也就是 `resolveScopedSlots` 函数的第三个参数决定的，由于这个 `_u` 是由编译器生成 `render` 函数时生成的的，那么就到 `codegen` 的逻辑中去看：

```javascript
  let needsForceUpdate = el.for || Object.keys(slots).some(key => {
    const slot = slots[key]
    return (
      slot.slotTargetDynamic ||
      slot.if ||
      slot.for ||
      containsSlotChild(slot) // is passing down slot from parent which may be dynamic
    )
  })
  
```

简单来说，就是用到了一些动态语法的情况下，就会通知子组件对这段 `scopedSlots` 进行强制更新。

2. 也是 `$stable` 属性相关，旧的 `scopedSlots` 不稳定

这个很好理解，旧的`scopedSlots`需要强制更新，那么渲染后一定要强制更新。

3. 旧的 `$key` 不等于新的 `$key`

这个逻辑比较有意思，一路追回去看 `$key` 的生成，可以看到是 `_u` 的第四个参数 `contentHashKey`，这个`contentHashKey` 是在 `codegen` 的时候利用 `hash` 算法对生成代码的字符串进行计算得到的，也就是说，这串函数的生成的 `字符串` 改变了，就需要强制更新子组件。

```javascript
function hash(str) {
  let hash = 5381
  let i = str.length
  while(i) {
    hash = (hash * 33) ^ str.charCodeAt(--i)
  }
  return hash >>> 0
}
```

Vue 2.6 版本后对 `slot` 和 `slot-scope`  做了一次统一的整合，让它们全部都变为函数的形式，所有的插槽都可以在 `this.$slotScopes` 上直接访问，这让我们在开发高级组件的时候变得更加方便。

在优化上，Vue 2.6 也尽可能的让 `slot` 的更新不触发父组件的渲染，通过一系列巧妙的判断和算法去尽可能避免不必要的渲染。（在 2.5 的版本中，由于生成 `slot` 的作用域是在父组件中，所以明明是子组件的插槽 `slot` 的更新是会带着父组件一起更新的）