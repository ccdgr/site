---
date: '2023-10-13T14:23:47+08:00'
draft: false
title: '开发环境配置指南'
tags: ["dev", "tooling"]
author: "Gao Yuanming"
description: "新机器快速搭建开发环境：Go、Homebrew、npm、Maven、Git 等常用工具的换源与配置"
toc: true
---

> 换电脑或重装系统后的第一件事：配环境。记录一下常用工具的配置，省得每次去翻文档。

---

## Go

### 换国内代理

```bash
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOPRIVATE=gitlab.mycompany.com
```

验证：

```bash
go env GOPROXY
# https://goproxy.cn,direct
```

### 常用命令

```bash
# 初始化模块
go mod init example.com/project

# 整理依赖
go mod tidy

# 下载依赖到本地缓存
go mod download

# 查看依赖树
go mod graph

# 编译
go build -o bin/app ./cmd/server

# 运行
go run ./cmd/server

# 测试（含竞态检测）
go test -race -v ./...

# 格式化
go fmt ./...

# 静态检查
go vet ./...

# 交叉编译（Linux amd64）
GOOS=linux GOARCH=amd64 go build -o app .

# 查看可执行文件大小
go build -ldflags="-s -w" -o app .
```

### .gitignore

```
# Binaries
*.exe
*.exe~
*.dll
*.so
*.dylib
bin/
dist/

# Test binary
*.test

# Go workspace
go.work
```

---

## Homebrew（macOS 换源）

### 安装 Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 换中科大源

```bash
# 替换 brew.git
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.ustc.edu.cn/brew.git"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.ustc.edu.cn/homebrew-core.git"

# 替换 Bottles 源
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc
```

或者用清华源：

```bash
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"
export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles
```

### 常用命令

```bash
brew update          # 更新自身
brew upgrade         # 升级所有包
brew install <pkg>   # 安装
brew uninstall <pkg> # 卸载
brew list            # 已安装列表
brew info <pkg>      # 包信息
brew search <text>   # 搜索
brew cleanup         # 清理旧版本
```

---

## npm / pnpm

### 换淘宝源

```bash
# npm
npm config set registry https://registry.npmmirror.com

# pnpm
pnpm config set registry https://registry.npmmirror.com
```

### 常用命令

```bash
# 全局安装
npm install -g pnpm

# 初始化项目
npm init -y
pnpm init

# 安装依赖
npm install
pnpm install

# 运行脚本
npm run dev
npm run build
npm run test

# 查看全局包
npm list -g --depth=0

# 清理缓存
npm cache clean --force
```

---

## pnpm Workspace（Monorepo）

```bash
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"

# 常用命令
pnpm add <pkg> --filter <workspace>
pnpm run build --filter <workspace>
pnpm -r run test
```

---

## Maven（Java）

### 换阿里源

编辑 `~/.m2/settings.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
  <mirrors>
    <mirror>
      <id>aliyun</id>
      <mirrorOf>central</mirrorOf>
      <name>Aliyun Maven</name>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
  </mirrors>
</settings>
```

### 常用命令

```bash
# 编译
mvn clean compile

# 打包（跳过测试）
mvn clean package -DskipTests

# 运行测试
mvn test

# 查看依赖树
mvn dependency:tree

# 强制更新快照
mvn clean install -U

# 指定 profile
mvn clean package -P production
```

### Gradle 换源

`~/.gradle/init.gradle`：

```groovy
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/google' }
        mavenCentral()
        google()
    }
}
```

---

## Vue / Vite

### 创建项目

```bash
# Vite + Vue
pnpm create vite my-app --template vue-ts

# Nuxt
npx nuxi@latest init my-app
```

### 常用命令

```bash
# Vite 开发
pnpm dev
pnpm build
pnpm preview

# Vitest
pnpm test
pnpm test -- --coverage

# ESLint
pnpm lint
```

### 推荐 VSCode 插件

- Vue - Official（Volar）
- ESLint
- Prettier
- Tailwind CSS IntelliSense

---

## Git

### 基本配置

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# 换行符自动转换（macOS）
git config --global core.autocrlf input

# 默认分支名
git config --global init.defaultBranch main

# 别名
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"
```

### SSH 密钥

```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
# 复制公钥到 GitHub/GitLab
```

### .gitignore 全局

```bash
git config --global core.excludesfile ~/.gitignore_global

cat > ~/.gitignore_global << 'EOF'
.DS_Store
Thumbs.db
*.swp
.idea/
.vscode/
EOF
```

---

## Docker

### 换源

`/etc/docker/daemon.json`（Linux）或 Docker Desktop Settings（macOS）：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ]
}
```

### 常用命令

```bash
# 清理
docker system prune -a

# 进入容器
docker exec -it <container> sh

# 查看日志
docker logs -f <container>

# Compose
docker compose up -d
docker compose down
```

---

## Python

### pip 换源

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### 虚拟环境

```bash
python -m venv .venv
source .venv/bin/activate  # macOS/Linux
```

### uv（快速包管理器）

```bash
# 安装 uv
brew install uv

# 创建项目
uv init my-project
uv add requests
uv run main.py
```

---

## Rust

### 安装与换源

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 换源 ~/.cargo/config.toml
```

```toml
[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "sparse+https://mirrors.ustc.edu.cn/crates.io-index/"
```

---

## 常用命令速查

| 工具 | 安装 | 换源 | 初始化 |
|---|---|---|---|
| Go | `brew install go` | `GOPROXY` | `go mod init` |
| Node | `brew install node` | `npm config set registry` | `npm init` |
| pnpm | `npm i -g pnpm` | `pnpm config set registry` | `pnpm init` |
| Java | `brew install openjdk` | `settings.xml` | `mvn archetype:generate` |
| Python | `brew install python` | `pip config set` | `python -m venv .venv` |
| Rust | `curl --proto ... sh.rustup.rs` | `config.toml` | `cargo init` |
| Docker | `brew install --cask docker` | `daemon.json` | — |

---

> 一个小技巧：把常用的 alias 和环境变量写到 `~/.zshrc` 里，换电脑时直接复制过去。
