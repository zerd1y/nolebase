# Redis
## 安装配置

>[!tip] 注意
> 基于 Windows + WSL

### 第一步：安装 wsl
### 第二步：安装 Redis  

1. 打开 wsl
2. [Install Redis on Windows](https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/install-redis-on-windows/)
### 第三步：配置

用 vim 编辑器进入配置文件

```Shell
sudo vim /etc/redis/redis.conf
```

>[!tip]
>修改一下配置：
>1. 允许访问的地址，默认是127.0.0.1，会导致只能在本地访问。修改为0.0.0.0则可以在任意IP访问，生产环境不要设置为0.0.0.0
>```shell
>bind 0.0.0.0
>```
>2. 守护进程，修改为yes后即可后台运行
>```shell
>daemonize yes 
>```
>3. 密码，设置后访问Redis必须输入密码
>```shell
>requirepass 101024
>```
>4. 日志文件，默认为空，不记录日志，可以指定日志文件名
>```shell
>logfile "redis.log"
>```

设置开机自启（只有在打开 WSL 时启动Redis）

>[!tip]
>1. 进入配置文件
>```Shell
>nano ~/.bashrc
>```
>2. 添加 Redis 启动命令
>```Shell
># Start Redis server if not already running
>if ! pgrep -x redis-server > /dev/null; then
>sudo service redis-server start
>fi
>```
>3. 保存并退出
>`Ctrl+O` -> `Enter` -> `Ctrl+X`
>
>4. 生效命令
>```Shell
>source ~/.bashrc

检查 Redis 是否运行：

```Shell
sudo service redis-server status
```

## Redis 客户端

下载 Another Redis Desktop Manager

获取 WSL 动态 IP（**每次启动 WSL 都会更新 IP**）

```Shell
ip addr show eth0 | grep -oP 'inet\s+\K[\d.]+'
```

进入客户端连接