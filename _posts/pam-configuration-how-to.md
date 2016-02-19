layout: post
title: PAM 配置简介
date: Fri Feb 19 22:00:00 2016 +0800
tags:
- PAM
- 技术
---

## PAM 配置简介

写这篇文章主要是想总结一下近来折腾 PAM 配置的收获和感想。

### 背景

PAM 是用来进行鉴定授权的一套框架，其主要目的就是分离这三个东西：

*	有鉴定需求的应用程序
*	实施的鉴定方法
*	对鉴定方法的组合策略

那么每一个这样的应用程序就是一个 PAM Application，一种鉴定方法就是一个 PAM Moudule，而由用户配置的鉴定方法组合策略，则是 PAM Configuration

<!--more-->

### 基本流程

PAM 有四组相互独立的功能，分别是鉴定（Auth）、账户管理 （Account）、会话管理（Session）、密码修改（Password）

这四个功能的用途可以从名字上略知一二，不过我先按下不表。因为这四个功能是相互独立的，即 `pam.conf` 里是分开写的，Module 里这几个接口是分开提供的，Application 则是分开调用这些接口的。在讨论 pam 执行流程的时候，区分这些功能没有什么太大意义。

那么一个功能是怎么被使用起来的呢？这个就是要说到的基本流程了。 从应用程序的角度，PAM 工作的基本流程如下：

1.	Application 收集如下信息：要鉴定的用户名（username）；要鉴定的服务名（如 login, sshd 等）；
2. Application 开启一个 PAM 事务，初始化 PAM；
3. 调用某个 PAM 功能；
4. 根据返回值判断功能是否成功，如果失败，则可以根据返回值判断出错原因；
5. 关闭 PAM 事务；
6. 结束。

就是这么简单。有人可能会问：我的密码是怎么输进去的呢？收集密码的功能是 PAM Module 具体操作的，当 PAM Module 想要收集密码的时候，会通知 PAM，PAM 则会调用事先 Application 注册的回调函数来收集密码。这个回调函数有可能是打印提示符，从标准输入读入密码（比如 login su sudo 等），也可能是向远程客户端发出收集密码的指令（比如 ssh），具体怎么收集密码，是 Application 实现的。

当 Application 去调用某个 PAM 功能时，PAM 会去依照 `pam.conf` 里设定的规则，依次载入并调用配置文件里记载的若干 Module，于是每个 Module 的相应功能被调用，并进行一些操作，返回一个返回值。PAM 根据一定的规则，综合这些 Module 的返回值，最终得出一个“总的”返回值，返回给 Application。

而 `pam.conf` 则是由系统管理员配置的，它控制着模块调用的顺序和规则，以及“总的”返回值得出的方式。

### `pam.conf` overview

`pam.conf` 一般保存在 `/etc/pam.d/` 下，每个 Service 的配置文件都以其名字命名。例如 `/etc/pam.d/sshd`。习惯上，一个使用 PAM 的 Application 的 Service 名字一般取作其可执行文件的名字，但这并不绝对，因为一个 Application 究竟使用什么作为 Service 名字，是这个 Application 在初始化 PAM 事务的时候，作为参数传给传给 PAM 的。

配置文件举例如下：


```
# /etc/pam.d/su
# su: auth account session
auth       sufficient     pam_rootok.so 
auth       required       pam_opendirectory.so
account    required       pam_group.so no_warn group=admin,wheel ruser root_only fail_safe
account    required       pam_opendirectory.so no_check_shell
password   required       pam_opendirectory.so
session    required       pam_launchd.so
```

相信你已经看出来了，这个文件每一行就是一条配置，每条配置由这么几项组成：

*	功能名，这个我之前有说过，一共有四种，分别是 `auth`、`account`、`password`、`session`；现在我们只需要知道这四种功能相互独立，处理逻辑是一致的就好了，我们暂时不需要管着四种功能具体是干什么的。
*	控制标记。这个很关键，主要用于控制模块调用的顺序和返回值的生成。取值有 `optional`、`sufficient`、`required`、`requisite`、`binding`
*	模块。这个就是指定的 PAM Module，如果不指定路径，则会自动在默认路径中搜素。否则使用指定的绝对路径。
*	其他参数。用空格分开的其他参数，这些参数会被全权交给 PAM Module 处理。

