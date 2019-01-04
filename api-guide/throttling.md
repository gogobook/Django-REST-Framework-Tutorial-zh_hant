# Throttling
Throttling類似於權限，它決定是否應該授權請求。Throttles表示一個臨時狀態，用於控制客戶端對API的請求率。

與權限一樣，可以使用多個throttles。您的API可能對未經身份驗證的請求有限制性的節流，對經過身份驗證的請求的限制也比較少。

如果您需要對API的不同部分施加不同的約束，那麼可能需要使用多個throttles的另一個場景是，由於某些服務特別佔用資源。

如果您想要同時施加節流率和持續的節流率，也可以使用多個節流。例如，您可能希望將用戶限制為每分鐘最多60個請求，以及每天1000個請求。

Throttles並不僅僅指限速請求。例如，存儲服務可能還需要對帶寬進行限制，而付費數據服務可能希望對正在訪問的某些記錄進行限制。

## 節流是如何決定
與權限和身份驗證一樣，REST framework中的節流始終被定義為類的列表。

在運行視圖的主體之前，檢查列表中的每個節流。如果任何節流檢查失敗，則有例外。將會提出`exceptions.Throttled`的異常，並且視圖的主體不會運行。

## 設置節流政策
預設的節流策略可以使用`DEFAULT_THROTTLE_CLASSES`和`default_throttle_rate`設置全局設置。為例。

```py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ),
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    }
}
```
在`default_throttle_rate`中使用的速率描述可能包括秒、分鐘、小時或日作為節流週期。

您還可以使用`APIView`類的視圖，在每個視圖或每個viewset的基礎上設置節流策略。

Or, if you're using the @api_view decorator with function based views.

```py
@api_view(['GET'])
@throttle_classes([UserRateThrottle])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```
## 客戶是如何確定的
`X-Forwarded-For`和`REMOTE_ADDR WSGI`變量，用於唯一地標識客戶端IP地址以進行調節。如`X-Forwarded-For`，那麼它將被使用，否則將使用來自WSGI環境的`REMOTE_ADDR`變量的值。

如果您需要嚴格地識別唯一的客戶端IP地址，您需要首先配置應用程序代理的數量，通過設置num_proxy設置來運行該API。該設置應該為零個或多個整數。如果將其設置為非零，那麼客戶端IP將被標識為x -轉發中的最後一個IP地址——對於頭，一旦任何應用程序代理IP地址被排除在外。如果設置為0，那麼REMOTE_ADDR值將始終被用作標識IP地址。

重要的是要理解，如果您配置了num_proxy設置，那麼在一個唯一的NAT的網關後面的所有客戶端將被當作一個單獨的客戶機來對待。

關於`X-Forwarded-For`如何工作，以及如何識別遠程客戶端IP的進一步上下文可以在這裡找到。

## Setting up the cache
REST framework提供的節流類使用Django的緩存後端。您應該確保設置了適當的緩存設置。對於簡單的設置，`LocMemCache`後端的預設值應該是可以的。有關更多細節，請參見Django的緩存文檔。

如果您需要使用除「預設」之外的緩存，您可以通過創建一個自定義節流類和設置緩存屬性來實現。例如:
```py
class CustomAnonRateThrottle(AnonRateThrottle):
    cache = get_cache('alternate')
```
You'll need to remember to also set your custom throttle class in the 'DEFAULT_THROTTLE_CLASSES' settings key, or using the throttle_classes view attribute.

# API Reference
##AnonRateThrottle
AnonRateThrottle只會限制未經身份驗證的用戶。傳入請求的IP地址被用來生成一個惟一的鍵來抑制。

允許的請求率由下面的一個(按優先順序)確定。

該類的rate屬性，可以通過重寫AnonRateThrottle並設置屬性來提供。
DEFAULT_THROTTLE_RATES(「不久」)設置。

如果您想要限制來自未知來源的請求率，那麼AnonRateThrottle是合適的。
## UserRateThrottle

UserRateThrottle將通過API將用戶限制到給定的請求率。用戶id被用來生成一個惟一的鍵來抑制。未經身份驗證的請求將會返回到使用傳入請求的IP地址，以生成一個惟一的鍵來阻止。

允許的請求率由下面的一個(按優先順序)確定。

類的速率屬性，可以通過覆蓋UserRateThrottle和設置屬性來提供。
DEFAULT_THROTTLE_RATES(「用戶」)。

一個API可能同時具有多個UserRateThrottles。為此，重寫UserRateThrottle並為每個類設置一個惟一的「範圍」。

例如，可以通過使用以下類來實現多個用戶節流率。
```py
class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'

```
...and the following settings.
```py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'example.throttles.BurstRateThrottle',
        'example.throttles.SustainedRateThrottle'
    ),
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day'
    }
}
```
如果您想要對每個用戶進行簡單的全域速率限制，那麼`UserRateThrottle`是合適的。

## ScopedRateThrottle
`ScopedRateThrottle`類可以用來限制對API特定部分的訪問。只有當被訪問的視圖包含`.throttle_scope`屬性時，才會應用這個節流。然後，通過使用唯一的用戶id或IP地址將請求的「範圍」連接起來，從而形成唯一的節流鍵。

允許的請求率是由`default_throttle_rate`設置的，它使用一個來自請求「範圍」的鍵來設置。

例如，給定以下視圖…
```py
class ContactListView(APIView):
    throttle_scope = 'contacts'
    ...

class ContactDetailView(APIView):
    throttle_scope = 'contacts'
    ...

class UploadView(APIView):
    throttle_scope = 'uploads'
    ...
```
...and the following settings.
```py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'rest_framework.throttling.ScopedRateThrottle',
    ),
    'DEFAULT_THROTTLE_RATES': {
        'contacts': '1000/day',
        'uploads': '20/day'
    }
}
```
用戶對`ContactListView`或`ContactDetailView`的請求將被限制為每天1000個請求。用戶對`UploadView`的請求將被限制為每天20個請求。

## Custom throttles
要創建自定義節流閥，請重寫`BaseThrottle`並實現`.allow_request(self, request, view)`。如果請求允許，則該方法應該返回True，否則將返回False。

另外，您也可以重寫.wait()方法。如果實現，.wait()應該在嘗試下一個請求之前返回一個建議的秒數，或者沒有。只有.allow_request()之前返回False，才會呼叫.wait()方法。

如果.wait()方法被實現，請求被抑制，那麼在response 中將包含一個重試頭。

### 例子
下面是一個速率節流閥的例子，它將隨機地控制每10個請求中的1個。
```py
import random

class RandomRateThrottle(throttling.BaseThrottle):
    def allow_request(self, request, view):
        return random.randint(1, 10) != 1
```