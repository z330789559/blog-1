---
title: "Pretty Terminal"
date: 2022-01-07T23:44:05+08:00
---


## 概述

很多人看到我的终端觉得很酷，问我怎么配置的？今天这篇文章我就来教教大家怎么配置一个骚气的终端，先让大家看看配置好的终端效果图吧：

![Go Project](https://tva1.sinaimg.cn/large/008i3skNgy1gy5c5pc15nj317v0u0n2w.jpg)


![普通项目](https://files.mdnice.com/user/8699/0f364320-886a-495a-946c-7d2b457a774c.png)



![rust project icon支持有点问题，不能正常显示rust图标](https://files.mdnice.com/user/8699/08bc8bb0-e28b-4218-a383-a86cdd101b1e.png)

是不是很骚气？？？

## 颜色配置

- `iTerm2`：号称 `Mac` 下最好的终端工具。
- `Zsh`：一款强大的终端工具，自定义可玩性非常高，如果你是`Linux`用户也可以安装。


**1. 第一步安装`iTerm2`，如果你已经安装了`brew`软件包管理工具，可以执行下面命令：**

```bash
brew install iterm2
```

如果你没有安装`brew`工具，你可以执行下面命令进行安装：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
但是这里有问题的，那就是大家懂得`网络问题`，可能`dns`污染，不能正常访问到`GitHub`，如果你是`Mac`用户把你的`dns`设置为`8.8.8.8`，如下图：

![添加一个dns](https://tva1.sinaimg.cn/large/008i3skNgy1gy5cylbg7vj30w40u0jso.jpg)

配置这个`dns`之前，你得先有`🪜`梯子你懂得，在大陆编程，我觉得程序员第一件事就是要解决网络问题，我感觉`80%`问题都是网络引起的问题。。

**2. 下载和配置主题，首先去设置背景颜色，位置在 `iTerm2 -> Preferences -> Profiles -> Terminal。`**

![终端颜色配置为 xterm-256color](https://tva1.sinaimg.cn/large/008i3skNgy1gy5dqmzed3j31920u0tct.jpg)

把配色设置为`xterm-256color`。

**3. 虽然 `iTerm2` 有 `7 `种自带的配色，像我这种对颜值要求高的，肯定是满足不了我需求了，既然有像我这种有需求的，那肯定有人提供这种高颜值的主题，于是有大神在`GitHub`开源了一个仓库`https://github.com/mbadolato/iTerm2-Color-Schemes`，里面收录各种主题配色方案。**

通过终端回到你用户根目录，执行下面命令：

```bash
mkdir ~/.iterm2 && cd ~/.iterm2

git clone https://github.com/mbadolato/iTerm2-Color-Schemes
```

![主题目录](https://tva1.sinaimg.cn/large/008i3skNgy1gy5e4fvxz9j316e0u043f.jpg)


然后导入主题到你终端里面：

![选择你刚刚那个目录](https://tva1.sinaimg.cn/large/008i3skNgy1gy5e5vj92xj31750u0q67.jpg)


选择 `schemes` 文件夹内的所有配色方案：

![选择 schemes 文件夹内的所有配色方案](https://tva1.sinaimg.cn/large/008i3skNgy1gy5e6n39cpj318e0owdiv.jpg)

然后根据自己喜欢的选择自己的配色方案。


## 安装字体

光有好配色，还达不到高颜值的效果，还需要找一款字体，并且这个字体要支持`emoji`，而这些图标字体其实是非 `ASCII` 码字体，在 `iTerm2` 中可以进行配置，所以先要安装这个字体，如下：

这个字体项目地址：`https://github.com/ryanoasis/nerd-fonts`，你可以看着说明自己配。

```bash
brew tap homebrew/cask-fonts

brew install --cask font-hack-nerd-font
```
安装字体同样有网络问题，大家自行解决吧，如果这个你都解决不了，我觉得你不适合写程序。


安装好之后，你需要更改终端配置，在 `iTerm2 -> Preferences -> Profiles -> Text -> Font -> Change Font` 栏位中，`Text` 下面勾选 `Use a different font for non-ASCII text`，然后在 `Non-ASCII font` 点击 `Change font` 修改，如下图：

![配置字体](https://tva1.sinaimg.cn/large/008i3skNgy1gy5ezi0drrj31cn0u0wio.jpg)

选择好`hack-nerd-font`字体即可。


## 安装 `zsh` 和 `oh-my-zsh`

安装`zsh`很容易一条命令即可：

```bash
brew install zsh
```

然后需要把默认的`shell`更改成`zsh`：

```bash
sudo sh -c "echo $(which zsh) >> /etc/shells"

chsh -s $(which zsh)
```
中途可能需要密码，你需要输入你的`root`密码，剩下就是安装`oh-my-zsh`，它可以帮你管理配置`zsh`，执行下面命令：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

![我这里已经安装了，如果你是第一次执行可能不是这样的，自己看提示信息安装](https://tva1.sinaimg.cn/large/008i3skNgy1gy5fcm54gpj316e0u0jvo.jpg)

安装好之后会产生一个名为` .zshrc `的配置文件，在用户家目录下面，我们以后主要就是修改它了。


## 自定义你的终端

你可以打开 `~/.zshrc` 文件看看，有一行 `ZSH_THEME="robbyrussell" `


![我这里已经更改了配置](https://tva1.sinaimg.cn/large/008i3skNgy1gy5gx348aij316d0u0djr.jpg)

我使用的主题是`powerlevel9k`，这个项目地址是`https://github.com/Powerlevel9k/powerlevel9k`

你只需要把这个主题项目下载到本机，然后配置到你配置文件里面，命令如下：

```bash
git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
```
然后改配置文件里面的主题目录：

```bash
ZSH_THEME="powerlevel9k/powerlevel9k"
```
你们可以自定义，你只需要配置配置正确即可，使用 `source` 命令生效配置，刷新环境变量：

```bash
source ~/.zshrc
```

![重新打开终端，你的效果和我这个差不多](https://tva1.sinaimg.cn/large/008i3skNgy1gy5fpte0v9j316e0u078b.jpg)

这这里的背景颜色用的是`Dracula`，你需要在配置里面设置一下即可。主题配置可以自定义的，一些配置可以查看官方仓库里面的配置教程自定义主题，仓库地址如下：`https://github.com/Powerlevel9k/powerlevel9k`。

## zsh 插件推荐
上面配置完成了终端，只能说打造一个漂亮终端，还没有发挥`zsh`潜力，`zsh`有很多插件，大家可以自定义安装吧。

- `autojump` 这个插件主要帮助我们记住目录，一键直达。
- `zsh-syntax-highlighting` 输入正确命令可以高亮显示
- `zsh-autosuggestions` 这是一个神奇的终端自动提示插件，历史命令记录，类似于`redis-cli`一样。

- `colors` 是最主要的文件夹`icon`显示的插件，通过`gem install colorls`，即可！

可能需要其他插件，大家自己去查找吧，`good luck~`。