此外还有一种语法是：

	function-class include other-service-name
	
作用是将其它的 service 直接包含进来。

### PAM Module 调用流程

下面则是比较关键的一个环节，就是我讲了这么半天的“模块调用的顺序和返回值的生成”。

模块的调用和生成返回值遵从以下过程：

1.	设定要执行的功能 （Function）
2.	取出该功能相应的 PAM 配置链（chain）
3.	设定返回值 `ret` 为 `PAM_SUCCESS`
4.	设定出错标记 `fail` 为 `false`
5.	设定成功次数 `success` 为 `0`
6.	从前向后依次遍历配置链，调用相应配置项的 PAM Module 的相应功能函数。
7.	PAM Module 做一些事情，给出返回值
8.	获得该模块的返回值`r`
9.	若 `r` 为 `PAM_IGNORE` 则表示该模块希望 PAM 忽略这一结果，于是转 6， 继续处理下一个配置项；若都处理完毕，则转 12
10. 按下表格处理

	|控制标记 | `PAM_SUCCESS` | 其它 |
	|--- | ------- | ------- |
	|`optional` | `success` ++ | 不处理 |
	|`required` | `success` ++ | 若 `fail` 为 `false`，则 `fail` 置为 `true`，且将 `ret` 置为 `r`；否则不处理 |
	|`requisite` | 同上 | 同 `requisite`；并立刻终止遍历，转 12 |
	|`sufficient` | `success` ++；若 `fail` 为 `false`，则终止遍历，转 12 | 不作处理 |
	|`binding` | 同 `sufficient` | 同 `required` |
	  
11.	转 6，继续处理下一条配置项
12. 若 `success` 为 `0` 但 `ret` 为 `PAM_SUCCESS`，则将 `ret` 置为 `PAM_SYSTEM_ERR`
13. 返回 `ret`

总结一下，PAM 配置执行过程中有这么几个要点：

*	按照 PAM 配置文件从前向后**依次**调用 PAM Module
*	每个 PAM Module 都有自己的**一个**返回值
*	`optional` 的模块会被调用，但结果会被忽略
*	一旦有一个 `required` 的 Module 不成功，则整条配置链注定失败，且返回值就是这个 Module 的返回值，但后边的模块都会被调用，它们的返回值会被忽略（这主要是为了避免被知道是哪一个 Module 失败了）
*	一旦有一个 `requisite` 的 Module 不成功，则整条配置链会被立刻终止执行，但返回值视情况而定。若之前已经有 `required` 的 Module 不成功，那么返回值取之前的那个返回值；反之则取这个 Module 的返回值
*	若一个 `sufficient` 的 Module 成功了，且之前没有 `required` 的 Module 失败，则整条配置链停止执行，鉴定成功；若之前有 `required` 的 Module 失败，则会忽略这个成功的结果，继续执行配置链
*	`bind` 的 Module 成功时，处理方法和 `sufficient` 一致；失败时，和 `required` 一致
*	若要鉴定成功，则必须至少一个 Module 成功。

因此，某些资料有一些“sufficient 是鉴定成功的充分条件”的说法是不准确的。

### 四种功能

终于回到这四种功能上了。首先这四种功能，原则上，PAM 模块可以作任何想做的事情。但实际上还是有一定约定和习惯的。

