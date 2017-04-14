#Tutorial 4: Authentication & Permissions

[src](http://django-rest-framework.org/tutorial/4-authentication-and-permissions/)

目前為止，我們的API對誰能編輯或刪除snippet（代碼片段）還沒有任何限制。我們將增加一些擴展功能來確保以下：

    * snippets(即資料庫內的實例)總是關聯著創建者；
    * 只有認證後的用戶才能創建一個snippets.
    * 只有創建者才能更新或刪除snippet；
    * 非認證請求只擁有只讀權限。

## 1. 為model增加信息

我們先需要對Snippet的model做些修改。首先，增加一些fields. 其中一個用來表示創建者，另一個用來存儲代碼中的HTML高亮。

在`models.py`中的`Snippet`模型中增加這兩個field。

    owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
    highlighted = models.TextField()

我們還需要確保model存儲時，我們能使用 `pygments` 代碼高亮庫以生成語法高亮的field內容，。

首先需要一些額外的imports:

    from pygments.lexers import get_lexer_by_name
    from pygments.formatters.html import HtmlFormatter
    from pygments import highlight
    
我們現在為模型增加一個 `.save() `方法：
- `# models.py中只有一個class Snippet(model.Model):` 
- `# 因為要進行語法高亮化之後的儲存動作，所以需要一個額外的def save`

```python
    def save(self, *args, **kwargs):
        """
        Use the `pygments` library to create a highlighted HTML
        representation of the code snippet.
        """
        lexer = get_lexer_by_name(self.language)
        linenos = self.linenos and 'table' or False
        options = self.title and {'title': self.title} or {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos,
                                full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super(Snippet, self).save(*args, **kwargs)
```
這些完成後還需要更新一下資料庫裡面的表。通常我們要創建一個資料庫遷移（database migration）來完成，但教程中，我們就只是刪除資料庫然後重新創建：

    rm -f tmp.db db.sqlite3
    rm -r snippets/migrations
    python manage.py makemigrations snippets
    python manage.py migrate

你可能想創建一些其他用戶來測試這些API，最快捷的方式是利用 createsuperuser 命令。

    python ./manage.py createsuperuser 

## 2. 為用戶模型增加endpoints

現在我們需要一些用戶，我們最好把用戶呈現型也增加到API上，創建一個新的serializer很容易，在`serializers.py`加入以下程式碼： `#這個User是系統預設的，所以models.py內沒有`



```python
    from django.contrib.auth.models import User
 
    class UserSerializer(serializers.ModelSerializer):
        snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

        class Meta:
            model = User
            fields = ('id', 'username', 'snippets')
```
因為 `'snippets'` 和用戶model是反向關聯（reverse relationship，即多對一），所以在使用`ModelSerializer`時並不會預設加入，所以我們需要加入一顯式field的來實現。

我們還需要加一些views到views.py，對用戶呈現型上，我們偏好使用唯讀式的view，所以將使用 `ListAPIView` 和 `RetrieveAPIView` 泛型class-based Views。

```python
from django.contrib.auth.models import User

    class UserList(generics.ListAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

    class UserDetail(generics.RetrieveAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer
```
同時記得將`UserSerializer` import 進來(views.py中)

    from snipperts.serializers import UserSerializer
    
最後，我們需要修改URL conf：

    url(r'^users/$', views.UserList.as_view()),
    url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()), 

## 3. 把Snippets與Users關聯

現在，如果我們創建一個code snippet(物件)，還沒有方法可關連到其創建者。User並沒有作為序列化呈現型的一部分發送，而是作為incoming request的一個property。

這裡的處理方法是overriding(重載)snippet view中的 .preform_create() (這裡有改過，之前是.pre\_save())方法，以使實例物件的儲存可被管理，並且可以讓我們處理傳入的request或requested URL中隱式的信息。

在 `SnippetList`的view類中，添加如下的方法：

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

現在我們的序列化器的`create()`方法將傳遞一額外的`owner`欄位，並帶有來自request的驗證過的data

## 4. 更新 serializer

現在snippets已經和創建者關聯起來了，我們接下來還需要更新`SnippetSerializer`，在`serizlizers.py`中增加一個新的field：

    owner = serializers.ReadOnlyField(source='owner.username') # 要有這一行，才能正確顯示名稱吧，不然就會直接用id nunber來代替。

Note: 確定你有在class Meta:的field列表中也加入了 'owner'。 

這個field所做的一些事是十分有趣。`source` 參數控制某屬性以做為一個field，可以指向序列化實例的任何屬性。它可以採用如上的點式表示（dotted notation），在這個案例它將直接遍歷到指定的屬性。在Django's template中使用時，也是採用類似的方式。

我們增加field是一個無類型 `ReadOnlyField` 類，而我們之前的field都是有類型的，例如 `CharField`, `BooleanField` etc... 
無類型`ReadOnlyField`field總是只讀的，它們只用在序列化呈現型中，而在反序列化時（修改model）不被使用。在此(反序列化)我們也可以使用`CharField(read_only=True)`

