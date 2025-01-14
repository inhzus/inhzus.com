---
title: "非 root 包管理器的选择"
date: 2025-01-14T19:28:00+08:00
tags: [linux]
---

## 背景

在工作中遇到两种情况，需要非 root 包管理器：

1. 开发环境在 docker 中，往往只对 `/home` 等做持久化，而不会持久化 `/usr`；
2. 多人共用开发机，不提供 root 权限。

以下提供几种选项，注意：

1. 我所遇到的操作系统环境含 CentOS 7、Debian 10，其它环境未经验证；
2. 以下提供的具体安装方式，仅在当下具有时效性，仍推荐去相关工具的官方文档中查找。

## Homebrew

Homebrew 被人熟知的使用场景是在 macOS 上，但其实在 Linux 的完成度也相当之高。它内置了 glibc  package，brew 安装的大多数二进制，都会依赖于该 glibc，而不会受系统自带 glibc 版本过低的影响。

默认情况下，即 [brew.sh](https://brew.sh/) 中提供的方式，将安装至 `/home/linuxbrew/.linuxbrew`，如果你有 `/home` 权限，那这是最佳的安装方式。

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

这也是我走通的方式之一，我安装了数十个二进制都没有任何的问题。进一步地，我推荐 `ln -s /home/linuxbrew/.linuxbrew /opt/brew` 将该路径软连接至更短的路径名，方便后续的使用。

将该路径添加至 `PATH`，就已经可以使用 `brew install xxx` 安装你想要的包了。



如果没有 `/home` 路径的权限，可以[自定义安装位置](https://docs.brew.sh/Installation#untar-anywhere-unsupported)（`HOMEBREW_PREFIX`）：

```shell
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip-components 1 -C homebrew
```

在这种方式下，大多数包都需要从源码编译，且 Homebrew 官方不会支持这种情况下编译安装问题的修复。但如果你的操作系统环境相对较新，可以进行尝试，或许运气好，不会遇到任何的问题。

## Nix

[Nix](https://nixos.wiki/wiki/Nix_Installation_Guide#Installing_without_root_permissions) 也提供了无 root 权限的安装方式。

通过以下命令之一判断环境是否支持 user namespace：

```shell
$ unshare --user --pid echo YES
YES
$ zgrep CONFIG_USER_NS /proc/config.gz
CONFIG_USER_NS=y
$ grep CONFIG_USER_NS /boot/config-$(uname -r)
CONFIG_USER_NS=y
```

如果支持，推荐使用 [nix-user-chroot](https://github.com/nix-community/nix-user-chroot) 安装 Nix：

```shell
mkdir -m 0755 ~/.nix
nix-user-chroot ~/.nix bash -c 'curl -L https://nixos.org/nix/install | sh'
```

安装后，使用如下的指令，即可进入该 namespace 使用 nix，注意，如果要使用安装的包，也必须使用 `nix-user-chroot` 进入该 namespace 后才能使用。（略有些麻烦）

```shell
nix-user-chroot ~/.nix bash
nix-env -i openssl
```

## Mise

Homebrew 和 Nix 都是包管理器，扩展到开发环境管理工具，推荐使用 [Mise](https://mise.jdx.dev/)，使用如下指令安装 mise 至 `~/.local/bin/mise`。

```shell
curl https://mise.run | sh
```

Mise 可以安装常用的开发工具或运行时，如 node，python 等，且能在不同的作用域（如全局或某文件夹下）指定不同的版本。该工具保持了对 [asdf](https://asdf-vm.com/) 的兼容。

```shell
mise use --global python@latest
# 使用如上指令即可安装、全局使用 Python 最新版本
```

注意，如果系统的 glibc 运行时版本过低，这样直接安装的二进制可能无法运行，可以设置 [MISE_ALL_COMPILE](https://mise.jdx.dev/configuration/settings.html#all_compile)，即会从源码编译，解决该问题。

## Lang specific

更简单妥协的做法，使用各编程语言的包管理器，如 go，pip，rustup，npm 等，当然，这些包管理器都可以通过 Mise 安装。如：

- rustup: ripgrep, fd, choose-rust, bat, rargs
- pip: black, httpie, ranger
- go: kubectl, fzf, hugo
- npm: pm2

## Build from source

下下策，源码编译，安装至如 `/opt` 目录。