*	`auth` 用于鉴定用户身份。通常来说，就是用于收集秘密信息，鉴定声称是身份的 visitor 是否是该身份的持有者。鉴定方法有各种各样的，比如要求输入密码、对某种不可复制的物品进行鉴定等。
*	`account` 用于管理用户登录资格。具体而言，就是在成功鉴定了来者是他所声称的那个身份后，用于确认用户在相应的上下文中是否有资格访问系统。比如限定登录时间、限定登录位置（tty）、限定远程主机等。
*	`session` 则是会话管理。在打开和关闭用户会话时调用，具体用途有记录登录日志、设定登录环境、启动和停止计费等。
*	`password` 用于更改用户密码。它的特殊性在于，当应用程序尝试执行修改密码的功能时，整条配置链会被执行两次，第一次用于预判是否能够修改密码（比如判断是否有足够的写入权限、如果密码存在网络上，判断网络连接是否正常等），第二次用于修改密码。这两次执行的时候，PAM 会为 Module 传入不同的 flag，因此不会混淆。当判断修改的权限时，**`suffcient` 会被当作 `optional` 对待**

### 模块参数

一般的来讲，Module 的参数是由模块全权处理的，但是不同的 Module 接受的参数还是有一定共性的约定的。下面是一些常见的参数。注意，不一定所有的  Module 都接受这些参数，这些参数的意义也有可能因 Module 的不同而有所变化，请以 Module 的文档为准。

*	`debug` 输出调试信息
*	`use_first_pass` 意味着 Module 不提示用户输入密码，而是用上一个模块输入的密码；如果之前没有模块输入密码，则使用 Application 在调用 PAM 鉴定功能前设定的密码。如果 Application 没有设定密码，则 Module 会获知这一情况，进行相应的处理。
*	`try_first_pass` 跟 `use_first_pass` 类似，但是当上一个模块输入的密码或 Application 提供的密码不正确或不存在时，提示用户输入密码，重新鉴定。
*	`nullok` 允许空密码

### Linux-PAM 高级语法

Linux-PAM 在控制标记字段支持一种高级的语法，即跟据 Module 返回值指派处理动作：

	[val1=act1 val2=act2 ... default=act3]
	
其中 `val` 是返回值，合法的取值有：

*	`success`
*	`open_err`
*	`symbol_err`
*	`service_err`
*	`system_err`
*	`buf_err`
*	`perm_denied`
*	`auth_err`
*	`cred_insufficient`
*	`authinfo_unavail`
*	`user_unknown`
*	`maxtries`
*	`new_authtok_reqd`
*	`acct_expired`
*	`session_err`
*	`cred_unavail`
*	`cred_expired`
*	`cred_err`
*	`no_module_data`
*	`conv_err`
*	`authtok_err`
*	`authtok_recover_err`
*	`authtok_lock_busy`
*	`authtok_disable_aging`
*	`try_again`
*	`ignore`
*	`abort`
*	`authtok_expired`
*	`module_unknown`
*	`bad_item`
*	`conv_again`
*	`incomplete`
*	`default`

这些返回值（除了 `success`）具体的意义则是 Module 相关的，可以通过阅读文档或源代码得到。

`act` 则是采取的动作，合法的取值有：

*	`ignore`
*	`bad`
*	`die`
*	`ok`
*	`done`
*	N （一个无符号整数）
*	`reset`

而原有的 `optional`、`sufficient`、`requisite`、`required` 则等价于：

	required
		[success=ok new_authtok_reqd=ok ignore=ignore default=bad]
	
	requisite
		[success=ok new_authtok_reqd=ok ignore=ignore default=die]
	
	sufficient
		[success=done new_authtok_reqd=done default=ignore]
	
	optional
		[success=ok new_authtok_reqd=ok default=ignore]

而整个处理流程则变成：

