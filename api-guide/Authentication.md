身份驗證是將傳入請求與一組標識憑據（例如請求來自的用戶或其簽名的令牌）相關聯的機制。然後，權限和限制策略可以使用這些憑據來確定是否應該允許該請求。
REST框架提供了許多開箱即用的身份驗證方案，還允許您實現自定義方案。
認證總是在view最開始的時候執行，在permission與throttling check發生之前，並在任何程式碼被允許處理之前。
`request.user` property 通常被設定為`contrib.auth` package's `User` class的實例。
`request.auth` property 通常作為任何額外的認證資訊，例如它可能用來呈現由request 所簽認的一個認證令符。

**注意:** 別忘了，認證本身並不能阻止一個連入request，它只是簡單的識別request的憑證。
如何為你的API設定權限策略，請見權限文件。

## 認證如何被決定
認證綱要總是被定義成類別的串列， REST framework將嘗試使用列表中的每個類進行認證，並在認證成功後將第一個類別的返回值設定為`request.user`與`request.auth`。

如果沒有類別認證，`request.user`將成為`django.contrib.auth.models.AnonymousUser`的實例，而且`request.auth`將設為`none`。
未經認證的request的`request.user`與 `request.auth`的值，可以被`UNAUTHENTICATED_USER` and `UNAUTHENTICATED_TOKEN` settings改變。

## 設定認證網要

預設的認證網要，可能是全域的，使用`DEFAULT_AUTHENTICATION_CLASSES` setting, 例如
<!--TODO: 這裡有更新過，原本是tuple，現在是list-->
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    )
}
```
你也可以在per-view or per-viewset基礎上設定認證網要，使用`APIView` CBV. 

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    authentication_classes = (SessionAuthentication, BasicAuthentication)
    permission_classes = (IsAuthenticated,)

    def get(self, request, format=None):
        content = {
            'user': unicode(request.user),  # `django.contrib.auth.User` instance.
            'auth': unicode(request.auth),  # None
        }
        return Response(content)
```

或著，假如你正在使用`@api_view` decorator的FBV. 

```python
@api_view(['GET'])
@authentication_classes((SessionAuthentication, BasicAuthentication))
@permission_classes((IsAuthenticated,))
def example_view(request, format=None):
    content = {
        'user': unicode(request.user),  # `django.contrib.auth.User` instance.
        'auth': unicode(request.auth),  # None
    }
    return Response(content)
```
## 未認證及回應禁止
當未經身份驗證的請求被拒絕時，有兩個不同的錯誤代碼可能是合適的。

    * HTTP 401 Unauthorized
    * HTTP 403 Permission Denied

HTTP 401響應必須始終包含一個WWW-Authenticate標頭，指示客戶端如何進行身份驗證。HTTP 403響應不包括WWW-Authenticate標頭。

The kind of response that will be used depends on the authentication scheme. Although multiple authentication schemes may be in use, only one scheme may be used to determine the type of response. **The first authentication class set on the view is used when determining the type of response.**

Note that when a request may successfully authenticate, but still be denied permission to perform the request, in which case a `403 Permission Denied` response will always be used, regardless of the authentication scheme.

## Apache mod_wsgi specific configuration
Note that if deploying to Apache using mod_wsgi, the authorization header is not passed through to a WSGI application by default, as it is assumed that authentication will be handled by Apache, rather than at an application level.

If you are deploying to Apache, and using any non-session based authentication, you will need to explicitly configure mod_wsgi to pass the required headers through to the application. This can be done by specifying the `WSGIPassAuthorization` directive in the appropriate context and setting it to `'On'`.

```
# this can go in either server config, virtual host, directory or .htaccess
WSGIPassAuthorization On
```

# API Reference

## BasicAuthentication
使用`HTTP BasicAuthentication`...僅適用於測試

