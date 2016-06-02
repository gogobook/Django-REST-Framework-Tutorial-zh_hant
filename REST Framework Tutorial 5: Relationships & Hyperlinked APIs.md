# Tutorial 5: Relationships & Hyperlinked APIs

[src](http://django-rest-framework.org/tutorial/5-relationships-and-hyperlinked-apis.html)

到目前為止，在我們的API中關係（relationship）還是通過主鍵來表示的。在這部分的教程中，我們將用超鏈接方式來表示關係，從而提升API的統一性和可發現性。

## 1. 為API根創建一個endpoint

到目前為止，我們已經有了'snippets'和'users'的endpoint, 但是我們還沒有為我們的API單獨創立一個端點入口。我們可以用常規的基於函數的view和之前介紹的 `@api_view `修飾符來創建。

    from rest_framework import renderers
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from rest_framework.reverse import reverse

    @api_view(('GET',))
    def api_root(request, format=None):
        return Response({
            'users': reverse('user-list', request=request, format=format),
            'snippets': reverse('snippet-list', request=request, format=format)
        })

請注意，我們用 REST framework 的 reverse 函數來返回完全合規的URLs.

## 2. 為高亮的Snippet創建一個endpoint

我們目前還沒有為支持代碼高亮的Snippet創建一個endpoints.

與之前的API endpoints不同, 我們將直接使用HTML呈現，而非JSON。在 REST framework中有兩種風格的HTML render, 一種使用模板來處理HTML，一種則使用預先處理的方式。在這裡我們使用後者。

另一個需要我們考慮的是，對於高亮代碼的view並沒有具體的泛型view可以直接利用。我們將只返回實例的一個屬性而不是物件實例本身。

沒有具體泛型view的支持，我們使用基類來表示實例，並創建我們自己的 `.get() `方法。在你的 `snippets.views `中增加：

    from rest_framework import renderers
    from rest_framework.response import Response

    class SnippetHighlight(generics.SingleObjectAPIView):
        model = Snippet
        renderer_classes = (renderers.StaticHTMLRenderer,)

        def get(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted) 

和以往一樣，我們需要為新的view增加新的URLconf，如下增加urlpatterns:

    url(r'^$','api_root'),

還需要為代碼高亮增加一個urlpatterns：

    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),

## 3. API超鏈接化

在Web API設計中，處理實體間關係是一個有挑戰性的工作。我們有許多方式來表示關係：

    * 使用主鍵；
    * 使用超鏈接；
    * 使用相關實體唯一標識的字段；
    * 使用相關實體的默認字符串表示；
    * 在父級表示中嵌入子級實體；
    * 其他自定義的表示。

REST framework支持所有這些方式，包括正向或者反向的關係，或者將其應用到自定義的管理類中，例如泛型外鍵。

在這部分，我們使用超鏈接方式。為了做到這一點，我們需要在序列化器中用 `HyperlinkedModelSerializer` 來替代之前的`ModelSerializer`.

`HyperlinkedModelSerializer` 與 `ModelSerializer` 有如下的區別:

    * 缺省狀態下不包含 pk 字段；

    * 具有一個 url 字段，即HyperlinkedIdentityField類型.

    * 用HyperlinkedRelatedField表示關係，而非PrimaryKeyRelatedField.

    * 我們可以很方便的改寫現有代碼來使用超連接方式：
```python
    class SnippetSerializer(serializers.HyperlinkedModelSerializer): owner = serializers.Field(source='owner.username') highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

      class Meta:
          model = models.Snippet
          fields = ('url', 'highlight', 'owner',
                    'title', 'code', 'linenos', 'language', 'style')

    class UserSerializer(serializers.HyperlinkedModelSerializer): snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail')

      class Meta:
          model = User
          fields = ('url', 'username', 'snippets')
```
注意：我們也增加了一個新的 'highlight' 字段。該字段與 url 字段相同類型。不過它指向了 'snippet-highlight'的 url pattern, 而非'snippet-detail' 的url pattern.

因為我們已經有一個 '.json'的後綴，為了更好的表明`highlight`字段鏈接的區別，使用一個 '.html' 的後綴。

## 4. 正確使用URL patterns

如果要使用超鏈接API，就必須確保正確的命名和使用 URL patterns. 我們來看看我們需要命名的 URL patterns：

    * 指向 'user-list' 和 'snippet-list' 的API根.
    * snippet的序列化器，包括一個 'snippet-highlight'字段.
    * user序列化器，包含一個 'snippet-detail'字段.
    * snippet 和user的序列化器，包含 'url' 字段（會缺省指向'snippet-detail' 和 'user-detail'.

一番工作之後，最終的 'urls.py' 文件應該如下所示：

    # API endpoints
    urlpatterns = format_suffix_patterns(patterns('snippets.views',
        url(r'^$', 'api_root'),
        url(r'^snippets/$',
            views.SnippetList.as_view(),
            name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$',
            views.SnippetDetail.as_view(),
            name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
            views.SnippetHighlight.as_view(),
            name='snippet-highlight'),
        url(r'^users/$',
            views.UserList.as_view(),
            name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$',
            views.UserDetail.as_view(),
            name='user-detail')
    ))

    # Login and logout views for the browsable API
    urlpatterns += patterns('',    
        url(r'^api-auth/', include('rest_framework.urls',
                                namespace='rest_framework')),
    ) 

## 5. 添加分頁

列表view有時會返回大量的實例結果，所以我們應該把結果分頁顯示，以便用戶使用。

通過在`settings.py` 中添加如下配置，我們就能在結果列表中增加分頁的功能:

    REST_FRAMEWORK ={'PAGINATE_BY':10}

請注意REST framework的所有配置信息都是存放在一個叫做 'REST_FRAMEWORK'的dictionary中，以便於其他配置區分。

如有必要，你也可以自定義分頁的方式，這裡不再贅述。

## 6. Browsing the API

If we open a browser and navigate to the browsable API, you'll find that you can now work your way around the API simply by following links.

You'll also be able to see the 'highlight' links on the snippet instances, that will take you to the highlighted code HTML representations.

In part 6 of the tutorial we'll look at how we can use ViewSets and Routers to reduce the amount of code we need to build our API.

