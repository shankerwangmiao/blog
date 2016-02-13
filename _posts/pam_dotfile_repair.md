layout: post
title: pam_dotfile 修理手记
date: Wed Feb 10 20:50:00 2016 +0000
tags:
- 技术
- PAM
---

## `pam_dotfile` 修理手记

### 源起

修理这个 `pam_dotfile` 的起因是[@dotkrnl](http://www.dotkrnl.com) 。他认为用 yubikey 作为登录的鉴定的充分凭据是不当的。因为 yubikey 是由所有者持有的（What you have.），存在失窃的风险，需要和所有者知道的（What you know.）搭配来使用，才科学。但是二者搭配起来作鉴定，即同时鉴定 yubikey 和系统登录密码又显得很麻烦，没有体现出 yubikey 的方便之处。于是一个这样的设想被提出来，才引发了下面一连串的血案。

<!--more-->

	bigeag1e 11:57:32    YubiKey 4 简介与配置 | K.I.S.S
	                     https://bigeagle.me/2016/02/yubikey-4/
	dotkrnl  12:02:10	   auth sufficient
	dotkrnl  12:02:41	   看起来就很不安全  
	shankerwangmiao  12:25:54	 怎么不安全了？
	dotkrnl  12:26:32 YubiKey 一插进去就可以登录用户，丢失权限就直接被获得了啊
	dotkrnl  12:26:41	 没有第二步密码
	shankerwangmiao	12:27:11	 密码还是少用为妙
	shankerwangmiao	12:28:00	 你yubikey还能和电脑一起丢了啊？
	bigeag1e 12:28:25 你可以改成 auth required 啊
	dotkrnl  12:28:44	 我觉得 required 比较靠谱呢
	bigeag1e 12:29:03 还是TOTP，U2F和TOTP有其一即可
	dotkrnl  12:29:09	 存在这种可能啊，所以我觉得比密码危险
	shankerwangmiao	12:29:19	 再说了，电脑丢了，有多少密码也挡不住啊
	dotkrnl  12:29:46	 全盘加密 🐶
	dotkrnl  12:29:48	 
	shankerwangmiao	12:30:01	 那要yubikey
	dotkrnl  12:30:19	 解锁啊
	dotkrnl  12:30:30	 你也不会关机的吧 🐶
	shankerwangmiao	12:31:35	 何用？
	shankerwangmiao	12:31:41	 额
	shankerwangmiao	12:31:58	 反正我解锁电脑是sufficient
	shankerwangmiao	12:32:35	 login是required
	dotkrnl  12:32:39	 我的期望是能 1、输完整密码 2、插 key 同时输一个弱密码。二选一。但是不知道怎么做。
	shankerwangmiao	12:33:12	 你需要配置一个pam
	shankerwangmiao	12:34:09	 这个pam仅仅验证输入的密码是不是123456
	shankerwangmiao	12:34:09	 然后在pam里配置三个规则
	shankerwangmiao	12:36:29	 第一条yubikey的，成功继续往下走，失败则跳过下面的一条
	shankerwangmiao	12:36:29	 第二条是弱口令的，是sufficient
	shankerwangmiao	12:36:29	 第三条是系统原来的
	dotkrnl  12:42:12	 这个怎么做
	shankerwangmiao	12:44:08	 就是在原来sufficient的位置处填一个表达式
	shankerwangmiao	12:44:20	 这个表达式在pam.conf的man里有讲解

当时提出这个配置 pam 的方法，是因为刚刚看过 linux pam 的 man 页面，才知道 linux pam 的 conf 还可以这样写：

	For the more complicated syntax valid control values have the following
	form:

	          [value1=action1 value2=action2 ...]

	Where valueN corresponds to the return code from the function invoked
	in the module for which the line is defined. It is selected from one of
	these: success, open_err, symbol_err, service_err, system_err, buf_err,
	perm_denied, auth_err, cred_insufficient, authinfo_unavail,
	user_unknown, maxtries, new_authtok_reqd, acct_expired, session_err,
	cred_unavail, cred_expired, cred_err, no_module_data, conv_err,
	authtok_err, authtok_recover_err, authtok_lock_busy,
	authtok_disable_aging, try_again, ignore, abort, authtok_expired,
	module_unknown, bad_item, conv_again, incomplete, and default.

	The last of these, default, implies 'all valueN's not mentioned
	explicitly. Note, the full list of PAM errors is available in
	/usr/include/security/_pam_types.h. The actionN can take one of the
	following forms:

	ignore
	    when used with a stack of modules, the module's return status will
	    not contribute to the return code the application obtains.

	bad
	    this action indicates that the return code should be thought of as
	    indicative of the module failing. If this module is the first in
	    the stack to fail, its status value will be used for that of the
	    whole stack.

	die
	    equivalent to bad with the side effect of terminating the module
	    stack and PAM immediately returning to the application.

	ok
	    this tells PAM that the administrator thinks this return code
	    should contribute directly to the return code of the full stack of
	    modules. In other words, if the former state of the stack would
	    lead to a return of PAM_SUCCESS, the module's return code will
	    override this value. Note, if the former state of the stack holds
	    some value that is indicative of a modules failure, this 'ok' value
	    will not be used to override that value.

	done
	    equivalent to ok with the side effect of terminating the module
	    stack and PAM immediately returning to the application.

	N (an unsigned integer)
	    equivalent to ok with the side effect of jumping over the next N
	    modules in the stack. Note that N equal to 0 is not allowed (and it
	    would be identical to ok in such case).

	reset
	    clear all memory of the state of the module stack and start again
	    with the next stacked module.

	Each of the four keywords: required; requisite; sufficient; and
	optional, have an equivalent expression in terms of the [...] syntax.
	They are as follows:
	required
	    [success=ok new_authtok_reqd=ok ignore=ignore default=bad]

	requisite
	    [success=ok new_authtok_reqd=ok ignore=ignore default=die]

	sufficient
	    [success=done new_authtok_reqd=done default=ignore]

	optional
	    [success=ok new_authtok_reqd=ok default=ignore]

我来概括一下上面的说明的意思：除了可以配置为 `required`、`sufficient` 之类的以外，还可以针对每个鉴定模块的不同鉴定结论作出不同的动作，包括忽略鉴定结论，鉴定失败，鉴定失败并退出，继续鉴定，鉴定成功并退出、重置鉴定结论和_跳过一些鉴定模块_（这里的概括稍微有些不准确，实际的动作的含义要更复杂一些，我之后会单独介绍）。

于是很自然的，为了实现“1、输完整密码 2、插 key 同时输一个弱密码”都可以完成鉴定的功能，我们可以采取这样的方法：

	      fail    +-------------+
	  /-----------| pam_yubikey |
	  |           +-------------+
	  |                   |
	  |                   | success
	  |                  \|/
	  |           +-------------+   success   +----------+
	  |           | pam_弱口令   | ----------->|  鉴定成功 |
	  |           +-------------+             +----------+
	  |                   |                      /|\
	  |                   | fail                  |
	  |                  \|/                      |
	  |           +-------------+    success      |
	  \-------->  | pam_强口令   | ----------------/
	              +-------------+
	                      |
	                      | fail
	                     \|/
	              +-------------+
	              | 鉴定失败     |
	              +-------------+

不难想到，这个鉴定流程很容易用 linux pam 的“跳过鉴定模块”的功能来实现。其中“`pam_强口令`”可以用系统原先的 pam 模块，唯一缺的就是 “`pam_弱口令`”。不过这个自己写也不会很难，照着别的 pam 模块的代码改一改应该就能成。

这就是我在上面的聊天里作出回复的思维过程，当时因为在外边走路，感觉这样可以实现，于是就这么回答了 [@dotkrnl](http://www.dotkrnl.com)。

### 发展

有一句名言是这么说的：

> Talk is cheap, show me the f**k code.

更何况我也认为这样使用 yubikey 比较 make sence，所以就着手实现上边的思路。首先我要确认的是，pam.conf 能不能像我想的那样写——毕竟，改配置文件比写代码容易多了。测试配置文件写法的时候可以先用 `pam_permit` 和 `pam_deny` 代替没有实现的那个 `pam_弱口令`。

说干就干，为了测试 pam 的配置，肯定不能拿我的 login 和 sudo 开刀，否则肯定是作死的节奏。测试 pam 的配置的程序原则上也可以自己写，不过这种轮子肯定有人造过。于是打开万能的<del>度娘</del> Google ，拿“test pam configuration”搜索一下，找到一篇 ubuntu 的 manual，讲的是 pamtest 这个工具，刚好是我需要的。

由于我和 [@dotkrnl](http://www.dotkrnl.com) 都是将 OS X 作为日常使用的系统，因此，要找到这个程序的源代码。这个程序是由一个叫做 `libpam-dotfile` 的包提供的。等一下，这个包的名字似乎说明，这个包主要的功能应该是一个 pam 的模块。于是直接前往 man 页面中声称的[项目页面](http://0pointer.de/lennart/projects/pam_dotfile/)。

简单的浏览了一下这个 pam 模块的功能。这个 pam 模块的中心思想是，在一个用户的 `home` 目录下创建若干 secret 文件，用于存储不同的 pam service 所使用的密码的散列值。在鉴定的时候根据鉴定的 pam service，读取相应的散列值，用于鉴定用户提供的作为鉴定凭据的密码。这正是我们需要的 `pam_弱口令` 。真是踏破铁鞋无觅处，得来全不费工夫，既然已经有人帮我们造好了轮子，我们为啥不用呢？于是赶紧找下载地址＋源码仓库。结果无意中瞟到了 “Requirements” 一节，里面赫然写着：

> `pam_dotfile` was developed and tested on Debian GNU/Linux "testing" from July **2003**, it should work on most other Linux distributions

当时心中飘过一阵草泥马。但是本着“宁可修老轮子也不造轮子”的思想，我还是把源代码下载了下来，心想，pam 的接口一直没怎么变过，说不定还能用。然而，这时我立了一个响当当的 flag。


### 高潮

我把源代码下载下来，开始执行 `./configure`。竟然没报错！看来有戏。继续 `make`，妥妥的报错了。

提示是缺头文件，尼玛。。。。缺头文件你 `configure` 的时候不说，那你 `configure` 在干些啥？

于是果断在 <del>gay</del>github 上 clone 一份这个代码，然后操刀开始改。这时我突然想起我不太会用 autotools 啊。本着“没吃猪肉但是见过猪跑”的精神，鉴于我编译过这么多遍 gcc ，autotools 的大致思想还是能搞懂的。然后我就同步地打开 `pam_yubico` 的代码，照着边学习边改。

整个项目分这么几个部分：

*	`pam_dotfile` 这个是一个 shared library，就是我们要用的 pam 模块。
*	`pam-dotfile-helper` 这个是一个 setuid 的 helper 程序，当 pam 模块的执行权限不足以读取事先保存的 secret 文件时，使用这个 helper 程序提升权限，将鉴定工作委托给这个程序。
*	`pam-dotfile-gen` 这个是一个用于生成 secret 文件的程序。
*	`pamtest` 这个用于测试 pam 的工作情况。

修改的细节我就不一一叙述了。实际上作的改动主要是适配不同的 pam 库的头文件的区别。OS X 使用的 pam 库是 `OpenPAM`，而 linux 上的 pam 库是 `Linux PAM`，因此头文件的名字和存放位置需要进行适配。只需要一一探测就好。改动的要点如下：

*	在所有的 .c 文件里包含 `config.h`，在所有的包含的头文件的两侧加上 `#ifdef HAVE_XXX_H` 和 `#endif` （我很诧异为啥原作者使用了 autotools，生成了 `config.h` 却没包含）

*	添加对 `security/pam_appl.h`, `security/pam_modules.h`, `security/_pam_macros.h`,  `security/pam_modutil.h` 的侦测
*	添加对 `security/pam_appl.h` 的包含

之后再编译，似乎就可以通过了。一阵输出之后，突然编译停了下来，提示这个错误：

	pamtest.c: 在函数‘main’中:
	pamtest.c:42:35: 错误：‘misc_conv’未声明(在此函数内第一次使用)
	     static struct pam_conv pc = { misc_conv, NULL };
	                                   ^

这个 `misc_conv` 是个啥东西？通过观察 `struct pam_conv` 的定义：

	struct pam_conv {
	        int     (*conv)(int, const struct pam_message **,
	            struct pam_response **, void *);
	        void    *appdata_ptr;
	};

可知，`misc_conv` 应该是个函数，错误应该是由于少包含了头文件导致的。于是立刻查找这个函数的头文件，Google 告诉我，这个函数在 `security/pam_misc.h`。立刻检测并包含之。且慢！我的系统上并没有这个头文件啊？这下坑爹了。于是用 misc_conv+ OSX 在 Google 上搜索，并没有啥结论。于是返回看 linux 下关于这个函数的 manual，其中提到：

> The `misc_conv` function is part of `libpam_misc` and not of the standard `libpam` library.

于是转而查找 `pam_misc` 这个 library 在 OSX 上的替用品，结果也没找到什么结论。于是继续看这个函数干什么用的。

>   This function will prompt the user with the appropriate comments and obtain the appropriate inputs as directed by authentication modules.

唔，原来是用来输入密码的呀。OS X 上显然应该有相应的函数啊。这种函数会在哪出现呢？于是我展开了联想。

几乎不出两秒钟，我就得到了答案：`su` 和 `sudo`，只要翻一下 OS X 上的这两个程序的代码就可以知道了。于是我立刻找到了 `su` 的[源代码](http://www.opensource.apple.com/source/shell_cmds/shell_cmds-162/su/su.c)（因为 `su` 比 `sudo` 简单），映入眼帘的就是：

	int
	main(int argc, char *argv[])
	{
		static char	*cleanenv;
		struct passwd	*pwd;
		struct pam_conv	conv = { openpam_ttyconv, NULL };
		enum tristate	iscsh;

哈哈，原来是`openpam_ttyconv`，搜索之，得到 `security/openpam.h`。

然后就是如何在两个函数中选择了，这里我参考了[这里](https://github.com/TinLe/macports/blob/082d31e48a3498fde4cfe0c479983128a479a791/net/quagga/files/quagga-patch2.diff) 的代码：

	  AC_CHECK_HEADER([pam/pam_misc.h],
	    [AC_DEFINE(HAVE_PAM_MISC_H,,Have pam_misc.h)
	     AC_DEFINE(PAM_CONV_FUNC,misc_conv,Have misc_conv)
	     pam_conv_func="misc_conv"
	    ],
	    [], QUAGGA_INCLUDES)
	   AC_CHECK_HEADER([security/openpam.h],
	     [AC_DEFINE(HAVE_OPENPAM_H,,Have openpam.h)
	      AC_DEFINE(PAM_CONV_FUNC,openpam_ttyconv,Have openpam_ttyconv)

大概的实现方式就是在 `configure` 的时候探测 `pam/pam_misc.h` 和 `security/openpam.h`，并定义宏 `PAM_CONV_FUNC` 为相应的函数名，然后在代码中直接使用：

	static struct pam_conv pc = { PAM_CONV_FUNC, NULL };

OK，最后修复了这个位置，然后编译就通过了。

### 尾声

编译通过了就要简单的测试一下代码还能否 work。于是先执行

	$ pam-dotfile-gen -a test

然后输入一个简单的密码 `12345`，于是就生成了 `~/.pam-test`，然后写一个 `/etc/pam.d/test`

	auth       required       /usr/local/lib/security/pam_dotfile.so try_first_pass

最后执行：

	$ pamtest test $USER

先输入一个错误的密码，再输入一个正确的密码，于是可以看出，这个模块在这个给定的输入下是可以工作的。鉴于这份代码已经是成型的代码，我并没有改动什么逻辑的部分，于是可以推定这份代码应该是没什么问题了。

于是本来我们是想先解决 `pam.conf` 的问题，结果不小心解决了 `pam_弱口令` 的问题。此时我们有了 `pamtest` 于是就可以顺利地测试我们期望的那个配置文件了：

	auth       [success=ok ignore=ignore default=1]       pam_yubico.so mode=challenge-response
	auth       sufficient     /usr/local/lib/security/pam_dotfile.so try_first_pass
	auth       required       pam_opendirectory.so try_first_pass  # 这个是 OS X 上原有的 pam 模块，用于标准的鉴定过程。

结果：

	$ pamtest test $USER
	Trying to authenticate <shanker> for service <test>.
	Failure starting pam: system error

在 syslog 中赫然写着：

	pamtest[72280]: in openpam_read_chain(): /etc/pam.d/test(2): invalid control flag '[default=1]'

唔，原来 OS X 上的 `OpenPAM` 不支持这种语法。

卒。。。。。（未完待续）

### 结论

于是我们得到了一个在 OS X 上能用的 `pam_dotfile`。下面简单介绍一下这个模块的工作过程和使用方法。

当针对某个用户的鉴定开始的时候， `pam_dotfile` 会在下面的地点依次寻找存放密码的散列值的文件：

1.	`~/.pam-<service>`
2.	`~/.pam/<service>`
3.	`~/.pam-other`
4.	`~/.pam/other`

其中，`<service>` 代表被请求鉴定的服务的名字，如 `sshd`、`sudo` 等。当其中某个文件的权限不是 `x00` 时，或者是符号链接时，或者父目录是组可写或他人可写的时候，这个文件会被忽略。

然后用户输入的作为鉴定凭据的密码将会被求散列值，与之对比，若成功，则鉴定成功；否则鉴定失败。

通过 `pam-dotfile-gen -a <service>` 来生成 `~/.pam-<service>`。你需要输入两遍密码。

当不加参数直接调用 `pam-dotfile-gen` 时，该程序从标准输入读入多组明文密码（每行一个），并相应的输出它们的散列值。

我在使用这个程序的时候，往往是将生成的 `~/.pam-<service>` 移动到 `~/.pam/<service>`，这样可以让 `home` 目录下的 dotfiles 整齐些。

### 后记

我改好的 `pam_dotfile` 放在了 [Github](https://github.com/shankerwangmiao/pam_dotfile) 上，欢迎测试。

另外，这个项目的地址是 [http://0pointer.de/lennart/projects/pam_dotfile/](http://0pointer.de/lennart/projects/pam_dotfile/)。从地址上来看，*Lennart Poettering* 这个人应该搞过一些别的项目，唔，应该就在 [http://0pointer.de/lennart/projects/](http://0pointer.de/lennart/projects/)。看来还是这个人还是很高产的一个程序作者。

什么？你说什么？他还写过 `systemd` ？你没开玩笑吧？

我擦，这个 pam 模块竟然是 `systemd` 的作者的作品。。。orz......
