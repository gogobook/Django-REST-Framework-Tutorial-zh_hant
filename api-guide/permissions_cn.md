# 權限
與 authentication 和 throttling 一起，permission 決定是應該接受還是拒絕訪問請求。
權限檢查總是在視圖的最開始處運行，在任何其他代碼被允許進行之前。權限檢查通常會使用 request.user 和 request.auth 屬性中的認證信息來確定是否允許傳入請求。
權限用於授予或拒絕不同類別的用戶訪問 API 的不同部分。
最簡單的權限是允許通過身份驗證的用戶訪問，並拒絕未經身份驗證的用戶訪問。這對應於 REST framework 中的 IsAuthenticated 類。
稍微寬鬆的權限會允許通過身份驗證的用戶完全訪問，而未通過身份驗證的用戶只能進行只讀訪問。這對應於 REST framework 中的 IsAuthenticatedOrReadOnly 類。

## 如何確定權限
REST framework 中的權限總是被定義為權限類的列表。
在運行視圖的主體之前，檢查列表中的每個權限。如果任何權限檢查失敗，則會引發 exceptions.PermissionDenied 或 exceptions.NotAuthenticated 異常，並且視圖的主體不會再運行。
當權限檢查失敗時，根據以下規則，將返回 「403 Forbidden」 或 「401 Unauthorized」 response ：

該請求已成功通過身份驗證，但權限被拒絕。 — 將返回 403 Forbidden response 。
該請求未成功通過身份驗證，並且最高優先級身份驗證類未添加 WWW-Authenticate header。— 將返回 403 Forbidden response 。
該請求未成功通過身份驗證，不過最高優先級身份驗證類添加了 WWW-Authenticate header。— 返回一個 HTTP 401 Unauthorized response ，並會帶上一個適當的 WWW-Authenticate header。

## 對象級權限
REST framework 權限還支持對象級權限。對象級權限用於確定是否允許用戶對特定對象進行操作，該特定對象通常是指模型實例。
.get_object() 被調用時，對象級權限由 REST framework 的通用視圖執行。與視圖級權限一樣，如果用戶不被允許對給定對象進行操作，則會引發 exceptions.PermissionDenied 異常。
如果您正在編寫自己的視圖並希望強制執行對象級權限，或者如果您在通用視圖上重寫了 get_object 方法，那麼將需要顯式地在你檢索該對象時調用 .check_object_permissions(request, obj) 方法。
這將引發 PermissionDenied 或 NotAuthenticated 異常，或者只是在視圖具有適當的權限時才返回。
例如：
def get_object(self):
    obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
    self.check_object_permissions(self.request, obj)
    return obj
```
對象級權限的限制
出於性能原因，通用視圖在返回對象列表時不會自動將對象級權限應用於查詢集中的每個實例。
通常，當您使用對象級權限時，您還需要適當地過濾查詢集，以確保用戶只能看到他們被允許查看的實例。
設置權限策略
默認權限策略可以使用 DEFAULT_PERMISSION_CLASSES setting 全局設置。例如。
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}
```
如果未指定，則此設置默認為允許無限制訪問：
'DEFAULT_PERMISSION_CLASSES': (
   'rest_framework.permissions.AllowAny',
)
```
您還可以在基於 APIView 類的視圖上設置身份驗證策略。
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    permission_classes = (IsAuthenticated,)

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```
或者在基於 @api_view 裝飾器的函數視圖上設置。
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@permission_classes((IsAuthenticated, ))
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```
注意： 當你通過類屬性或裝飾器設置新的權限類時，settings.py 文件中的默認設置會被忽略。

API 參考
AllowAny
AllowAny 權限類將允許不受限制的訪問，而不管該請求是否已通過身份驗證或未經身份驗證。
也不一定非要用此權限，可以通過為權限設置空列表或元組來實現相同的結果，但是你會發現，使用此權限使意圖更加清晰。
IsAuthenticated
IsAuthenticated 權限類將拒絕任何未通過身份驗證的用戶的訪問。
如果你希望 API 只能由註冊用戶訪問，則可以使用此權限。
IsAdminUser
IsAdminUser 權限僅允許 user.is_staff 為 True 用戶訪問，其他任何用戶都將被拒絕。
如果你希望 API 只能被部分受信任的管理員訪問，則可以使用此權限。
IsAuthenticatedOrReadOnly
IsAuthenticatedOrReadOnly 允許通過身份驗證的用戶執行任何請求。未通過身份驗證的用戶只能請求 「安全」 的方法： GET， HEAD 或 OPTIONS。
如果你希望 API 允許匿名用戶擁有讀取權限，並且只允許對已通過身份驗證的用戶執行寫入權限，則可以使用此權限。
DjangoModelPermissions
此權限類與 Django 的標準 django.contrib.auth 模型權限綁定。此權限只能應用於具有 .queryset 屬性集的視圖。只有在用戶通過身份驗證並分配了相關模型權限的情況下，才有權限訪問。

POST 請求要求用戶在模型上具有 add 權限。
PUT 和 PATCH 請求要求用戶在模型上具有 change 權限。
DELETE 請求要求用戶在模型上具有 delete 權限。

默認行為也可以被重寫以支持自定義模型權限。例如，你可能想要包含 GET 請求的 view 模型權限。
要自定義模型權限，請繼承 DjangoModelPermissions 並設置 .perms_map 屬性。有關詳細信息，請參閱源代碼。
使用不包含 queryset 屬性的視圖。
如果你將此權限與重寫 get_queryset() 方法的視圖一起使用，則視圖上可能沒有 queryset 屬性。在這種情況下，我們建議使用 sentinel 查詢集標記視圖，以便此類可以確定所需的權限。例如：
queryset = User.objects.none()  # Required for DjangoModelPermissions
```DjangoModelPermissionsOrAnonReadOnly
與 DjangoModelPermissions 類似，但也允許未經身份驗證的用戶對 API 進行只讀訪問。
DjangoObjectPermissions
該權限類與 Django 的標準對象權限框架綁定，該框架允許對每個模型對象進行權限驗證。為了使用此權限類，你還需要添加支持對象級權限的權限後端，例如 django-guardian。
與 DjangoModelPermissions 一樣，此權限只能應用於具有 .queryset 屬性或 .get_queryset() 方法的視圖。只有在用戶通過身份驗證並且具有相關的每個對象權限和相關的模型權限後，才有權限訪問。

