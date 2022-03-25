###### tags: `Third Party API`

# Google API

測試串接Google API

## Google Drive API

基本照官方步驟處理, 但有部分不好理解

1. 通常先進入google develop console
    * https://console.developers.google.com/apis/dashboard
2. 到**程式庫**找到Google Drive API 並啟用
3. 到OAuth同意畫面**編輯應用程式**, 並新增使用者(Gmail帳號)
4. 之後到**憑證**建立**OAuth用戶端ID**, 並設定**已授權的重新導向 URI**
    * 須設定好redirect URI才會正常驗證成功, 同時該domain是之後測試時訪問跟重新導向用的domain
5. 正常到這邊就可以使用設定的domain做訪問, 可參考官方Flask server範例
    * https://developers.google.com/identity/protocols/oauth2/web-server#python_5

* Drive API文件
    *  https://developers.google.com/drive/api/v3/quickstart/python