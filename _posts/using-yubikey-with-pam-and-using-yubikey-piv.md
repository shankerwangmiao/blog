---
title: YubiKey 的花样用法：填时隔半年的坑
date: 2016-08-07 15:39:28
tags: 
- PAM
- 技术
- 折腾
---

突然想起来，我之前说过的要介绍一下 YubiKey 的花样用法的文章。半年来，一直被拖延症给拖延了，就没写成。有一天 [@Blink](https://blog.blink.moe) 提醒我，这还有个大坑要填呢。想起来，半年以来，确实给好多人，比如 [@Youngcow](https://youngcow.net) 老师安利过 Yubikey，还口头介绍过很多次这玩意的用法，看来还是有必要简单地总结一下这个东西的原理和一些花样用法。

# YubiKey4 简介

YubiKey4 是一个比较开放的硬件密码学设备。它的功能比较丰富，下面是它的所有功能的列表（来源：[YubiKey 4 and YubiKey 4 Nano Product Sheet](https://www.yubico.com/wp-content/uploads/2016/02/Yubico_YubiKey4YubiKey4Nano_ProductSheet.pdf)）

* Config Set 1 & 2

	下面四个功能中可以选两个，配置进两个独立的 slot
	
	* Yubico OTP
	* OATH-HOTP
	* Challenge-Response
	* Static Pssword

* OpenPGP Smart Card
* PIV Smart Card
* U2F

上面这四大类功能相互独立，互不影响。

# Yubico OTP

Yubico OTP 是 Yubico 官方推荐的一种功能模式。Yubikey 在出厂的时候，会在 Slot 1 内配置好一个 Yubico OTP 的配置。当功能被触发（Slot 1 是触摸了按键，Slot 2 是长按按键）时，yubikey 会模拟发送键盘按键动作，将 OTP 发出来，比如

```
cccjgjgkhcbbirdrfdnlnghhfgrtnnlgedjlftrbdeut
```

## Modhex

Yubico OTP 面临的第一个问题，就是如何通过键盘把 OTP 发出来。有人可能会很疑惑，这有什么困难呢？实际上，我们大多数人，可能没有遇到过类似的问题。但事实上，有一个东西叫做键盘布局。键盘在与计算机通信的时候，发出的是 **[Scan Code](https://en.wikipedia.org/wiki/Scancode)**，而特定的 scan code 对应的字母，则是由