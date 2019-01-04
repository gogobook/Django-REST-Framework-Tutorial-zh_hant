https://hk.saowen.com/a/1f9937358ac81e71a88bf6e2fb023f6f74e10a3ee5f1a66ceacc168d72e0025f
# vue-resource pos提交t數據時碰到Django csrf
2017-08-04 / views: 69

　　最近在用Vue寫前端代碼，再用vue-resource向後台提交數據。項目後台是用python+Django開發的。下面我就覆盤一下我出現問題的經過。

　　首先，想用vue進行數據交互只能引入vue-resource。

<script src="js/vue.js"></script>
<script src="js/vue-resource.js" ></script>

　　之後我使用vm.$http方法向後台發送數據，其實和jquery的AJAX方法差不多，需要定義發送的地址、請求類型和發送數據等。
```js
this.$http({
    url:'/data/',
    data:JSON.stringify(Strdata),
    method:'POST'
}).then(function(res){
    alert(res.data);                    
});
```
由於POST請求需要修改發送date的Content-Type為application/json，所以要設置emulateJSON為true，就變成這樣了
```js
this.$http({
    url:'/data/',
    data:JSON.stringify(Strdata),
    method:'POST',
    emulateJSON:true
}).then(function(res){
    alert(res.data);                    
});
```
正常來説這樣就OK了，就可以拿到後台數據了。之後打開控制枱，查看network，找到發出的data,看到HTTP狀態碼為302。

怎麼會被重定向到了登錄頁面，難道是登錄超時？？

 

然後我看了看後台服務.....

不應該是登錄的問題啊，後面有請求成功返回啊，於是我對比了一下成功的請求和失敗請求的HTTP請求頭，發現好像是少了一個叫X-CSRFToken的東西。這是什麼東西呢，於是我就google了一下，得到如下答案：

    Django 提供的 CSRF 防護機制

    django 第一次response 來自某個客户端的請求時，會在服務器端隨機生成一個 token，把這個 token 放在 cookie 裏。然後每次 POST 請求都會帶上這個 token，

    這樣就能避免被 CSRF 攻擊。

        在返回的 HTTP response 的 cookie 裏，django 會為你添加一個 csrftoken 字段，其值為一個自動生成的 token
        在所有的 POST 表單時，必須包含一個 csrfmiddlewaretoken 字段 （只需要在模板里加一個 tag， django 就會自動幫你生成，見下面）
        在處理 POST 請求之前，django 會驗證這個請求的 cookie 裏的 csrftoken 字段的值和提交的表單裏的 csrfmiddlewaretoken 字段的值是否一樣。如果一樣，則表明這是一個合法的請求，否則，這個請求可能是來自於別人的 csrf 攻擊，返回 403 Forbidden.
        在所有 ajax POST 請求裏，添加一個 X-CSRFTOKEN header，其值為 cookie 裏的 csrftoken 的值。

 也就是説我每次向後台發送POST請求的時候，Django為了防止跨站請求偽造，即csrf攻擊，提供了CsrfViewMiddleware中間件來防禦csrf攻擊。這時我就知道為什麼之前將請求方式改為GET後成功了。

之後就想辦法在HTTP請求頭中設置X-CSRFToken了，我查了很多資料，看到最多的一種方法是這樣：

```html
<meta id="token" name="token" value="{ csrf_token() }">
```
```js
Vue.http.headers.common['X-CSRFToken'] = document.querySelector('#token').getAttribute('value');
```
但是我試過還是重定向到登錄頁，不知道為什麼。之後想了想，實際就是獲取cookie裏面的csrftoken值，然後在賦值給HTTP請求頭裏面的X-CSRFToken就行了。
```js
function getCookie(name) {
    var cookieValue = null;
     if (document.cookie && document.cookie != '') {
        var cookies = document.cookie.split(';');
    for (var i = 0; i < cookies.length; i++) {
        var cookie = jQuery.trim(cookies[i]);
        if (cookie.substring(0, name.length + 1) == (name + '=')) {
        cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
        break;
        }
    }
    }
    return cookieValue;
}

Vue.http.headers.common['X-CSRFToken'] = getCookie('csrftoken');            
```
之後HTTP請求頭上的X-CSRFToken就有值了，response 也就成功了。

