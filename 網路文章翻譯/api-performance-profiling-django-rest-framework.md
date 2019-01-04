[原始出處](https://www.dabapps.com/blog/api-performance-profiling-django-rest-framework/)  

在討論Web服務的可伸縮性時，一些開發人員似乎傾向於過度關注正在使用的框架，這是基於錯誤的假設，即小的意味著更快，而不是考慮整個應用程序的架構。我們已經看到了一些開發人員在開始構建他們的API之前做出這些假設的案例，他們認為Django不能滿足他們的需求，或者決定不使用Django REST framework這樣的Web API框架，因為他們需要一些輕量級的東西。

我要說明的是，需要構建高性能API的Web API開發人員應該關注架構的一些核心基礎，而不是專注於他們正在使用的Web框架或API框架的原始性能。

使用一個成熟的、功能齊全的、支持良好的API框架會給您帶來巨大的好處，並且可以讓您將開發精力集中在真正重要的事情上，而不是耗費精力去重新發明輪子。

本文的大部分內容都是Django/Python的，但是一般的建議應該適用於任何語言或框架。

## 消除一些神話

在我們開始之前，我想先反駁一些我認為是錯誤的建議，這些建議是我們給那些編寫Web api的開發人員提供的，他們需要為每秒數千次的請求提供服務。

### 滾你自己的框架。

:請不要這樣做。如果您除了使用Django REST framework的plain APIView，而忽略了一般的視圖、序列化、路由器和其他高級功能之外，您還將為您自己提供一個比使用普通Django編寫API更大的優勢。你的服務將會是一個更好的網絡公民。

### 使用一個輕量級的框架。

不要合併一個框架，它的功能是緊密耦合的、整體的或緩慢的。REST framework包含大量的功能，但是核心視圖類非常簡單。

### REST framework被綁定到Django模型。

不。有一組預設的通用視圖，您可以很容易地使用Django模型和預設的序列化子類，它們與Django模型很好地工作，但是這些REST framework是完全可選的，並且與Django的ORM完全沒有緊密耦合。

### Django/Python/REST framework太慢。

正如本文所討論的，Web api最大的性能提升不是通過代碼調整，而是通過適當緩存數據庫查找、精心設計的HTTP緩存，以及在可能的情況下運行共享的服務器端緩存。

## 分析我們的觀點

我們將對一個簡單的Django REST framework API進行廣泛的分析，以瞭解它如何與使用普通Django視圖進行比較，並瞭解可以獲得最大性能改進的地方。

這裡使用的分析方法絕對不是一個全面的性能指標，而是給出了API中各個組件相對性能的高級概述。使用Django的開發服務器和本地PostgreSQL數據庫在我的筆記本上執行基準測試。

對於我們的測試用例，我們將分析一個簡單的API列表視圖。
```py
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'username', 'email')

class UserListView(APIView):
    def get(self, request):
        users = User.objects.all()
        serializer = UserSerializer(users)
        return Response(serializer.data)
```
這裡有我們感興趣的各種組件。

- 數據庫查詢。這裡，我們從Django ORM到原始數據庫訪問，都在計時。為了使其獨立於序列化，我們將在列表()中封裝queryset呼叫，以迫使它進行評估。
- Django請求/response 週期。在視圖方法運行之前或之後發生的任何事情。這包括預設的中間件、請求路由和在每個請求上進行的其他核心機制。我們可以通過將其連接到request_started和request_finished信號來計時。
- 序列化。將模型實例序列化為簡單的原生python數據結構所需的時間。我們可以將此作為序列化器實例化和.data訪問中發生的所有事情的時間。
- 視圖代碼。一旦有了APIView，就會呼叫任何東西。這包括REST framework的身份驗證、權限、節流、內容協商和請求/response 處理的機制。
- response 呈現。REST framework的response 物件是一個TemplateResponse類型，這意味著在視圖返迴response 之後，呈現過程就發生了。為了達到這個目的，我們可以包裝APIView。在超類中分派，在返回之前強制執行response 。

我們不會使用Python的剖析模組，而是要讓事情變得簡單，並將代碼的相關部分封裝在時間塊中。它是粗糙的並且已經準備好了(不用說，你不會用這種方法來衡量一個「真實的」應用程序)，但是它會完成這項工作。

對數據庫查找和序列化的計時涉及到對視圖的`.get()`方法的修改:
```py
def get(self, request):
    global serializer_time
    global db_time

    db_start = time.time()
    users = list(User.objects.all())
    db_time = time.time() - db_start

    serializer_start = time.time()
    serializer = UserSerializer(users)
    data = serializer.data
    serializer_time = time.time() - serializer_start

    return Response(data)
```
In order to time everything else that happens inside REST framework we need to override the .dispatch() method that is called as soon as the view is entered. This allows us to time the mechanics of the APIView class, as well as the response rendering.
```py
def dispatch(self, request, *args, **kwargs):
    global dispatch_time
    global render_time

    dispatch_start = time.time()
    ret = super(WebAPIView, self).dispatch(request, *args, **kwargs)

    render_start = time.time()
    ret.render()
    render_time = time.time() - render_start

    dispatch_time = time.time() - dispatch_start

    return ret
```
Finally we measure the rest of the request/response cycle by hooking into Django's request_started and request_finished signals.
```py
def started(sender, **kwargs):
    global started
    started = time.time()

def finished(sender, **kwargs):
    total = time.time() - started
    api_view_time = dispatch_time - (render_time + serializer_time + db_time)
    request_response_time = total - dispatch_time

    print ("Database lookup               | %.4fs" % db_time)
    print ("Serialization                 | %.4fs" % serializer_time)
    print ("Django request/response       | %.4fs" % request_response_time)
    print ("API view                      | %.4fs" % api_view_time)
    print ("Response rendering            | %.4fs" % render_time)

request_started.connect(started)
request_finished.connect(finished)
```
We're now ready to run the timing tests, so we create ten users in the database. The API calls are made using curl, like so:
`curl http://127.0.0.1:8000`
We make the request a few times, and take an average. We also discount the very first request, which is skewed, presumably by some Django routing code that only need to run at the point of the initial request.

Here's our first set of results.
Action 	Time (s) 	Percentage
Database lookup 	0.0090 	65.7%
Serialization 	0.0025 	18.2%
Django request/response 	0.0015 	10.9%
API view 	0.0005 	3.6%
Response rendering 	0.0002 	1.5%
Total 	0.0137 	
Removing serialization

Let's simplify things slightly. At the moment we're returning fully fledged model instances from the ORM, and then serializing them down into simple dictionary representations. In this case that's a little unnecessary - we can simply return simple representations from the ORM and return them directly.
```py
class UserListView(APIView):
    def get(self, request):
        data = User.objects.values('id', 'username', 'email')
        return Response(data)
```
Serializers are really great for providing a standard interface for your output representations, dealing nicely with input validation, and easily handling cases such as hyperlinked representations for you, but in simple cases like this they're not always necessary.

Once we've removed serialization, our timings look like this.
Action 	Time (s) 	Percentage
Database lookup 	0.0090 	80.4%
Django request/response 	0.0015 	13.4%
API view 	0.0005 	4.5%
Response rendering 	0.0002 	1.8%
Total 	0.0112 	

It's an improvement (we've shaved almost 20% off the average total request time), but we haven't dealt with the biggest issue, which is the database lookup.
Cache lookups

The database lookup is still by far the most time consuming part of the system. In order to make any significant performance gains we need to ensure that the majority of lookups come from a cache, rather than performing a database read on each request.

Ignoring the cache population and expiry, our view now looks like this:
```py
class UserListView(APIView):
    def get(self, request):
        data = cache.get('users')
        return Response(data)
```
We set up Redis as our cache backend, populate the cache, and re-run the timings.
Action 	Time (s) 	Percentage
Django request/response 	0.0015 	60.0%
API view 	0.0005 	20.0%
Redis lookup 	0.0003 	12.0%
Response rendering 	0.0002 	8.0%
Total 	0.0025 	

As would be expected, the performance difference is huge. The average request time is over 80% less than our original version. Lookups from caches such as Redis and Memcached are incredibly fast, and we're now at the point where the majority of the time taken by the request isn't in view code at all, but instead is in Django's request/response cycle.
Slimming the view

The default settings for REST framework views include both session and basic authentication, plus renderers for both the browsable API and regular JSON.

If we really need to squeeze out every last bit of possible performance, we could drop some of the unneeded settings. We'll modify the view to stop using proper content negotiation using the IgnoreClientContentNegotiation class demonstrated in the documentation, remove the browsable API renderer, and turn off any authentication or permissions on the view.
```py
class UserListView(APIView):
    permission_classes = []
    authentication_classes = []
    renderer_classes = [JSONRenderer]
    content_negotiation_class = IgnoreClientContentNegotiation

    def get(self, request):
        data = cache.get('users')
        return Response(data)
```
Note that we're losing some functionality at this point - we don't have the browsable API anymore, and we're assuming this view can be accessed publicly without any authentication or permissions.
Action 	Time (s) 	Percentage
Django request/response 	0.0015 	71.4%
Redis lookup 	0.0003 	14.3%
API view 	0.0002 	9.5%
Response rendering 	0.0001 	4.8%
Total 	0.0021 	

That makes some tiny savings, but they're not really all that significant.
### Dropping middleware

At this point the largest potential target for performance improvements isn't anything to do with REST framework, but rather Django's standard request/response cycle. It's likely that a significant amount of this time is spent processing the default middleware. Let's take the extreme approad and drop all middleware from the settings and see how the view performs.
Action 	Time (s) 	Percentage
Django request/response 	0.0003 	33.3%
Redis lookup 	0.0003 	33.3%
API view 	0.0002 	22.2%
Response rendering 	0.0001 	11.1%
Total 	0.0009 	

In almost all cases it's unlikely that we'd get to the point of dropping out Django's default middleware in order to make performance improvements, but the point is that once you're using a really stripped down API view, that becomes the biggest potential point of savings.
### Returning regular HttpResponses

If we're still need a few more percentage points of performance, we can simply return a regular HttpResponse from our views, rather than returning a REST framework Response. That'll give us some very minor time savings as the full response rendering process won't need to run. The standard JSON renderer also uses a custom encoder that properly handles various cases such as datetime formatting, which in this case we don't need.
```py
class UserListView(APIView):
    permission_classes = []
    authentication_classes = []
    renderer_classes = [JSONRenderer]
    content_negotiation_class = IgnoreClientContentNegotiation

    def get(self, request):
        data = cache.get('users')
        return HttpResponse(json.dumps(data), content_type='application/json; charset=utf-8')
```
This gives us a tiny saving over the previous case.
Action 	Time (s) 	Percentage
Django request/response 	0.0003 	37.5%
Redis lookup 	0.0003 	37.5%
API view 	0.0002 	25%
Total 	0.0008 	

The final step would be to stop using REST framework altogether and drop down to a plain Django view. This is left as an exercise for the reader, but hopefully by now it should be clear that we're talking about tiny performance gains that are insignificant in all but the most extreme use-cases.
Comparing our results

Here's the full set of results from each of our different cases.

The areas in pink/red tones indicate bits of REST framework, the blue is Django, and the green is the database or Redis lookup.
請記住，這些數字實際上只是為了粗略地瞭解各個組件的相對計時。具有更複雜的數據庫查找、分頁結果或寫操作的視圖將具有非常不同的特性。需要指出的重要一點是，數據庫查找是限制因素。

在我們的基本未優化的情況下，您的應用程序將以與其他簡單的Django視圖相同的速度運行。我們並沒有在這些設置上運行ApacheBench，但是您可以合理地期望能夠在一個適當的設置上實現每秒幾百個請求的速率。對於緩存的情況，您可能會看到更接近每秒幾千個請求。

