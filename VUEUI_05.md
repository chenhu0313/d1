# `VUEUI Unit05`

#### 初始化加载首页文章列表

1. 把`article_item.vue`中的资源整理到`Home.vue`。
   1. 复制`template`与`style`中的代码，到相应位置。
   2. 把`App.vue`中的`style`整个删掉即可。
2. 发送请求。
   1. 什么时候发这个请求？`mounted`
   2. 如何发请求？以接口文档为准  `GET  axios.get()`
   3. 获取响应后执行后续业务。
3. 渲染文章列表数据。
   1. 一旦获取文章列表信息，则把文章数组存入`data`。
   2. 使用`v-for`在页面中渲染即可。



##### `vue`列表渲染时的图片路径问题

为了能动态显示图片，需要为`src`添加冒号变为动态属性。但是发现动态修改路径后，无法访问图片。这是因为`webpack`在编译打包时，对`src`的静态地址与动态地址处理方式不同。

```html
<img src="../assets/article/1.jpg">    静态地址
<img :src="`../assets/article/${item.image}`">   动态地址
```

如果是静态路径（不带冒号），`webpack`将会通过`vueloader`对路径进行二次处理。如果是动态路径（带冒号），`webpack`将原封不动的把路径字符串设置给`src`。

`vueloader`对`url`资源路径的处理规则：

> 1. 如果路径是绝对路径（`http://`开头、  `/`开头），则`vueloader`不会对该`url`进行处理，直接输出。
> 2. 但是如果路径是相对路径，则`vueloader`将会把该资源当做项目中的模块进行加载，并且对资源进行统一管理，对于图片会重命名后放到`/Img`下，并且为`src`设置该绝对路径。

例如：

假设`<template>`中有如下代码：

```html
<img src="../assets/article/1.jpg">    静态地址
```

将会被编译：

```javascript
let newurl = require('../assets/article/1.jpg')
// 通过require() 方法可以将一个相对路径 转换为 webpack打包后的路径
// newurl:  /img/xxxxxxx.xxxxx.jpg
<img :src='newurl'>  
<img src='/img/xxxxxxx.xxxxx.jpg'>  
```



### 实现切换顶部选项卡时更新文章列表

由于每个选项卡的布局都一样，所以没有必要为每一个类别设计一个独立的面板。切换导航时仅仅更新文章列表即可。干掉三个，留下第一个即可。

当激活某个顶部导航时，获取当前激活项的`id`（`watch`监听`navactive`的变化），发送`http`请求，获取响应数据，替换文章列表。

**实现步骤**

> 编写`watch`监听`navactive`的变化。
>
> 一旦触发了`watch`，获取`newval`（当前激活项`id`），作为`cid`发送请求。
>
> 获取响应，渲染列表。



### 列表的触底分页加载

当列表滚动到底部时，将会触发某个事件，我们可以通过绑定该事件，加载下一页数据。

#### `InfiniteScroll`指令（无限滚动指令）

`InfiniteScroll`用于无限滚动，监听触底事件，其基本使用方法：

```html
<div infinite-scroll-distance="定义离底部的距离阈值"
     v-infinite-scroll="触底事件产生后将执行的函数名称"
     infinite-scroll-disabled="变量名|busy">
    <div>...</div>
    <div>...</div>
	...
</div>
```

案例：`http://localhost:8080/infinite`

1. 新建页面：`testing/Infinite.vue`，在该页面中定义长列表。在该页面中测试无限滚动指令的使用。
2. 配置路由。



#### 在项目中实现列表触底加载下一页

1. 为`container`容器添加无限滚动指令，触底后执行`showmore`方法。
2. 在`showmore`方法内部，尝试发请求加载下一页数据。
   1. 需要在`data`中维护一个变量：`page`（用于描述当前页码），该变量需要在每次触底后递增。++
   2. 当需要发送请求，加载下一页时：`/articles?cid=1&page=this.page`
   3. 获取到响应数据后，不应该直接把列表替换掉，而是向当前列表的末尾进行追加。



#### 解决滚动到底部的问题

**实现思路：**

1. 每次`showmore`发送请求获取响应后，服务端将会返回`pagecount`，表达当前类别下的最大页数。所以我们可以比较当前`page`与`pagecount`谁大谁小，从而知道到底有没有到达最后一页。

2. 如果到底了，可以在页面底部显示提示`div`。我是有底线的。

   实现方案可以在`data`中声明一个变量`reachBottom`用来保存是否到达最后一页，若为`true`则表示已经到达最后一页。

3. 判断`page`与`pagecount`之间的大小，更新`reachBottom`变量的值。

4. 通过`v-if`来动态显示**底线**，`showmore`发请求之前也需要判断`reachBottom`，若到底了则不再发送请求。



### 封装`loadArticles`函数用于加载文章列表

由于`mounted`、`nav`更新、`showmore`时都需要向`/articles`发送请求，都需要传递`cid`、`page` 参数，所以可以封装一个函数`loadArticles`专门用于发送请求，接收响应，处理图片路径等操作。

**实现步骤**

