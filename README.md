# Rails Docked 

安装 `Rails` 开发环境，对于新手来说，非常棘手：

- 在中国大陆，由于网络环境不够友好，导致安装 `Ruby` 和 `RubyGems` 非常困难。
- `Rails` 项目开发中，经常需要安装一些由 `C` 或 `Rust` 等语言开发的 `Gem` 包。这些包在 `Windows` 中编译安装非常困难。
- 对于一些老旧 `macOS`，无法使用 `Homebrew` 正确安装第三方依赖。例如 [Active Storage](https://guides.rubyonrails.org/active_storage_overview.html) 中所需要的图片分析工具 [vips](https://github.com/libvips/libvips)，在 `macOS Monterey` 上已无法正确安装了。

为了让大家无论使用什么操作系统的电脑，都能简单、顺利的开发 `Ruby On Rails` 应用，于是有了 `Rails Docked` 这个项目。其中，主要参考了 [Docked Rails CLI](https://github.com/rails/docked) 的相关配置。

打包好的镜像中，已包含了以下内容：

- Ruby 3.4.1 + 默认开启 YJIT
- Node 22.12.0 + Yarn + 中国镜像
- [Ruby China 镜像](https://gems.ruby-china.com/)
- Rails 8.0.1

## 安装 Docker

首先需要先安装 [Docker](https://www.docker.com/products/docker-desktop/)，如果安装中出现了问题，请参考 [Docker 安装教程](https://clwy.cn/chapters/fullstack-node-mysql)。

## 创建 Docker 卷

创建一个名为 `ruby-bundle-cache` 的卷，用于保存 `Ruby` 项目的依赖包。

```bash
docker volume create ruby-bundle-cache
```

## 创建项目

### macOS 系统

创建一个名为 `docked` 的别名：

```bash
alias docked='docker run --rm -it -v ${PWD}:/rails -u $(id -u):$(id -g) -v ruby-bundle-cache:/bundle -p 3000:3000 registry.cn-hangzhou.aliyuncs.com/clwy/rails-docked'
```

> 提示：`macOS`，还可以将此命令添加到`~/.bash_profile`, `~/.zshrc`, `~/.profile`, 或 `~/.bashrc`文件中，便于下次使用。

创建 `rails` 项目：

```bash
docked rails new weblog -d postgresql
```

> 提示：如果需要使用 `MySQL`，请将命令最后的 `-d postgresql` 改为 `-d mysql`。

### Windows 系统

创建 `rails` 项目：

```bash
docker run --rm -it -v ${PWD}:/rails -v ruby-bundle-cache:/bundle -p 3000:3000 registry.cn-hangzhou.aliyuncs.com/clwy/rails-docked rails new weblog -d postgresql
```

> 说明：由于我不清楚如何在 `Windows` 中配置 `alias`，所以这里的示例使用了完整的命令。如有知晓的朋友，请提交 PR。

## 使用 Docker Compose 配置容器

建好后，用编辑器打开项目。并在项目根目录下，增加 `docker-compose.yml` 文件，添加以下内容：

```yml
services:
  web:
    image: "registry.cn-hangzhou.aliyuncs.com/clwy/rails-docked"
    ports:
      - "3000:3000"
    depends_on:
      - postgresql
      - redis
    volumes:
      - .:/rails
      - ruby-bundle-cache:/bundle
    tty: true
    stdin_open: true
    command: ["tail", "-f", "/dev/null"]
  postgresql:
    image: postgres:17
    ports:
      - "5432:5432"
    environment:
      POSTGRES_HOST_AUTH_METHOD: trust
    volumes:
      - ./data/pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7.4
    ports:
      - "6379:6379"
    volumes:
      - ./data/redis:/data
volumes:
  ruby-bundle-cache:
    external: true
```

其中包含：

- PostgreSQL 17
- Redis 7.4

## 修改数据库连接

修改项目中的 `config/database.yml` 文件，增加如下数据库配置信息，这样才能连接到容器中的数据库：

```yml
default: &default
  # ...
  host: postgresql
  username: postgres
```

## 启动项目

- 启动容器

```bash
cd weblog
docker-compose up -d
```

- 进入容器

```bash
docker-compose exec web bash
```

- 安装 Ruby Gems

```bash
bundle install
```

- 创建数据库

```bash
rails db:create
```

- 使用脚手架，自动生成增删改查功能（可选）

```bash
# 创建路由、模型和迁移文件
rails generate scaffold post title:string body:text

# 迁移数据库
rails db:migrate
```

- 启动服务

```bash
rails s
```

等待服务顺利启动后，请访问 [http://localhost:3000/posts](http://localhost:3000/posts)