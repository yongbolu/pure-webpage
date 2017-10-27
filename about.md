# 搭建一个多页面的无依赖的工程化项目

前端到如今这个阶段，再让大家接手一个没有工程化的项目，肯定内心非常抵触了。试想这么一个项目，手动link资源，不能写less/sass，不能写ES6，不能依赖管理，不能编译打包...，哦天，想都不敢想。可是工程化这事在实际业务中却没有大家想象中的那么顺利。

目前大家的工程化方案多是一整套全家桶，如vue-cli的webpack模板。可实际上我们会遇到这样的阻碍：

1. 项目多展示页面，交互逻辑简单，不需要vue，react这些框架。
2. 需要考虑SEO，不想走单页应用，更没必要部署node走服务端渲染。
3. 由于历史等原因，路由必须过后端，前后端无法完全分离，还需要后端套页面。

举个例子，一些公司的企业网站。遇到这些项目，往往会有这些问题：

1. 由于页面是后端渲染，需要部署后端程序（php,java之类），各种环境配置相当麻烦。
2. 前端的html代码依托于服务端，导致前端做工程化时，很难对接好前后端工程。

也就是说，我们需要做一个多页面的工程化项目。虽然这个项目页面是交于后端渲染的，但这个项目也不想跟后端耦合，可以不依托于服务端程序本地跑开发环境。

怎么办呢，其实也很简单，我们只要改造下现有的轮子，把[vue-cli](https://github.com/vuejs/vue-cli)的webpack模板改造一下，删了没必要的vue-loader，给它增加一下多页面入口就好了。

## 修改Vue脚手架项目模板

### 第一步：理解 [vuejs-templates/webpack](https://github.com/vuejs-templates/webpack)

```javascript
npm install -g vue-cli

vue init webpack my-project
```

既然要改人家的模板，先得理解人家都做了什么。这里就不带大家读代码了，根据package.json的命令一个个文件的代码看过去就知道了，很直接很暴力。

### 第二步：删

既然我们不需要用vue，那么对于vue文件处理的相关逻辑我们就不需要了。根据刚刚对这个模板的了解，我们知道`vue-loader`跟`vue-style-loader`是不需要的。所以删除对应的代码跟package.json里面的包就好了。

多提一点的是，`vue-style-loader`虽然不需要，`style-loader`还是需要的，所以需要用后者替换前者。

### 第三步：加

做减法容易，做加法就没这么轻松了。根据我们刚刚的需求，我们应该给它加个多页面入口。网上有非常多的webpack多入口配置教程。然而他们不一定就能满足我们的需求。他们普遍存在如下问题：

1. **入口文件需要自己配置**。在一个页面较多的项目中，入口文件应当从约定的目录中自动读取，也更符合约定优于配置。
2. **多入口是针对js的**。由于业界普遍是在用单页应用，页面由js生成，故多页面只要多个js入口就好，不需要直接写html。而我的需求不是，我希望的多入口是针对html文件而言的。

不过当我们解决了上述两个问题后，我们还会有一个新的问题。我们不同的html文件，其实又是有公共的部分的。比如都有 header，footer。也就是说，我们需要给这些html文件增加一个模板。我们可以通过webpack的loader来实现，但是没有现成的loader可以比较好的解决。那怎么办呢？可以参考我另外一篇文章。[编写自己的Webpack Loader](https://juejin.im/post/59df06e6f265da430d5701d0)。

## 静态资源的版本控制

上述问题解决后，我们的工作并未完成。现在这个项目的静态资源是以文件哈希值来控制的。可惜有的项目的静态资源是要后端来更新时间戳控制的。虽然这不是个好方案，但有些工程却依旧是这样。没办法，为了适应他们，我们必须得去掉哈希值。可是这样的话，当我们想更新css内引用的图片时又没辙了，因为css内链的图片后端没法控制版本。

这个该怎么解决呢？感谢webpack，我们可以通过如下的配置来实现：
```javascript
{
  test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
  oneOf: [
    {
      issuer: /\.html$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: utils.assetsPath('img/[name].[ext]',)
      }
    },
    {
      issuer: /\.(css|less)$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: utils.assetsPath('img/[name].[hash:7].[ext]')
      }
    }
  ]
}
```
意思就是如果图片是在html中引用的则不加哈希值，在css和less中引入的则加上。

## 完工

这样我们就完成了一个简单的项目架构。它能帮助我们实现文件的打包、编译，html的模板控制等功能。最终能build出一份html+静态资源的web页面直接发布cdn。当然也可以把它们直接扔给后端。

不过这个架子还不是非常的完善，应用场景也有限，比较适用于一些**交互较少、页面较多、看重seo或者传统后端套页面**的网站。另外，作为工程化中非常重要的组件化与测试，由于没有任何框架的引入，这点也需要使用者自己再去摸索。

最后，如果这个架子对您有用，欢迎戳开[github](https://github.com/wuomzfx/pure-webpage)

