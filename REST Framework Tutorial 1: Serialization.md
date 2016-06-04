# Tutorial 1: 序列化 Serialization

`新的教學好像改了很多東西，201606更新`

## 1. 設置一個新的環境

在我們開始之前， 我們首先使用virtualenv要創建一個新的虛擬環境，以使我們的配置和我們的其他項目配置徹底分開。

    $mkdir ~/env
    $virtualenv  ~/env/tutorial
    $source ~/env/tutorial/bin/avtivate

現在我們處在一個虛擬的環境中，開始安裝我們的依賴包

    $pip install django
    $pip install djangorestframework
    $pip install pygments   ////使用這個包，做代碼高亮顯示

需要退出虛擬環境時，運行deactivate。更多信息，virtualenv document

## 2. 開始

環境準備好只好，我們開始創建我們的項目

    $ cd ~
    $ django-admin.py startproject tutorial
    $ cd tutorial

項目創建好後，我們再創建一個簡單的app

    $python manage.py startapp snippets

我們使用sqlite3來運行我們的項目tutorial，編輯tutorial/settings.py,
將資料庫的默認引擎engine改為sqlite3, 資料庫的名字NAME改為tmp.db

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': 'tmp.db',
            'USER': '',
            'PASSWORD': '',
            'HOST': '',
            'PORT': '',
        }
    }

同時更改settings.py文件中的INSTALLD_APPS,添加我們的APP snippets和rest_framework

    INSTALLED_APPS = (
        ...
        'rest_framework',
        'snippets.apps.SnippetsConfig',
    )

在tutorial/urls.py中，將snippets app的url包含進來 `這裡被移到後面去了`

    urlpatterns = patterns('',
        url(r'^', include('snippets.urls')),
    )

## 3. 創建Model

這裡我們創建一個簡單的snippets model，目的是用來存儲代碼片段。

    from django.db import models
    from pygments.lexers import get_all_lexers
    from pygments.styles import get_all_styles

    LEXERS = [item for item in get_all_lexers() if item[1]]
    LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
    STYLE_CHOICES = sorted((item, item) for item in get_all_styles())

    class Snippet(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        title = models.CharField(max_length=100, default='')
        code = models.TextField()
        linenos = models.BooleanField(default=False)
        language = models.CharField(choices=LANGUAGE_CHOICES,
                                    default='python',
                                    max_length=100)
        style = models.CharField(choices=STYLE_CHOICES,
                                default='friendly',
                                max_length=100)

        class Meta:
            ordering = ('created',)

完成model時，記得sync下資料庫

    python manage.py makemigrations snippets
    python manage.py migrate

## 4. 創建序列化類

我們要使用我們的web api，要做的第一件事就是提供snippets實例序列化和反序列化的方法， 以使snippets實例能轉換為可表述的內容，例如json.
我們宣告一個串行器serializer，該串行器與django 的表單形式很類似。在snippets目錄下面，創建一個serializers.py ，並將下面內容拷貝到文件中。

    from rest_framework import serializers
    from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES

    class SnippetSerializer(serializers.Serializer):
        pk = serializers.IntegerField(read_only=True)  
        title = serializers.CharField(required=False,
                                    allow_blank=True,
                                    max_length=100)
        code = serializers.CharField(style={'base_template': 'textarea.html'})                                    
        linenos = serializers.BooleanField(required=False)
        language = serializers.ChoiceField(choices=LANGUAGE_CHOICES,
                                        default='python')
        style = serializers.ChoiceField(choices=STYLE_CHOICES,
                                        default='friendly')

        def create(self, validated_data):
            """
            Create or return a new 'Snippet' instance, given the validated data.
            """
            return Snippet.objects.create(**validated_data)
        def update(self, instance, validated_data):
            """
            update and return an existing `Snippet` instance, given the validated data.
            """
            instance.title = validated_data.get('title', instance.title)
            instance.code = validated_data.get('code', instance.code)
            instance.linenos = validated_data.get('linenos', instance.linenos)
            instance.language = validated_data.get('language', instance.language)
            instance.style = validated_data.get('style', instance.style)
            instance.save()
            return instance

            
            
該序列化類的前面部分，定義了要序列化和反序列化的欄位(字段)(所以都是serializaers.什麼什麼)。
當呼叫`serializer.save()`,`create()`和`update()` 方法定義了如何通過反序列化數據，生成或修改為正確的物件實例(所以都是validated_data.get(屬性)，然後instance.save()，最後返回instance)。

一個序列化類與Django Form 類十分相似，且包含了相似的確效旗標在各個欄位上，比如`required`, `max_length`, `default`。

在某些情況下，欄位旗標也可以用來控制serializer 如何被顯示。比如繪出為HTML時，上述的`{'base_template': 'textarea.html'}` 是等於Django Form class的 `widget=widgets.Textarea`。
這對於控制`browsable API`如何被顯示十分有用，我們將在稍後的教學中看到。

我們也可以使用ModelSerializer來快速生成，後面我們將看到如何使用它。

## 5. 使用 Serializers

在我們使用我們定義的新的Serializers之前，我們先到Django shell.

    $python manage.py shell

進入shell終端後，輸入以下代碼：

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser

    snippet = Snippet(code='print "hello, world"\n')
    snippet.save()

我們現在獲得了一個Snippets的實例可以利用，現在我們對他進行以下序列化

    serializer = SnippetSerializer(snippet)
    serializer.data
    # {'pk': 1, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}
    # type(serializer.data)
    # rest_framework.utils.serializer_helpers.ReturnDict


這時，我們將該實例轉成了python原生的數據類型。下面我們將該數據轉換成json格式，以完成序列化：

    content = JSONRenderer().render(serializer.data)
    content
    # '{"pk": 1, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'
    # type(content)
    # bytes

反序列化也很簡單，首先我們要將一個輸入流（content），轉換成python的原生數據類型
`這裡可能可以做一些修改，使用json.loads(content.decode())，可返回python 的dict資料類型`

    import StringIO

    stream = StringIO.StringIO(content)
    data = JSONParser().parse(stream)

然後我們將該原生數據類型，轉換成物件實例

    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.object
    # <Snippet: Snippet object>

注意這些API和django表單的相似處。這些相似點， 在我們講述在view中使用serializers時將更加明顯。

We can also serialize querysets instead of model instances. To do so we simply add a many=True flag to the serializer arguments.

    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data

```sh    
    ReturnDict([('pk', 3),
            ('title', ''),
            ('code', 'print("hello,world"\n'),
            ('linenos', False),
            ('language', 'python'),
            ('style', 'friendly')])
    #一些其他的例子
    
    
```
## 6. 使用 ModelSerializers

SnippetSerializer使用了許多和Snippet中相同的代碼。如果我們能把這部分代碼去掉，看上去將更佳簡潔。

類似與django提供`Form`類和`ModelForm`類，Rest Framework也包含了`Serializer` 類和 `ModelSerializer`類。

打開snippets/serializers.py ,修改SnippetSerializer類：

    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ('id', 'title', 'code', 'linenos', 'language', 'style')

一個serializers的良好特性是你可以探索在serializer實例中所有的欄位，藉由印出其表現型，打開Django shell，試著依下面進行。

    from snippets.serializers import SnippetSerializer
    serializer = SnippetSerializer()
    print(repr(serializer))
    # SnippetSerializer():
    #    id = IntegerField(label='ID', read_only=True)
    #    title = CharField(allow_blank=True, max_length=100, required=False)
    #    code = CharField(style={'base_template': 'textarea.html'})
    #    linenos = BooleanField(required=False)
    #    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    #    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...

重要請記得，`ModelSerializer`並沒有做什麼魔法，它們只是簡單的創造出一個serializer 類別

    * An automatically determined set of fields.
    * Simple default implementations for the create() and update() methods.

## 7. 通過Serializer編寫Django View

讓我們來看一下，如何通過我們創建的serializer類編寫API view。這裡我們不使用rest framework的其他特性，僅編寫正常的django view。

我們創建一個HttpResponse 子類，這樣我們可以將我們返回的任何數據轉換成json。

在snippet/views.py中添加以下內容：

    from django.http import HttpResponse
    from django.views.decorators.csrf import csrf_exempt
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

    class JSONResponse(HttpResponse):
        """
        An HttpResponse that renders it's content into JSON.
        """
        def __init__(self, data, **kwargs):
            content = JSONRenderer().render(data)
            kwargs['content_type'] = 'application/json'
            super(JSONResponse, self).__init__(content, **kwargs)

我們API的目的是，可以通過view來列舉全部的Snippet的內容，或者創建一個新的snippet

    @csrf_exempt
    def snippet_list(request):
        """
        List all code snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return JSONResponse(serializer.data)

        elif request.method == 'POST':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data, status=201)
            else:
                return JSONResponse(serializer.errors, status=400)

注意，因為我們要能夠自client向該view 丟一個`Post`請求，所以我們要將該view 標註為csrf_exempt, 以說明不是一個CSRF事件。
這並不是一個正常你想要做的事，且**REST framework** view事實上使用一個比csrf token更靈敏的行為，但我們現在就是要這麼做。

我們也需要一個view來操作一個單獨的Snippet，以便能取回/更新/刪除該sinppet物件。

    @csrf_exempt
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a code snippet.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JSONResponse(serializer.data)

        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data)
            else:
                return JSONResponse(serializer.errors, status=400)

        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)

將views.py保存，在Snippets目錄下面創建urls.py,添加以下內容：
    from django.conf.urls import url
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
    ]
We also need to wire up the root urlconf, in the tutorial/urls.py file, to include our snippet app's URLs.

    from django.conf.urls import url, include

    urlpatterns = [
        url(r'^', include('snippets.urls')),
    ]
注意我們有些邊緣事件沒有處理，服務器可能會拋出500異常。

It's worth noting that there are a couple of edge cases we're not dealing with properly at the moment. If we send malformed json, or if a request is made with a method that the view doesn't handle, then we'll end up with a 500 "server error" response. Still, this'll do for now.

## 8. 測試

現在我們啟動server來測試我們的Snippet。

在python mange.py shell終端下執行（如果前面進入還沒有退出）

    quit()

執行下面的命令， 運行我們的server：

    python manage.py runserver

    Validating models...

    0 errors found
    Django version 1.8.3, using settings 'tutorial.settings'
    Development server is running at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.

新開一個terminal來測試我們的server,來測試我們的server
我們可以用curl或httpie來測試我們的API.httpie 是一個使用者友善的http client，由python寫成，讓我安裝它

    pip install httpie

最後我們可得一個所有的snippets物件的清單
    http http://127.0.0.1:8000/snippets/

    HTTP/1.1 200 OK
    ...
    [
        {
            "id": 1,
            "title": "",
            "code": "foo = \"bar\"\n",
            "linenos": false,
            "language": "python",
            "style": "friendly"
        },
        {
            "id": 2,
            "title": "",
            "code": "print \"hello, world\"\n",
            "linenos": false,
            "language": "python",
            "style": "friendly"
        }
    ]
或者由其id取得特定的snippet物件。
    http http://127.0.0.1:8000/snippets/2/

    HTTP/1.1 200 OK
    ...
    {
        "id": 2,
        "title": "",
        "code": "print \"hello, world\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

## Where are we now
我們目前還可以，我們已經有一個序列化API，有點像Django's Forms API，和一些Django views。
我們的API此時無法做任何特別的動作，除了提供`json`回應，也需要錯誤處理機制，但這已經是一個有功能的API。

心得

SnippetSerializer類別繼承了rest_framework的serializer.Serializer類別，因而有些方法在程式碼上看不到；
像是SS.is_valid()和SS.save()，save()之後會直接將資料丟進資料庫。作者想表達的是從資料庫物件經序列化到json(b)，
再從json經反序列化回到資料庫物件，都是使用serializer.Serializer類別來完成。
在教學的最部分則是用views只是把json包裝一下()，再Response出來。