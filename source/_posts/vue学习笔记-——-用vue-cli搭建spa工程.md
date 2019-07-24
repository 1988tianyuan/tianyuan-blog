title: vue学习笔记（1） —— 用vue-cli搭建spa工程
author: 天渊
date: 2019-03-18 13:09:22
tags:
---
使用webpack和vue搭建搭建单页面应用程序（SPA，Single-Page Application）是前端开发的发展趋势之一，现在来学习一下在Intellij IDEA中使用`vue-cli`搭建一个基于vue框架的SPA demo程序，并部署到nginx服务器
<!--more-->

### 初始化工程

1. 首先保证本机安装有最新版的node.js和npm，使用`node -v`查看版本，不赘述

2. 命令行执行`npm install -g vue-cli`全局安装`vue-cli`
3. 进入想要构建工程的目录，执行命令`vue init webpack project-name`，webpack默认版本目前是2.0
4. 接下来需要为初始化工程进行配置，根据提示按需配置，一般来说直接默认就够了

### 配置IDEA

需要在IDEA中安装vue相关的插件

1. `File -> Settings -> Plugins -> Browse respositoties`搜索`Vue.js`安装，然后重启IDEA

2. `File -> Settings -> Editor -> File Types -> HTML`，将`.vue`文件配置为默认的html类型

3. `File -> Settings -> Language & Frameworks -> JavaScript`，将js版本设置为ES6

4. 使用IDEA打开之前初始化完成的vue工程，点击工具栏的`Edit Configurations`进行启动配置：command选择`run`，Scripts选择`dev`环境

5. 点击启动：
   
	![upload successful](\blog\images\pasted-4.png)

6. 打开`http://localhost:8080/`即可看到官方的HelloWorld页面，接下来在此工程的基础上进行开发即可



### 认识vue-cli工程结构

在vue-cli工程初始化完成后的工程中，目录如下：

![upload successful](\blog\images\pasted-5.png)

1. 最重要的文件夹是`src`，这下面包含了跟页面app有关的所有源代码，包括各类js, css, .vue模板，以及router
2. 除了`src`文件夹，最外层的`index.html`即为项目主页，即`SPA`中那个`single-page`
3. `static`文件夹存放其他类型的静态资源如图片和字体等
4. 其他文件夹（test, build, config）是与项目构建，编译和测试相关的配置，现阶段暂时不用管

为何vue工程有自己独特的`.vue`文件？官网叫其为单文件组件，通过webpack源码转换，会全部转换为对应的文件，通常用于自定义组件模板，包括`template`的html模板，`style`样式以及js脚本，如HelloWorld工程的`App.vue`文件：

![upload successful](\blog\images\pasted-6.png)

`template`为html页面框架，`style`为当前组件的css样式，`script`则主要用于编写并导出当前组件脚本。

### vue-cli工程运行流程

SPA工程遵循一定的规则和流程对页面进行渲染，以当前HelloWorld工程为例：

1. 首先打开主页面`index.html`，vue基于`el`属性，对`id="app"`的这个div进行渲染，脚本位于入口js文件`main.js`中，有关的js脚本以及router文件都需要导入到这个入口js文件中来：

   ```html
   <body>
       <div id="app"></div>
   </body>
   ```

   ```javascript
   new Vue({
     el: '#app',
     router,
     components: { App },
     template: '<App/>'
   })
   ```

2. 如上，`router`为vue-router路由对象，用于配置需要导入的组件页面，本例子中router配置于`/router/index.js`文件中：

   ```javascript
   Vue.use(Router)
   export default new Router({
     routes: [
       {
         path: '/',
         name: 'HelloWorld',
         component: HelloWorld
       }
     ]
   })
   ```

   将`HelloWorld`组件配置到router对象中

3. `components`用于配置自定义组件 `App`，来自于`App.vue`文件：

   ```html
   <template>
     <div id="app">
       <img src="./assets/logo.png">
       <router-view/>
     </div>
   </template>
   
   <script>
   // 命名并导出当前组件
   export default {
     name: 'App'
   }
   </script>
   
   <style>
   ...
   </style>
   ```

4. `template`用于将当前html替换为自定义组件模板`App`，也是来自于`App.vue`文件

5. `<App/>`中的`<router-view/>`标签用于渲染之前router对象中配置的页面，即`HelloWorld.vue`中的内容

至此整个流程完成