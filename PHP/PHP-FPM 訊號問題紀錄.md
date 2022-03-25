###### tags: `PHP` `問題記錄`

# PHP-FPM 訊號問題紀錄

## 熱更新時, PHP-FPM未等待請求做完就直接停止服務的問題

### 問題敘述

狀況: 正式區服務處理轉帳請求, 做到一半整個請求沒任何錯誤訊息就被中斷, 是否是什麼原因導致做到一半被kill掉?  
問題: 排除其他原因, 被中斷當下剛好在上版, 因此熱更新上版是否是導致運行中的服務的請求沒做完就被中斷的原因?  

### 研究

1. docker swarm熱更新方式, 直接送出SIGTERM, 並等待stop_grace_period 時間, 超過後還沒有停止就會送出SIGKILL強制停止  
2. docker stop原理, 直接送出SIGTERM, 並等待stop_grace_period 時間, 超過後還沒有停止就會送出SIGKILL強制停止  
3. 根據上述原理, 使用docker stop來做訊號測試
4. PHP-FPM使用master-worker架構來管理服務, 有關的設定值如下
    ```
    php.ini
    max_input_time = 60  # 讀取輸入資料的最大時間, 基本沒機會觸發
    max_execution_time = 30  # 此為CPU執行時間, sleep或db訪問等為io 等待, 不計入CPU執行時間

    php-fpm.conf
    process_control_timeout = 30  # 此為接收到SIGQUIT 之後的等待時間, 超過後送SIGTERM 並等待1秒(實測1秒內worker就會自動停止), 若超過1秒就送SIGKILL 強制中止
    ```
5. 當docker stop 發出SIGTERM訊號, PHP-FPM master收到後會送出SIGQUIT去中止worker, 若超過process_control_timeout的時間, 會改送出SIGTERM去停止worker, 1秒內SIGTERM未停止則會送出SIGKILL 強制中止


**根據研究分析當前設定並實測**:  

有設定max_input_time 跟 max_execution_time, 沒設定process_control_timeout, 實測docker stop送出SIGTERM訊號, PHP-FPM收到會立即送出SIGTERM去停止所有worker, 此時會看到請求未做完即被強制停止  

**若加上process_control_timeout設定**:  

實測docker stop送出SIGTERM訊號, PHP-FPM收到會立即送出SIGQUIT去停止所有worker, master此時會等待process_control_timeout時間, 此時

1. 若超過process_control_timeout時間worker還未執行完成, master會送出SIGTERM去停止所有worker, 此時會看到請求未做完即被強制停止  
2. 若在process_control_timeout時間內worker執行完成, worker停止後, master會普通的停止服務  

### 結論

根據上述研究跟實測, 出現請求無錯誤及被中斷的情況, 跟PHP-FPM未設定process_control_timeout參數有關, 因為設定該參數導致熱更新時服務會直接被停止, 導致問題  
解決方法則根據上述研究, 在PHP-FPM設定檔中加入process_control_timeout參數, 即可以避免該問題

## 簡單版

問題: 正式區服務處理轉帳請求, 做到一半整個請求沒任何錯誤訊息就被中斷  
原因: 因使用熱更新, 送出停止容器的訊號SIGTERM時, PHP-FPM未做任何等待就直接停止了服務, 而PHP-FPM直接停止服務的原因, 是因為設定檔沒有設定process_control_timeout參數, 因此預設收到停止訊號就會直接停止服務

## 參考資料

* https://segmentfault.com/a/1190000013675862
* https://learnku.com/articles/31303
