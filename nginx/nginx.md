# nginx相关问题
## nginx安装过程(最小安装centos7)
```
yum install gcc-c++
yum install pcre
yum install pcre-devel
yum install zlib
yum install zlib-devel
yum install openssl
yum install openssl-devel
cd /opt/nginx-1.12.2/
./configure
make
make indtall
```
以上完毕之后不出意外的话，就算安装完毕了，因为`./configure` 并没有指定安装路径，所以安装默认路径`/usr/local/nginx`  
**修改配置文件**  
使kibana反向代理输入账号密码  
生成账号密码文件  
```
yum install httpd-tools
mkdir -p /etc/nginx/passwd
htpasswd -c -b /etc./nginx/passwd/kibana.passwd user password
```
修改nginx.conf
```
server {
        listen 192.168.175.130:5601;

        auth_basic "Kibana Auth";
        auth_basic_user_file /etc/nginx/passwd/kibana.passwd;

        location / {
        proxy_pass http://127.0.0.1:5601;
        proxy_redirect off;
        }
        }
```
nginx启动：`./nginx`  
nginx重启：`./nginx -s reload`  
nginx 停止: `./nginx -s stop`  

### nginx 启动报错
```
netstat -ntlp
kill nginx相关命令 线程
```
### 配置nginx开机自启动
[niginx开机自启配置](https://blog.csdn.net/qq_39681301/article/details/79529498)