POST 請求要求用戶對模型實例具有 add 權限。
PUT 和 PATCH 請求要求用戶對模型實例具有 change 權限。
DELETE 請求要求用戶對模型實例具有 delete 權限。

請注意，DjangoObjectPermissions 不需要 django-guardian 軟件包，並且同樣支持其他對象級別的後端。
與 DjangoModelPermissions 一樣，你可以通過繼承 DjangoObjectPermissions 並設置 .perms_map 屬性來自定義模型權限。有關詳細信息，請參閱源代碼。

注意: 如果你需要獲取 GET，HEAD 和 OPTIONS 請求的對象級 view 權限，則還需要考慮添加 DjangoObjectPermissionsFilter 類，以確保列表端點只返回包含用戶具有查看權限的對象的結果。


自定義權限
要實現自定義權限，請繼承 BasePermission 並實現以下方法中的一個或兩個：

.has_permission(self, request, view)
.has_object_permission(self, request, view, obj)

如果請求被授予訪問權限，則方法應該返回 True，否則返回 False。
如果你需要測試一個請求是一個讀操作還是一個寫操作，你應該根據常量 SAFE_METHODS 檢查請求方法， SAFE_METHODS 是一個包含 'GET'，'OPTIONS' 和 'HEAD' 的元組。例如：
if request.method in permissions.SAFE_METHODS:
    # Check permissions for read-only request
else:
    # Check permissions for write request
```
Note: 只有在視圖級別 has_permission 檢查已通過時才會調用實例級別的 has_object_permission 方法。還要注意，為了運行實例級檢查，視圖代碼應該顯式調用 .check_object_permissions(request, obj)。如果你使用的是通用視圖，那麼默認情況下會為您處理。（基於函數的視圖將需要明確檢查對象權限，在失敗時引發 PermissionDenied。）

如果測試失敗，自定義權限將引發 PermissionDenied 異常。要更改與異常相關的錯誤消息，請直接在你的自定義權限上實現 message 屬性。否則將使用 PermissionDenied 的 default_detail 屬性。
from rest_framework import permissions

class CustomerAccessPermission(permissions.BasePermission):
    message = 'Adding customers not allowed.'

    def has_permission(self, request, view):
         ...
```舉個栗子
以下是一個權限類的示例，該權限類將傳入請求的 IP 地址與黑名單進行比對，並在 IP 被列入黑名單時拒絕該請求。
from rest_framework import permissions

class BlacklistPermission(permissions.BasePermission):
    """
    Global permission check for blacklisted IPs.
    """

    def has_permission(self, request, view):
        ip_addr = request.META['REMOTE_ADDR']
        blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
        return not blacklisted
```除了針對所有傳入請求運行的全局權限，還可以創建對象級權限，這些權限僅針對影響特定對象實例的操作執行。例如：
class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission to only allow owners of an object to edit it.
    Assumes the model instance has an `owner` attribute.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Instance must have an attribute named `owner`.
        return obj.owner == request.user
```請注意，通用視圖將檢查適當的對象級權限，但如果你正在編寫自己的自定義視圖，則需要確保檢查自己的對象級權限。您可以通過在擁有對象實例後從視圖中調用 self.check_object_permissions(request, obj) 來完成此操作。如果任何對象級權限檢查失敗，此調用將引發適當的 APIException，否則將簡單地返回。
另請注意，通用視圖將僅檢查單個模型實例的視圖的對象級權限。如果你需要列表視圖的對象級過濾，則需要單獨過濾查詢集。

作者：wcode
链接：https://juejin.im/post/5aab380bf265da239a5f8d8c
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。