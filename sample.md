### 简单部署样例（简单3台centos虚拟机包含一台nfs，两台k8s集群）
* 以k8s-cluster搭建k8s集群
* k8s-master 和 baas-kubeengine 部署同一台centos
  * 将k8s master的$HOME/.kube/config文件 替换 kubeconfig/config
  * 修改配置文件 keconfig.yaml  
* nfs服务器和 baas-fabricengine 部署同一台centos
  * 设置 GOPATH 环境变量
  * 创建baas根目录
    * 复制 baas-template到其下
    * 创建nfs共享目录 baas-nfsshared 
  * 修改配置文件 feconfig.yaml  
  * nfs安装和配置
    * yum -y install nfs-utils rpcbind
    * id (查看当前用户的uid和gid)
    * vim /etc/exports (添加配置,相应修改)  
      ```
      /baas根目录/baas-nfsshared 192.168.1.0/24(rw,sync,insecure,anonuid=当前用户的uid,anongid=当前用户的gid)
      ```
    * exportfs -r (配置生效)
    * service rpcbind start &&  service nfs start (启动rpcbind、nfs服务)
  * 启动 baas-fabricengine 
* baas-gateway 随便部署到其中一台centos
  # 部署Gateway
 
- 进入baas-gateway目录下，通过docker安装mysql

``sudo docker pull mysql:5.7``
![image name](https://github.com/2643920101/baasmanager/blob/main/d6e5d54a3fb20d9f310ead746fcfe40.png)


- 通过docker启动MySQL容器


**记住添加后返回的容器ID**
```
sudo docker run -p 3306:3306 --name apimysql \
> -e MYSQL_ROOT_PASSWORD=123456 \
> -d mysql:5.7
```
![image name](https://github.com/2643920101/baasmanager/blob/main/2.png)



-  查看所有docker容器，是否有mysql的容器

``sudo docker images``
![image name](https://github.com/2643920101/baasmanager/blob/main/3.png)

- 以容器的形式安装mysql 需要进入容器内部才可以使用mysql

- 进入 MySQL 容器的 shell

``sudo docker exec -it apimysql bash``
![image name](https://github.com/2643920101/baasmanager/blob/main/4.png)

- 将数据库配置文件复制进容器内

```
sudo docker cp mysql.sql 67af2612f19b78061329b3f169def2dc9c4485c6cf3acb7774ac5ad62a203fb8:/root
```
![image name](https://github.com/2643920101/baasmanager/blob/main/5.png)


- 重新进入 MySQL 容器的 shell

``sudo docker exec -it apimysql bash``

- 在容器中使用 mysql 命令登录 MySQL：密码123456

``mysql -u root -p``

- 查看数据库管理系统中的所有数据库

``show databases;``

![image name](https://github.com/2643920101/baasmanager/blob/main/6.png)

- 配置mysql.sql数据库

``source /root/mysql.sql``

![image name](https://github.com/2643920101/baasmanager/blob/main/7.png)

- 再次查看数据库管理系统中的所有数据库

``show databases;``

![image name](https://github.com/2643920101/baasmanager/blob/main/8.png)

- 修改配置文件 gwconfig.yaml

``vim gwconfig.yaml``

![image name](https://github.com/2643920101/baasmanager/blob/main/9.png)

- 配置Go环境

```
go env -w GO111MODULE=on

go env -w GOPROXY=https://goproxy.cn,direct
```
![image name](https://github.com/2643920101/baasmanager/blob/main/10.png)

- 安装gcc，g++
```
sudo apt install gcc

sudo apt install g++
```
![image name](https://github.com/2643920101/baasmanager/blob/main/11.png)

- 更新go运行环境

``go build ./main.go``

- 执行运行

``go run main.go``
![image name](https://github.com/2643920101/baasmanager/blob/main/12.png)
![image name](https://github.com/2643920101/baasmanager/blob/main/13.png)

完成


* baas-frontend 随便部署到其中一台centos
  * npm run build:prod 打包
  * 用nginx部署，把打包生成的dist文件夹复制并重命名/usr/share/nginx/baas
  * 配置nginx.conf反向代理(相应修改baas-gateway地址)
    ```
    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    
    events {
    	worker_connections 768;
    	# multi_accept on;
    }
    
    http {
        include       mime.types;
        default_type  application/octet-stream;
    
        log_format  logformat  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for" '
                          '"[$request_time]" "[$upstream_response_time]" '
                          '"[$connection]" "[$connection_requests]" '
                          '"$http_imei" "$http_mobile" "$http_type" "$http_key" "$cookie_sfpay_jsessionid"';
        access_log  /var/log/nginx/access.log logformat;
    
    
        sendfile        on;
        #tcp_nopush     on;
        underscores_in_headers on;
    
        keepalive_timeout  65;
        proxy_connect_timeout 120;
        proxy_read_timeout 120;
        proxy_send_timeout 60;
        proxy_buffer_size 16k;
        proxy_buffers 4 64k;
        proxy_busy_buffers_size 128k;
        proxy_temp_file_write_size 128k;
        proxy_temp_path /tmp/temp_dir;
        proxy_cache_path /tmp/cache levels=1:2 keys_zone=cache_one:200m inactive=1d max_size=30g;
    
        client_header_buffer_size 12k;
        open_file_cache max=204800 inactive=65s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 1;
    
    
    
        gzip  on;
        gzip_types       text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png image/jpg;
        # baas-gateway地址
        upstream baasapi {
            server 127.0.0.1:6991;
        }
    
       
        # HTTP server
        #
        server {
            listen       8080;
            server_name  baasadmin;
    
            location /nginx_status {
                    stub_status on;
                    access_log off;
            }
            location /api/{
                proxy_pass  http://baasapi/api/;
                proxy_set_header  X-Real-IP  $remote_addr;
                proxy_set_header Host $host;
    
            }
            location /dev-api/{
                proxy_pass  http://baasapi/api/;
                proxy_set_header  X-Real-IP  $remote_addr;
                proxy_set_header Host $host;
    
            }
            location /stage-api/{
                proxy_pass  http://baasapi/api/;
                proxy_set_header  X-Real-IP  $remote_addr;
                proxy_set_header Host $host;
    
            }
    
            location / {
                root   baas;
                index  index.html index.htm;
            }
    
            location ~ ^/favicon\.ico$ {
                root   baas;
            }
             
        }
    }
    ```
  * 启动nginx
    ``` 
    sudo service nginx start
    ```
  * 访问 http://ip:8080 


