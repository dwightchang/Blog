---
title: "ProxyFactory"
date: 2022-01-22T22:46:03+08:00
draft: false
---

我們經常需要在系統中，呼叫別人的服務，要理解別人的API服務，傳送的Json參數內容，參數型態、預設值、必填參數等等。呼叫完成後，同樣要解讀收到的Json內容，反序列化成物件，檢查呼叫的結果是否成功。而這個服務可能在另一個系統上也會使用，又要再一次地依API規格撰寫發送及解讀API回覆的程式碼。

如果將呼叫別人服務的程式，包裝成模組，甚至讓API呼叫，就像執行某個物件裏的方法，那將會變得很方便。

```r
curl --location --request POST 'http://localhost:63392/Order/FindOrder' \
--header 'Token: x12345678' \
--data-raw '{
    "OrderSn": 1
}'
```

```json
{
    "success": true,
    "message": null,
    "result": {
        "sn": 1,
        "customerNo": 133,
        "productNo": 15,
        "quantity": 1,
        "createdTime": "2020-04-04T15:30:31"
    }
}
```