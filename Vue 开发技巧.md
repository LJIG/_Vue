# Vue 开发技巧

## 路由参数解耦

一般在组件内使用路由参数，大多数人会这样做：

```javascript
export default{
	methods: {
        getParamsId(){
            return this.$route.params.id
        }
    }	
}
```

在组件中使用 `$route` 会使之与其对应路由形成高度耦合，从而使组件只能在某些特定的 URL 上使用，限制了其灵活性。

正确做法通过`props` 解耦

```javascript
const router = new VueRouter({
    routes:[{
        path: '/user/:id',
        component: User,
        porps: true
    }]
})
```

将路由的`props`属性设置为`true`后，组件内可以通过props接收到`params`参数

```javascript
export default{
    props:['id'],
    methods:{
        getParamsId(){
            return this.id
        }
    }
}
```

还可以通过函数模式来返回`props`

```javascript
const router = new VueRouter({
    router:[{
        path:'/user/:id',
        component:User,
        props:(route)=>({
            id: route.query.id
        })
    }]
})
```

文档：[router.vuejs.org/zh/guide/es…](https://router.vuejs.org/zh/guide/essentials/passing-props.html)

## 函数式组件

函数式组件是无状态，他无法实例化，没有任何的生命周期和方法。创建函数式组件也很简单，只需要在模板添加`functional`声明即可。一般适合只依赖于外部数据的变化而变化的组件，因其轻量，渲染性能也会有所提高。

组件需要的一切都是通过`context`参数传递。他是一个上下文对象，具体属性查看[文档](https://cn.vuejs.org/v2/guide/render-function.html#函数式组件)。这里`props`是一个包含所有绑定属性的对象。

函数式组件

```vue
<template functional>
	<div class="list">
        <div class="item"
             v-for="item in props.list" :key="item.id"
             @click="props.itemClick(item)"
       >
            <p>{{item.title}}</p>
            <p>{{item.content}}</p>
	    </div>
    </div>
</template>
```

父组件使用

```vue
<template>
	<div>
        <List :list="list" :itemClick="item=>(currentItem=item)"/>
    </div>
</template>
<script>
import List from '@/components/List.vue'
export default{
    components:{List},
    data(){
        return {
            list:[{
                title: 'title',
                content: 'content'
            }],
            currentItem: ''
        }
    }
}
</script>
```

文档：[cn.vuejs.org/v2/guide/re…](https://cn.vuejs.org/v2/guide/render-function.html#函数式组件)

## 样式穿透

在开发中修改第三方组件样式很常见，但由于`scoped`属性的样式隔离，可能需要去除`scoped`或是另起一个`style`。这些做法都会带来副作用(组件样式污染、不够优雅)，样式穿透在css预处理器中使用才生效。

我们可以使用`>>>`或`/deep/`解决这一问题

```scss
<style>
外层>>>.el-checkbox{
    display:block;
    font-size:26px;
    
    .el-checkbox_label{
        font-size:16px;
    }
}
</style>
```

``` scss
<style scoped>
/deep/ .el-checkbox{
    display:block;
    font-size: 26px;
    
    .el-checkbox_label{
        font-size:16px;
    }
}
</style>
```

## watch高阶使用

### 立即执行

`watch`是在监听属性改变时才会触发，有时候，我们希望组件创建后`watch`能够立即执行可能想到的方法就是在`create`生命周期中调用一次，但这样的写法不优雅，或许我们可以使用这样的方法

```javascript
export default{
    data(){
        return {
            name: 'Jack'
        }
    },
    watch:{
        name: {
            handler: 'sayName',
            immediate: true
        }
    },
    methods:{
        sayName(){
            console.log(this.name)
        }
    }
}
```

### 深度监听

在监听对象时，对象内部的属性被改变时无法触发`watch`,我们可以为其设置深度监听

```javascript
export default{
    data(){
        return {
            studen:{
                name: 'Jack',
                skill:{
                    run:{
                        speed: 'fast'
                    }
                }
            }
        }
    },
    watch:{
        studen:{
            handler: 'sayName',
            deep: true
        }
    },
    mwtchods:{
        sayName(){
            console.log(this.studen)
        }
    }
}
```

### 触发监听执行多个方法

使用数组可以设置多项，形式包括字符串、函数、对象

```javascript
export default{
    data(){
        return {
            name: 'Jack'
        }
    },
    watch:{
        name:[
            'sayName1',
            function(newVal,oldVal){
                this.sayName2()
            },
            {
                handler:'sayName3',
                immeadiate:true
            }
        ]
    },
    methods:{
        sayName1(){
            console.log('sayName1==>',this.name)
        },
        sayName2(){
            console.log('sayName2==>',this.name)
        },
        sayName3(){
            console.log('sayName3==>',this.name)
        }
    }
}
```

文档：[cn.vuejs.org/v2/api/#wat…](https://cn.vuejs.org/v2/api/#watch)

### watch监听多个变量

watch本身无法监听多个变量。但我们可以将需要监听的多个变量通过计算属性返回对象，在监听这个对象来实现"监听多个变量"

```javascript
export default{
    data(){
        return {
            msg1: 'apple',
            msg2: 'banana'
        }
    },
    compouted:{
        msgObj(){
            const {msg1,msg2} = this
            return {
                msg1,
                msg2
            }
        }
    },
    watch:{
        msgObj:{
            handler(newVal,oldVal){
                if(newVal.msg1!=oldVal.msg1){
                    console.log('msg1 is change')
                }
                if(newVal.msg2!=oldVal.msg2){
                    console.log('msg2 is change')
                }
            }
        }
    }
}
```

## 事件参数$event

`$event`是事件对象的特殊变量，在一些场景能给我们实现复杂功能提供更多可用的参数

### 原生事件

在原生事件中表现和默认的事件对象相同

```vue
<template>
	<div>
        <input type="text" @input="inputHandler('hello', $event)" />
    </div>
</template>
<script>
export deault{
    methods:{
        inputHandler(msg,e){
            console.log(e.target.value)
        }
    }
}
</script>
```

### 自定义事件

在自定义事件中表现为捕获从子组件抛出的值

`my-item.vue`

```javascript
export default{
    methods:{
        customEvent(){
            this.$emit('custom-event','some value')
        }
    }
}
```

`App.vue`

```vue
<template>
	<div>
        <my-item v-for="(item,index) in list" :key="index"
        	@custom-event="customEvent(index,$event)"
        />
    </div>
</template>
<script>
export default{
    methods:{
        customEvent(index,e){
            console.log(e) // some value
        }
    }
}
</script>
```

文档：[cn.vuejs.org/v2/guide/ev…](https://cn.vuejs.org/v2/guide/events.html#内联处理器中的方法)

[cn.vuejs.org/v2/guide/co…](https://cn.vuejs.org/v2/guide/components.html#使用事件抛出一个值)

## 自定义组件双向绑定

组件`model`选项：

> 允许一个自定义组件在使用v-model时定制prop和event。默认情况下，一个组件上的v-model会把value用作prop且把input用作event,但是一些输入类型比如单选框和复选框按钮可能使用value prop来达到不同的目的。使用model选项可以回避这些情况产生冲突。

`input`默认作为双向绑定的更新事件，通过`$emit`可以更新绑定的值

```vue
<my-switch v-model="val" ></my-switch>
```

```javascript
export default{
    props:{
        value: {
            type: Boolean,
            default: false
        }
    },
    methods:{
        switchChange(val){
            this.$emit('input',val)
        }
    }
}
```

修改组件的`model`选项，自定义绑定的变量和事件

```vue
<my-swich v-model="num" value="some value"></my-swich>
```

```javascript
export default{
    model:{
        prop: 'num',
        event: 'update'
    },
    props:{
        value:{
            type:String,
            default: ''
        },
        num:{
            type:Number,
            default: 0
        }
    },
    methods:{
        numChange(){
            this.$emit('update',num++)
        }
    }
}
```

文档：[cn.vuejs.org/v2/api/#mod…](https://cn.vuejs.org/v2/api/#model)

## 监听组件生命周期

通常我们监听组件生命周期会使用`$emit`,父组件接收事件来进行通知

子组件

```javascript
export default{
    mounted(){
        this.$emit('listenMuounted')
    }
}
```

父组件

```vue
<template>
	<div>
        <List @listenMounted=="listenMounted"></List>
    </div>
</template>
```

其实还有一种简洁的方法，使用`@hook`即可监听组件生命周期，组件内无需做任何改变。同样的，`created`、`updated`等也可以使用此方法。

```vue
<template>
	<List @hook:mounted="listenMounted"></List>
</template>
```

## 程序化的事件侦听器

比如，在页面挂载时定义计时器，需要在页面销毁时清除定时器。这看起来没什么问题。但仔细看`this.timer`唯一的作用只是为了能够在`beforeDestroy`内取到计时器序号，除此之外没有任何用处。

```javascript
export default{
    mounted(){
        this.timer=setInterval(()=>{
            console.log(Date.now())
        },1000)
    },
    beforeDestroy(){
        clearInterval(this.timer)
    }
}
```

如果可以的话最好只有生命周期钩子可以访问到它。这并不算严重的问题，但是它可以被视为杂物。

我们可以通过`$on`或`$once`监听页面生命周期销毁来解决这个问题：

```javascript
export default{
    mounted(){
        this.creatInterval('hello')
        this.creatInterval('world')
    },
    creatInterval(msg){
        let timer = serInterval(()=>{
            console.log(msg)
        },1000)
        this.$once('hook:beforeDestroy',function(){
            clearInterval(timer)
        })
    }
}
```

使用这个方法后，即使我们同时创建多个计时器，也不影响效果。因为它们会在页面销毁后程序化的自主清除。

文档：[cn.vuejs.org/v2/guide/co…](https://cn.vuejs.org/v2/guide/components-edge-cases.html#程序化的事件侦听器)

## 手动挂载组件

在一些需求中，手动挂载组件能够让我们实现起来更加优雅。比如一个弹框组件，最理想的用法是通过命令式调用，就像`elementUI`的`this.$message`。而不是在模板中通过状态切换，这种实现真的糟糕。

先来个最简单的例子：

```javascript
import Vue from 'vue'
import Message from './Message.vue'

// 构造子类
let MessageConstructor = Vue.extend(Message)
// 实例化组件
let messageInstance = new MessageConstructor()
// $mount 可以传入选择器字符串，表示挂载到该选择器
// 如果不传入选择器，将渲染为文档之外的元素，你可以想象成 document.createElement()在内存中生成dom
messageInstance.$mount()
// messageInstance.$el获取的是dom元素
document.body.appendChild(messageInstance.$el)
```

`Message/index.vue`

```vue
<template>
    <div class="wrap">
        <div class="message" :class="item.type" v-for="item in notices" :key="item._name">
            <div class="content">{{item.content}}</div>
        </div>
    </div>
</template>
<script>
// 默认选项
const DefaultOptions = {
    duration: 1500,
    type: 'info',
    content: '这是一条提示信息！',
}
let mid = 0
export default {
    data() {
        return {
            notices: []
        }
    },
    methods: {
        add(notice = {}) {
            // name标识 用于移除弹窗
            let _name = this.getName()
            // 合并选项
            notice = Object.assign({
                _name
            }, DefaultOptions, notice)

            this.notices.push(notice)

            setTimeout(() => {
                this.removeNotice(_name)
            }, notice.duration)
        },
        getName() {
            return 'msg_' + (mid++)
        },
        removeNotice(_name) {
            let index = this.notices.findIndex(item => item._name === _name)
            this.notices.splice(index, 1)
        }
    }
}

</script>
<style>
// 默认选项
const DefaultOptions = {
    duration: 1500,
    type: 'info',
    content: '这是一条提示信息！',
}
let mid = 0
export default {
    data() {
        return {
            notices: []
        }
    },
    methods: {
        add(notice = {}) {
            // name标识 用于移除弹窗
            let _name = this.getName()
            // 合并选项
            notice = Object.assign({
                _name
            }, DefaultOptions, notice)

            this.notices.push(notice)

            setTimeout(() => {
                this.removeNotice(_name)
            }, notice.duration)
        },
        getName() {
            return 'msg_' + (mid++)
        },
        removeNotice(_name) {
            let index = this.notices.findIndex(item => item._name === _name)
            this.notices.splice(index, 1)
        }
    }
}

</style>
```

`Message/index.js`

```javascript
import Vue from 'vue'
import Index from './index.vue'

let messageInstance = null
let MessageConstructor = Vue.extend(Index)

let init = () => {
    messageInstance = new MessageConstructor()
    messageInstance.$mount()
    document.body.appendChild(messageInstance.$el)
}

let caller = (options) => {
    if (!messageInstance) {
        init(options)
    }
    messageInstance.add(options)
}

export default {
    // 返回 install 函数 用于 Vue.use 注册
    install(vue) {
        vue.prototype.$message = caller
    }
}

```

`main.js`

```javascript
import Message from '@/components/Message/index.js'

Vue.use(Message)

```

使用

```
this.$messagae({
    type: 'success',
    content: '成功信息提示',
    duration: 3000
})

```

文档：[cn.vuejs.org/v2/api/#vm-…](https://cn.vuejs.org/v2/api/#vm-mount)