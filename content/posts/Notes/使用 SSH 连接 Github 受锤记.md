---
url: "connection_closed_and_permission_denied"
title: 使用 SSH 连接 Github 受锤记
toc: true
authors:
  - viyi
tags:
- SSH
- Github
categories:
  - 札记
date: 2023-06-01
lastmod: 2023-06-02
---

在试图学习用 Git，那当然第一步是要先连上 Github 啦。


<!--more-->


## 孤岛简中

依旧使用 ssh (Secure Shell) 来连接到 GitHub，起初还是下意识想要[创建新的密钥](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)，毕竟[之前](https://viyi.cc/building_a_blog.html)也这么干过，还灵光一现顺手改了个名字（记住这个下意识和顺手，后面要考🚬）。把公钥给到 GitHub 之后，用 `ssh -T git@github.com` 进行连接测试，结果报错 `Connection closed` ，这才开始启动大脑。

查了好半天原因被溜回[官方文档](https://docs.github.com/zh/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)，于是默默把它在寻找这类问题解决方法的路径优先级调高至 top。

按照文档所言，先使用 `ssh -T -p 443 git@ssh.github.com` 进行测试。其中，选项 `-T` (test) 表测试，会在 ssh 成功连接后立即断开；`-p` (port) 指定端口号为 HTTPS 协议默认端口 44；`git` 为 GitHub 上 Git 服务器的用户名，通常没有实际操作权限，只是用于通信的身份认证；`ssh.github.com` 为所请求的服务器域名。结果报了新错 `Permission denied (publickey)`。看意思是能收到连接请求了，但没通过验证。

先回看一下第一个错误，结合文档提及的代理服务器（proxy servers），考虑大概率是因为习惯科学养鹅，惯用的那家机场可能做了什么限制。确认 443 端口可以连接之后，考虑将它作为 ssh 的连接端口。参照文档中的具体操作，在 ssh 客户端的配置文件（ `~/.ssh/config` ）中添加以下内容

```
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
```

其中，`Host` 用于指定该配置项作用的主机名（若后接 `*` 则表全局），`Hostname` 用于指定请求通信对象的目标地址，`Port` 指定了连接所使用的远程端口，`User` 用于指定登陆远程主机所用的用户名。

## 我很好奇

![冰菓](https://zyin-1309341307.cos.ap-nanjing.myqcloud.com/note/1694227110862.png))

官方文档提供的解决路径是把 ssh 的连接端口换成 HTTPS 的默认端口，网上[查到也有说](https://www.v2ex.com/t/807649)可以通过在 config 中添加 `ProxyCommand nc -v -x 127.0.0.1:7890 %h %p` 来解决。顺手查了些拓展资料，还挺有意思的。

`ProxyCommand` (代理命令) 是 ssh 配置中的一个参数，用于实现通过 Jump Host（跳板机/堡垒机）来连接到远程主机。大概意思就是阿珍（本地主机）想和阿强（目标主机）在赛博空间 say hi，但阿珍出于害羞（安全）不想让阿强知道自己是谁，于是让阿狗（跳板机）传达一下。在这个过程中，阿狗需要知道 ta 需要知道向哪个阿强（目标主机的 IP/域名）转达阿珍的 hi （流量）以及阿强的档期（目标主机的端口号）。具体在命令行中的呈现，是两个占位符—— `%h` (host) 和 `%p` (port)，在实际中会被替换为对应的内容。

这个参数除了 ~~围观阿珍和阿强的爱情故事~~ 通过跳板机转发 ssh 流量给到目标主机之外，还可以指定执行 ssh 连接之前要执行的命令。于是接着就出现了 `nc` (netcat) 命令[^1]，这个命令用于设置路由器。麻瓜式理解路由器的话，把它当作是在网络中传输数据的阿猫就行，和之前的阿狗差不多意思。选项 `-v` (verbose) 是指显示指令执行过程，也就是说会开启调试模式输出详细信息。`-x` 是用于指定代理主机地址和通信端口号，开启这个选项后会默认使用 SOCKES5(Socket Secure 5) 协议。也就是说，后面的 `127.0.0.1:7890` 是指代理服务器的 IP 地址，端口号是 `7890`，使用的通信协议是 SOCKES5。

后两个也挺有意思， `127.0.0.1` 是本地回环接口地址(localhost)，这是很特殊的一个 IPv4 地址，总是会指向本机上的网络接口。说人话，感觉这基本和我给我电脑取名叫小雷一样，让计算机帮我和 `127.0.0.1` 说谢谢，和我在致谢里写"感谢小雷"的区别是计算机只能听懂那串数字。换句话，`ProxyCommand nc -v -x 127.0.0.1:7890 %h %p` 的意思大概是假装自己是阿狗（运行在本地的代理软件），把原本在 ssh 默认 22 端口的流量通过 7890 端口用 SOCKS 协议转发给阿强（目标主机地址 `%h`）的端口（`%p`）。至于为什么是 7890，结合科学养鹅盲猜是因为常用工具 Clash 的默认端口号是这个，通常的配置中也会用 SOCKS5 协议。再简单来说，就是手动为 ssh 科学养鹅设个代理。

进一步逆推它的奏效原因，结合[此处](https://www.cnblogs.com/tsalita/p/16185135.html#gallery-2)提及 window 下的 git 默认没有用系统代理，科学养鹅之后会出现在浏览器里可以正常顺利访问，但在命令行中就没办法直接用这个命令连接的情况 ~~（我恨孤岛简中世界）~~。罪魁祸首大概率是我用的 WSL(Windows Subsystem for Linux)，我他喵可没注意到 WSL 和 Windows 本身的网络配置是分开的啊！WSL 默认读取环境变量 `http_proxy` 和 `https_proxy` 里的代理配置，这里头我从没动过当然不会有我在 Windows 里的代理配置啊QAQ。

## 背景知识

把时间线拉回最初用 `ssh -T git@github.com` 测试连接后报错查解法的时候，看到有新手说 ta 最后发现是因为自己“自作聪明”地把测试命令中的 `git@github.com` 换成了生成密钥时填在注释中的个人邮箱，于是卡在这一步折腾很久。

个人猜想是因为通常教程中会习惯性地把个人邮箱放在注释中，会提示说这里要改成自己的邮箱（`-C "your_email@example.com"`）。但很少有人会进一步解释这只是个注释，是为了方便管理和识别，和生成密钥的过程本身其实没啥关系。于是操作者很可能会以为这是个必要操作，且保险起见通常也会逐步照搬。

接着下一步中会出现 ssh 测试命令 `ssh -T git@github.com` 也同样出现了 @ 符号，于是产生误导，以为这里也需要替换成自己邮箱（尽管文档没说，但下意识脑补撰写者忘记提醒orz）。同样，很少有文档会进一步解释这条命令中 @ 前后内容的具体含义，这是指在进行 ssh 连接时，登陆远程主机所使用的用户名和远程主机的地址，有别于在使用邮箱服务时的用户名和电子邮箱网站地址。

仔细想想，这事挺有意思的。首先官方文档大多时候是提示操作过程以期达成用户目的，它通常不会充当教育角色给足背景信息讲解说明，因为这样未免过于冗长，最多在[故障排误](https://docs.github.com/zh/authentication/troubleshooting-ssh/error-permission-denied-publickey#%E5%A7%8B%E7%BB%88%E4%BD%BF%E7%94%A8-git-%E7%94%A8%E6%88%B7)说明一下。而其受众大多也是有明确问题期望得到解决的用户，大多数更希望提供明确路径来完成，通常没有耐心且认为了解原理无甚必要。其次，新手所犯的绝大多数错误是常识性错误，操作说明中所省略掉的解释，绝大多数是撰写者下意识默认为常识的背景知识。

矛盾之处就在于此，一方面文档必然无法涵盖所有情况，但又无法给予操作者必要的最小背景知识，以期其能够自行做出初步推断。另一方面困住新手的常识性错误，低级到甚至不会出现在故障排误中又切实造成困扰，只能寄希望于搜索引擎的呈现内容，而往往又因无法准确归因导致检索效率低下。

我一直觉得不写什么和写什么同样重要，但一方面受限于知识诅咒，[^2] 很难完全把自己放置在读者角色上来正确预设共识。另一方面又担忧过多说明背景信息会使得文章冗长，不太符合我的美学。通常情况下我会放弃挣扎，生怕自己哪儿没写清楚，任由文章变得冗长。

就算我偏好将所涉及到的专有名词和背景信息补足呈现，我也无法处理那些难以在一篇文章中说清的背景知识。比如我预设这篇文章的读者应当是和我一样的初学者，但其实我并没有办法在这篇文章中从宏观上说明在客户端和服务器之间经由网络进行交互的大致过程（因为过于复杂甚至能再写篇文章），而我的确认为这对于理解上述内容是很有帮助的。我总不能按头拉着（至今也许不存在的）读者和我一样去自习郑烇老师的[计算机网络课程](https://www.bilibili.com/video/BV1JV411t7ow/)，所以目前做法是尽可能补全可拓展信息（我爱超链接）。

最近在适应使用命令行，时不时就会发现自己被那些默认忽略的背景共识背刺了。其中之一是非英母语者对常用缩写命令的低敏感度，没办法立刻颅内补足完整命令，以至于理解记忆起来都有些磕绊（ [zsh-autocomplete](https://github.com/marlonrichert/zsh-autocomplete) 赛高！[^3]）。我当初还疑惑了半天为什么 Linux 把内容输出是 `cat` ，不能用 `dog` 是因为我们汪汪不可爱吗（bushi。查了下其实是取了"concatenate"(连接)的缩写，而不是通常理解中的喵喵。至于为什么是连接，后来在理解数据流（Data Stream）查管道（Pipeline）资料的时候才算是明白它的含义。

再就又是“仅提供解法略过说明”情况的某种例证，诸多大佬大概是惯用命令行中的短选项（short option, `-单个缩写字母`），虽然在实际使用上确实是真的会方便很多啦。但对非英母语的初学者来说无形就增加了学习难度啊喂！比如上文提及 `nc` 命令的 `-x` 选项是用于指定代理地址（proxy_address）和端口号（port）的，通常来说命令行短选项的缩写是首字母，很自然会觉得说怎么不是 `-p` 对吧？！ `-x` 就出现得非常让人疑惑。

查了一圈发现很可能是某种 QWERTY 现象[^4]的遗留产物——代理服务器作为中继很可能在没啥事的时候下线，这种情况被称为 "exit"(退出)，所以通常计网中出现 "exit proxy" 或者是 "external host" 都和代理服务器有关。而又因为英语中的元音容易和其他选项混淆，选项名称中的 e 会被省略（其实常用写法是 "eXit"），于是传统上代理服务器选项的缩写通常是 `-x` 。而在 `nc` 命令中，`-p` 也确实是用于设置本地主机使用的通信端口号，通常用于监听，而不是指定代理服务器信息。

我，单方面理直气壮地建议所有新手教程中涉及命令行的都放上长选项（long option, `--完整单词`）的解释！如果你不愿意，我只好嘤嘤嘤求你照顾下麻瓜（bushi。

## 拒绝许可

至于我开篇第二个 `Permission denied (publickey)` [公钥权限被拒绝的错误](https://docs.github.com/zh/authentication/troubleshooting-ssh/error-permission-denied-publickey)，那就又是另一个例证了。文档里的常规做法和软件中的默认设置，在很大程度上可以规避掉很多因为麻瓜搞出奇怪初始配置出现的诡异问题，而我不幸的就是那个完美避开防御性设计的踩雷麻瓜。

> 人类的理智往往不断地重复自身，失去理智的情况亦然。——《失明症漫记》，若泽·萨拉马戈

人在大多数时候依赖经验记忆行事，而非理性逻辑。启动大脑后想穿越回去拜托自己想想，在同一台主机上和多台远程服务器连接有必要搞新的密钥吗？！私钥又没泄露！这有道理吗！它被发明出来不就是为了只要把公钥撒出去就能建立安全连接这点很方便吗！这就是为什么[文档里要人检查是否存在已有密钥](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/checking-for-existing-ssh-keys)的原因，但在写这篇文章之前我并没有优先看文档的习惯，我通常直接上手硬刚~~（然后就受锤了吧，让你不看说明书，幸灾乐祸.jpg）~~。

虽然在我已经有个密钥的情况下，新建密钥其实毫无必要，但是理论上我把公钥给到 GitHub 之后，就可以通过它连接了对不对，它不应该再报错。起初我以为是私钥权限的问题，但确认后发现不是。还记得我创建的时候灵光一现顺手改了个名吗？也就是说我的公钥和私钥都不是默认的常规命名（id_算法），这才是问题所在。

我又是怎么发现的呢？终于学会优先翻文档本人虽然没完全在故障排除里被治愈，但[结合提示](https://docs.github.com/zh/authentication/troubleshooting-ssh/error-permission-denied-publickey#make-sure-you-have-a-key-that-is-being-used)里的「在大多数系统中，默认私钥（ `~/.ssh/id_rsa` 和 `~/.ssh/identity`）会自动添加到 SSH 身份验证代理中。 应无需运行 `ssh-add path/to/key`，**除非在生成密钥时覆盖文件名。**」，以及在[说明](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/checking-for-existing-ssh-keys)步骤中的「 **默认情况下**，GitHub 支持的公钥的文件名是以下之一：id_rsa.pub、id_ecdsa.pub、id_ed25519.pub」，不难推断出是顺手一改名的锅TAT。

让你灵光一闪又不顺手改配置！这顺得还不够啊喂！解决方法也很简单，用 `ssh-add path/to/文件名` 把专用密钥添加到 ssh-agent 的高速缓存中，再告诉 ssh 在连接到 GitHub 中所使用的指定公钥文件是什么应该就行。具体是在 `~/.ssh/config` 配置文件里加上

```
Host github.com
    IdentityFile ~/.ssh/公钥文件名
```

其中，「公钥文件名」替换为 ".pub"(public) 后缀的那个文件名字。至于为什么是“应该”，因为我其实用的之前那个默认名称的密钥 :D，顺手查了原因但懒得再复现一次验证（我忏悔）。

## 只是个 c

早些年我也许还是很傲慢的，对一些特别简单问题甚至想用「[让我帮你百度一下](https://btfy.ur1.fun/)」回复嘲讽（虽然我从没这么干……但我真的考虑过要不要这么干，再度忏悔）。但总是每天都有人在学习那些我已然熟稔于心的东西，总是有人才刚刚开始学习怎么在电脑上双击打开一款软件，就像我从头学习如何在命令行中使用工具一样。~~（果然受锤之后才变得能谦卑）~~

另一方面，在基于关键字的传统搜索引擎中，小白和大佬的检索效率完全不在一个量级，这也是那些下意识被当作共识忽略的背景知识所衍生出来的问题。单单是某些领域内的专有名词，都能拉开一大截检索质量。而两者对信源的优先级认知，无形中又拉了一截。很多时候并不是小白没有检索，而是以现在的认知水平没办法有效检索。

这时候 chatGPT 真的就很有帮助，最近也在高频向 ta 提问，就算起初只能提出宽泛的问题，也能够从 ta 的回应里挖出有效信息进一步查相关资料，应对刚开始意识到可能“不知道自己不知道”的那一类情况很有效。不过它基于大力出奇迹的概率原理做出的回答，其实无法百分百保证真实和正确性，起初还小小的纠结了一下这个缺点。但仔细想了想，我并不止有这一个学习渠道，就算它说的是错的，也会随着逐渐深入学习被交叉印证出来，问题也不是特别大。当然有时候当场就会发现它前后逻辑矛盾……就只好怒斥“你们 AI 能不能努努力！”（bushi。

看得出来我的 AI 焦虑症状减轻了很多，昨天我有个浴室沉思是说：所谓「精通」也许是从“使用”到“设计”甚至再到“创造”的过程。之前网上冲浪看到有人说，「能做出 copilot 的人自己得是 pilot」，十分同意，甚至想补一句“能够用好它的人自己也得是 copilot”。至于我？我现在最多算个 c 吧（。

[^1]: 吐槽一下，一开始只查到了 [Netcat](https://netcat.sourceforge.net/) 的主页和 ncat 的[文档](https://man7.org/linux/man-pages/man1/ncat.1.html)，最后在奇怪的地方查到了它的[解释文档](https://docs.oracle.com/cd/E56344_01/html/E54075/netcat-1.html)。还疑惑了一下 ncat 是什么，然后才发现它可能是 Netcat 里带的另一个工具……另外，发现 Michael Kerrisk 维护的 [Linux 文档](https://man7.org/linux/man-pages/index.html) 中搜索是用 Google 的 site: 来实现的，好讨巧，转念又觉得挺合理的。
[^2]: 一种认知偏差，是指掌握内容后很难想象不知道的情况，容易下意识地假设对方拥有理解我们熟知内容的背景知识。这使得我们在向他人传授时忽略必要信息，难以从听者角度思考。
[^3]: 一个可以「实时预输入自动补全」（real-time type-ahead autocompletion）的 zsh 插件，完美拯救记不住选项名称的金鱼脑。
[^4]: 这并不是什么专有名词，只是我通常用它来指代那些出于经济学家所说的转换成本（Switching Barrier） 而倾向遵循旧习，不愿意尝试更换其他更优解决方案的现象。QWERTY 的键盘布局放现在来说不算是漂亮的方案，但不妨碍它如今的统治地位。