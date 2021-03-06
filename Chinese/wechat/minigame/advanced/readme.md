# 微信小游戏综合介绍

### 一、LayaAirIDE中的小游戏发布目录详解

![img](img/1-1.png)  

（图1）

#### 1、目录内容介绍

`libs`是引擎库目录

`res`是本地资源目录（关于本地资源目录后面会重点讲）

`code.js`是采用LayaAirIDE发布为小游戏后生成的游戏项目入口文件。

`game.js`是微信小游戏的入口文件，比如游戏项目入口JS文件与适配库JS等都是在这里进行引入。

`game.json` 是小游戏的配置文件，开发者工具和客户端需要读取这个配置，完成相关界面渲染和属性设置。比如屏幕的横竖屏方向，状态栏的显示等，都是在这里配置。具体如何配置，以及参数的使用，可以[查看微信小游戏的开发文档](https://mp.weixin.qq.com/debug/wxagame/dev/index.html?t=2018115)。

`project.config.json`是微信小游戏的项目配置文件。包括了项目的一些信息，如果想修改appid等信息，可以直接在这里面编辑。

> 项目参数libVersion的值一定要是game，这里一般不会出错。但是，万一出现了LayaAirIDE里发布正常，也引用了适配库，发布为小游戏后，在开发者工具里还是有报错的话，可以检查libVersion里的值是不是game。不是的话要改为game。

`weapp-adapter.js` 是微信小游戏适配库文件，LayaAirIDE的发布工具在发布时会自动引用。



#### 2、本地资源目录应用详解

通常我们开发项目的时候，会直接使用本地路径，比如示例中引用的就是本地路径，

```typescript
Laya.Texture2D.load("res/layabox.png");
```

如果项目的目录中，全部大小加起来不超过4M的话，只要能找到本地的资源，怎么写也没问题。

但是，

微信小游戏的本地包有4M的限制，一旦超过这个限制，那就不允许上传，不允许真机预览。

所以，我们的**项目要是大于4M后，怎么处理呢？**

要进行资源目录的规划，分为本地加载与网络动态加载，两种模式结合使用。

本地加载的规划里，我们除了入口文件和必用的配置文件外，只放一些预加载必用的素材，比如加载进度（Loading）页用到的背景与图形等。总之，就是不能超过4M。

> Tips：4M的本地包内容无法动态更新或删除。每次修改必须要重新上传发布，所以尽量只放预加载必须的，不常变的内容。常用的内容，可以放到缓存中进行管理与使用。微信小游戏有50M物理缓存供开发者使用，在首次加载后，一旦存下来，二次使用就是从本地直接加载。

**网络动态加载的路径怎么处理呢**。在本地加载的`load()`方法之后使用`URL.basePath`方法。

例如：

```typescript
material.diffuseTexture = Laya.Texture2D.load("res/layabox.png");
box.meshRender.material = material;
Laya.URL.basePath = "https://XXXX.com";//请把XXXX换成自己的真实网址；
//在此之下，再使用load加载资源，都会自动加入URL网址。从网络上动态加载。
```

使用`URL.basePath`方法后，再使用load加载本地路径，都会自动加上URL.basePath里的网址。这样就实现了本地与网络加载的结合。

如果是内网地址开发，那怎么动态网络加载呢，在微信小项目的项目设置里，勾选不校验安全域名这个选项，否则，必须为HTTPS的安全域名。当然，正式上线还是必须要用HTTPS的。

在设置不校验之后，然后确保内网地址可正常访问，再把资源复制到对应的加载目录，供小游戏网络动态加载。

![img](img/2.png) 

**这样就结束了吗？并没有！**

按刚刚的写法，`res/layabox.png`明明已经上传到微信小游戏的本地目录，但是如果在使用`URL.basePath`之后，再次加载`res/layabox.png`并不会从本地加载使用，而是从网络动态加载使用。这并不是我们要的结果。

所以，引擎针对使用`URL.basePath`之后，如何再次使用本地加载，进行了**特殊目录的处理**。

**只要是包含了wxlocal之个目录名，引擎会自动将该目录视为本地目录**，即便使用了URL.basePath，对于包含了wxlocal目录名的目录，都不会从网络动态加载，只会从本地加载，所以wxlocal必须要放到微信小游戏项目内，作为4M上传的一部分。

目前的LayaAirIDE还没有自动生成wxlocal目录，经过与引擎同事沟通后，下个版本在小游戏示例项目中会自动创建wxlocal目录，小游戏项目里除了根目录那些必用的入口文件和项目配置文件与适配文件外，其它的预加载资源建议都放到wxlocal目录内。这样，下个版本里，再发布微信小游戏时，就只复制带有本地特殊目录关键字的目录（wxlocal）。避免全部复制小游戏项目内，导致无法直接预览，还要把多余的删掉，从网络加载。当然，如果是老项目，还是需要开发者自己手动建立一个wxlocal用于小游戏的发布。



**目录提醒**：

> 需要提一下的是，最早引擎适配库是将layaNativeDir作为本地目录，但是根据开发者的反馈来看，有的开发者把layaNativeDir误认为是和LayaNative有直接关系。所以新增了wxlocal目录关键字并在后续的文档与教程中只推荐wxlocal目录。不过原目录名考虑有一些开发者在使用，也继续保持兼容。也就是说，只要将目录命名为wxlocal与layaNativeDir都不会从网络中动态加载，所以请开发者注意。



### 二、一些踩过的“坑”

#### 1、管理项目，只能创建、切换和删除。

在微信开发者工具里，项目一旦建立，没有编辑修改项目信息的地方。如果想修改项目信息，不用费力气去找这个功能了，直接删除项目重新创建即可。因为工具里的删除只会删除项目信息相关的内容，项目本身不会被删除。

在**项目**菜单里点击**查看所有项目**可以查看到当前存在的项目列表，直接点击项目，可以实现切换，点击加号创建按钮可以创建一个新项目。

如果想删除项目，点击管理项目可进入项目批量删除的界面。

![img](img/4.jpg) 

**2、读本地文件必须是ASCII编码**

之前提到每个游戏有4M本地物理存储空间，这里需要特别注意的，如果需要读取本地物理空间内的配置文件，比如json文件。由于浏览器加载文件编码没有限制，引擎没有预留编码设置接口。而小游戏里读本地资源会校验编码，所以，当小游戏本地文件的编码格式不是ASCII，那就会报错。如果有配置文件存在4M的本地包内。目前必须要改为ASCII编码，后续引擎版本计划支持编码读取设置。

> Tips：本地程序文件之间的引用（比如require或import）或者才是从网络中动态加载读取，都没有编码的校验限制。

**3、不支持loader预加载声音使用**

用LayaAir引擎开发小游戏时，要注意，不支持通过loader预加载声音文件的使用方式，声音播放直接用SoundManger音频管理类即可。

**4、缓存的大小**

微信小游戏的缓存是物理缓存，只有50M，所以对于较大型的项目，要注意控制缓存资源的使用，手动管理不常用的缓存。



### 三、实时调试的建议

在和开发者沟通中，也遇到不少的开发者存在这样的需求：就是先不使用真机调试，每次项目编译完之后，直接在开发者工具中调试，如何处理？

由于4M本地包的限制，所以我们真机调试的正式发布项目，一定要严格遵循这个规范。

假如每次编译后只是想开发工具下调试，而不是真机调试。那每次编译都调用**项目发布**功能其实没有必要，而且项目发布的速度并不快。

这种情况下，开发者可以在微信开发者工具内再创建一个仅用于调试的临时项目。这个项目创建的时候，可以将开发者工具的项目目录设置为IDE编译后的目录bin（AS3是bin/h5）,如下图所示。
![图](img/8.png)  