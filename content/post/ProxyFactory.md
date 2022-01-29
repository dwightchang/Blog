---
title: "ProxyFactory"
date: 2022-01-22T22:46:03+08:00
draft: false
---

我們經常需要在系統中，呼叫別人的服務，在撰寫程式碼時，要先理解該API服務介面的規格，例如傳送的Json參數內容，參數的型態、預設值、必填參數等等。呼叫完成後，同樣要解讀收到的Json內容，檢查呼叫的結果是否成功。而這個服務可能在另一個系統上也會使用，這時，又要再一次地重覆這個工作。

如果將呼叫 API 服務的程式，包裝成模組，甚至讓 API 呼叫，就像呼叫某個物件的方法，那將會變得很方便。

首先從以下的 API 服務 `/Order/FindOrder` 呼叫開始
```r
curl --location --request POST 'http://localhost:63392/Order/FindOrder' \
--header 'Token: x12345678' \
--data-raw '{
    "OrderSn": 1
}'
```
發送內容包含一個訂單編號(OrderSn)，以及授權呼叫的 token，用物件包裝如下
```c#
public class FindOrderReq
{
    public int OrderSn { get; set; }        
}
```
API回覆的Json內容如下
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
將此Json格式用物件包裝起來
```c#
public class FindOrderResp
{
    public bool Success { get; set; }
    public string Message { get; set; }
    public FindOrderResult Result { get; set; }
}

public class FindOrderResult
{
    public int Sn { get; set; }
    public int CustomerNo { get; set; }
    public int ProductNo { get; set; }
    public int Quantity { get; set; }
    public DateTime CreatedTime { get; set; }
}
```
還需要一個物件來執行呼叫動作
```c#
public class OrderProxy : ProxyBase
{
    public OrderProxy(string ApiRootUrl, string ApiToken): base(ApiRootUrl, ApiToken)
    {
        
    }

    public FindOrderResult FindOrder(FindOrderReq req)
    {
        var data = new ApiRequestData()
        {
            apiPath = "Order/FindOrder",
            contentType = ApiRequestData.ContentType.application_json,
            method = ApiRequestData.Method.POST,
            postData = JsonConvert.SerializeObject(req)
        };

        var resp = InvokeApi<FindOrderResp>(data);

        if (!resp.Success)
        {
            throw new Exception(JsonConvert.SerializeObject(resp));
        }

        return resp.Result;
    }
}
```
在第3行建構式中，傳入該服務的 url 位置，以及授權呼叫該服務的 token。

第8行`FindOrder()`是提供給呼叫端呼叫`/Order/FindOrder` 服務的方法，輸入參數為`FindOrderReq`物件。
`ApiRequestData`物件的內容，則是依該服務介面規格來撰寫，例如呼叫的方法是GET、POST或是PUT等等。

接下來第18行執行呼叫，並將結果反序列化為`FindOrderResp`物件，程式碼如下
```c#
using (var response = (HttpWebResponse)request.GetResponse())
{
    //HttpStatusCode=200才算呼叫成功
    if (response.StatusCode == HttpStatusCode.OK)
    {
        using (var sr = new StreamReader(response.GetResponseStream()))
        {
            value = sr.ReadToEnd();
        }
    }
    else
    {
        throw new Exception(((int)response.StatusCode).ToString() + ":" + response.StatusDescription);
    }

    return JsonConvert.DeserializeObject<T>(value);
}
```
檢查呼叫的結果是不是HTTP狀態碼200，如果不是成功的狀態，則抛出 Exception，同樣的，在剛才`FindOrder()`程式碼收到反序列化的物件後，依該服務的規格定義，當 success=true 時才表示成功，因此再進一步檢查此屬性，若不為 true 則一樣拋出 Exception，如下
```c#
    var resp = InvokeApi<FindOrderResp>(data);

    if (!resp.Success)
    {
        throw new Exception(JsonConvert.SerializeObject(resp));
    }

    return resp.Result;
```
請注意最後回傳的物件是`resp.Result`而不是`resp`，我們再看一次回覆的Json內容，其中 success、message 等屬性，是該 API 服務用來告訴我們呼叫的結果是否成功，對於呼叫端的系統來說，呼叫失敗會收到 Exception 錯誤，成功的話只需要收到訂單查詢的結果即可，也就是`result`的內容。
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
我們再將建構`OrderProxy`物件時所需的 API Url 以及 token 包裝起來
```c#
public class ProxyFactory
{
    public static OrderProxy OrderProxy()
    {
        return new OrderProxy("http://localhost:63392", "f496896f-446f-4a5b-97c4-b6d12f66a22c");
    }
}
```
最後，當呼叫端使用這個`OrderProxy`呼叫服務時，就會像這樣
```c#
var orderData = ProxyFactory.OrderProxy().FindOrder(new ApiProxy.Models.Order.FindOrderReq()
{
    OrderSn = 1
});
```            
如此一來，呼叫端在呼叫此 API 服務時，就像呼叫物件的方法一樣，不用處理 HTTP 以及 API 呼叫相關的細節。
當我們在另一個系統也要呼叫 `/Order/FindOrder` 服務時，只需要將`OrderProxy`這個套件拿到另一個系統上使用即可。