## Basic Controllers

### What is a Controller?

A controller is a class associated with one or more routes and is responsible for handling requests for those routes. Controllers organize related request handling logic together. For example:
- `ProductController` should contain only product-related logic
- `CarController` should contain only car-related logic

This separation keeps your code organized and maintainable.

### Generate Controller with Artisan

```bash
php artisan make:controller CarController
```

This generates `CarController.php` in the `app/Http/Controllers` folder with the following default content:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class CarController extends Controller
{
    //
}
```

### Create Route and Associate Action

Open `CarController` and add an `index` method:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class CarController extends Controller
{
    public function index()
    {
        return "Index method from CarController";
    }
}
```

Open `routes/web.php` and add a route associated with the `index` method:

```php
Route::get("/car", [\App\Http\Controllers\CarController::class, 'index']);
```

You can define other methods in `CarController` the same way and associate them with new routes.

**Note:** Controller methods that are associated with routes are often called **actions**.

Access http://127.0.0.1:8000/car in your browser to see the output:

```
Index method from CarController
```

## Group Actions by Controller

If you want to associate multiple methods of your controller with routes, you can specify the controller name only once using route groups:

```php
Route::controller(\App\Http\Controllers\CarController::class)->group(function () {
    Route::get("/car", "index");
    Route::get("/my-cars", "myCars");
    Route::get("/search", "search");
});
```

This approach reduces repetition and makes your route definitions cleaner.

## Single Action Controllers

If a controller grows too large, it's recommended to split it into multiple smaller controllers or create **Single Action Controllers**.

**Single Action Controllers** are controllers associated with a single route only.

### Generate Single Action Controller

Execute the following artisan command:

```bash
php artisan make:controller ShowCarController --invokable
```

This generates `ShowCarController.php` in `app/Http/Controllers` with the following content:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class ShowCarController extends Controller
{
    /**
     * Handle the incoming request.
     */
    public function __invoke(Request $request)
    {
        //
    }
}
```

The `__invoke` method is executed when the route is matched. Associate the controller with a route like this:

```php
Route::get("/car", \App\Http\Controllers\ShowCarController::class);
```

Notice that you only provide the controller class name - no method name is needed for **Single Action Controllers**.

### Using __invoke in Regular Controllers

A regular controller can also have an `__invoke` method alongside other methods. You can modify `CarController`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class CarController extends Controller
{
    public function __invoke(Request $request)
    {
        return "From __invoke";
    }

    public function index()
    {
        return "Index method from CarController";
    }
}
```

In routes, you can use `CarController` both ways:

```php
Route::get("/car", [CarController::class, 'index']);
Route::get("/car/invokable", CarController::class);
```

For `/car` you use a regular method, but for `/car/invokable` you pass the entire controller class name and Laravel automatically executes the `__invoke` method.

## Resource Controllers

A resource controller provides a convenient way to handle typical CRUD (Create, Read, Update, Delete) operations for a resource such as a database table.

### Generate Resource Controller

Execute the following command:

```bash
php artisan make:controller ProductController --resource
```

This creates a controller class with predefined methods to handle various HTTP verbs and corresponding actions:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class ProductController extends Controller
{
    /**
     * Display a listing of the resource.
     */
    public function index()
    {
        //
    }

    /**
     * Show the form for creating a new resource.
     */
    public function create()
    {
        //
    }

    /**
     * Store a newly created resource in storage.
     */
    public function store(Request $request)
    {
        //
    }

    /**
     * Display the specified resource.
     */
    public function show(string $id)
    {
        //
    }

    /**
     * Show the form for editing the specified resource.
     */
    public function edit(string $id)
    {
        //
    }

    /**
     * Update the specified resource in storage.
     */
    public function update(Request $request, string $id)
    {
        //
    }

    /**
     * Remove the specified resource from storage.
     */
    public function destroy(string $id)
    {
        //
    }
}
```

Resource controllers let you write cleaner code for CRUD operations without defining each route and controller action individually. They integrate seamlessly with Laravel's route model binding, making it easy to work with model instances directly in controller methods.

### Use Resource Controller in Route

```php
Route::resource('/products', \App\Http\Controllers\ProductController::class);
```

This single line registers 7 routes. Inspect them by executing `php artisan route:list` in your terminal. You'll see routes for:
- GET `/products` - index
- GET `/products/create` - create
- POST `/products` - store
- GET `/products/{product}` - show
- GET `/products/{product}/edit` - edit
- PUT/PATCH `/products/{product}` - update
- DELETE `/products/{product}` - destroy

### Partial Resource Routes

To exclude certain methods from resource routes, use the `except` method:

```php
Route::resource('/products', \App\Http\Controllers\ProductController::class)
    ->except(['destroy']);
```

To include only specific routes, use the `only` method:

```php
Route::resource('/products', \App\Http\Controllers\ProductController::class)
    ->only(['index', 'show']);
```

### API Resource Routes

For API routes that don't need HTML forms, exclude `create` and `edit` routes using `apiResource`:

```php
Route::apiResource('/products', \App\Http\Controllers\ProductController::class);
```

This registers only 5 routes (excluding `create` and `edit`).

To skip generating `create` and `edit` methods in the controller, use the `--api` flag:

```php
php artisan make:controller CarController --api
```

You can register multiple API resources at once:

```php
Route::apiResources([
    '/products' => \App\Http\Controllers\ProductController::class,
    '/cars' => \App\Http\Controllers\CarController::class
]);
```

## Challenge

### Problem

Generate a controller with 2 methods: `sum` and `subtract`. Define routes `/sum` and `/subtract` associated with the corresponding controller methods.

Both methods should accept two arguments (both must be numbers). Accessing `/sum` should print the sum of the arguments, and accessing `/subtract` should print the difference.

### Solution

Execute the following artisan command to create `MathController`:

```bash
php artisan make:controller MathController
```

Open the generated `MathController` and add the `sum` and `subtract` methods:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class MathController extends Controller
{
    public function sum(float $a, float $b)
    {
        return $a + $b;
    }

    public function subtract(float $a, float $b)
    {
        return $a - $b;
    }
}
```

Open `routes/web.php` and define the routes:

```php
Route::get('/sum/{a}/{b}', [MathController::class, 'sum'])
    ->whereNumber(['a', 'b']);
Route::get('/subtract/{a}/{b}', [MathController::class, 'subtract'])
    ->whereNumber(['a', 'b']);
```

The `whereNumber` constraint ensures both parameters are numeric values.

**Testing:**
- Access http://localhost:8000/sum/5/8 - you should see **13**
- Access http://localhost:8000/subtract/5/8 - you should see **-3**
- Access http://localhost:8000/sum/5/abc - you should see **404 Not Found Page** (because 'abc' is not a number)
