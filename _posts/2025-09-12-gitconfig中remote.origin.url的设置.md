---
layout:     post
title:      "gitconfig中remote.origin.url的设置"
date:       2025-09-12
author:     "suhiymof"
header-img: "img/post-bg-2015.jpg"
tags:
    - git push两步验证
    - git 2FA
---

由于github开启了2步验证-Two Factor Authentication (2FA),所以我们push到github仓库的时候如果还是使用账号密码会报错.

我在windows上用 `gitbash` 执行 `git push origin master`的时候输出如下:
```bash
remote: Invalid username or token. Password authentication is not supported for
Git operations.
fatal: Authentication failed for 'https://github.com/xxx/xxx.github.io
.git/'

```

这时候需要我们去[github](https://github.com/settings/tokens)上申请一个access token, 选择 `Personal access tokens (classic)`, 填入note,选择自己想要的选项去生成一个就好了。

> :memo: **注意：** : Make sure to copy your personal access token now. You won’t be able to see it again(必须保存好这个token，因为不会出现第二次。)

打开.git目录下的config文件,修改[remote "origin"]下的url类似这样：

`https://xxx:accesstoken@github.com/xxx/xxx.github.io.git`,`accesstoken` 就是我们新生成的 `token`。

接下来 `git push` 推送吧

                                                             enjoy!!!!
