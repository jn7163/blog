---
title: Python 学习手记：使用 smtplib 发送邮件
date: 2019-01-19 12:04:16
tags:
  - python
  - smtp
---

高中时期就用 Python 写过一个邮件发送脚本，但当时的逻辑处理得相当凌乱 。这次利用了一个周末晚上总结规整了一下 `smtplib` 与 `email` 模块的用法，并写了一个鲁棒性比较强且支持模板填充的[邮件群发脚本](https://gist.github.com/queensferryme/fac00cb791b7d53e49ad12a72b19d754) 。这篇博客就简单记录一下 。

<!--more-->

## 启用 SMTP 服务

首先我们必须了解，`SMTP` 是一种邮件发送协议（Simple Mail Transfer Protocol）。我们在本地使用 python 脚本作为客户端发送请求，还需要有一个服务器来处理这些 SMTP 请求 。你可以选择自己搭建一个 SMTP 服务器，亦或是使用第三方邮件服务提供商的 SMTP 服务 —— 为了方便起见，我选择了后者，因为几乎所有邮箱（gmail，qq，126）等都给用户提供免费 SMTP 服务 。一般来说你可以在**设置**里找到开启 SMTP 服务的选项；如果对此过程有问题，请在搜索引擎上查找对应邮箱的设置方法 。

如果一切顺利，你现在应该有一对用户名+密码用来认证登录远程 SMTP 服务器 。然后我们就可以尝试连接并登录了：

```python
from smtplib import SMTP, SMTPException

config = {
    'host': 'smpt.126.com',
    'port': 25,
    'user': 'your-user-name',
    'passwd': 'your-pass-word'
}

try:
	server = SMTP(config['host'], config['port'])
	server.login(config['user'], config['passwd'])
except SMTPException:
    print('ERROR: login failed')
else:
    print('INFO: login success')
```

在我的情况下，`user` 字段就是我的 126 邮箱帐号，端口号 `port` 一般来说都是 25（如果使用 SSL 连接可能会是 465，具体情况请查看相关文档）。

## 普通文本邮件

登录后就可以开始发送邮件了 。首先从最简单的纯文本文件说起：

```python
config['sender'] = 'sender-email-addr'
config['receiver'] = 'receiver-email-addr'
config['message'] = '''
Subject: Title

Python SMTP Test Mail.
'''

try:
	server.send_mail(config['sender'], config['receiver'], config['message'])
except SMTPExecption as error:
    print('ERROR: mail failed to send', error)
else:
    print('INFO: mail successfully sent')
```

注意 `Subject` 头部字段后连续两个换行是必须的 。如果一切正常，`config['receiver']` 的邮箱就会收到一封标题为 `Title`，正文内容为 `Python SMTP Test Mail` 的邮件 —— 但是大多邮箱服务商现在都有垃圾邮件过滤机制，你可能会在 error 里看见类似 SPAM 这样的信息 —— 这种情况下你可以选择将邮件内容改的真实一些或是先发送给自己，这都可以避免垃圾邮件过滤 。

我们还可以使用 `email` 模块来更优雅的构建邮件，例如：

```python
from email.header import Header
from email.mime.text import MIMEText

message = MIMEText('Python SMTP Test Mail', 'plain', 'utf-8')
message['Subject'] = Header('Title', 'utf-8')
message['From'] = Header(config['sender'], 'utf-9')
message['To'] = Header(config['receiver'], 'utf-8')
server.send_mail(config['sender'], config['receiver'], message.as_string())
```

## HTML 富文本邮件

含有 HTML 内容的邮件同样可以使用 `MIMEText` 来构建，只不过我们需要吧类型从 `text/plain` 修改为 `text/html` 。例如：

```python
config['message'] = '''
<h1>WELCOME</h1>
<p>we sincerely invite you to join our party.</p>
<p>your entrance id is <code>0021</code>.</p>
'''
message = MIMEText(config['message'], 'html', 'utf-8')
```

之后的操作基本相似 。

需要注意的是，如果你想向邮件的 HTML 里添加图片，直接使用外链大多数情况下是不行的；因为处于安全考量，邮件服务商一般会屏蔽这些带有外链的图片 。我们需要从本地以一种特殊方式添加图片资源 。

```python
from email.mime.image import MIMEImage
from email.mime.multipart import MIMEMultipart

config['message'] = config['message'] + '<img src="cid:0"/>'
message = MIMEMultipart()
message.attach(MIMEText(config['message'], 'html', 'utf-8'))
with open('invitation.png', 'rb') as fin:
    image = MIMEImage(fin.read(), _subtype='png')
image.add_header('Content-ID', '0')
message.attach(image)
```

图片的链接应被写作 `cid:<cid>` 的形式，然后对应的图片资源要加入 `Content-ID:<cid>` 头部后附加到 `message` 中 。

## 为邮件添加附件

添加附件并不困难，你同样只需要向一个 `Multipart` 对象中添加相应的附件即可：

```python
from email.mime.base import MIMEBase
from encoders import encode_base64

att = MIMEBase('application', 'octet-stream')
att.add_header('Content-Disposition', 'attachment;filename=invitation.png')
with open('invitation.png', 'rb') as fin:
    att.set_payload(fin.read())
encode_base64(att)
message.attach(att)
```

附件头部中的 `filenam` 字段才是邮箱里显示的附件名，这意味着你可以让一个文件看起来像其他类型的文件 。

## 使用 SSL 加密连接

现在是 https 的时代，我们的 smtp 同样可以使用 ssl 来加密 。加密方式也相当简单：

```python
from ssl import create_default_context

server = SMTP(config['host'], config['port'])
server.starttls(context=create_default_content())
```

在 `starttls` 之后的连接就都是加密的了，这能有效地防止你的 SMTP 服务密码泄漏 。

------

参考文献：

- [Real Python: Sending Emails With Python](https://realpython.com/python-send-email/#make-a-csv-file-with-relevant-personal-info)
- [廖雪峰 python 教程：SMTP 发送邮件](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432005226355aadb8d4b2f3f42f6b1d6f2c5bd8d5263000)