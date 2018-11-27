---
layout:     post
title:      "[译]Dropbox是如何安全地存储用户密码的"
date:       2016-11-05 21:22:00 +0800
author:     "ssj"
category: Web开发
catalog: true
comments: true
tags:
    - Dropbox
    - 安全
---

本文最早发于 [简书](https://www.jianshu.com/p/5fa8289c9249)。

> 最初看到这篇文章是在 @登州知府 的微博上看到的，他的微博上分享了很多好的技术博客，推荐。由于本人英语学的比较烂，翻译的错漏之处请大家指正。原文在这里：[How Dropbox securely stores your passwords](https://blogs.dropbox.com/tech/2016/09/how-dropbox-securely-stores-your-passwords/)

# 概述
众所周知，存储明文密码是一件很糟糕的事情。一旦数据库存储了明文密码，那么用户账号就危险了。因为这个原因，早在1976年，工业界就提出了一套使用单向哈希机制来安全地存储密码的标准(从Unix Crypt开始）。很不幸的是，尽管这种方式可以阻止你直接读取到密码，但是所有的哈希机制都不能阻止攻击者在离线环境下暴力破解它，攻击者只需要遍历一个可能包含正确密码的列表，对每个可能的密码进行哈希然后跟获取到的密码（使用哈希机制存储的密码）比对即可。在这种环境下，安全哈希函数如SHA在用于密码哈希的时候有一个致命的缺陷，那就是它们运算起来太快了。一个现代的商用CPU一秒钟可以生成数百万个SHA256哈希值。一些特殊的GPU集群的计算速度甚至可以达到每秒数十亿次。

过去的这些年，为了应对攻击，我们对密码哈希方法进行了数次升级。在本文中，我们将会为各位分享我们关于密码存储机制的更多的细节以及我们为什么要这么做的原因。我们的密码存储方案依赖三个不同层级的密码保护，如下图所示。为了方便说明，在下图中以及接下来我们省略了字节编码(base64)。

![Dropbox存储密码的层级图](http://upload-images.jianshu.io/upload_images/286774-3f8b649e400edb58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们采用bcrypt作为我们的核心哈希算法，每个用户都有一个独立的salt以及一个加密的key（这个key也可以是一个全局的，通常也叫pepper），salt和key是分开存储的。我们的方法与基础的bcrypt算法在一些重要的方面是不同的。

首先，用户的明文密码通过SHA512算法转换成了一个哈希值。这一步主要是针对bcrypt的两个突出的问题。有些bcrypt的实现中会将用户输入截取为72个字节大小以降低密码熵，而另外有一些实现并没有截取用户输入导致其容易受到DoS攻击，因为它们允许任意长度的密码输入。通过使用SHA，我们可以快速的将一些的确很长的密码转换为一个512比特的固定长度，解决了上述两个问题-即避免降低密码熵和预防[DoS攻击](http://arstechnica.com/security/2013/09/long-passwords-are-good-but-too-much-length-can-be-bad-for-security/)。**[译者注：关于第一点，熵是信息学里面的一个概念，这里引入信息学中的信息熵（我们常听人说这个信息多、那个信息少，对信息“多少”的量化就是信息熵），用它来作为密码强度的评估标准。信息熵计算公式为 H = L * log 2 N，其中，L表示密码的长度，N是字符种类，密码强度 （H） 与密码长度 （L） 和密码包含字符的种类 （N） 这两个因素有关。也就是说密码包含的字符种类越多，密码长度越长，熵越大，更多细节参见[这篇文章](http://www.guokr.com/article/61644/))。由于一些bcrypt算法截断了用户密码为72个字节长度，从而导致超过72个字节的用户输入无效，一定程度降低了密码熵。而第二点是有文章提到如果不限制用户输入的密码长度，很容易遭到DoS攻击，比如django之前有个版本没有限制密码长度，而它用的又是PBKDF2哈希算法(PBKDF2是是一个CPU计算密集型算法，但是对GPU效果不如bcrypt，[这里有个比较](http://security.stackexchange.com/questions/211/how-to-securely-hash-passwords))，这样如果攻击者输入的密码长度达到1M的话，对密码进行哈希需要几分钟的计算时间从而在大量这样的请求下导致服务器无法正常服务，这里使用SHA512先进行一次哈希的优缺点分析还可以参见[这个帖子](http://stackoverflow.com/questions/16594613/how-to-hash-long-passwords-72-characters-with-blowfish/16597402)]**

然后，对SHA512哈希后的值使用bcrypt算法再次哈希，使用的工作因子是10，每个用户都有一个单独的salt。不像其他的哈希算法比如SHA等，bcrypt算法很慢，它很难通过硬件和GPU加速。设置工作因子为10，在我们的服务器上执行一次bcrypt大概需要100毫秒。**[译者注：使用python的bcrypt模块，默认的工作因子为12，在我的电脑上执行一次大概是300毫秒左右，而如果工作因子设置为20，这个时间大概为89秒]**

最后，使用bcrypt哈希过后的结果再次使用AES256算法进行加密，使用的密钥是所有用户同意的，我们称之为*pepper*。pepper是我们基于深度考量的一种防御措施，pepper以一种攻击者难以发现的方式存储起来(比如不要放在数据库的表中)。由此，如果只是密码被拖库了，通过AES256加密过的哈希密码对于攻击者来说毫无用处。


# 为什么不用{scrypt,argon2}
我们也曾考虑过使用scrypt，但是我们对bcrypt有更多的经验。关于这几种算法那种更好的讨论一直都有，大部分的安全领域的专家都认为scrypt和bcrypt的安全性上相差无几。

我们考虑在下一次升级中使用argon2算法：因为在我们采用当前的方案的时候，argon2还没有赢得 [Password Hashing Competition](https://password-hashing.net/)。此外，尽管我们认为argon2是非常棒的密码哈希函数，我们更倾向于采用bcrypt，因为从1999年以来，bcrypt还没有发现有任何重大的攻击存在。

# 为什么使用一个全局的密钥(pepper)替代哈希函数
如前面提到的，采用一个全局的密钥是我们深度权衡后的一个防御措施，而且，pepper我们是单独存储的。但是，单独存储pepper也意味着我们要考虑pepper泄露的可能性。如果我们只是用pepper对密码进行哈希，那么一旦pepper泄露，我们无法从哈希后的结果反解得到之前bcrypt哈希过的密码值。作为一个替代方案，我们使用了AES256加密算法。AES256算法提供了差不多的安全性，同时我们还可以反解回原来的值。尽管AES256这个加密函数的输入是随机的，我们还是额外加上了一个随机的初始化向量(IV)来增强安全性。

下一步，我们考虑将pepper存储到一个硬件安全模块([HSM](https://en.wikipedia.org/wiki/Hardware_security_module)），对我们来说，这虽然是一个相当复杂的任务，但是它能极大的降低pepper泄露的风险。同时，我们也计划在下一次升级中增强bcrypt的强度。

# 展望
我们相信使用SHA512,加上bcrypt和AES256是当前保护密码最稳妥和流行的方法之一。同时，所谓道高一尺魔高一丈。我们的密码哈希程序只是加固Dropbox的众多举措之一，我们还部署了额外的保护措施-比如针对暴力攻击者密码尝试次数的速度限制，验证码，以及其他一些方法等。如之前图片中所示，我们积极的在各个层级进行投入以确保安全。当然，也很期待能够听到诸位的高见。

# 译者注
总结一下这篇文章，说道的Dropbox的加密方法大致就是三点：其一，使用SHA512把明文密码哈希，既避免降低密码的熵，又能防止DoS攻击。其二，使用bcrypt二次哈希，工作因子为10，每个用户都有一个独立的salt。最后，使用一个全局密钥(pepper)通过AES256算法对二次哈希的值进行加密存储。




