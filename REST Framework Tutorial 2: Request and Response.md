# Tutorial 2: Requests and Response

[src](http://django-rest-framework.org/tutorial/2-requests-and-responses.html) 

從本節我們開始真正接觸rest framework的核心部分。首先我們學習一下一些必備知識。

## 1. Request Object ——Request物件

rest framework 引入了一個繼承自HttpRequest的`Request`物件，該物件提供了對requests的更靈活解析。`Request`物件的核心部分是request.data屬性，
類似於request.POST, 但在使用WEB API時，request.data更有用。下面是request.data與request.POST的比較。

    request.POST # Only handles form data. Only works for 'POST' method. 
    request.data # Handles arbitrary data. Works any HTTP request with content.

## 2. Response Object ——Response物件

rest framework引入了一個Response 物件，它是TemplateResponse的類型。它獲得未渲染的內容並通過內容協商(content negotiation) 來決定正確的content type返回給client。

    return Response(data)  # Renders to content type as requested by the client.

## 3. Status Codes

在views當中使用數字化的HTTP狀態碼，會使你的代碼不宜閱讀，且不容易發現代碼中的錯誤。rest framework為每個狀態碼提供了更明確的標識。例如`HTTP_400_BAD_REQUEST`在status module。相比於使用數字，在整個views中使用這類標識符將更好。

## 4. 可包裝的API views

在編寫API views時，REST Framework提供了兩種wrappers：

1. The `@api_viwe` decorator for working with function based views.
2. The `APIView` class for working with class based views.

這兩種包裝器提供了許多功能，例如，確保在view當中能夠接收到`Request`實例；往`Response`中增加內容以便內容協商(content negotiation) 機制能夠執行。

包裝器也提供一些行為，例如在適當的時候返回`405 Methord Not Allowed`回應；在不正確輸入`request.data`時，處理任何的`ParseError`異常。

## 5. 彙總

我們開始用這些新的組件來寫一些views。

我們不再需要`views.py`中的`JESONResponse` 類（在前一篇中`view.py`中創建，它的作用就是做為一個包裝器，將json資料進行包裝），將它刪除。刪除後我們開始稍微重構下我們的view

    from rest_framework import status
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

    @api_view(['GET', 'POST'])
    def snippet_list(request):
        """
        List all snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        elif request.method == 'POST':
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

上面的代碼是對我們之前代碼的改進。看上去更簡潔，也更類似於django的forms api形式。我們也採用了狀態碼，使返回值更加明確。 下面是對單個snippet操作的view更新：

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a snippet instance.
        """              
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        elif request.method == 'PUT':
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        elif request.method == 'DELETE':
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

這應感覺非常熟悉，這與一般的Django views並不會差很多。

注意，我們不在有沒有明確的要求requests或者responses給出content type。`request.data`可以處理輸入的`json` requests，也可以輸入yaml和其他格式。
類似的在response返回帶數據的物件時，允許REST Framework繪出response到正確的content type給我們。

## 6. 給URLs增加可選的格式後綴

利用在response時不需要指定content type這一優點，我們在API端增加格式的後綴。使用格式後綴，可以明確的指出使用某種格式，意味著我們的API可以處理類似`http://example.com/api/items/4.json.`的URL。

增加`format`參數到function views中，如：

    def snippet_list(request, format=None):

and

    def snippet_detail(request, pk, format=None):

現在稍微更新urls.py文件，在現有的URLs中添加一個格式後綴pattterns (format\_suffix\_patterns):

    from django.conf.urls import patterns, url
    from rest_framework.urlpatterns import format_suffix_patterns

    urlpatterns =[
        url(r'^snippets/$', 'snippet_list'),
        url(r'^snippets/(?P<pk>[0-9]+)$', 'snippet_detail'),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns) 

We don't necessarily need to add these extra url patterns in, but it gives us a simple, clean way of referring to a specific format.
## 7. How's it looking?

Go ahead and test the API from the command line, as we did in tutorial part 1. Everything is working pretty similarly, although we've got some nicer error handling if we send invalid requests.

We can get a list of all of the snippets, as before.

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
We can control the format of the response that we get back, either by using the Accept header:

    curl http://127.0.0.1:8000/snippets/ -H 'Accept: application/json'  # Request JSON
    curl http://127.0.0.1:8000/snippets/ -H 'Accept: text/html'         # Request HTML

Or by appending a format suffix:

    curl http://127.0.0.1:8000/snippets.json  # JSON suffix
    curl http://127.0.0.1:8000/snippets.api   # Browsable API suffix

Similarly, we can control the format of the request that we send, using the Content-Type header.

    # POST using form data
    http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
        "id": 3,
        "title": "",
        "code": "print 123",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

    # POST using JSON
    http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

    {
        "id": 4,
        "title": "",
        "code": "print 456",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

Now go and open the API in a web browser, by visiting `http://127.0.0.1:8000/snippets/`.

## 8. Browsability

Because the API chooses the content type of the response based on the client request, it will, by default, return an 
HTML-formatted representation of the resource when that resource is requested by a web browser. This allows for the API to 
return a fully web-browsable HTML representation.

Having a web-browsable API is a huge usability win, and makes developing and using your API much easier. It also dramatically 
lowers the barrier-to-entry for other developers wanting to inspect and work with your API.

See the browsable api topic for more information about the browsable API feature and how to customize it.

## 心得
這篇教學就是教使用預設有包裝器功的request.data與Response(data)以及format(後綴)的使用。