1. 封装`loadArticles`方法

   ```javascript
   // 异步加载文章列表  通过callback执行回调
   loadArticles(cid, page, callback){
       let url = `/articles?cid=${cid}&page=${page}`;
       this.axios.get(url).then(result=>{
           // 处理图片路径
           result.data.results.forEach(item=>{
               if(item.image){
                   // 通过require处理图片路径，重新赋值
                   item.image = require('../assets/articles/'+item.image)
               }
           })
           // 判断是否已经到达底部 
           if(this.page >= result.data.pagecount){
               // 当前页已经大于等于最大页数  => 到底了
               this.reachBottom = true;
           }
           // 执行回调方法  
           callback(result.data.results)
       })
   },
   ```

   

2. 重构`mounted`

   ```javascript
   mounted() {
       // 发送http请求，获取UI类别下的第一页数据
       this.loadArticles(1, 1, (list)=>{
           // list即是发送请求后 响应中的文章列表
           this.articles = list;
       })
   
       // 发送http请求，获取类别列表
       this.axios.get('/category').then(result=>{
           console.log(result)
           // 把服务端返回的类别数组存入data中
           // result.data.results里存储着类别数组 [{},{},{},{}]
           this.category = result.data.results
       })
   
       // 初始化轮播图的高度
       this.initSwipe();
   },
   ```

   

3. 重构`watch` `navactive`

   ```javascript
   watch:{
       // 监听顶部导航的更新
       navactive(newval){
           // 把page变量重置为1 
           this.page = 1;
           this.loadArticles(newval, 1, (list)=>{
               this.articles = list;
           })
       },
   
       // 如果watch中只接受了一个参数，vue将会自动传入新值
       selected(newval){
           if(newval=='home'){
               this.$router.push('/')
           }else if(newval=='me'){
               this.$router.push('/me')  // 需要新建Me.vue
           }
       }
   },
   ```

   

4. 重构`showmore`

   ```javascript
   // 当触发触底事件后执行
   showmore(){
       if(this.reachBottom){
           return;
       }
       // busy = true 发请求，拿响应，追加列表
       this.busy = true;
       let cid = this.navactive;
       this.page++;
       this.loadArticles(cid, this.page, (list)=>{
           // 把list，追加到当前列表末尾
           this.articles = this.articles.concat(list);
           this.busy = false;
       })
   
   }
   ```

   



### 文章详情页的展示

需求是点击首页中其中一个列表项，跳转到详情页，并且传递当前选中文章的`id`。在详情页中接收`id`参数，发送`http`请求，拿到详情数据，渲染页面。

**`vue`程序在页面跳转过程中传参的方式有`2`种**

> 第一种：使用`?`的方式向第二个页面传参
>
> `A`页面向`B`页面跳转时携带参数：
>
> ```html
> <router-link to="/article?id=3&name=zs"></router-link>
> ```
>
> `B`页面接收参数：
>
> ```javascript
> mounted(){
> 	let id = this.$route.query.id
>     let name = this.$route.query.name
> }
> ```
>
> 第二种：使用`路径参数`的方式向第二个页面传参：
>
> `A`页面携带参数跳转到`B`页面：
>
> ```html
> <router-link to="/article/237"></router-link>
> ```
>
> 这种方式跳转，需要在`router`的`index.js`做配置，否则无法跳转到`B`。
>
> ```javascript
> {
>     path: '/article/:id',
>     name: '',
>     component: Article
> }
> ```
>
> `B`页面中接收参数：
>
> ```javascript
> mounted(){
>     let id = this.$route.params.id;
> }
> ```
>
> 

**实现步骤：**

1. 整理文章详情页的页面结构。复制`Article.vue`，配置路由。
2. 为列表项中的每一个`item`新增`routerlink`。跳转的过程中顺便传递`id`。
3. 在详情页中接收`id`参数，发送`http`请求，获取详情数据。
4. 渲染页面。



#### `moment.js`

`moment.js`是一个日期时间的`JS`类库。可以运行在浏览器端与`Node`环境下。

**常用操作**

1. 创建`moment`对象，表达一个时刻。

   ```javascript
   let day = moment();
   let day = moment(131232312312) 
   let day = moment('2020-10-10')
   let day = moment('2020-10-11 22:12', 'YYYY-MM-DD HH:mm')
   ```

2. 通过`moment`对象输出相应格式的字符串。

   ```javascript
   let day = moment()
   day.format()  =>  字符串  
   day.format('YYYY年MM月DD日')
   ```

安装`moment.js`

```shell
cd scaffolding
npm install --save moment
```

`main.js`中引入`moment`

```javascript
import moment from 'moment'
Vue.prototype.moment = moment
```

































## 出的错误

1. `Module not found: Error: Can't resolve 'elment-ui/lib/theme-chalk/index.css' in 'D:\code2012\VUEUI\day01\demo\ele_project\src'` 
2. `unknown custom element <el-button>`
3. `Failed to compile. SyntaxError: D:\code2012\VUEUI\day01\demo\ele_project\src\router\index.js: Identifier 'Button' has already been declared (5:7)`
4. `unknown custom element <mt-header>`









