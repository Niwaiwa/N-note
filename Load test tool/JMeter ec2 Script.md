###### tags: `壓測`

# JMeter ec2 Script

本專案使用Script來啟動AWS EC2(Ubuntu 18.04)並且自動安裝Java 11、Jmeter-5.1.1、複製Jmeter Plugin(jp@gc)、Jmeter測試計畫，之後自動在每個EC2上啟動相同的測試計畫進行壓力測試，執行完畢後會將測試結果從每個EC2上回收並彙總，最後關閉所有EC2結束整個測試

## Getting Started

### 事前準備

* AWS EC2帳號
* AWS CLI
* 測試計畫必須含有Generate Summary Results Listener(產生總計結果)

### Setup

 1. 複製專案到自己的位置
 2. 解壓縮 `example-project.zip` 到專案底下
 3. 修改 `jmeter-ec2.properties` 裡面的參數為自己的環境
 4. 記得要有AWS CLI SSH 用的 pem 檔
 5. 複製Jmeter jmx 測試計畫到 jmx 資料夾底下，並且更名為專案名稱
 6. 複製需要的 plugin 到 plugins 資料夾底下(已預先放進常用的plugin ex. jp@gc)
 7. 準備好之後在 CLI 輸入 `count="1" ./path/to/jmeter-ec2.sh` 啟動壓測，count 數量為 EC2 啟動數量，EC2的啟動數量取決於帳號裡的限制，EC2規格也有相對應的限制，詳細請到自己的AWS EC2帳號裡查看EC2 limit
 8. 用 Jmeter plugin jp@gc 的 Ultimate Thread Group 及 Throughput Shaping Timer 並設定 100 threads, 100 req/s 即可使用1台EC2達成穩定的 100 並發，10台EC2即可達成 1000 並發，使用其他的Thread Group有可能無法達成穩定的並發，此時請自行斟酌使用(原生的Thread Group會被拆分Thread，請自行查看jmeter-ec2.sh的實作)
 9. 可用 Grafana + influxDB + Jmeter Backend Listener 進行監控，或只將測試結果用Jmeter jp@gc listener打開即可得到圖表結果

### 進階用法

請參照原始的README.md

### 額外資料

The source repository is at:
  [https://github.com/oliverlloyd/jmeter-ec2](https://github.com/oliverlloyd/jmeter-ec2)

參考資料:
  [http://sj82516-blog.logdown.com/posts/4738430/-use-jmeter-jmeter-ec2-to-do-stress-tests]
