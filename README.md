### 1. 配置docker compose的.env环境参数，acme取证书用的Cloudflare上的KEY
### 2. 以oracle arm机DD debian系统来布署
#### 1. 创建实例时选择ubuntu系统,ssh登陆
```
curl -fLO https://raw.githubusercontent.com/bohanyang/debi/master/debi.sh
chmod a+rx debi.sh
sudo ./debi.sh --architecture arm64 --user root --password password
sudo reboot
```
#### 2. 等待几分钟系统DD好后 再ssh root password 登陆 开启BBR
```
# 先更新系统
apt-get update
apt-get install -y wget curl

# 开启原生BBR：
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p

# 检测是否正常开启BBR：
sysctl net.ipv4.tcp_available_congestion_control
lsmod | grep bbr
```
#### 3. 安装docker (本身已经是root登陆了，不需要sudo)
```
# 安装docker依赖包
apt-get install \
    ca-certificates \
    gnupg \
    lsb-release
   
# 安装官方gpg key 
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 添加源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
  
# 最后安装docker engine
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
docker -v
```
#### 4. 安装docker compose
```
# 创建目录
mkdir -p /usr/local/lib/docker/cli-plugins
# 拉取官方compose v2文件
curl -SL https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
# 附上执行权限
chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
# 测试
docker compose version
```
### 4. acme申请zeroSSL证书，zeroSSL官网里在develop里拿到api相关的key
#### 1. 拉取本项目到app目录
```
git clone https://github.com/ousqi/deploy-xray.git app
cd app
```
#### 2. 先把acme容器运行起来,注意改你的邮箱和域名,以DNS方式申请证书
```
docker compose up -d acme
docker compose exec acme acme.sh  --register-account  -m your@gmail.com --server zerossl
docker compose exec acme acme.sh  --register-account  --server zerossl \
        --eab-kid  your-kid  \
        --eab-hmac-key  your-hmac-key
docker compose exec acme acme.sh --issue  -d  *.yourdomain.xyz --dns dns_cf --keylength ec-256
```
### 5. 申请证书成功就编辑xray/config.json，证书位置和你的clients id值
```
//...省略
{
    "id": "your-id",
    "flow": "xtls-rprx-direct",
    "level": 0,
    "email": "your@example.com"
}
//...省略
{
    "ocspStapling": 86400,
    "certificateFile": "/acme/yourdomain_ecc/fullchain.cer",
    "keyFile": "/acme/yourdomain_ecc/*.yourdomain.xyz.key"
}
```

### 6. nginx的伪站反代了bootstrap4的站，可以自行修改其他
```
server {
    listen 80 default_server;    
    return 301 https://$host$request_uri;
}

server {
    listen 8001 proxy_protocol;
    listen 8002 http2 proxy_protocol;

    #关闭访问日志
    access_log off;

    location / {
        resolver 1.1.1.1;
        # 你喜欢的站
        proxy_pass https://themes.getbootstrap.com;
        proxy_ssl_server_name on;
    }
}
```

### 7. 一切准备就续，启动所有容器
```
docker compose down
docker compose up -d
docker compose ps
```