上面這張圖上的甜蜜點顯然是左邊第三條。優化您的數據庫訪問是目前最重要的事情:在此之後，刪除框架的核心部分提供了遞減的性能回報。

下一步:ETags和網絡級緩存。

如果性能是您的API嚴重關注的問題，那麼您將希望停止關注優化Web服務器，轉而專注於獲取HTTP緩存。

HTTP緩存的潛力在不同的API之間會有很大的不同，這取決於使用模式和多少API是公開的，並且可以緩存到共享緩存中。

在上面的示例中，我們可以將API放在共享的服務器端緩存後面，比如清漆。如果用戶列表可以接受幾秒鐘的延遲，那麼我們可以設置適當的緩存頭，這樣緩存只會在一小段時間過期後重新驗證服務器的response 。

獲得正確的HTTP緩存並在共享的服務器端緩存後服務API會導致巨大的性能提升，因為絕大多數請求都是由緩存處理的。這取決於API顯示的可緩存屬性，比如公開，以及主要處理讀取請求，但如果有可能，收益可能是巨大的。像Varnish這樣的緩存可以愉快地處理每秒成千上萬的請求。

您可以使用Django REST framework來使用Django的預設緩存行為，但是即將發佈的REST framework2.4.0將包括更好的內置支持。

改進的範圍

與任何軟件一樣，總是有改進的餘地。

儘管數據庫查找將是大多數REST frameworkapi的主要性能問題，但是序列化速度可能會有所提高。在REST framework的核心部分，另一個可能被調整為小改進的領域是內容協商。

總結

那麼，帶回家的是什麼?

### 1。讓你的ORM查找正確。

由於數據庫查找是視圖中最慢的部分，所以您的ORM查詢是非常重要的。使用.select_related()和.prefetch_related()在必要時在泛型視圖的.queryset屬性上。如果您的模型實例很大，您還可以考慮使用延遲()或僅()來部分填充模型實例。

### 2。數據庫查找將成為瓶頸。

如果您的API正在努力處理請求的數量，那麼您的第一個代碼改進應該是適當地緩存模型實例。

您可能需要考慮使用Django緩存機器或Johnny Cache。或者，將緩存行為寫入視圖級別，以便緩存查找和序列化。

例如:
```py
class UserListView(APIView):
    """
    API View that returns results from the cache when possible.

    Note that cache invalidation would also be required.

    Invalidation should hook into the model save signal,
    or in the model `.save()` method.
    """
    def get(self, request):
        data = cache.get('users')
        if data is None:
            users = User.objects.all()
            serializer = UserSerializer(users)
            data = serializer.data
            cache.set('users', data, 86400)
        return Response(data)
```
### 3。有選擇地改進性能改進。

記住，過早的優化是萬惡之源。不要開始嘗試改進API的性能，直到您能夠開始分析API客戶端所展示的使用特性。然後有選擇地優化視圖，首先針對最關鍵的端點。

REST framework的優點之一是，因為它使用了常規的Django視圖，您可以輕鬆地開始優化和刪除單個視圖，同時還能獲得使用適當的API框架而不是使用普通Django的好處。

### 4。您並不總是需要使用序列化器。

對於性能關鍵視圖，您可以考慮將序列化器完全刪除，並在數據庫查詢中簡單地使用.values()。如果使用超鏈接表示或其他複雜資料欄位，您可能會發現在返迴response 之前，還需要對數據結構進行一些post處理。REST framework的設計是為了簡化這個過程。
最後的話

要構建高性能的Web api，要關注正確的事情。使用成熟的、支持良好的、經過測試的、功能齊全的Web API框架(如REST framework)，並致力於改進API的體系結構，而不是框架代碼的細節。對於序列化的模型實例的適當緩存和HTTP緩存和服務器端緩存的最優使用，將會看到在花費時間試圖精簡您正在使用的服務器框架時的巨大性能提升。

想瞭解更多嗎?DabApps提供Django REST framework支持和培訓，以及定製的API開發服務。要想瞭解更多，請聯繫。
