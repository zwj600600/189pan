# GitHub Actions 使用教程 - 天翼云盘自动签到

本文档介绍如何使用GitHub Actions自动执行天翼云盘签到任务。

## 目录

1. [什么是GitHub Actions](#什么是github-actions)
2. [配置步骤](#配置步骤)
3. [工作流文件说明](#工作流文件说明)
   - [签到工作流](#签到工作流)
   - [GitHub Pages工作流](#github-pages工作流)
4. [添加账号信息](#添加账号信息)
   - [单账户配置](#单账户配置)
   - [多账户配置](#多账户配置)
5. [手动触发工作流](#手动触发工作流)
6. [查看执行结果](#查看执行结果)
7. [常见问题](#常见问题)

## 什么是GitHub Actions

GitHub Actions是GitHub提供的持续集成/持续部署(CI/CD)服务，可以自动化执行各种任务，如构建、测试和部署代码。在本项目中，我们使用GitHub Actions自动执行天翼云盘的签到操作。

## 配置步骤

### 1. Fork本仓库

首先，您需要Fork本仓库到您自己的GitHub账号下。点击页面右上角的"Fork"按钮即可。

### 2. 设置Secrets

在您Fork的仓库中，需要设置天翼云盘的账号和密码：

1. 进入您的仓库
2. 点击"Settings"（设置）
3. 在左侧菜单中选择"Secrets and variables" > "Actions"
4. 点击"New repository secret"添加以下两个secret：
   - 名称：`TYYP_USERNAME`，值：您的天翼云盘账号（手机号）
   - 名称：`TYYP_PSW`，值：您的天翼云盘密码

### 3. 启用Actions

如果您是第一次使用GitHub Actions，可能需要手动启用：

1. 进入您的仓库
2. 点击"Actions"选项卡
3. 点击"I understand my workflows, go ahead and enable them"

## 工作流文件说明

本项目包含两个工作流文件：

### 签到工作流

签到工作流配置文件位于`.github/workflows/main.yml`，主要负责执行天翼云盘签到操作：

```yaml
name: 云盘签到

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]
  schedule:
    - cron: '30 1,13 * * *'  # 每天北京时间9:30和21:30执行
  workflow_dispatch:  # 支持手动触发
  watch:
    types: started  # 当有人star仓库时触发

permissions: write-all

jobs:
  build:
    runs-on: ubuntu-latest
    env:
       TZ: Asia/Shanghai  # 设置时区为中国时区
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        
      - name: 打印IP地址
        run: echo "My IP address is $(curl -s ifconfig.me)"
        
      - name: Set up Python 3.8
        uses: actions/setup-python@v4
        with:
          python-version: 3.8
          cache: 'pip'  # 缓存pip依赖
          
      - name: 缓存Python依赖
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
            
      - name: 安装环境
        run: pip install -r requirements.txt
        
      - name: 签到
        run: |
          (echo "签到时间 $(date "+%F %T") [![签到状态](https://github.com/${{ github.repository }}/actions/workflows/main.yml/badge.svg?branch=${{ github.ref_name }})](https://github.com/${{ github.repository }}/actions/workflows/main.yml)" && python3 ./main.py) | tee >(sed 's/^/- /' > index.md)
        env:
          TYYP_USERNAME: ${{ secrets.TYYP_USERNAME }}
          TYYP_PSW: ${{ secrets.TYYP_PSW }}
          
      - name: Git Auto Commit
        uses: stefanzweifel/git-auto-commit-action@v4.16.0
        with:
          commit_message: "自动签到更新 [skip ci]"
          file_pattern: 'index.md log.md'
```

### GitHub Pages工作流

GitHub Pages工作流配置文件位于`.github/workflows/jekyll-gh-pages.yml`，主要负责将签到结果部署到GitHub Pages：

```yaml
name: GitHub Pages

on:
  push:
    branches: ["main"]
  schedule:
    - cron: '35 5,17 * * *'  # 每天北京时间13:35和01:35执行
  workflow_dispatch:  # 支持手动触发

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
       TZ: Asia/Shanghai  # 设置时区为中国时区
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        
      - name: Setup Pages
        uses: actions/configure-pages@v3
        
      - name: Cache Jekyll Build
        uses: actions/cache@v3
        with:
          path: _site
          key: ${{ runner.os }}-jekyll-${{ hashFiles('**/*.html', '**/*.md') }}
          restore-keys: |
            ${{ runner.os }}-jekyll-
            
      - name: Build with Jekyll
        uses: actions/jekyll-build-pages@v1
        with:
          source: ./
          destination: ./_site
          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v3
```

## 添加账号信息

本项目支持单账户和多账户配置，您可以根据需要选择合适的方式：

### 单账户配置

如果您只有一个天翼云盘账号，可以使用以下方式添加账号信息：

1. **GitHub Secrets（推荐）**：在仓库的Secrets中设置：
   - `TYYP_USERNAME`：您的天翼云盘账号（手机号）
   - `TYYP_PSW`：您的天翼云盘密码

2. **环境变量文件**：在项目根目录创建`.env`文件：
   ```
   TYYP_USERNAME=您的手机号
   TYYP_PSW=您的密码
   ```
   注意：如果使用此方法，请确保不要将`.env`文件提交到GitHub

### 多账户配置

如果您有多个天翼云盘账号，可以使用以下方式添加多账户信息：

1. **GitHub Secrets（推荐）**：在仓库的Secrets中设置：
   - `TYYP_USERNAME`：多个账号用`&`分隔，例如：`13800000001&13800000002`
   - `TYYP_PSW`：对应账号的密码，同样用`&`分隔，例如：`password1&password2`

   注意：用户名和密码的顺序必须一一对应

2. **环境变量文件**：在项目根目录创建`.env`文件：
   ```
   TYYP_USERNAME=13800000001&13800000002
   TYYP_PSW=password1&password2
   ```

多账户配置后，程序会自动依次处理每个账号的签到和抽奖操作，并在输出中分别显示每个账号的结果。

## 手动触发工作流

除了定时触发外，您还可以手动触发工作流：

1. 进入您的仓库
2. 点击"Actions"选项卡
3. 在左侧选择"云盘签到"或"GitHub Pages"工作流
4. 点击"Run workflow"按钮
5. 选择分支并点击"Run workflow"确认

## 查看执行结果

工作流执行后，您可以通过以下方式查看结果：

1. 在仓库的"Actions"选项卡中查看执行日志
2. 查看自动生成的`index.md`文件，其中包含签到记录
3. 通过GitHub Pages查看签到记录：`https://[您的用户名].github.io/189pan/`

## 常见问题

### 1. 工作流没有按时执行

GitHub Actions的定时任务可能会有延迟，特别是在GitHub服务器负载较高的时候。这是正常现象，通常会在计划时间后的几分钟内开始执行。

### 2. 签到失败

可能的原因：
- 账号或密码错误
- 天翼云盘服务器问题
- IP被限制（GitHub Actions的IP可能被天翼云盘识别为异常）

解决方法：
- 检查您设置的账号和密码是否正确
- 尝试手动触发工作流，查看详细错误信息
- 如果频繁失败，可以尝试减少签到频率

### 3. 如何修改签到时间

编辑`.github/workflows/main.yml`文件中的`cron`表达式：

```yaml
schedule:
  - cron: '30 1,13 * * *'  # 默认为每天北京时间9:30和21:30
```

cron表达式格式为`分 时 日 月 周`，注意GitHub Actions使用UTC时间，比北京时间晚8小时。

### 4. GitHub Pages不更新

如果您的GitHub Pages没有及时更新，可以尝试：

1. 手动触发"GitHub Pages"工作流
2. 检查GitHub Pages设置是否正确（在仓库设置的Pages选项中）
3. 确认`jekyll-gh-pages.yml`工作流是否成功执行

### 5. 多账户配置问题

如果您使用多账户配置遇到问题：

1. 确保用户名和密码的数量一致，且顺序对应
2. 确保分隔符使用`&`，不要使用其他字符
3. 检查账号和密码中是否包含特殊字符，如果有，可能需要进行转义 
