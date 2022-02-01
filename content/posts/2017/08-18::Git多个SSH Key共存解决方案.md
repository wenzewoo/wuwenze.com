+++
title = "Git多个SSH Key共存解决方案"
date = "2017-08-18 07:44:00"
url = "archives/251"
tags = ["Git"]
categories = ["开发工具"]
+++

目前手上不止一个git账号，平台也不一致，这就比较尴尬了：

1.  同一个`SSH Key`中不允许两个账号
2.  再次生成新的`SSH Key`会将上次的覆盖

目前的解决办法是，生成多个`SSH Key`并命别名，通过配置文件指定域

### 用ssh-keygen生成多个key ###

```bash
$ ssh-keygen -t rsa -C "wuwz@live.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/wuwenze/.ssh/id_rsa): /Users/wuwenze/.ssh/github
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/ubuntu/.ssh/id_rsa_github.
Your public key has been saved in /home/ubuntu/.ssh/id_rsa_github.pub.
The key fingerprint is:
SHA256:T4s5hxOOX3ABGNHZNMPX6v7J1rgYxfzXMCuCipIh5Ng wuwz@live.com
The key's randomart image is:
+---[RSA 2048]----+
|      o=.=+  .   |
|      . o.oo. .  |
|          .. .   |
| .         ..o   |
|+.      S o.  *  |
|o.E    o @ ... =.|
| . o  . O *.o .o+|
|  o  . o = ..=o.o|
|   .. . .   .o=. |
+----[SHA256]-----+
```

根据提示，重新指定文件名，假设我有三个平台需要同时管理，即

```bash
/Users/wuwenze/.ssh/github
/Users/wuwenze/.ssh/gitee
/Users/wuwenze/.ssh/coding
```

依次按照上述操作，添加3个ssh-key，生成后的文件位于~/.ssh

![591f211d-da18-4843-992f-504c017abe0f.png][]

### 添加私钥 ###

git自动把新生成的私钥写到known\_hosts中

```bash
$ ssh-add ~/.ssh/github
$ ssh-add ~/.ssh/gitee
$ ssh-add ~/.ssh/coding
```

如果执行ssh-add时提示"`Could not open a connection to your authentication agent`"，可以执行命令：

```bash
$ ssh-agent bash
```

然后再运行ssh-add命令，添加完成后，通过以下命令来验证：

```bash
### 可以通过 ssh-add -l 来确私钥列表
$ ssh-add -l

### 可以通过 ssh-add -D 来清空私钥列表，清空后重复上一步操作
$ ssh-add -D
```

![328ce3e9-4879-4f51-87b9-7bb76af93b66.png][]

### 配置文件 ###

```bash
$ vi ~/.ssh/config

### coding
Host coding.net
    HostName coding.net
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa_coding

### github
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa_github

### github
Host gitee.com
    HostName gitee.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa_gitee
```

### 配置公钥并验证 ###

依次登录对应的平台，配置SSH公钥，如：github

![e3b26b2b-547f-43d0-9b54-51e15de0cdd1.png][]![0073d91e-9fa4-49b9-8dac-2e1832d3ea64.png][]

依次验证：

```bash
$ ssh -T git@github.com
$ ssh -T git@gitee.com
$ ssh -T git@git.coding.net
```

![c0d4c621-5040-4a75-b866-ab5d32da72fa.png][]


[591f211d-da18-4843-992f-504c017abe0f.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170818/591f211d-da18-4843-992f-504c017abe0f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[328ce3e9-4879-4f51-87b9-7bb76af93b66.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170818/328ce3e9-4879-4f51-87b9-7bb76af93b66.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e3b26b2b-547f-43d0-9b54-51e15de0cdd1.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170818/e3b26b2b-547f-43d0-9b54-51e15de0cdd1.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[0073d91e-9fa4-49b9-8dac-2e1832d3ea64.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170818/0073d91e-9fa4-49b9-8dac-2e1832d3ea64.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c0d4c621-5040-4a75-b866-ab5d32da72fa.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170818/c0d4c621-5040-4a75-b866-ab5d32da72fa.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg