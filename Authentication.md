認證是一個關連識別證書與連入request的機制，例如當使用者request所帶，或令符所簽用。權限與節流閥政策然後使用這些證書來決定request是否被允許。
REST framework 提供一些即可用的認證綱要，並允許你實作自訂綱要。
認證總是在view最開始的時候執行，在permission與throttling check發生之前，並在任何程式碼被允許處理之前。
`request.user` property 通常被設定給`contrib.auth` package's `User` class的實例。
`request.auth` property 通常作為任何額外的認證資訊，例如它可能用來呈現由request 所簽認的一個認證令符。

**注意:** 別忘了，認證本身並不能阻止一個連入request，它只是簡單的識別request的證書。
如何為你的API設定權限政策，請見權限文件。

## 認證如何被決定
認證綱要總是被定義成類別的串列， REST framework將試著認證串列內的每個類別，並在認證成功後將第一個類別的返回值設定為`request.user`與`request.auth`。

如果沒有類別認證，`request.user`將成為`django.contrib.auth.models.AnonymousUser`的實例，而`request.auth`將設為`none`。
給未認證的request的`request.user`與 `request.auth`的值，可以被`UNAUTHENTICATED_USER` and `UNAUTHENTICATED_TOKEN` settings改變。

## 設定認證網要

預設的認證網要，可能是全域的，使用`DEFAULT_AUTHENTICATION_CLASSES` setting, 例如

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
## 未認及回應禁止
When an unauthenticated request is denied permission there are two different error codes that may be appropriate.

    * HTTP 401 Unauthorized
    * HTTP 403 Permission Denied

HTTP 401 responses must always include a `WWW-Authenticate` header, that instructs the client how to authenticate. HTTP 403 responses do not include the `WWW-Authenticate` header.

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
客端進行認，token key應包含於`Authentication` HTTP header. 此key 應前綴字串"Token" 並有空白鍵分隔二字串，例如

```
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```
----
**注意:** 假如你想要在header使用不同的keyword，像是`Bearer`，簡單地子類`TokenAuthentication`並設置`keyword`類別變數。

----
假如成功認證，`TokenAuthentication`提供下列證書。
* `request.user` 將會是一個Django `User`的實例。
* `request.auth` 將會是一個`rest_framework.authtoken.models.Token`的實例。

末過認證的回應為拒絕權限將導致一個`HTTP 401 unauthorized` response並帶一個適當的`WWW-Authenticate` header, 例如

```
WWW-Authenticate: Token
```

`curl`命令列工具可用於測試 token authenticated APIs，例如

```
curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'
```
----
**注意:** 假如你在生產環境使用`TokenAuthentication` 你一定要確定你的api僅能通過`https`使用。

----

### 產生令符 -藉由使用signals
假如你想要每一個使用者都有一個自動產生的Token，你可以簡單的捕捉使用者的`post_save`訊號。

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
你將需要確定你將此段代碼放在一個已安裝的`models.py`模組中，或其位置而那是Django 啟動時就載入的。
假如你已建立一些使用者，你可以為所有既存的使用都產生tokens，用如下方法。

```python
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token

for user in User.objects.all():
    Token.objects.get_or_create(user=user)
```

### 產生令符 -藉由曝露一個api endpoint

當使用`TokenAuthentication` 你可能想要提供一個機制給客端，使得到一個token，由給定的username與password，  REST framework 提供一個內建的view以提供這個行為。在你的URLconf中加上`obtain_auth_token` view，以使用它。

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

<!--未完待續-->[](http://www.django-rest-framework.org/api-guide/authentication/#authentication)