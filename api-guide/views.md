source:decorators.py views.py

# 基於類的視圖
Django的基於類的視圖是對舊式視圖的脫離。
Django's class-based views are a welcome departure from the old-style views.

- Reinout van Rees

REST框架提供了一個`APIView`類，它是Django View類的子類。

`APIView`類View通過以下方式與常規類不同:

- 傳遞給處理程序方法的請求將是REST框架的`Request`實例，而不是Django的HttpRequest實例。
- 處理程序方法可能會返回REST框架`Response`，而不是Django HttpResponse。該視圖將管理內容協商並在response 上設置正確的渲染器。
- 任何`APIException`例外都將被捕獲並調解為適當的response 。
- 將傳入的請求進行身份驗證，並在將請求分派給處理程序方法之前運行適當的權限和/或限制檢查。

使用`APIView`該類與使用常規`View`類幾乎相同，像往常一樣，傳入的請求被分派到適當的處理程序方法，如.get()或.post()。另外，可以在控制API策略的各個方面的類上設置許多屬性。

例如:
```py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import authentication, permissions
from django.contrib.auth.models import User

class ListUsers(APIView):
    """
    View to list all users in the system.

    * Requires token authentication.
    * Only admin users are able to access this view.
    """
    authentication_classes = (authentication.TokenAuthentication,)
    permission_classes = (permissions.IsAdminUser,)

    def get(self, request, format=None):
        """
        Return a list of all users.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```
注: 完整的方法，屬性，並與Django的REST框架的關係`APIView`，Generic`APIView`各種Mixins，並且Viewsets可以初步複雜。除了此處的文檔之外，Classy Django REST Framework資源還為Django REST Framework的每個基於類的視圖提供了可瀏覽的引用，包括完整的方法和屬性。

## API策略屬性
以下屬性控制API視圖的可插入方面。

.renderer_classes
.parser_classes
.authentication_classes
.throttle_classes
.permission_classes
.content_negotiation_class

## API策略實例化方法
REST框架使用以下方法來實例化各種可插入API策略。您通常不需要覆蓋這些方法。

.get_renderers(self)
.get_parsers(self)
.get_authenticators(self)
.get_throttles(self)
.get_permissions(self)
.get_content_negotiator(self)
.get_exception_handler(self)

## API策略實現方法
在分派到處理程序方法之前調用以下方法。

.check_permissions(self，request)
.check_throttles(self，request)
.perform_content_negotiation(self，request，force = False)

## 派遣方式
視圖的.dispatch()方法直接調用以下方法。這些執行需要之前或調用處理方法，如之後發生的任何動作.get()，.post()，put()，patch()和.delete()。

.initial(self，request，* args，** kwargs)
執行在調用處理程序方法之前需要執行的任何操作。此方法用於強制執行權限和限制，以及執行內容協商。

您通常不需要覆蓋此方法。

.handle_exception(self，exc)
處理程序方法拋出的任何異常都將傳遞給此方法，該方法返回Response實例或重新引發異常。

默認實現處理任何子類rest_framework.exceptions.APIException，以及Django Http404和PermissionDenied異常，並返回適當的錯誤response 。

如果您需要自定義API返回的錯誤response ，則應該對此方法進行子類化。

.initialize_request(self，request，* args，** kwargs)
確保傳遞給handler方法的請求對像是一個實例Request，而不是通常的Django HttpRequest。

您通常不需要覆蓋此方法。

.finalize_response(self，request，response，* args，** kwargs)
確保Response從處理程序方法返回的任何對像都將呈現為正確的內容類型，這由內容協商確定。

您通常不需要覆蓋此方法。

# 基於函數的視圖
說[基於類的視圖]總是優越的解決方案是一個錯誤。

- 尼克科格蘭

REST框架還允許您使用基於常規功能的視圖。它提供了一組簡單的裝飾器，它們包裝基於函數的視圖，以確保它們接收Request(而不是通常的Django HttpRequest)的實例並允許它們返回Response(而不是Django HttpResponse)，並允許您配置請求的處理方式。

## @api_view()
簽名: `@api_view(http_method_names=['GET'])`

此函數的核心是api_view裝飾器，它接收視圖應response 的HTTP方法列表。例如，這就是你如何編寫一個非常簡單的視圖，只需手動返回一些數據:
```py
from rest_framework.decorators import api_view

@api_view()
def hello_world(request):
    return Response({"message": "Hello, world!"})
```
此視圖將使用設置中指定的默認渲染器，解析器，身份驗證類等。

默認情況下，只GET接受方法。其他方法將response “405 Method Not Allowed”。要更改此行為，請指定視圖允許的方法，如下所示:
```py
@api_view(['GET', 'POST'])
def hello_world(request):
    if request.method == 'POST':
        return Response({"message": "Got some data!", "data": request.data})
    return Response({"message": "Hello, world!"})
```
## API策略修飾器
要覆蓋默認設置，REST框架提供了一組可添加到視圖中的其他裝飾器。這些必須在裝飾者之後(下面)@api_view。例如，要創建一個使用限制的視圖以確保它只能由特定用戶每天調用一次，請使用@throttle_classes裝飾器，傳遞一個節流類列表:
```py
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
        rate = '1/day'

@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```
這些裝飾器對應於`APIView`上面描述的子類上設置的屬性。

可用的裝飾器是:

- @renderer_classes(...)
- @parser_classes(...)
- @authentication_classes(...)
- @throttle_classes(...)
- @permission_classes(...)

這些裝飾器中的每一個都接受一個參數，該參數必須是類的列表或元組。

## 查看架構裝飾器
要覆蓋基於函數的視圖的默認模式生成，您可以使用`@schema`裝飾器。這必須在 裝飾者之後(下面)`@api_view`。例如:
```py
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # override view introspection here...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
# 此裝飾器採用Schemas文檔中描述的單個AutoSchema實例，AutoSchema子類實例或ManualSchema實例。您可以傳遞以從模式生成中排除視圖。None

@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Will not appear in schema!"})
```