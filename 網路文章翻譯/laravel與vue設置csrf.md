https://laravel-china.org/articles/3967/two-methods-of-setting-up-x-csrf-token-in-laravel54-vue-framework

Laravel5.4 Vue 框架中 X-CSRF-TOKEN 的兩種設置方法

創建於 1年前 / 閱讀數 9296 / 評論數 8

Laravel5.4中，Vue框架引用了Axios HTTP 庫來處理後端請求，提供了一種比Jquery更輕量的解決方案，新框架也試圖讓一些操作更簡潔和統一，比如常用的 X-CSRF-TOKEN令牌設置。

第一次使用，通常會遇到一個X-CSRF-TOKEN的錯誤，編譯完成，前端頁面啟動過程拋異常：
```sh
Uncaught TypeError: Cannot read property 'csrfToken' of undefined
    at Object.<anonymous> (app.js:40921)
    at __webpack_require__ (app.js:20)
    at Object.<anonymous> (app.js:11198)
    at __webpack_require__ (app.js:20)
    at Object.<anonymous> (app.js:40943)
    at __webpack_require__ (app.js:20)
    at app.js:66
    at app.js:69
```
查看編譯後的app.js，發現有一段對X-CSRF-TOKEN的處理，默認是從window.Laravel中獲取：
```js
window.axios.defaults.headers.common = {
  'X-CSRF-TOKEN': window.Laravel.csrfToken,
  // 'X-CSRF-TOKEN':document.querySelector('meta[name="csrf-token"]').getAttribute('content'),
  'X-Requested-With': 'XMLHttpRequest'
};
```
這個變量沒有設置，因此報錯。

順藤摸瓜，查源頭，入口文件app.js 引用了一個資源文件 resources/assets/js/bootstrap.js，其中對X-CSRF-TOKEN有定義
```js
window.axios = require('axios');

window.axios.defaults.headers.common = {
    'X-CSRF-TOKEN': window.Laravel.csrfToken,
    'X-Requested-With': 'XMLHttpRequest'
};
```
正是這段處理，導致了前端頁面的報錯

其實，框架應該是有意為之，在初次調試的時候報錯提示可以在這裡統一處理X-CSRF-TOKEN，但方式其實可以做得更優雅，應該在文檔中給出最佳實踐。

這裡給出兩種方案，大家可以按照實際情況採用：
一、從meta標記中獲取（推薦）

因為很多時候中間件也會檢查CSRF-TOKEN，所以通常我們都習慣在模板文件中設置meta標記來保存CSRF-TOKEN，因此從這裡統一獲取，處理方式一致

`<meta name="csrf-token" content="{{ csrf_token() }}">`

因此，修改bootstrap.js對應代碼為：
```js
window.axios = require('axios');

window.axios.defaults.headers.common = {
    'X-CSRF-TOKEN':document.querySelector('meta[name="csrf-token"]').getAttribute('content'),
    'X-Requested-With': 'XMLHttpRequest'
};
```
這樣的修改，通用性也更好。
二、直接在頁面設置

當然，如果使用通用模板，有些頁面可能meta沒有單獨設置csrf-token標記，這種情況下，直接在頁面用js腳本設置變量值也是一個辦法，注意變量名和bootstrap.js中定義的保持一致
```php
 window.Laravel = <?php echo json_encode([
              'csrfToken' => csrf_token(),
          ]); ?>
```
其他方式：

通過命令行 php artisan make:auth 可以自動生成 layouts模板文件：app.blade.php，裡面包含了以上兩種方法。
這個模板中還有其他關於登錄態驗證的最佳實踐，大家可以參考（感謝 @yanyin 提供tips）。

隨著Laravel版本的迭代，更多前端開發流程框架被引入，節省了大量配置、調試時間。對比最近折騰webpack+vue遇到的各種坑，這種框架本身自帶的流程框架確實非常有效率。
