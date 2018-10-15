up vote
4
down vote

If you are using Vue, here's the way to go:

Vue.http.interceptors.push(function (request, next) {
request.headers['X-CSRF-TOKEN'] = Laravel.csrfToken;
next();
});

or

<script>
window.Laravel = <?php echo json_encode([
    'csrfToken' => csrf_token(),
]); ?>
</script>

shareimprove this answer
