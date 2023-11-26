### 简单部署样例（简单3台centos虚拟机包含一台nfs，两台k8s集群）
### 地址规划
| 主机名 |IP地址  |节点 |
|--|--|--|
| djtu02 | 192.168.101.14 |主节点
|djtu07|192.168.101.52|从节点
|djtu17|192.168.101.23|从节点
* 克隆程序
  `git clone https://github.com/mikewang68/baasmanager`
### 部署k8s
  * 系统初始化 （***所有节点都需要做***） 
     * 配置hosts

  修改所有主机hosts，把每台主机IP和主机名对应关系写入host文件
  `sudo vim /etc/hosts`

  > 192.168.101.14 	djtu02
  > 
  > 192.168.101.23 	djtu17
  > 
  > 192.168.101.52 	djtu07
  
  * 关闭swap分区
    `sudo swapoff -a  &&  sudo  sed  -i  's/^\/swap.img\(.*\)$/#\/swap.img \1/g' /etc/fstab &&  free`

    验证是否关闭:```free -m```
  * 关闭防火墙
    ```
    sudo systemctl stop ufw.service
    sudo systemctl disable ufw.service
    #查看防火墙状态:
    sudo ufw status
    ```
    * 将网桥的ip4流量转接到iptables
      ```
      cat > /etc/sysctl.d/k8s.conf << EOF
      net.bridge.bridge-nf-call-ip6tables = 1
      net.bridge.bridge-nf-call-iptables = 1
      EOF
      ```
    执行`sysctl --system`生效
    
### 安装docker
 * 更新apt软件包缓存
   ```
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io
   apt update
   apt-get install ca-certificates curl gnupg lsb-release
   ```
  * 安装证书
    ```
    curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    ```
  * 写入软件源信息
    ```
    sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
    ```
  * 查看可安装的版本
    ```
    apt-cache madison docker-ce
    ```
  * 安装
    ```
    sudo apt-get install docker-ce=5:20.10.22~3-0~ubuntu-focal docker-ce-cli=5:20.10.22~3-0~ubuntu-focal containerd.io
    ```
  * 启动docker
    ```
    systemctl start docker
    ```
  * 重启，设置开机自启
    ```
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    sudo systemctl enable docker
    ```
  * 查看版本信息
    ```
    docker version
    ```

    
### nfs服务器和 baas-fabricengine 部署同一台centos
  * 查看docker和docker-compose和go是否安装，如未安装，按照上述步骤从新安装
  * Fabric的安装，创建目录并进入
    ```
    mkdir -p $GOPATH/src/github.com/hyperledger
    cd $GOPATH/src/github.com/hyperledger
    ```
  * 下载fabric源码
    ```
    git clone https://github.com/hyperledger/fabric.git
    ```
  * 脚本下载
    ```
    sudo ./bootstrap.sh
    ```
  * 测试网络
    ```
    cd ./fabric-samples/first-network/
    ./byfn.sh up
    ```
  * 关闭网络
    ```
    ./byfn.sh down
    ```
  * 在baasmanager路径下，创建baas根目录
    ```
    mkdir baas
    ```
  * 复制baas-template到其下
    ```
    cp -r baas-template baas
    ```
  * 在baas路径下创建nef共享目录baas-nfsshared,并使其生效
    ```
    cd bass
    mkdir baas-nfsshared
    chmod 755 -R baas-nfsshared/
    ```

* 安装nfs
  * 在baasmanager路径下,下载安装nfs
    ```
    yum -y install nfs-utils rpcbind
    ```
  * 在baas路径下，修改配置文件
    ```
    vim /etc/exports
    /home/djtu17/baasmanager/baas/baas-nfsshared 192.168.101.0/24(rw,sync,insecure,anonuid=0,anongid=0,subtree_check)
    ```
  * 使用exportfs -r命令使NFS配置生效
    ```
    exportfs -r
    service rpcbind start && service nfs start (启动rpcbind、nfs服务)
    ```
  * nfs服务启动后，使用rpcinfo -p查看端口号是否生效
    ```
    rpcinfo -p
    ```
  * 我们可以使用 showmount 命令来查看服务端(本机)是否可连接
    ```
    showmount
    ```
  * 启动 baas-fabricengine
    ```
    go build ./main.go
    go run main.go
    ```

### 随便部署到其中一台centos,我这里还是部署到了安装nfs的那台机器上 
* 安装mysql
  - 进入baas-gateway目录下，通过docker安装mysql

  ``sudo docker pull mysql:5.7``
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/1.png)


- 通过docker启动MySQL容器


  **记住添加后返回的容器ID**
```
  sudo docker run -p 3306:3306 --name apimysql \
  > -e MYSQL_ROOT_PASSWORD=123456 \
  > -d mysql:5.7
```
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/2.png)



-  查看所有docker容器，是否有mysql的容器

  ``sudo docker images``
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/3.png)

- 以容器的形式安装mysql 需要进入容器内部才可以使用mysql

- 进入 MySQL 容器的 shell

  ``sudo docker exec -it apimysql bash``
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/4.png)

- 将数据库配置文件复制进容器内

```
  sudo docker cp mysql.sql 67af2612f19b78061329b3f169def2dc9c4485c6cf3acb7774ac5ad62a203fb8:/root
```
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/5.png)

- 重新进入 MySQL 容器的 shell

  ``sudo docker exec -it apimysql bash``

- 在容器中使用 mysql 命令登录 MySQL：密码123456

  ``mysql -u root -p``

- 查看数据库管理系统中的所有数据库

  ``show databases;``

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/6.png)

- 配置mysql.sql数据库

  ``source /root/mysql.sql``

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/7.png)
- 再次查看数据库管理系统中的所有数据库

  ``show databases;``

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/8.png))
- 修改配置文件 gwconfig.yaml

  ``vim gwconfig.yaml``

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/9.png)

- 配置Go环境

  ```
  go env -w GO111MODULE=on

  go env -w GOPROXY=https://goproxy.cn,direct
  ```
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/10.png）

- 安装gcc，g++
  ```
  sudo apt install gcc

  sudo apt install g++
  ```
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/11.png)

- 更新go运行环境

  ``go build ./main.go``

- 执行运行

  ``go run main.go``
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/12.png)
![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/13.png)
完成！

## 安装go1.20.6

- 获取安装包

  ``wget [http://go.dev/dl/go1.20.6.linux-amd64.tar.gz](http://go.dev/dl/go1.20.6.linux-amd64.tar.gz)``

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/go1.png)

- 解压文件

  ``sudo tar -C /usr/local -xzf go1.20.6.linux-amd64.tar.gz``

- 找到对应文件夹

  ``ls /usr/local/go``

- 设置环境变量

  ``nano ~/.bash_profile``

- 添加代码如下：export PATH=$PATH:/usr/local/go/bin  

- 运行以下命令应用配置更改：

  ``source ~/.bash_profile``

- 使用go version验证是否安装成功

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/go2.png)

![image name](https://github.com/mikewang68/baasmanager/blob/master/baas-gateway/go3.png)

完成！

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


