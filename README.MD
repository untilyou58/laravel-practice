# Laravel best practice

## Nguyên tắc trách nhiệm duy nhất(Single responsibility principle)

Một class và một method nên có duy nhất một trách nhiệm

Bad:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Good:

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

## Models béo, controllers gầy(Fat models, skinny controllers)

Đặt tất cả logic liên quan đến DB vào các Eloquent models hoặc vào các Repository classes nếu bạn đang sử dụng
Query Builder hoặc các truy vấn raw SQL.

Bad:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Good:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

## Validation

Di chuyển validation từ controllers sang các Request class.

Bad:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Good:

```php
public function store(PostRequest $request)
{
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

## Business logic phải ở trong service class (Business logic should be in service class)

Một controller phải có một trách nhiệm duy nhất, vì vậy hãy chuyển business logic từ các contoller sang các service class.

Bad:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }

    ....
}
```

Good:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

## Đừng lặp lại(mememememememememe) chính mình (DRY) (Don't repeat yourself (DRY))

Sử dụng lại code khi bạn có thể. SRP đang giúp bạn tránh trùng lặp. Ngoài ra, sử dụng lại các Blade templates, sử dụng Eloquent scopes, v.v.

Bad:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Good:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

## Ưu tiên sử dụng Eloquent hơn là sử dụng QUery Builder và các truy vấn raw SQL. Ưu tiên các collection hơn các array (Prefer to use Eloquent over using Query Builder and raw SQL queries. Prefer collections over arrays)

Eloquent cho phép bạn viết code đọc dễ  hiểu và bảo trì. Ngoài ra, Eloquent có các công cụ tích hợp tuyệt vời như xóa mềm(soft deletes), sự kiện(events), phạm vi(scopes), v.v.

Bad:

```php
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Good:

```php
Article::has('user.profile')->verified()->latest()->get();
```

## Gán giá trị hàng loạt (Mass assignment)

Bad:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

Good:

```php
$category->article()->create($request->validated());
```

## Không thực hiện truy vấn trong Blade view và sử dụng eager loading (N + 1 query)

Bad (Cho 100 người dùng, 101 truy vấn sẽ được thực hiện):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Good (cho 100 người dùng, 2 truy vấn sẽ được thực hiện):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

## Comment code, nhưng ưu tiên tên hàm và tên biến có ý nghĩa hơn comment

Bad:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Better:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

Good:

```php
if ($this->hasJoins())
```

## Không đặt JS và CSS trong các Blade view và không đặt bất kỳ HTML nào trong các PHP class

Bad:

```php
let article = `{{ json_encode($article) }}`;
```

Better:

```html
<input id="article" type="hidden" value='@json($article)'>
```

Hoặc

```html
<button class="js-fav-article" data-article='@json($article)'>{{ $article->name }}<button>
```

Trong một javascript file:

```js
let article = $('#article').val();
```

## Sử dụng các files config và language, const thay vì viết luôn vào trong code

Bad:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Good:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

##  Sử dụng các công cụ của Laravel được cộng đồng chấp nhận

Thích sử dụng các built-in và chức năng của Laravel tích hợp thay vì sử dụng các packages và công cụ của bên thứ 3.
Bất kỳ nhà phát triển nào sẽ làm việc với ứng dụng của bạn trong tương lai sẽ cần học các công cụ mới. Ngoài ra, cơ
hội nhận được sự giúp đỡ từ cộng đồng Laravel thấp hơn đáng kể khi bạn đang sử dụng gói hoặc công cụ của bên thứ 3.
Đừng bắt khách hàng của bạn trả tiền cho điều đó.

| Task | Standard tools | 3rd party tools |
|------|----------------|-----------------|
| Authorization|Policies|Entrust, Sentinel and other packages|
|Compiling assets|Laravel Mix|Grunt, Gulp, 3rd party packages|
|Development Environment|Homestead|Docker|
|Deployment|Laravel Forge|Deployer and other solutions|
|Unit testing|PHPUnit, Mockery|Phpspec|
|Browser testing|Laravel Dusk|Codeception|
|DB|Eloquent|SQL, Doctrine|
|Templates|Blade|Twig|
|Working with data|Laravel collections|Arrays|
|Form validation|Request classes|3rd party packages, validation in controller|
|Authentication|Built-in|3rd party packages, your own solution|
|API authentication|Laravel Passport|3rd party JWT and OAuth packages|
|Creating API|Built-in|Dingo API and similar packages|
|Working with DB structure|Migrations|Working with DB structure directly|
|Localization|Built-in|3rd party packages|
|Realtime user interfaces|Laravel Echo, Pusher|3rd party packages and working with WebSockets directly|
|Generating testing data|Seeder classes, Model Factories, Faker|Creating testing data manually|
|Task scheduling|Laravel Task Scheduler|Scripts and 3rd party packages|
|DB|MySQL, PostgreSQL, SQLite, SQL Server|MongoDB|

## Tuân theo quy ước đặt tên của Laravel

Theo chuẩn [PSR standards](https://www.php-fig.org/psr/psr-2/).

Ngoài ra, hãy làm theo các quy ước đặt tên được chấp nhận bởi cộng đồng Laravel như sau:

|What|How|Good|Bad|
|----|---|----|---|
|Controller|singular|ArticleController|ArticlesController|
|Route|plural|articles/1|article/1|
|Named route|snake_case with dot notation|users.show_active|users.show-active, show-active-users|
|Model|singular|User|Users|
|hasOne or belongsTo relationship|singular|articleComment|articleComments, article_comment|
|All other relationships|plural|articleComments|articleComment, article_comments|
|Table|plural|article_comments|article_comment, articleComments|
|Pivot table|singular model names in alphabetical order|article_user|user_article, articles_users|
|Table column|snake_case without model name|meta_title|MetaTitle; article_meta_title|
|Model property|snake_case|$model->created_at|$model->createdAt|
|Foreign key|singular model name with _id suffix|article_id|ArticleId, id_article, articles_id|
|Primary key|-|id|custom_id|
|Migration|-|2017_01_01_000000_create_articles_table|2017_01_01_000000_articles|
|Method|camelCase|getAll|get_all|
|Method in resource controller|table|store|saveArticle|
|Method in test class|camelCase|testGuestCannotSeeArticle|test_guest_cannot_see_article|
|Variable|camelCase|$articlesWithAuthor|$articles_with_author|
|Collection|descriptive, plural|$activeUsers = User::active()->get()|$active, $data|
|Object|descriptive, singular|$activeUser = User::active()->first()|$users, $obj|
|Config and language files index|snake_case|articles_enabled|ArticlesEnabled; articles-enabled|
|View|kebab-case|show-filtered.blade.php|showFiltered.blade.php, show_filtered.blade.php|
|Config|snake_case|google_calendar.php|googleCalendar.php, google-calendar.php|
|Contract (interface)|adjective or noun|AuthenticationInterface|Authenticatable, IAuthentication|
|Trait|adjective|Notifiable|NotificationTrait|

## Sử dụng cú pháp ngắn hơn và dễ đọc hơn nếu có thể

Bad:

```php
$request->session()->get('cart');
$request->input('name');
```

Good:

```php
session('cart');
$request->name;
```

More examples:

|Common syntax|Shorter and more readable syntax|
|-------------|--------------------------------|
|Session::get('cart')|session('cart')|
|$request->session()->get('cart')|session('cart')|
|Session::put('cart', $data)|session(['cart' => $data])|
|$request->input('name'), Request::get('name')|$request->name, request('name')|
|return Redirect::back()|return back()|
|is_null($object->relation) ? null : $object->relation->id|optional($object->relation)->id|
|return view('index')->with('title', $title)->with('client', $client)|return view('index', compact('title', 'client'))|
|$request->has('value') ? $request->value : 'default';|$request->get('value', 'default')|
|Carbon::now(), Carbon::today()|now(), today()|
|App::make('Class')|app('Class')|
|->where('column', '=', 1)|->where('column', 1)|
|->orderBy('created_at', 'desc')|->latest()|
|->orderBy('age', 'desc')|->latest('age')|
|->orderBy('created_at', 'asc')|->oldest()|
|->select('id', 'name')->get()|->get(['id', 'name'])|
|->first()->name|->value('name')|

## Sử dụng IoC container hoặc facade thay vì Class mới

Cú pháp new Class tạo ra sự kết hợp chặt chẽ giữa các class và làm phức tạp việc kiểm tra.
Sử dụng IoC container hoặc facade thay thế. Khó hiểu nhìn code :v

Bad:

```php
$user = new User;
$user->create($request->validated());
```

Good:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->validated());
```

## Không lấy dữ liệu trực tiếp từ file .env

Đưa dữ liệu vào file config thay vào đó, rồi sau đó dùng chức năng helper `config()` để sử dụng dữ liệu trong ứng dụng

Bad:

```php
$apiKey = env('API_KEY');
```

Good:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

## Chứa ngày tháng trong format tiêu chuẩn. Sử dụng accessors và mutators để sửa đổi định dạng ngày

Bad:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Good:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at'];
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

## Những best practices khác

- Không bao giờ đẩy logic vào file routes
- Giảm thiểu việc sử dụng Vanilla PHP trong Blade templates.
(Thay vì viết {{ csrf_token() }} hoặc @csrf bạn lại đi viết `<?php echo csrf_field(); ?>`)

Source: [laravel-best-practice](https://github.com/alexeymezenin/laravel-best-practices)
