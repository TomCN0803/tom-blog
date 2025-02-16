---
title: "「Go系列」漏洞管理体系以及 govulncheck 漏洞检测工具介绍"
date: 2023-07-08T10:30:06+08:00
draft: false
---

## 体系概览

Golang官方提供了一套比较完整的漏洞管理体系，可以帮助开发人员检测、评估和解决可能被攻击者利用的错误和漏洞。该体系的底层架构主要分为如图所示三部分：**数据源**、**Go漏洞数据库**、**工具集成**。

![Go漏洞管理体系](https://files.mdnice.com/user/17908/2d8c8a86-4781-4a0a-b75a-8ded3fb615a5.png)

1. Go漏洞**数据源**主要包括[National Vulnerability Database (NVD)](https://nvd.nist.gov)、[Github Advisory Database](https://github.com/advisories)、[Go维护者提交的Issue](https://go.dev/s/vulndb-report-new)等，通过一个data pipeline对这些数据进行采集、格式化处理；
2. [Go漏洞数据库](https://pkg.go.dev/vuln)用于存储从data pipeline传输来并格式化后的漏洞数据。Go安全团队会对数据库中所存储的全部漏洞数据进行研判和修正。存储的漏洞数据均为[Open Source Vulnerability (OSV)](https://ossf.github.io/osv-schema)格式，同时对外提供查询服务的[API](https://go.dev/security/vuln/database#api)；
3. 为了方便Go开发者对自己的项目进行漏洞检测，Go漏洞管理体系同样提供了[pkg.go.dev](https://pkg.go.dev/)以及[govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)的**工具集成**。其中govulncheck是一个Go编写的二进制工具，开发者运行该工具就可以检查Go项目中的漏洞。

## govulncheck工具

govulncheck是前文提到的Go官方编写的Go项目漏洞检测二进制工具，开发者运行该工具就可以检查Go项目中的漏洞。govulncheck通过对Go项目源码或者二进制文件中的符号表进行静态分析，以得到对项目安全性造成影响的安全漏洞。

### 基本原理

在默认情况下，运行govulncheck会向Go官方的[漏洞数据库](https://pkg.go.dev/vuln)发起HTTP请求，并返回查询结果。govulncheck所发起的请求中只会包含项目中所用到的go module路径，而不会包括源码、项目配置等其它数据信息。

govulncheck根据不同的分析目标会有不同的分析策略，例如在分析Go源码文件（*.go）时，会使用当前本地环境变量中默认的Go配置信息；而分析Go二进制文件时（仅支持go1.18及以上），会使用编译出该二进制文件的Go配置信息。不同的Go配置信息可能会得到不同的漏洞检测结果。

### 基本使用

安装最新版本govulncheck：

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
```

基本使用（在项目根目录下）：

```bash
govulncheck ./...
```

官方完整文档链接见下文

### 接入外部漏洞数据库

执行govulcheck时可以通过`-db`标识指定一个外部Go漏洞数据库，这个外部数据库必须实现Go漏洞数据库[API](https://go.dev/security/vuln/database#api)。**企业单位其实可以利用这一特性，将自己内部的漏洞库适配govulncheck**。

## 集成至编辑器

目前官方仅支持VSCode和Vim/NeoVim的Go漏洞扫描集成，且需要开启v0.11.0版本以上的[gopls](https://pkg.go.dev/golang.org/x/tools/gopls)（Go官方语言服务器LSP）。目前共有两种漏洞检测模式：

- Imports-based：此模式**不需要govulncheck的支持**，直接对go.mod（如果启用go workspace则会应用go.work中的依赖规则）中的依赖进行扫描，速度较快但是容易有“假阳性”的结果；
- Govulncheck-based：基于govulncheck工具，速度较慢需，要手动触发，但准确度、集成度更高；

其中Vim/NeoVim仅支持Imports-based，而VSCode两者全部支持。

#### VSCode

VSCode的[官方Go插件](https://marketplace.visualstudio.com/items?itemName=golang.go)已经自动集成了gopls，只需在VSCode配置文件中添加如下配置项：

```json
"go.diagnostic.vulncheck": "Imports", // enable the imports-based analysis by default.
"gopls": {
  "ui.codelenses": {
    "run_govulncheck": true // "Run govulncheck" code lens on go.mod file.
  }
}
```

一些效果图：
![](https://files.mdnice.com/user/17908/246c375b-cbea-4b6e-92e9-259c3505cf26.png)

![](https://files.mdnice.com/user/17908/3b93a6a5-2b25-4d75-8a82-619c567f8391.png)

![](https://files.mdnice.com/user/17908/bc5a96f8-83f3-4106-98de-b0598e0e1941.png)

#### Vim/NeoVim

使用[coc.nvim](https://www.vim.org/scripts/script.php?script_id=5779)，添加如下配置：

```json
{
    "codeLens.enable": true,
    "languageserver": {
        "go": {
            "command": "gopls",
            "initializationOptions": {
                "vulncheck": "Imports",
            }
        }
    }
}
```

## 相关资源

- Go漏洞管理官方介绍：<https://go.dev/security/vuln>
- Go漏洞数据库索引：<https://vuln.go.dev>
- 网页版Go漏洞数据库（集成至[pkg.go.dev](https://pkg.go.dev/)）：<https://pkg.go.dev/vuln>
- Go漏洞数据库数据格式、API说明文档：<https://go.dev/security/vuln/database>
- 漏洞检测工具govulncheck文档：<https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck>
- govulncheck集成VSCode：<https://go.dev/security/vuln/editor>