## TokenAuthentication 
此認證網要使一個簡單的token-based HTTP Authentication scheme，適用於主從式設定，像是原生桌面與移動客端。
為了使用`TokenAuthentication`網要，你將需要[設置認證類別](http://www.django-rest-framework.org/api-guide/authentication/#setting-the-authentication-scheme)以包入`TokenAuthentication`及額外包入的`rest_framework.authtoken`到你的`INSTALLED_APPS` setting:

```python
INSTALLED_APPS = (
    ...
    'rest_framework.authtoken'
)
```
----
**注意:** 確定執行`manage.py migrate` 在你改變設定之後，`rest_framework.auth` app提供django database migrations

----
你也需要為你的使用者建立tokens

```python
from rest_framework.authtoken.models import Token

token = Token.objects.create(user=...)
print token.key
```
<!--上述例子還差一些東西
from django.contrib.auth.models import User
users=User.objects.all()
token=Token.objects.create(user=users[0])
print(token.key)

-->
<!--token 應該是僅作為簡便認證使用，而不該用於需要嚴謹認證的地方，比如發言，提問，但如果要進行交易則應該要進一步的認證。-->
<!--注意認證綱要也要增加TokenAuthentication的部份，否則報401錯誤，請看前面"設定認證網要"-->
客端進行認證，token key應放在`Authentication` HTTP header之中，此key 應前綴字串"Token" 並有空白鍵分隔二字串，例如
<!--記得http header也要放入Content-Type: application/json-->
```
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```
----
**注意:** 假如你想要在header使用不同的keyword，像是`Bearer`，簡單地子類`TokenAuthentication`並設置`keyword`類別變數。

----
假如成功認證，`TokenAuthentication`提供下列憑證。
* `request.user` 將會是一個Django `User`的實例。
* `request.auth` 將會是一個`rest_framework.authtoken.models.Token`的實例。

末通過認證的回應為拒絕權限將導致一個`HTTP 401 unauthorized` response並帶一個適當的`WWW-Authenticate` header, 例如

```
WWW-Authenticate: Token
```

`curl`命令列工具可用於測試 token authenticated APIs，例如

```
curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'
```
----
**注意:** 假如你在生產環境使用`TokenAuthentication` 你一定要確定你的api僅能通過`https`使用。
<!--很明顯地，這是由於token 是透過GET 來使用的，-->
----

### 產生token -藉由使用signals
假如你想要每一個使用者都有一個自動產生的Token，你可以簡單的捕捉使用者的`post_save`訊號。
<!--所以這可以在建立使用者時，自動地給使用者設定token -->
```python
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)
```
你將需要確定你將此段代碼放在一個已安裝的`models.py`模組中，或其他位置而那是Django 啟動時就載入的。
假如你已建立一些使用者，你可以為所有既存的使用都產生tokens，用如下方法。

```python
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token

for user in User.objects.all():
    Token.objects.get_or_create(user=user)
```

### 產生令符 -藉由曝露一個api endpoint

當使用`TokenAuthentication` 你可能想要提供一個機制給客端，使得到一個token，由給定的username與password，REST framework 提供一個內建的view以提供這個行為。在你的URLconf中加上`obtain_auth_token` view，以使用它。

```python
from rest_framework.authtoken import views
urlpatterns += [
    url(r'^api-token-auth/', views.obtain_auth_token)
]
```

注意這個URL pattern可以在任何地方使用。

`obtain_auth_token` view將返回一個JSON response當一個有效的username與password欄位被POSTed到view中，藉由form data或JSON. 

```json
{ 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }
```
注意預設的`obtain_auth_token` view 明確地使用JSON requests與responses，而不是使用在你的settings中預設的renderer與parser classes。假如你需要`obtain_auth_token` view的客製化版本，你可以藉如overriding `ObtainAuthToken` view class，並且將之用在你的url conf中。

預設中，`obtain_auth_token`並未套用任何permissions或throttling，假如你想要應用任何throttling，你將需要override view class, 並使用`thorttle_classes` attribute含入它們。
[1060226](http://www.django-rest-framework.org/api-guide/authentication/#authentication)

### 藉由 Django admin
透過admin 介面可手動建立Tokens，以防萬一你正使用一個大user base，我們建議...

## SessionAuthentication

本認證網要使用Django 的預設session backend 作為認證，Session Authentication 是適合給AJAX客端用的，這會執行同樣的session context在你的website上。
假如成功認證，`SessionAuthentication` 提供下列憑證。
    * `request.user` 將會是一個Django `User`物件
    * `request.auth` 將會是`None`

未通過認證將回應拒絕權限，並導`HTTP 403 Forbidden` 回應。
假如你正在使用一個帶AJAX style API 的SessionAuthentication，你將需要確認你有帶入一個有效的CSRF token 給任何不安全的 'HTTP'方法呼叫，例如`PUT` `PATCH` `POST` `DELETE`，見Django CSRF 文件以了解細節。

**警告** 總是在登入頁面，使用Django的標準登入view，這將確保你的登入views是有適當保護的。

CSRF 驗證在REST framework 工作與在標準的Django中會有些許差異，由於需要支持兩者session與non-session based 在同一個views上，這意味著僅有authenticatd requests要有CSRF tokens, 而匿名requests可能不帶CSRF token。這行為並不適合用來登入views，因為這總是要有CSRF驗證。

## RemoteUserAuthentication
這種身份驗證方案允許您將身份驗證委託給設置`REMOTE_USER` 環境變量的Web服務器。

要使用它，你必須有`django.contrib.auth.backends.RemoteUserBackend`（或者一個子類）在你的 `AUTHENTICATION_BACKENDS`設置中。默認情況下，為不存在的用戶名`RemoteUserBackend`創建`User`對象。要改變這個和其他行為，請參考 Django文檔。

如果成功通過身份驗證，則`RemoteUserAuthentication`提供以下憑據：

`request.user`將是一個Django `User`實例。
`request.auth將會None`。
有關配置身份驗證方法的信息，請參閱您的Web服務器的文檔，例如：

Apache身份驗證方法
NGINX（限制訪問）
## Custom authentication
略

## Example
以下示例將以名為'X_USERNAME'的自定義請求標頭中的用戶名給出的用戶身份驗證任何傳入請求。


```python
from django.contrib.auth.models import User
from rest_framework import authentication
from rest_framework import exceptions

class ExampleAuthentication(authentication.BaseAuthentication):
    def authenticate(self, request):
        username = request.META.get('X_USERNAME')
        if not username:
            return None

        try:
            user = User.objects.get(username=username)
        except User.DoesNotExist:
            raise exceptions.AuthenticationFailed('No such user')

        return (user, None)
```
## 第三方套件
### drfpasswordless
[drfpasswordless](https://github.com/aaronn/django-rest-framework-passwordless)為Django REST Framework自己的TokenAuthentication方案添加了無中心支持（Medium，Square Cash的靈感）。用戶登錄並註冊發送給聯繫點的令牌，如電子郵件地址或手機號碼。
這個有點像二因子認證，只是認證位置是e-mail或簡訊，而且不用密碼。認證有效時間為15分鐘。然後取得 drf的Token，token可以用於drf的token 認證。
