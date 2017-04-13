<!--
@Author: songqi
@Date:   2017-01-09
@Last modified by:   songqi
@Last modified time: 2017-04-13
-->

## 前言

阿里的 weex 给我们带来了用 js 写出原生应用的体验，也让我们可以通过一些办法使得业务代码可以进行远程更新了，通过远程更新带来的好处这里就不用赘述了，基本上就是解决了现在客户端的各种痛点。

在解决升级问题的期间，苹果禁止了 jsPatch，众多 APP 被警告，风吹的比实际的伤害大太多，让我们也是彷徨无助了很久，最终也是果敢的决定，我们要先提审一版，在方案做好之后提交审核的过程中做了不计其数的好人好事，终于也是顺利给过了。

希望能写一个系列对我们这次使用 weex 解决方案进行总结，首先不应该是搭建项目么？在技术调研期间如果不把更新问题考虑清楚，其实还不如使用原生开发，所以下面主要是更新的一些实践，基于当时的实现思路介绍一下我们的 weex 解决方案。

## 前置依赖

### 脚手架依赖

    sudo npm install -g BMFE_scaffold

### bsdiff 生成 zip 包

    网上查一下 bsdiff 安装，本地安装很快，开发机编译安装也只用了不到五分钟

## node_module 依赖

    cd demo && npm install

## 启动服务

    // 启动一个服务，此时可以访问到一个 bundle.js
    sudo BM server

## 模拟线上打包

    // 模拟打包机打包，如果需要生成 diff 包，需要配置本地文件夹 'zipFolder': '/home/app'
    sudo BM minWeex

## 哪些需要更新

进入正题，一个混合应用包含下面下面这些需要更新的部分：

> * native 代码部分
> * js 业务代码部分
> * 图片、iconfont 资源部分

#### native 代码部分

现在苹果禁止掉 jsPatch 之后，这个就不要再想了，只有等到什么时候苹果自己出了一个热更新了...

#### js 业务代码部分

我们认为所有脱离了 native 部分的代码都是业务代码，我们的页面非常多，并且我们有自己封装的路由，所以我们基本上是一个页面对应一个 js 文件，就像是回到了多年前的多页面时代。

我们认为所有的 js 代码只要发生改变就应该做增量的更新。

#### 图片、iconfont 资源部分

图片部分：由于图片我们的需求并不是特别高，图片目前只是少量，图片这块目前没有进行压缩打包，目前的方案是发送请求，然后缓存到本地，下次打开的时候就会快了。

iconfont：如果在样式解释的时候没有找到字体图标文件，字体图标就会找不到，并且 iconfont 请求完成之后不会更新当前视图，所以 iconfont 必须要做增量更新

#### 更新包

更新的部分包括：js 业务代码、iconfont。我们目前将这两部分的代码加上一个 md5.json，生成一个 .zip 压缩包。
![zip 包内容](media/14899833557250.jpg)￼

md5.json 内容如下：

	{
		// 所有页面对应的 js 文件，用于做单个文件完整性校验
	    "filesMd5": [{
	        "android": "1.0.0",
	        "iOS": "1.0.0",
	        "page": "/pages/home/index.js",
	        "md5": "c2125dfab756dc0e9cfe854a297a0512"
		}, {
	        "android": "1.0.0",
	        "iOS": "1.0.0",
	        "page": "/iconfont/iconfont.ttf",
	        "md5": "50ed903231bcdc851bfde9a0bf565e38"
	    }],
	    // 当前 zip 包依赖的最低的安卓版本号
	    "android": "1.0.0",
	    // 当前 zip 包依赖的最低的 iOS 版本号
	    "iOS": "1.0.0",
	    // 当前这个 appName，用于区分不同 APP 业务的 key
	    "appName": "demo",
	    // 当前这个 zip 包的版本号
	    "jsVersion": "56a4569f271294a05e4ff0f567d332b4",
	    // 生成当前 zip 包版本号的时间戳
	    "timestamp": 1489983294137
	}
filesMd5 中的每一项就是当前这个 zip 包文件中的文件信息，根据每个文件内容会生成一个 MD5 值，当前包的 MD5 值作为当前包的版本号。

客户端可以根据每一个文件来看下载下来的文件有没有缺失，如果有缺失，这里就需要看策略了，是直接去请求线上最新的这个 js 文件还是重新下载 zip 包。

## 如何增量

当出现第一个版本之后，我们就需要考虑每次升级的问题了，我们肯定不希望每次都去下载完整的 zip 包，浪费流量。也不可能实时 diff，这样更新肯定会慢，所以我们想用空间换时间的方式。

每生成一次完整的资源包，需要和当前已有的资源包进行一次 diff，生成若干差分包。每次 APP 更新的时候只需要下载差分包就行了。

如：

新版本(v3) - 旧版本(v1，v2) = 全新的 v3 包、v3-v1 差分包、v3-v2 差分包

然后，APP 根据自己的当前版本，比如 v2，下载对应的增量包 v3-v2。

然后本地进行合并。 v2 完整包 + v3-v2 差分包，等到完整的 v3 新包。

## 增量包生成

zip 包的生成依赖的是 bsdiff，这个需要本地装一个 bsdiff

	// 命令行生成差分包命令：
	bsdiff oldZip newZip diffZip

	//命令行合并差分包命令：
	bspatch oldZip diffZip newZip

生成的 diffZip 包命名可以有一些命名的算法，这样的命名在存储的时候就不需要存差分包和完整包的映射 了。

