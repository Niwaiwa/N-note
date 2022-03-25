###### tags: `Tool`

# Multipass

Windows 使用Linux新工具
※正常使用預設hyper-v虛擬化工具, 要可以正常訪問外網也要可以正常SSH遠端VM, 不行的話還是不太建議現在使用

## 問題排除

如果正常安裝可以正常使用對外網路(hyper-v)就沒有問題

### 網路無法對外連線的情況

不使用hyper-v, 改使用virtualBox
但版本需使用6.1.2, 最新版有可能有異常導致不能跟hyper-v同時使用
安裝完virtualBox之後重新安裝multipass並選擇使用virtualBox
※ 或者使用multipass cli手動改設定為使用virtualBox

#### 安裝docker-engine

```
sudo apt update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

#### 安裝docker-compose

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

#### ifconfig指令無法使用

安裝net-tools

```
apt-get install net-tools
```

#### 所有multipass vm都是同一個ip

修改netplan設定檔

```
vi /etc/netplan/50-cloud-init.yaml
```

加入address參數

```
network:
    ethernets:
        enp0s3:
            dhcp4: true
            addresses: [10.0.2.100/24]
            match:
                macaddress: 52:54:00:fa:a0:3c
            set-name: enp0s3
    version: 2
```

重新應用netplan

```
netplan apply
```