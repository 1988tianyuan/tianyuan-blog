title: Nginx -- root和alias路径映射
author: 天渊
tags:
  - Nginx
categories:
  - 基础知识
date: 2019-03-18 12:40:00
---
在nginx中，`root`和`alias`命令都用于将http请求和服务器上的资源地址进行映射，不过使用方式不太相同
<!--more-->
### 使用范围
两个命令在nginx.conf中的作用范围如下：
- root: http、server、location和if中均可配置，用于指定当前范围的根路径
- alias: 尽在location中有效

### 使用方法
root与alias主要区别在于nginx如何解释location后面的uri，这会使两者分别以不同的方式将请求映射到服务器文件上，两者使用方法均为 `root/alisa path`
#### root映射方式：

root路径 + location路径
如下配置中，nginx将会把`/blog/***`这样的请求路径映射到` /root/deploy/static-file/blog`目录下的资源中，因此`/blog/index.html`请求得到的资源即为`/root/deploy/static-file/blog/index.html`

```lua
location /blog {
    root /root/deploy/static-file;
}
```

#### alias映射方式：

  alias路径直接替换原请求路径
  对于alias，如下配置，`/blog/index.html`请求得到的资源仍然为`/root/deploy/static-file/blog/index.html`，

  ```lua
location /blog {
    alias /root/deploy/static-file/blog/;
}
  ```
  因此root和alias主要区别在于，当映射文件路径的时候，前者用于指定根路径，将原请求在根路径的基础上组合成新的路径，而后者用于指定url别名，将该别名作为新路径替换掉原请求路径

**注意**：alias后面的路径必须加`/`正斜杠

### proxy_pass
相应的，`proxy_pass`作为反响代理时，对`/`正斜杠也有一定的讲究
##### path加正斜杠
`proxy_pass`的path加正斜杠时，用法与`alias`一样，都是用新路径替换掉原路径：

```lua
location /tomcat {
    proxy_pass http://localhost:8080/;
}
```

如上配置，nginx监听80端口，当请求`localhost/tomcat`时，请求转发到`http://localhost:8080/`

##### path不加正斜杠

```lua
location /tomcat {
    proxy_pass http://localhost:8080;
}
```

如果去掉正斜杠，当请求`localhost/tomcat`时，请求转发到`http://localhost:8080/tomcat`

这个地方容易踩坑，需要注意