在生成差分包时需要指定所有完整的包目录，在 demo 中 config.js 有对应的配置，每次发布都会将完整的 zip 放在 打包机上，每次都会和留下的这些完整的包进行对比生成差分包。
至此，我们的打包工作就完成了

我们的脚手架对目录有一定的格式要求，所以可以去 [weex-demo](https://github.com/karynsong/weex-demo)拉下代码，然后加载依赖，尝试一下更新流程。

下面我们看看整个升级的流程

## 更新设计

![更新方案](https://lev-inf.benmu-health.com/resource/image/3dd2d0e046ea585f7ef5fd133170b9e8.jpg)


## 包资源上传

#### 增量包

增量包打包完成之后，将增量包发送至静态资源服务器进行存储，如果有 CDN 可以放到 CDN 上。

#### 版本描述信息

版本描述信息用于升级服务器判断 APP 当前版本是否需要升级，脚手架打包时也会生成对应的 version.json，并且会将此 json 文件上传至升级服务器。

	{
		// 当前 zip 包依赖最低安卓版本
	    "android": "1.0.0",
	    // 当前 zip 包依赖最低 iOS 版本
	    "iOS": "1.0.0",
	    当前这个 appName，用于区分不同 APP 业务的 key
	    "appName": "demo",
	    // 当前 zip 包的版本号
	    "jsVersion": "cd34091af113f98c2cbf4d81131ccdde",
	     // 生成版本号时的时间戳
	    "timestamp": 1489997936509,
	     // 对应的静态资源服务器的地址
	    "jsPath": "https://xxx.xxx.com/app/"
	}

## 检测更新

请求参数：

	{
		// 当前客户端版本号
		"android[or iOS]": "1.0.0",
		// 当前 APP 业务的名称
		"appName": "demo",
		// 当前 zip 包的版本号
		"jsVersion": "cd34091af113f98c2cbf4d81131ccdde",
		// 是否需要差分包，如果是 false 则返回完整的 zip 包，完整 zip 包多用于特殊情况
		"isDiff": true
	}

### 更新场景1

发布了新版本，需要更新，这也是最常见的更新场景。发送请求参数如下：

	{
		// 当前客户端版本号
		"android[or iOS]": "1.0.0",
		// 当前 APP 业务的名称
		"appName": "demo",
		// 当前 zip 包的版本号
		"jsVersion": "cd34091af113f98c2cbf4d81131ccdde",
		// 是否需要差分包，如果是 false 则返回完整的 zip 包，完整 zip 包多用于特殊情况
		"isDiff": true
	}

#### 接口返回场景1

当前 jsVersion 在数据库中不存在，这种情况下无法生成差分包，也有可能是包的信息被篡改了，此时返回需要更新，并返回完整的更新包地址，客户端不做差分合并

	{
		// 此次请求有没有问题
		"resCode": 0,
		// 此次请求描述
		"msg": "当前版本已是最新，不需要更新",
		// 回溯的数据
		"data": {
			// 表示这是一个完整的包
			"isDiff": false,
			// 完整包的地址
			"jsPath": "https://xxx.xxx.com/app/cd34091af113f98c2cbf4d81131ccdde.zip"
		}
	}

#### 接口返回场景2

当前 jsVersion 在数据库中存在，当前客户端版本号下，jsVersion 已经是最新的，不需要更新

	{
		// 此次请求有没有问题
		"resCode": 1000,
		// 此次请求描述
		"msg": "当前版本已是最新，不需要更新",
		// 回溯的数据
		"data": {}
	}

#### 接口返回场景3

当前 jsVersion 在数据库中存在，当前客户端版本号下，jsVersion 还有更新的版本，获取差分包，进行合并再解压。

服务端查询当前库中最新的版本号，得到差分包的下载地址并返回。

	{
		// 此次请求有没有问题
		"resCode": 0,
		// 此次请求描述
		"msg": "当前版本已是最新，不需要更新",
		// 回溯的数据
		"data": {
			// 当前下载的 js 的 verison
			"jsVersion": "cd34091af113f98c2cbf4d81131ccdde",
			// 表示这是一个完整的包
			"isDiff": true,
			// 完整包的地址
			"jsPath": "https://xxx.xxx.com/app/cd34091af113f98c2cbf4d81131ccdde.zip"
		}
	}

#### 接口返回场景4

当前 jsVersion 在数据库中存在，当前客户端版本号下，jsVersion 还有更新的版本，但是获取差分包失败，这个情况下客户端会重新发起一个请求，获取当前客户端版本号下完整的 zip 包，

### 更新场景2

本地不存在包，或者校验包的完整性发现 zip 包有问题时，直接请求最新的 zip 包。

	{
		// 当前客户端版本号
		"android[or iOS]": "1.0.0",
		// 当前 APP 业务的名称
		"appName": "demo",
		// 是否需要差分包，如果是 false 则返回完整的 zip 包，完整 zip 包多用于特殊情况
		"isDiff": false
	}

#### 接口返回场景1

当前 jsVersion 在数据库中不存在，这种情况下无法生成差分包，也有可能是包的信息被篡改了，此时返回需要更新，并返回完整的更新包地址，客户端不做差分合并

	{
		// 此次请求有没有问题
		"resCode": 0,
		// 此次请求描述
		"msg": "当前版本已是最新，不需要更新",
		// 回溯的数据
		"data": {
			// 表示这是一个完整的包
			"isDiff": false,
			// 完整包的地址
			"jsPath": "https://xxx.xxx.com/app/cd34091af113f98c2cbf4d81131ccdde.zip"
		}
	}
