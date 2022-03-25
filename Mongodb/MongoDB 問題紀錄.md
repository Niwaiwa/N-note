###### tags: `MongoDB`

# MongoDB 問題紀錄

## mongos無回應

根據查詢, mongod跟mongos都無任何異常LOG監控也正常, 只在mongos查詢到下面的錯誤訊息, 但看不出來問題發生原因

    DBException handling request, closing client connection: ClientDisconnect: operation was interrupted

### 異常情況

* mongod 都正常, 監控顯示無任何異常
* mongos 無反應, 無法建立連接, 無法使用mongo shell進入
* log無異常只出現下列訊息

    DBException handling request, closing client connection: ClientDisconnect: operation was interrupted

### 原因

* 查看原因為下列官方BUG回報, 原因發生在config server

    https://jira.mongodb.org/browse/SERVER-52654

* 該BUG發生原因為config server內有一個signing keys更新的功能, 該功能會定期(預設90天)保持一個mongos對mongod連線用的key, 若未過期則mongos可以正常訪問mongod, 反之無法建立連線, 而根據BUG詳細為config server對signing keys更新有個變數類型的BUG導致無法正常更新signing keys導致問題

* 暫定解決方法為問題發生時重啟config server, 或是90天內主動將primary降級切換出去

* 永久解決方法為升級到4.2.12以上的版本

* 正式區未觸發此問題的原因為, 因經常性的網路延遲, 因此config server沒有primary可以持續90天的, 因此未觸發此問題

* 已先請MIS定時詢問RD是否有做該暫定解法的操作, 直到實際升級
