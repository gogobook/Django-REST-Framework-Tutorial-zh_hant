源代碼：test.py

## 測試
沒有測試的代碼就是壞的。

- Jacob Kaplan-Moss

REST framwork包含一些擴展自Django現有測試框架的助手類()，並改進對API requests 的支持。

## APIRequestFactory
擴展了[Django的現有`RequestFactory`類](https://docs.djangoproject.com/en/2.1/topics/testing/advanced/#django.test.client.RequestFactory)。

### 創建測試請求
本`APIRequestFactory`類支持幾乎相同的API來Django的標準RequestFactory類。這意味著，標準.get()，.post()，.put()，.patch()，.delete()，.head()和.options()方法都是可用的。
```py
from rest_framework.test import APIRequestFactory

# Using the standard RequestFactory API to create a form POST request
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'})
```
#### 使用format參數
創建請求主體的方法（例如post，put和patch）包含一個format參數，這使得使用除多部分錶單數據以外的內容類型生成請求變得容易。例如：
```py
# Create a JSON POST request
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'}, format='json')
```

默認情況下，可用的格式是'multipart'和'json'。為了與Django現有RequestFactory的默認格式兼容'multipart'。

要支持更多的請求格式或更改默認格式，請參閱配置部分。

#### 顯式編碼請求主體
如果你需要明確地編碼請求主體，你可以通過設置content_type標誌來完成。例如：

`request = factory.post('/notes/', json.dumps({'title': 'new idea'}), content_type='application/json')`
#### PUT和PATCH與表單數據
Django RequestFactory和REST framwork之間值得注意的一個區別APIRequestFactory是多部分錶單數據將被編碼為除了正義之外的其他方法.post()。

例如，使用`APIRequestFactory`，你可以像這樣做一個表單PUT請求：
```py
factory = APIRequestFactory()
request = factory.put('/notes/547/', {'title': 'remember to email dave'})
```
使用Django的`RequestFactory`，你需要自己明確地編碼數據：
```py
from django.test.client import encode_multipart, RequestFactory

factory = RequestFactory()
data = {'title': 'remember to email dave'}
content = encode_multipart('BoUnDaRyStRiNg', data)
content_type = 'multipart/form-data; boundary=BoUnDaRyStRiNg'
request = factory.put('/notes/547/', content, content_type=content_type)
```
### 強制認證
當使用請求工廠直接測試視圖時，能夠直接驗證請求通常很方便，而不必構造正確的驗證憑證。

要強制驗證請求，請使用該force_authenticate()方法。
```py
from rest_framework.test import force_authenticate

factory = APIRequestFactory()
user = User.objects.get(username='olivia')
view = AccountDetail.as_view()

# Make an authenticated request to the view...
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user)
response = view(request)
```
該方法的簽名是`force_authenticate(request, user=None, token=None)`。撥打電話時，可以設置用戶和令牌中的任一個或兩個。

例如，當使用令牌強行進行身份驗證時，您可能會執行以下操作：
```py
user = User.objects.get(username='olivia')
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user, token=user.auth_token)
```
注意：`force_authenticate`直接設置`request.user`到內存中的`user`實例。如果您`user`在多個測試中重新使用同一個實例來更新保存的`user`狀態，則可能需要refresh_from_db()在測試之間進行調用。

注意：使用時`APIRequestFactory`，返回的對像是Django的標準`HttpRequest`，而不是REST framwork的`Request`對象，只有在調用視圖後才會生成該對象。

這意味著直接在請求對像上設置屬性可能並不總是有您期望的效果。例如，`.token`直接設置將不起作用，`.user`直接設置僅在使用會話認證時才有效。
```py
# Request will only authenticate if `SessionAuthentication` is in use.
request = factory.get('/accounts/django-superstars/')
request.user = user
response = view(request)
```
### 強制CSRF驗證
默認情況下，創建的請求APIRequestFactory在傳遞到REST framwork視圖時將不會應用CSRF驗證。如果您需要明確打開CSRF驗證，則可以通過enforce_csrf_checks在實例化工廠時設置標誌來實現。

`factory = APIRequestFactory(enforce_csrf_checks=True)`
注意：值得注意的是，Django的標準RequestFactory不需要包含這個選項，因為當使用常規Django時，CSRF驗證發生在中間件中，當直接測試視圖時，中間件不運行。在使用REST framwork時，CSRF驗證發生在視圖內，因此請求工廠需要禁用視圖級CSRF檢查。

## APIClient
擴展了Django的現有Client類。

### 發出請求
本`APIClient`類支持相同的請求接口Django的標準Client類。這意味著這一標準`.get()`，`.post()`，`.put()`，`.patch()`，`.delete()`，`.head()`和`.options()`方法都是可用的。例如：
```py
from rest_framework.test import APIClient

client = APIClient()
client.post('/notes/', {'title': 'new idea'}, format='json')
```
要支持更多的請求格式或更改默認格式，請參閱配置部分。

### 認證
#### .login（** kwargs）
該`login`方法的功能與Django常規Client類的功能完全相同。這使您可以根據包含的任何視圖驗證請求`SessionAuthentication`。
```py
# Make all requests in the context of a logged in session.
client = APIClient()
client.login(username='lauren', password='secret')
```
要註銷，請logout照常調用方法。
```py
# Log out
client.logout()
```
該`login方`法適用於測試使用會話認證的API，例如包含AJAX與API交互的網站。

#### .credentials（** kwargs）
該`credentials`方法可用於設置標題，這些標題將包含在測試客戶端的所有後續請求中。
```py
from rest_framework.authtoken.models import Token
from rest_framework.test import APIClient

# Include an appropriate `Authorization:` header on all requests.
token = Token.objects.get(user__username='lauren')
client = APIClient()
client.credentials(HTTP_AUTHORIZATION='Token ' + token.key)
```
請注意，`credentials`再次調用將覆蓋任何現有憑證。您可以通過調用沒有參數的方法來取消任何現有的憑證。
```py
# Stop including any credentials
client.credentials()
```
該`credentials`方法適用於測試需要身份驗證頭的API，例如基本身份驗證，OAuth1a和OAuth2身份驗證以及簡單的令牌身份驗證方案。

#### .force_authenticate（user = None，token = None）
有時您可能想完全繞過認證，強制測試客戶端的所有請求被自動視為已認證。

如果您正在測試API但是不想構建有效的身份驗證憑據以進行測試請求，這可能是一個有用的捷徑。
```py
user = User.objects.get(username='lauren')
client = APIClient()
client.force_authenticate(user=user)
```
要未經`force_authenticate`身份驗證後續請求，請設置用戶和/或令牌`None`。

`client.force_authenticate(user=None)`
### CSRF驗證
默認情況下，CSRF驗證在使用時不適用`APIClient`。如果您需要明確啟用CSRF驗證，則可以通過`enforce_csrf_checks`在實例化客戶端時設置標誌來實現。

`client = APIClient(enforce_csrf_checks=True)`
像往常一樣，CSRF驗證將僅適用於任何會話驗證視圖。這意味著CSRF驗證只有在客戶端通過調用`login()`登錄後才會發生。

## RequestsClient
REST framework還包含一個客戶端，用於使用流行的Python庫`requests`與您的應用程序進行交互。這可能是有用的，如果：

- 您期望主要從另一個Python服務與API進行交互，並且希望在與客戶端相同的級別測試該服務。
- 您希望以這樣的方式編寫測試，以便它們也可以在分段或實時環境中運行。（請參閱下面的“實時測試”。）
這暴露了與直接使用請求會話完全相同的界面。
```py
client = RequestsClient()
response = client.get('http://testserver/users/')
assert response.status_code == 200
```
請注意，請求客戶端要求您傳遞完全限定的URL。

### `RequestsClient` 並與數據庫合作
`RequestsClient`類，如果你想編寫測試，是專為與服務界面交互是有用的。這比使用標準的Django測試客戶端要嚴格一些，因為它意味著所有的交互應該通過API。

如果您正在使用，`RequestsClient`您需要確保測試設置和結果聲明是作為常規API調用執行的，而不是直接與數據庫模型進行交互。例如，而不是檢查`Customer.objects.count() == 3`是否列出客戶端點，並確保它包含三條記錄。

### 頭和認證(Headers & Authentication)
自定義標頭和身份驗證憑證的提供方式與使用標準requests.Session實例時的方式相同。
```py
from requests.auth import HTTPBasicAuth

client.auth = HTTPBasicAuth('user', 'pass')
client.headers.update({'x-test': 'true'})
```
### CSRF
如果你使用的是`SessionAuthentication`，那麼你就需要包括任何CSRF令牌在POST，PUT，PATCH或DELETE請求。

您可以通過遵循基於JavaScript的客戶端使用的相同流程來實現此目的。首先發出`GET`請求以獲取CRSF令牌，然後在以下請求中呈現該令牌。

例如...
```py
client = RequestsClient()

# Obtain a CSRF token.
response = client.get('/homepage/')
assert response.status_code == 200
csrftoken = response.cookies['csrftoken']

# Interact with the API.
response = client.post('/organisations/', json={
    'name': 'MegaCorp',
    'status': 'active'
}, headers={'X-CSRFToken': csrftoken})
assert response.status_code == 200
```
### 現場測試
通過仔細的使用，`RequestsClient`和`CoreAPIClient`都提供了編寫測試用例的能力，這些測試用例可以在開發中運行，也可以直接運行在您的臨時服務器或生產環境中。

使用這種風格來創建一些核心功能的基本測試，是驗證您的正在運作(live)服務的強大方法。這樣做可能需要對設置和teardown進行一些仔細的注意，以確保測試以一種不會直接影響客戶數據的方式運行。

## CoreAPIClient
CoreAPIClient允許您使用Python coreapi客戶端庫與您的API進行交互 。
```py
# Fetch the API schema
client = CoreAPIClient()
schema = client.get('http://testserver/schema/')

# Create a new organisation
params = {'name': 'MegaCorp', 'status': 'active'}
client.action(schema, ['organisations', 'create'], params)

# Ensure that the organisation exists in the listing
data = client.action(schema, ['organisations', 'list'])
assert(len(data) == 1)
assert(data == [{'name': 'MegaCorp', 'status': 'active'}])
```
### 頭和認證
自定義頭和身份驗證可以用類似於RequestsClient的類似方式使用CoreAPIClient。
```py
from requests.auth import HTTPBasicAuth

client = CoreAPIClient()
client.session.auth = HTTPBasicAuth('user', 'pass')
client.session.headers.update({'x-test': 'true'})
```
## API測試案例
REST framwork包含以下測試用例類，它們反映了現有的Django測試用例類，但是使用的APIClient不是Django的默認類Client。

- APISimpleTestCase
- APITransactionTestCase
- APITestCase
- APILiveServerTestCase
### 例
您可以使用任何REST框架的測試用例類，就像常規的Django測試用例類一樣。客戶端屬性將是一個APIClient實例。
```py
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase
from myproject.apps.core.models import Account

class AccountTests(APITestCase):
    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        data = {'name': 'DabApps'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Account.objects.count(), 1)
        self.assertEqual(Account.objects.get().name, 'DabApps')
```
## URLPatternsTestCase
REST框架還提供了一個測試用例類，用於在每個類基礎上隔離urlpatterns。注意，這繼承了Django的SimpleTestCase，並且很可能需要與另一個測試用例類混合。

### 例
```py
from django.urls import include, path, reverse
from rest_framework.test import APITestCase, URLPatternsTestCase


class AccountTests(APITestCase, URLPatternsTestCase):
    urlpatterns = [
        path('api/', include('api.urls')),
    ]

    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        response = self.client.get(url, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
```
## 測試響應
### 檢查響應數據
在檢查測試響應的有效性時，檢查響應的創建數據通常比較方便，而不是檢查完全響應。

例如，檢查`response.data`更容易：
```py
response = self.client.get('/users/4/')
self.assertEqual(response.data, {'id': 4, 'username': 'lauren'})
```
而不是檢查解析`response.content`的結果：
```py
response = self.client.get('/users/4/')
self.assertEqual(json.loads(response.content), {'id': 4, 'username': 'lauren'})
```
### 呈現響應
如果您直接使用`APIRequestFactory`測試視圖，則返回的響應將不會呈現，因為模板響應的呈現由Django的內部請求 - 響應循環執行。為了訪問`response.content`，您首先需要呈現響應。
```py
view = UserDetail.as_view()
request = factory.get('/users/4')
response = view(request, pk='4')
response.render()  # Cannot access `response.content` without this.
self.assertEqual(response.content, '{"username": "lauren", "id": 4}')
```
## 組態
### 設置默認格式
用於生成測試請求的預設格式可以使用`TEST_REQUEST_DEFAULT_FORMAT`設置鍵來設置。例如，在預設情況下，為測試請求使用JSON而不是標準的multipart表單請求，請在設置中設置以下內容。py文件:
```py
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_DEFAULT_FORMAT': 'json'
}
```
### 設置可用的格式
如果您需要使用multipart或json請求以外的其他方法來測試請求，則可以通過設置`TEST_REQUEST_RENDERER_CLASSES`設置來進行測試。

例如，要添加對format='html'在測試請求中使用的支持，您的settings.py文件中可能會有類似的內容。
```py
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_RENDERER_CLASSES': (
        'rest_framework.renderers.MultiPartRenderer',
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.TemplateHTMLRenderer'
    )
}
```