## 5. 給view增加權限控制 
現在代碼片段 snippets 已經關聯了用戶，我們需要確保只有認證用戶才能增、刪、改snippets.

REST framework 包括許多權限類可用於view的控制。這裡我們使用 `IsAuthenticatedOrReadOnly`, 它可確保認證的request獲取read-write權限，
而非認證的request只有read-only 權限.

現需要在views模組中增加 import。

    from rest_framework import permissions

然後需要在 `SnippetList` 和 `SnippetDetail` view類中都增加如下屬性：

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,) 

## 6. 為可瀏覽API（Browseable API）增加login

如果你打開瀏覽器，訪問可瀏覽API，你會發現只有登錄後才能創建新的snippet了，為此我們需要可以login為user。

我們可以編輯project-level 的urls.py來增加一個登錄view。首先增加新的import：

    from django.conf.urls import include

然後，在文件末尾增加一個pattern來為browsable API增加 login 和 logout views.

```python

    urlpatterns += [
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
    ]
```
具體的， `r'^api-auth/'` 部分可以用任何你想用的URL來替代。這裡唯一的限制就是 urls 必須使用`'rest_framework'` 命名空間。
在Django 1.9+，REST framework將會設定namespace，所以你可以不用理它。

現在如果你打開瀏覽器，刷新頁面會看到頁面右上方的 'Login' 鏈接。如果你用之前的用戶登錄後，你就又可以創建 snippets了。

一旦你創建了一些snippets，當導航至'/users/'時，你會看到在每個user的snippetsfield都包含了一系列snippet的pk。

## 7. 物件級別的權限

我們希望任何人都可以瀏覽snippets，但只有創建snippet的用戶才能編輯或刪除它。

為了實現這個需求，我們需要創建定製的權限（custom permission）。

在 snippets 應用中，創建一個新文件： `permissions.py` `一個獨立的permissions.py，代表權限不屬於views.py or models.py`

```python
    from rest_framework import permissions # 注意這個permissions是來自rest_framework

    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        Custom permission to only allow owners of an object to edit it.
        """

        def has_object_permission(self, request, view, obj):
            # Read permissions are allowed to any request,
            # so we'll always allow GET, HEAD or OPTIONS requests.
            if request.method in permissions.SAFE_METHODS:            
                return True

            # Write permissions are only allowed to the owner of the snippet
            return obj.owner == request.user 
```
現在我們可以為snippet實例增加定製權限了，需要編輯 `SnippetDetail` view class的 permission_classes 屬性： `# 這應該是在views.py內`

```python

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                        IsOwnerOrReadOnly,)
```
別忘了import 這個`IsOwnerOrReadOnly `類。

    from snippets.permissions import IsOwnerOrReadOnly 

現在打開瀏覽器，你可以看見 'DELETE' 和 'PUT' 動作只會出現在那些你的登錄用戶創建的snippet頁面上了.

## 8. 通過API認證

我們已經有了一系列的權限，如果我們需要編輯任何snippet，我們需要認證我們的request。因為我們還沒有建立任何 `authentication classes`,所以目前是默認的`SessionAuthentication` 和`BasicAuthentication`在起作用。

當我們通過Web瀏覽器與API互動時，我們登錄後、然後瀏覽器session可以為所有的request提供所需的驗證。

如果我們使用程序訪問這些API，我們則需要顯式的為每個request提供認證憑證（authentication credentials）。

如果我們試圖未認證就創建一個snippet，將得到錯誤如下：

    http POST http://127.0.0.1:8000/snippets/ -d "code=print 123"
    {
        "detail": "Authentication credentials were not provided."
    }

如果我們帶著用戶名和密碼來請求時則可以成功創建：

    http -a tom:password POST http://127.0.0.1:8000/snippets/ -d "code=print 789" 
    {
        "id": 5,
        "owner": "tom",
        "title": "foo",
        "code": "print 789",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

## 9. 小結

我們已經為我們的Web API創建了相當細粒度的權限控制和相應的系統用戶。

在教程第5部分 part 5 ，我們將把所有的內容串聯起來，為我們的高亮代碼片段創建HTML節點，並利用系統內的超鏈接關聯來提升API的一致性表現。

## 重點
  1. snippet model 要能關連到創造者，本質上就是要有一個onwer欄位，是外鍵到認證使用者。因而資料必須更新。
  2. serializers.py 要增加UserSerializer的類，同時要 `from django.contrib.auth.models import User`
  3. SnippetSerializer 也要加一個欄位owner
  4. 我們現在的views.py是使用CBV，要加入`permission_classes = (permissions.IsAuthenticatedOrReadOnly,)` 
  5. 我們還需要一個permissons.py，用以放置客製化的permission `class IsOwnerOrReadOnly(permissions.BasePermission):`然後在detail類中使用
  6. 然後還要給login 的連結，或者是通過api的認證。


## 心得
這一小節的主要重點是權限與認證使用者，當然有時候權限會很複雜，REST framework 可以做到單一實例的權限管制其實滿厲害的。
