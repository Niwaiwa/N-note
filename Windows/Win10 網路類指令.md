###### tags: `Win10`
# Win10 網路類指令

## VPN相關

Forticlient VPN, 抓不到Win10的VPN設定, 要改用Win10 App版也需要正式的憑證  
正常可透過刪掉VPN的0.0.0.0/0來自定義, 但forticlient vpn有硬限制變成無法刪除調整  
雖然想設定VPN split tunnel來處理, 但因上述原因Get-VPNConnection抓不到設定因此無法調整  
另有透過調整metric來處理的方式, 但測試後沒效, 應該是有什麼不足的  
因此只做為紀錄用  
※如果使用forticlient以外的VPN, 應該有機會調整  

```
Get-VPNConnection
Set-VPNConnection -Name "乙太網路 4" -SplitTunneling $True
netsh interface ipv6 show route
netsh interface ipv4 show route
netsh interface ipv4 show interface
netsh interface ipv4 delete route 0.0.0.0/0 "10.212.134.219"
netsh interface ipv4 delete route 0.0.0.0/0 "乙太網路 4" 10.212.134.201
netsh interface ipv4 add route 10.1.1.0/24 "乙太網路 4" 10.212.134.201
netsh interface ipv4 set route 10.1.1.0/24 "乙太網路 4" 10.212.134.201 metric=1
```

## Local 開啟服務PORT時卻無權限的情況

通常為系統占用, 執行以下指令後即可正常使用

```
net stop winnat
net start winnat
```