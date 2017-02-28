快速看一下使用REST framework 以建立一個簡單的model-backed API。
我們將建立一個read-write API 以存取我們專案的使用者的資訊。
REST framework API 的任何的全域設定被保存在一個單一的設定字典中，其名稱為`REST_FRAMEWORK`，並置於你的`settings.py`中。

```python
REST_FRAMEWORK = {
    # Use Django's standard `django.contrib.auth` permissions,
    # or allow read-only access for unauthenticated users.
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
    ]
}
```
別忘了將`rest_framework`加到`INSTALLED_APPS`中。
現在我們已經準備好了要建立我們的API了，以下是我們的根`urls.py`模組。

```python
from django.conf.urls import url, include
from django.contrib.auth.models import User
from rest_framework import routers, serializers, viewsets

# Serializers define the API representation.
class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ('url', 'username', 'email', 'is_staff')

# ViewSets define the view behavior.
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

# Routers provide an easy way of automatically determining the URL conf.
router = routers.DefaultRouter()
router.register(r'users', UserViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

你現在可以用你的瀏覽器打開API `http://127.0.0.1:8000`， 並且檢視你的新'使用者'API ，假如你使用登錄控制，你將能在系統中建立與刪除使用者。