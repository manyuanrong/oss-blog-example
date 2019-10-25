部署博客有很多选择，国内外都有很多服务可以用，各有各的优缺点：

| | [GithubPages](https://pages.github.com/) | [码云Pages](https://gitee.com/help/articles/4136) |[Netlify](https://www.netlify.com/) | [Heroku](https://www.heroku.com) | [阿里云OSS](https://www.aliyun.com/product/oss) |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 纯静态托管 | 是 | 是 | 是 | 否👍 | 是 |
| CDN加速 | 否 | 否 | 是👍 | 否 | 是👍 |
| 访问速度 | 慢 | 快👍 | 一般 | 一般 | 很快👍 |
| 支持404重定向 | 否 | 是👍 | 是👍 | 是👍 | 是👍 |
| 自定义重定向 | 否 | 否 | 是👍 | 是👍 | 否 |

具体选择哪个，根据个人对博客的诉求进行选择。

* 访问速度快：优先选择阿里云(国内CDN加速)、其次是码云(国内服务器)
* 功能最强：选择Heroku，支持Node.js、PHP等后端

本文章要讲的是如何用 GitHub + Github Action + 阿里云OSS 部署博客
因为静态博客系统有很多选择，Jekyll、Hugo、Hexo等。这里选择Hexo

Github地址
[https://github.com/manyuanrong/oss-blog-example](https://github.com/manyuanrong/oss-blog-example)

简书地址
[https://www.jianshu.com/p/99952652b2dd](https://www.jianshu.com/p/99952652b2dd)

#### 创建项目
```shell
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```

#### 开发和编写博客内容
这一块内容有很多文章讲解，请参考其他文章

#### 提交代码
![image.png](https://upload-images.jianshu.io/upload_images/1771862-2e464454f6b133c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
代码提交之后并不会发生什么事情。只是保持小步提交的好习惯。接下来我们开始使用GitHub Action来做持续集成，生成静态页面。

#### 开通并配置oss
去阿里云创建一个公共读权限的 Bucket，用来托管我们的静态网站。
选择香港区域是为了不用备案也能绑定自定义域名。
![image.png](https://upload-images.jianshu.io/upload_images/1771862-edbdf2c6fc9a49a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们在阿里云控制台拿到一份 accesskeys。用于后面的远程部署文件到OSS上
![image.png](https://upload-images.jianshu.io/upload_images/1771862-565ef73bf1ef58da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们将拿到的 accesskeys配置到GitHub项目的 secrets 里面，后面会在工作流脚本中用来掉调用API上传文件到OSS上面。
保存在 secrets 里面，这样其他人就不能获取到你的 accesskeys ，保证安全。
![image.png](https://upload-images.jianshu.io/upload_images/1771862-ed9df75fc8e49e1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 编写 Github Action 持续集成脚本
##### 1.创建持续集成配置脚本文件
```shell
# 使用mkdir创建目录(windows可以手动建）
mkdir -p .github/workflows
# 使用touch 创建配置文件，名称随意，后缀名为 ".yml" （window可以手动建立文件）
touch ci.yml
```
workflows下面每个文件就是一个工作流，可以有多个，一般建一个就可以了。

##### 2.编写脚本
```yml
# workflow的名称，会显示在工作流运行页面
name: MainWorkflow

# 工作流执行的契机：push表示每次push代码之后都会执行
on: [push]

jobs:
  # build job 我们用来做持续构建
  build:
    # 构建运行的环境
    runs-on: ubuntu-latest
    # 构建步骤
    steps:
    # 复用 actions/checkout@v1 action，拉取最新代码
    - uses: actions/checkout@v1
```

关于 Action 的详细写法，可以参考官方文档
[https://help.github.com/cn/github/automating-your-workflow-with-github-actions/about-github-actions](https://help.github.com/cn/github/automating-your-workflow-with-github-actions/about-github-actions)

上面我只写了 actions/checkout 这个action。这是官方提供的action，我们直接复用就可以了。它的作用是拉取最新代码

由于我们使用了 Hexo ，因此我们需要使用一个action来安装配置Node.js环境。
我们可以去GitHub的市场上寻找别人写好的Action。
[https://github.com/marketplace?type=actions](https://github.com/marketplace?type=actions)

我们找到并加入一个叫 ```actions/setup-node``` 的action，按照文档我们将工作流脚本进行修改

```yml
name: MainWorkflow

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - uses: actions/setup-node@v1
      with:
        node-version: "12.x"
```
此时我们安装了Node.js 12，后面的步骤我们就可以开始执行Hexo的构建操作了。

我们将配置改成如下

```yml
name: MainWorkflow

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - uses: actions/setup-node@v1
      with:
        node-version: "12.x"
    - name: Build Blog
      run: |
        npm install
        npm install -g hexo-cli
        hexo generate
```
此时我们添加了一个步骤，用于安装node依赖，安装hexo命令行。最后生成静态页面。

现在我们已经有了生成的静态页面，只需要将它部署到我们的OSS上，我们可以使用 ```manyuanrong/setup-ossutil``` 这个action来部署我们的文件

```yml
name: MainWorkflow

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - uses: actions/setup-node@v1
      with:
        node-version: "12.x"
    - name: Build Blog
      run: |
        npm install
        npm install -g hexo-cli
        hexo generate
    - uses: manyuanrong/setup-ossutil@v1.0
      with:
        # endpoint 可以去oss控制台上查看
        endpoint: "oss-cn-hongkong.aliyuncs.com"
        # 使用我们之前配置在secrets里面的accesskeys来配置ossutil
        access-key-id: ${{ secrets.ACCESS_KEY_ID }}
        access-key-secret: ${{ secrets.ACCESS_KEY_SECRET }}
    - name: Deply To OSS
      run: ossutil cp public oss://enok-blog/ -rf
```  

### 提交代码，查看Action运行情况

提交我们的代码，这个时候去项目的Action菜单下查看运行状态。等一会儿，你将能看到类似下图的情况，表示我们的工作流成功执行，并且将自动生成静态文件并上传到OSS上。
![image.png](https://upload-images.jianshu.io/upload_images/1771862-95c6c7218e6f41e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以去OSS上查看，确实上传成功了！
![image.png](https://upload-images.jianshu.io/upload_images/1771862-649ab1a89a337e48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

每个OSS Bucket都会分配一个二级域名。我们可以在地址栏上输入页面的路径进行访问啦
![image.png](https://upload-images.jianshu.io/upload_images/1771862-12b3570059567c5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不过此时我们如果省略掉index.html的话，并不能正确访问到页面。这是因为我们的OSS还没有配置静态网站托管选项。

##### 配置oss静态页面托管
在 oss 管理页面 找到 “基础设置” => “静态页面”
![image.png](https://upload-images.jianshu.io/upload_images/1771862-8d2d14ed6c3c1553.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###### 配置默认首页 
这样我们可以省略index.html访问了
###### 配置默认404页面。
 这样当我们输入任何不存在的路径是依然能显示首页。这在于一些SPA单页应用来说很重要
 
---

恭喜，你已经完成了在OSS上部署博客的工作！你可以享受OSS在国内外飞速的CDN服务。还能享受OSS提供的各种图片处理服务（压缩图片、缩略图等）

当然，或许你会担心OSS收费的问题。但我告诉你，如果只是一个博客，除非真的很多人访问，否则几乎每个月都不会产生什么费用，远远达不到最低收费标准。。。相当于是免费使用的

当然，如果你有自己的域名，不妨在阿里云控制台和OSS管理面板上自己探索一下，配置自定义域名访问。以及HTTPS等等。