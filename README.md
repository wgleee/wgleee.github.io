# 操作说明

本博客使用 Hexo 博客框架配合 nexT 主题搭建，托管于 GITHUB 仓库，基中 master 分支为博客静态页站点；dev 分支为 Hexo 框架生成原始代码。 使用 dev 分支对都博客进行修改，master 用于展示

## 环境搭建

### 1. 安装 [nodejs](https://nodejs.org)
### 2. 安装 [hexo](https://hexo.io/zh-cn/)

```
npm install hexo-cli -g
```

### 3. 使用 git 下载本博客 dev 分支, 并安装相应模块

```
git clone https://github.com/wgleee/wgleee.github.io.git -b dev
cd wgleee.github.io
npm install
```

### 4. 发布新文章 (提交更新至 master 分支)

```
hexo clean
hexo deploy
```

> Tips: 使用 Hexo 提供的部署工具进行部署操作，部署前先清理下缓存及临时文件

### 5. 对原始代码更新进行提交 (提交更新至 dev 分支)

```
git push origin dev
```

> Notes: 原始代码提交至 dev 分支


## Hexo 简单的操作命令

```
# 创建新文章
hexo new post 'filename'

# 创建页面
hexo new page 'pagename'

# 启动服务器。默认情况下，访问网址为： http://localhost:4000/
hexo server
```

> 关于 Hexo 命令详细文档请查看 https://hexo.io/zh-cn/docs/commands.html
