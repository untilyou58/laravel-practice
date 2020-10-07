# Prevent fat controller

## Introduction

### What is Fat Controller?

To put it simply, it has a lot of lines (I personally feel Fat when one method exceeds 50 lines in NCLOC with Controller, how about you), and because of the large number of lines, it is processed. I think it's a class that is difficult to follow and often causes problems. In this article, not only the number of lines is large, but also the effect of one change on other parts is detected due to overlapping processing in multiple methods or conditional branching in the method. I'm assuming a controller that is in a difficult state.

Để đơn giản trong việc định nghĩa là nó có rất nhiều dòng > 50 lines trong controller (với tui là thế). Trong bài viết này, không chỉ có số lượng dòng lớn mà còn có sự ảnh hưởng của một thay đổi trên các phần khác do xử lý chồng chéo trong nhiều phương pháp hoặc phân nhánh có điều kiện trong phương pháp

Even if it is a huge class, if the data and functions contained in it are highly aggregated and lowly coupled, there will be no problem. I don't think the important thing is that it "causes a glitch", not that every huge class is evil.

Ngay cả nếu nó là một class lớn, nếu dữ liệu và chức năng chứa nót trong sự tổng hợp cao và có sự liên kết với nhau thấp

## Problems with Fat Controller

- [Methods that are too long] Each method is long and it is difficult to follow the processing flow. As a result, local changes can affect the entire process, which can lead to unexpected glitches.
- [Duplicate processing] Due to duplication of processing between controllers, code changes due to specification changes may be omitted, causing unexpected problems.
- [Violation of the Single Responsibility Principle] By processing multiple patterns with the same Controller, changes related to one pattern may affect other patterns, which may cause unexpected problems.

The main reason for becoming a Fat Controller is that the Model layer is too poor. The processing that should be written in the Model is written in the Controller, resulting in poor readability, changes to one use case affect other use cases, and mass production of copy code between Controllers. Or something like that happens.

## 5 Tips to Prevent Fat Controller

1. Write the function that processes the request data in FormRequest
2. Write the function that processes the response data in ViewModel or Resource
3. Make it a single action controller
4. Separate into multiple Controllers
5. Use UseCaseInteractor

### 1. Write the function that processes the request data in FormRequest

```php
public function search(Request $request) {
    if (isset($request->query('name'))) {
        $whereParams['name'] = $request->query('name');
    }
    return User::where($whereParams)->get();
}
```

It is a pattern in which processing branches depending on the presence or absence of request parameters such as. There is only one example above, but it becomes much harder to read if there are a lot of them or if the conditions in the if statement become complicated.

Therefore, these processes are moved to the FormRequest side.

```php
// Controller
public function search(SearchUserRequest $request) {
    return User::where($request->filters())->get();
}

// FormRequest
public function filters(): array {
    $filters = [];
    if (isset($this->query('name'))) {
        $filters['name'] = $this->query('name');
    }
    return $filters;  
}
```

You might think that you just moved the process to another class, but the advantage of moving to FormRequest is that you can only focus on building the input parameters to pass to the model, for example. Since the validation rule is written in FormRequest, I think that it is difficult for omission of change to occur if only FormRequest needs to be changed when the input parameters increase. Also, if there are changes that process multiple models in succession, it will be easier to distinguish which model is related to which parameter.

```php
public function search(SearchUserRequest $request) {
    $users = User::where($request->userFilters())->get();
    $anotherModels = AnotherModels
        ::whereIn('user_id', $users->pluck('id'))
        ->where($request->anotherFilters())
        ->get();
    return ['users' => $users, 'anotherModels' => $anotherModels];
}
```

### 2. Write the function that processes the response data in ViewModel or Resource

```php
public function index() {
    $users = User::where(...)->get();
    foreach ($users as $user) {
        $user->full_name = $user->family_name . ' ' . $user->given_name;
    }
    return view('user.index', compact('users'));
}
```

I think it would be better to do the above example with Accessor, but please forgive me because I couldn't come up with a good example.

The point is that the return value from the Model received by the Controller must be processed before being passed to the View.

```php
// Controller
public function index() {
    $users = User::where(...)->get();
    $viewModel = new UsersViewModel($users);
    return view('user.index', ['users' => $viewModel->users()]);
}

// ViewModel
public function __construct(array $users) {
    $this->users = $users;
}
public function users(): array {
    return array_map([$this, 'transform'], $this->users);
}
private function transform(User $user): array {
    return [
        'id'   => $user->id,
        'name' => $user->family_name . ' ' . $user->given_name,
    ];
}
```

The transform can return an object. If you want to rewrite the conversion logic, you can add flexibility by allowing it to be replaced on the Controller side.

```php
// Controller
$viewModel = new UsersViewModel($users, new SummaryUserTransformer);

// ViewModel
public function __construct(array $users, UserTransformerInterface $transformer = null) {
    $this->users = $users;
    $this->transformer = $transformer ?? new DefaultUserTransformer;
}
public function users(): array {
    return array_map($this->transformer, $this->users);
}

// Transformer
public function __invoke(User $user) {
  return [
      // ...
  ];
}
```

### 3. Make it a single action controller

A single-action controller is a `__invoke` Controller class that has methods and is paired with a particular routing.

There is a feature in how to write the routing.

Usually `Route::get('/', 'HomeController@index')` as `@` the class name in front of, you write the method after, in the case of single-action controller `Route::get('/', 'HomeController')` (automatically at the time of the call and specify only the class name, as `__invoke` you are called).

For the so-called CRUD process, you can select the required method from index/create/store/show/edit/update/delete and use it, but if you use a single action controller for the other methods, I think the class will be refreshing.

### 4. Separate into multiple Controllers

Combining multiple different use cases into one action method tends to be a Fat Controller because it requires conditional branching by pattern.

If there is a conditional branch in the Controller that divides the process according to the request, it is possible that multiple different use cases have been combined, so consider whether it can be separated. 

```php
public function __invoke(SearchUserRequest $request) {
    if ($request->is_special) {
        $users = User::searchSpecial($request->specialFilters())->get();
    } else {
        $users = User::search($request->filters())->get();
    }
    // ...
}
```

It separates this into two Controllers.

```php
// SpecialContext\SearchUserController
public function __invoke(SearchUserRequest $request) {
    $users = User::searchSpecial($request->filters())->get();
    // ...
}
// GenericContext\SearchUserController
public function __invoke(SearchUserRequest $request) {
    $users = User::search($request->filters())->get();
    // ...
}
```

If necessary, you may also separate the Model.

```php
// SpecialContext\SearchUserController
// use SpecialContext\User
public function __invoke(SearchUserRequest $request) {
    $users = User::search($request->filters())->get();
    // ...
}
// GenericContext\SearchUserController
// use GenericContext\User
public function __invoke(SearchUserRequest $request) {
    $users = User::search($request->filters())->get();
    // ...
}
```

It's okay to say that the classes are the same and "separate into different action methods", but if you make the rule that you want to use a single action controller except for CRUD, you will have to separate the Controllers.

### 5. Use UseCaseInteractor

UseCaseInteractor is a part of clean architecture, a class that receives input (request) to application, performs core processing, and generates output to view (or API response).

In the above example, UseCaseInteractor does not return a value and passes it directly to the property of Middleware, but in this article, it is returned as it is (the name is also abbreviated as UseCase).

```php
// Controller
public function __invoke(SearchUserUseCase $useCase, SearchUserRequest $request) {
    $users = $useCase->invoke($request->filters());
    $responder = new UsersResponder($users);
    return $responder->createResponse();
}
// UseCase
public function invoke(array $filters) {
    return User::where($filters)->get();
}
// Responder with Blade view
public function createResponse() {
    $viewModel = new UsersViewModel($this->users, new SummaryUserTransformer);
    return view('user.index', ['users' => $viewModel->users()]);
}
// Responder with API resource
public function createResponse() {
    return UserResource::collection($this->users);
}
```

You can keep the Controller slim by confining the process inside the UseCase. As for the data to be passed to this class, it is better to convert the information in the HTTP world into primitive data or domain object.