1.	设定要执行的功能 （Function）
2.	取出该功能相应的 PAM 配置链（chain）
3.	设定返回值 `status` 为 `PAM_PERM_DENIED`
4.	设定印象 `impression` 为 `_PAM_UNDEF`
5.	从前向后依次遍历配置链，调用相应配置项的 PAM Module 的相应功能函数。
6.	PAM Module 做一些事情，给出返回值
7.	获得该模块的返回值`r`
8.	根据返回值 `r` 选取采取的动作 `action`
9. 按下表格处理
		
	|`action` | 处理方法 |
	| -------- | ------- |
	|`reset` | 恢复 `status` 为 `PAM_PERM_DENIED`；恢复 `impression` 为 `_PAM_UNDEF` |
	|`ok` |  当 `r` 为 `PAM_IGNORE` 时，不处理；否则，当 `impression` 为 `_PAM_UNDEF` 时，更新 `impression` 为 `_PAM_POSITIVE`，并将 `status` 更新为 `r`；当 `impression` 已经是 `_PAM_POSITIVE` 且 `status` 是 `PAM_SUCCESS` 时，将 `status` 更新为 `r`|
	|`done` | 同 `ok`，若 `impression` 为 `_PAM_POSITIVE`，则终止处理，转 11 |
	|`bad` | 若 impression 已经是 `_PAM_NEGATIVE`，则不作处理；否则将 impression 置为 `_PAM_NEGATIVE`，若 `r` 是 `PAM_IGNORE` 则将 `status` 置为 `PAM_PERM_DENIED`，否则将 `status` 置为 `r` |
	|`die` | 同 `bad`，但立刻终止处理，转 11 |
	|`ignore` | 不处理|
	|N （无符号整数）| 跳过 N 个配置项|
	  
10.	转 5，继续处理下一条配置项
11. 若 `status` 是 `PAM_SUCCESS` 但 `impression` 不是 `_PAM_POSITIVE`，则将 `status` 覆盖为 `PAM_PERM_DENIED`
12. 返回 `status`

摘录要点如下：

*	Linux PAM 引入了 `action` 来指示处理方式，这样代码就清晰了不少，同时增强了灵活性。
*	Linux PAM 引入 `impression` 来评估当前的局面。`impression` 仅被 `action` 控制，与具体的 Module 返回值没有关系。
*	除了 `reset` 以外，`impression` 只能由“未知”转向“正面”（`ok` 和 `done`），或是由任意状态转向“负面”（`bad` 和 `die`）。一旦进入了“负面”，则出了被 `reset` 以为，不会进入别的状态。
*	`status` 取决于 Module 的返回值，同时受采取的 `action` 和当前的 `impression` 影响。
	无论 `action` 是什么，某个 Module 的返回值只会有两种处理方式：用于覆盖当前的 `status` 或者被忽略。注意：Module 的返回值是**不会**被修改的。
		
	例如：
		
	```
	auth [success=bad ignore=ignore default=done] pam_xxx.so
	```
		
	这条配置的后果是，当 `pam_xxx.so` 失败的时候，虽然动作是 `ok`，但是 `status` 会被更新为一个非 `PAM_SUCCESS` 的值，最终导致鉴定失败。当 `pam_xxx.so` 成功的时候，虽然 `status` 被更新为了 `PAM_SUCCESS`，但是由于采取的动作是 `bad`，`impression` 会转为“负面”，最终在出口处（步骤 11）PAM 把 `status` 覆盖为 `PAM_PERM_DENIED`。
	
*	一旦 `impression` 转为“负面”（`bad` 或 `die`），则 `status` 会被更新，且被“锁定”，之后的 Module 的所有返回值都会被忽略。
*	若 `impression` 尚为“不确定”，则 `ok` 和 `done` 会接受这个返回值，更新 `status`；若 `impression` 为“正面”，但 `status` 却不是 `PAM_SUCCESS`，则 `status` 不会被覆盖为 `PAM_SUCCESS`。

*	`reset` 则会重置 `impression` 和 `status`
*	无论如何 `status` 不会被置为 `PAM_IGNORE`
*	`die` 无论如何都会终止执行，但 `done` 只有在“正面”的 `impression` 时才会终止执行。
*	最后，只有当 `status` 为 `PAM_SUCCESS` 且 `impression` 是“正面”的时候才会返回 `PAM_SUCCESS`，否则一律不能返回 `PAM_SUCCESS`。
*	最后＋1，只有 `status` 会被返回给 Application，用于其判断是否成功以及错误原因，`impression` 是 PAM 工作时的内部状态，不算数的。

### 配置举例

继续立 #flag，下篇文章会介绍 yubikey 的各种奇妙用法，会有一些有意思的 pam 配置。
