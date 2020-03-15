最近问度娘时发现一些有趣东西，就是在linux下利用PythonSMTP发送邮件。我并不会写python，大家轻喷。**亲测成功**
#### 由于大家生产环境的日志文件的格式都不一样，所以需要对os.popen()里的shell命令做适当的修改

在[菜鸟教程](https://www.runoob.com/python/python-email.html)里有PythonSMTP的实例代码，我把它改成了自己需要用的。

## SMTP介绍
SMTP是一种提供可靠且有效的电子邮件传输的协议。SMTP是建立在FTP文件传输服务上的一种邮件服务，主要用于系统之间的邮件信息传递，并提供有关来信的通知。SMTP独立于特定的传输子系统，且只需要可靠有序的数据流信道支持，SMTP的重要特性之一是其能跨越网络传输邮件，即“SMTP邮件中继”。使用SMTP，可实现相同网络处理进程之间的邮件传输，也可通过中继器或网关实现某处理进程与其他网络之间的邮件传输。（转自百度百科）

## 需要的环境
linux系统下需要安装sendmail
下面是ubuntu系统安装命令
```
sudo apt-get install sendmail
```
python里需要安装smtplib模块
首先是：
```
pip search smtplib
```
![smtplib.png](https://upload-images.jianshu.io/upload_images/18154407-b8bc4d6a0a0c3198.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里一般用PyEmail,所以直接pip install PyEmail 就行
**代码直接贴上了，有注释**
```python
#!/usr/bin/python3
# -*- coding: UTF-8 -*-

import os
import re
import sys
import time
import smtplib
from email.mime.text import MIMEText
from email.utils import formataddr
 
my_sender = 'abcdefg@163.com'                 # 发件人邮箱账号 这个邮箱必须开启smtp服务 一般默认是开启的
my_pass = 'abcdefg'                           # 发件人邮箱密码 这里可以不用密码 可以在自己的邮箱设置里申请授权码
my_user_list = ['abc@163.com','abcd@163.com'] # 收件人邮件列表(可添加多个邮箱)
pre_line_define = 10                          # 报错行往上10行信息
next_line_define = 60                         # 报错行往下60行信息
param = 'stack traceback'                     # 用于检索日志文件内的字符串(用于找到报错位置)
filename = 'game-world-trunk_sanguo.log'      # 日志文件
haveSendLineFile = 'lineRecord.txt'           # 记录已经发送邮件的日志文件里的行号

def open_SMTP_server():  # 创建一个smtp服务
    try:
        # smtplib.SMTP_SSL("smtp.163.com", 465) 改为这个也行 用ssl更安全
        # smtp的服务器和端口可以问度娘
        server = smtplib.SMTP("smtp.163.com", 25) # 发件人邮箱的smtp服务器和端口
        server.login(my_sender, my_pass)
        return server
    except Exception:  # 本人对python不熟悉 只能这样写了
        return -1

def close_SMTP_server(server): # 关闭smtp服务
    server.quit()

def get_message(context, user):  # 使用MIMEText构建一个msg(必须有From、To和Subject 不然会被认为是垃圾邮件)
    msg = MIMEText(context, 'html', 'utf-8')
    msg['From'] = formataddr(["测试1", my_sender])
    msg['To'] = formataddr(["测试2", user])
    msg['Subject'] = "发邮件测试" 
    return msg

def mail(context, line): # 发送邮件
    server = open_SMTP_server()
    if server == -1:
        print('SMTP open error')
        return

    for user in my_user_list: # 获取收件人
        try:
            msg = get_message(context, user) # 发送构建好的message
            server.sendmail(my_sender, [user,], msg.as_string())

        except Exception:
            print('发送邮件出错')

    close_SMTP_server(server) # 关闭smtp服务
    return True

# 处理html
def lines(file):
    for line in file:
        yield line
    yield '\n'

# 处理html
def blocks(file):
    block = []
    for line in lines(file):
        if line.strip('\n'):
            block.append(line)
        elif block:
            yield ''.join(block).strip('\n')
            block = []

# 处理html
def dealToHtml(msg):
    context = ''
    for block in blocks(msg):
        block = re.sub(r'\*(.+?)\*', r'<em>\1</em>', block)
        context = context + block + '<br/>'
    return context

def mailsend(lineArray):
    tempArray = []
    for line in lineArray:
        message = ''
        # 将line往上10行的信息和往下60行的信息按行读出 并处理为html
        message_result = os.popen('sed -n ' + '\'' + str(int(line) - pre_line_define) + ',' + str(int(line) + next_line_define) + 'p\' ' + filename)
        msg = message_result.read()
        for line_msg in msg.splitlines():
            message = message + line_msg + '\n'
        result = mail(dealToHtml(message), int(line))
        if result == True:
            tempArray.append(line)

    return tempArray

def getHaveSendLine(): # 拿到已经发送过邮件的行号 已经发送过的就不需要再发了
    lineArray = []
    try:
        file = open(haveSendLineFile, 'r')
        for line in file.readlines():
            line = line.strip()
            lineArray.append(line)
    except OSError:
        print('文件不存在')
    finally:
        file.close()
    
    return lineArray

def getNeedSendLine(): # 获取到需要发送的行号
    result = os.popen('grep -n \'' + param + '\' ' + filename + ' | cut -d\':\' -f 1')
    lineArray = result.read().splitlines()
    return lineArray

def isHaveElement(array, element):
    for value in array:
        if value == element:
            return True
    return False

def addToHaveSendFile(array): # 将已经发送的行号记录下来
    try:
        file = open(haveSendLineFile, 'a+')
        for line in array:
            file.write(line + '\n')
    except OSError:
        print('打开文件出错')
    finally:
        file.close()

def main():
    while True:
        haveSendLineArray = getHaveSendLine() # 拿到之前已经发送过的行号
        needSendLineArray = getNeedSendLine() # 拿到需要发送的行号

        differentLineArray = []

        for line in needSendLineArray:
            result = isHaveElement(haveSendLineArray, line) # 找到在haveSendLineArray中没有的行号 记录下来 需要发送的
            if result == False:
                differentLineArray.append(line)

        if len(differentLineArray) != 0:
            result = mailsend(differentLineArray)
            addToHaveSendFile(result) # 记录下已经成功发送的行号
            print(result)

        time.sleep(60) # 睡眠60再检测 看具体需求


if __name__ == "__main__":
    main()

```
---
2020.2.28  14:22  广州
