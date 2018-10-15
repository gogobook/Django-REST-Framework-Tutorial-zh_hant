X-CSRF-TOKEN

除了檢查 POST 參數中的 CSRF token 外，VerifyCsrfToken 中間件還會檢查 X-CSRF-TOKEN 請求頭。你可以將令牌保存在 HTML meta 標籤中：

<meta name="csrf-token" content="{{ csrf_token() }}">

一旦創建了 meta 標籤，你就可以使用類似 jQuery 的庫將令牌自動添加到所有請求的頭信息中。這可以為您基於 AJAX 的應用提供簡單、方便的 CSRF 保護。
```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```
X-XSRF-TOKEN

Laravel 將當前的 CSRF 令牌存儲在由框架生成的每個響應中包含的一個XSRF-TOKEN cookie 中。你可以使用該令牌的值來設置 X-XSRF-TOKEN 請求頭信息。

這個 cookie 作為頭信息發送主要是為了方便，因為一些 JavaScript 框架，如 Angular，會自動將其值添加到 X-XSRF-TOKEN 頭中.