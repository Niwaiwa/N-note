###### tags: `EFK`

# EFK 升級方式 (Elasticsearch, Kibana)

既有方法不依賴fluentd的plugin版本, 但如果使用了ILM功能(elasticsearch-xpack), 則有可能有影響
升級時還是要參照官方升級說明, 檢查是否有使用不相容或需先關閉的功能(ex. 機器學習)

## 注意事項

如果升級的是正式區, 因怕會有失敗導致fluentd超時並停止, 進而導致正式區服務連不到fluentd而停止  
因此正式區升級最好以新機器去建立EFK, 並再完成之後在將正式區服務的fluentd位置改去新位置(上版熱更新調整)  

## 既有升級方式(~20210225)

1. 停止&刪除 容器&volume
2. 各機器啟動新版本elasticsearch
3. 啟動新版本kibana

## 現行升級方式(20210225~)

1. 停止&刪除 容器
2. 各機器啟動新版本elasticsearch
3. 刪除es裡的.kibana相關的全部index
4. 啟動新版本kibana