[source](https://qiita.com/nunulk/items/6ed409345efb6ee4f660)

## Another tip

### 1. Process to update the state of Model, 2. Conditional branching according to the state of Model

The easiest way is to set the properties one by one when creating the following objects. It is effective to put them together because the number of property divisions will increase.

```php
$user = new User();
$user->name = $request->name;
$user->email = $request->email;
// ....
$user->save();
```

Rather, it is better to set the validation rules appropriately and summarize as follows.

```php
User::create($request->all());
```

If you need to convert the data, you can create a function on the FormRequest side and let it do it.

An example closer to the domain is, for example, when there is a rule that "the initial state of a task is'todo'" in the process of creating a task in a TODO application.

```php
$task = Task::create(['status' => 'todo'] + $request->all());
```

You can create a dedicated method in Model and have it in it as shown below.

```php
// Controller
$task = Task::createTodo($request->all());

// Model
public static function createTodo(array $attributes) {
    return parent::create([
        'status' => 'todo',
    ] + $attributes);
}
```

The number of lines itself does not decrease, but if you look at the code of the Controller, it will be easier to identify because there is a name specific to the role of "creating a TODO".

Also, when the TODO application has a domain rule that "only completed tasks can be deleted",

```php
if ($task->is_complete) {
    $task->delete();
}
```

Rather than writing like

```php
// Controller
$task->delete();

// Model
public function delete() {
    if ($this->is_complete) {
        parent::delete();
    }
    throw new \DomainException('...');
}
```

This reduces the number of lines (this has a side effect, and making the deleteable condition more methodatic makes it easier to deal with when the condition changes). I think it depends on the requirements of the application how to behave when the condition is not met, but you may need a flow that returns 422 Unprocessable Entity and 404 Not Found in Controller.

Currently, there is only one property, so the benefit is small, but if the conditions become complicated, it will be easier to prevent code duplication (and update omission due to it) by confining it in Model.

Personally, I think that it is okay to judge whether a certain process is satisfied by validation, but I think that it is better that the process to be executed and the executable condition are close, so above In the example of, it is put on the Model side (If this affects whether or not the entire process that spans multiple models can be executed depends on the state of the input parameters for the Controller, I think it is better to implement it on the validation side)

### 3. Query building

The combination of simple WHERE and ORDER BY can be left as it is, but I think it is better to make it into a method and put it in Model if there is a complicated conditional clause. It's a little difficult to write an example, but for example, if the query matches multiple conditions including subtasks, the query building will continue as shown below.

```php
$tasks = Task
    ::whereHas('sub_tasks', function ($builder) {
        $builder->where('...')
        // There are another conditions
        ;
    })
    ->where('...')
    // There are other conditions as well
;
```

Make it a scope like.

Or you can make it a normal method and call it.

```php
// Controller
$tasks = Task::getInSomeState();

// Model
public static function getInSomeState() {
    return static::whereHas(...)->where(...)->get();
}
```

Personally, I prefer to scope it, but there are also people who want to avoid problems with completion and jumps in the IDE (it is recognized by writing in PHPDoc, but it is not possible to jump to the definition source). I think that there. You can go there as you like.

## Bonus

```php
public function update(Request $request)
{
    $name = $request->name
    $status = $request->status
    $a = $request->a
    $b = $request->b
    $c = $request->c
    .
    .
    .

    $tasks = Task
        ::whereHas('sub_tasks', function ($builder) {
            $builder->where('...')
            // conditions
            ;
        })
        ->where('name', $name)
        ->where('status', $status)
        ->where('a', $a)
        ->where('b', $b)
        ->where('c', $c)
        .
        .
        .
    ;
```

There are two patterns, one is to scope each one separately and the other is to put them together.

Dismembered

```php
// Controller
$tasks = Task::hasA($a)->hasB($b)->get();

// Model
public function scopeHasA(Builder $builder, $a = null) {
    if (!$a) {
        return $builder;
    }
    return $builder->where('a', $a);
}
public function scopeHasB(Builder $builder, $b = null) {
    if (!$b) {
        return $builder;
    }
    return $builder->where('b', $b);
}
```

collect

```php
public function scopeMatched(Builder $builder, array $filters) {
        return $builder
            ->when(Arr::has($filters, 'a'), function ($query) use ($filters) {
                return $query->where('a', $filters['a']);
            })
            ->when(Arr::has($filters, 'b'), function ($query) use ($filters) {
                return $query->where('b', $filters['b']);
            })
            ->where('c', $filters['c'])
        ;
}
```

Which one to use is on a case-by-case basis, but if the individual extraction conditions are used independently, it is better to separate them.

It is also possible to use the ones defined separately, so I think it is better to use them properly according to the situation.

```php
// Controller
$importantTasks = Task::important()->get();

// Task Model
public function scopeNotCompleted(Builder $builder) {
    return $builder->whereIn('status', ['todo', 'doing']);
}

public function scopeExpired(Builder $builder) {
    return $builder->where('deadline', '<', Carbon::now();
}

public function scopeImportant(Builder $builder) {
    return $builder->notCompleted()->expired();
}
```