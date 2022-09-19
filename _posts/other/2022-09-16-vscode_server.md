---
title: 云服务器部署code-server无域名支持https
tagline: ""
last_updated: null
category : code-server
layout: post
tags : [https, code-server]
---
### 安装code-server
```
wget https://github.com/coder/code-server/releases/download/v4.6.1/code-server-4.6.1-linux-amd64.tar.gz
tar zxvf code-server-4.6.1-linux-amd64.tar.gz
cd code-server-4.6.1-linux-amd64/
```
### 启动code-server
```
 ./bin/code-server --port 8080 --host 0.0.0.0 --auth password
```

启动完就可以通过ip:8080 访问了，但是这个发现插件都没有生效

看官方是必须https方式才可以支持

#支持Https

本来打算nginx+certbot配置下就好了,折腾半天，死活搞不通，这是才想起一定是审查备案,奈何天X云不开放80/443端口，只好放弃   
为啥用天X云,羊毛撸的16G机器便宜,github速度数10M比20k腾X云好多了吧,用来部署code-server应该还是够的
偶然想起openssl可以支持ip

### OpenSSL自签发配置ip地址的证书

#### 安装openssl
```
yum install openssl openssl-devel -y
```
####  openssl配置文件openssl.cnf
```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = CH
stateOrProvinceName = State or Province Name (full name)
stateOrProvinceName_default = ZJ
localityName = Locality Name (eg, city)
localityName_default = HangZhou
organizationalUnitName  = Organizational Unit Name (eg, section)
organizationalUnitName_default  = THS
commonName = Internet Widgits Ltd
commonName_max  = 64

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]

# 天X云服务器ip
IP.1 = 公网ip
IP.2 = 0.0.0.0
IP.3 = 内网ip

```
#### 生成私钥
```
openssl genrsa -out abc.key 2048
```
#### 创建CSR文件
```
openssl req -new -out abc.csr -key abc.key -config openssl.cnf

```
#### 测试CSR文件
```
openssl req -text -noout -in abc.csr
```
#### 自签名并创建证书
```
openssl x509 -req -days 3650 -in abc.csr -signkey abc.key -out abc.crt -extensions v3_req -extfile openssl.cnf
```
最终生成的3个文件
```
abc.csr
abc.crt
abc.key abc.key
```
### 将abc.crt下载到本地并导入
这一步很关键，不然浏览器还是会报不信任证书
> windows 命令行输入: certmgr.msc
受信任的证书颁发机构->所有任务->导入
更多工具里点添加到桌面
最后启动code-server
```
./bin/code-server --port 8080 --host 0.0.0.0  --cert /home/openssl/abc.crt  --cert-key /home/openssl/abc.key

```


