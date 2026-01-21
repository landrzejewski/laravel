## Basic Routing

The default routes file is located in `routes/web.php`.

### Available Request Methods

Laravel supports the following request methods when defining routes:

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

### Route That Matches Multiple Request Methods

```php
Route::match(['get', 'post'], '/', function () {
    // ...
});
```

### Route That Matches All Request Methods

```php
Route::any('/', function () {
    // ...
});
```

### Redirect Routes

```php
Route::redirect('/home', '/');
```

You can specify the HTTP status code for the redirect:

```php
Route::redirect('/home', '/', 301);
// The same as:
Route::permanentRedirect('/home', '/');
```

### View Routes

When you only need to return a view without any additional logic, you can use view routes:

```php
Route::view('/contact', 'contact');

// Pass data to the view as the third parameter
Route::view('/contact', 'contact', ['phone' => '+995557123456']);
```

## Required Parameters

This code defines a route with a required parameter `id`:

```php
Route::get("/product/{id}", function(string $id) {
    return "Product ID=$id";
});
```

Matches the following routes:
- `/product/1`
- `/product/test`
- `/product/{any_string}`

You can define multiple route parameters:

```php
Route::get("{lang}/product/{id}", function(string $lang, string $id) {
    return "Product Lang=$lang. ID=$id";
});
```

Matches:
- `/en/product/1`
- `/ka/product/test`
- `/{any_string}/product/{any_string}`

Multiple parameters example:

```php
Route::get("{lang}/product/{id}/review/{reviewId}", function(string $lang, string $id, string $reviewId) {
    return "Product Reviews Lang=$lang. ID=$id. ReviewID=$reviewId";
});
```

Matches:
- `/en/product/1/review/123`
- `/ka/product/test/review/foo`
- `/{any_string}/product/{any_string}/review/{any_string}`

## Route Optional Parameters

**Important:** You must make the parameter in the callback function optional with a default value. Otherwise the route will fail when the parameter is not provided.

```php
Route::get('/posts/{category?}', function(string $category = null) {
    // ...
    return "Render Posts";
});
```

This route will match both `/posts` and `/posts/technology`.

## Route Parameter Validation

You can define the expected format for each route parameter.

### whereNumber() - Only Digits

```php
Route::get("/product/{id}", function(string $id) {
    // ...
})->whereNumber("id");
```

Matches:
- `/product/1`
- `/product/01234`
- `/product/56789`

**Does not match:**
- `/product/test`
- `/product/123test`
- `/product/test123`

### whereAlpha() - Only Letters

The `username` parameter must only contain uppercase and lowercase letters:

```php
Route::get('/user/{username}', function(string $username) {
    // ...
})->whereAlpha('username');
```

Matches:
- `/user/zura`
- `/user/thecodeholic`
- `/user/ZURA`

**Does not match:**
- `/user/zura123`
- `/user/123ZURA`

### whereAlphaNumeric() - Letters and Digits

The `username` parameter must only contain uppercase, lowercase letters and digits:

```php
Route::get('/user/{username}', function(string $username) {
    // ...
})->whereAlphaNumeric('username');
```

Matches:
- `/user/zura123`
- `/user/123thecodeholic`
- `/user/ZURA123`

**Does not match:**
- `/user/zura.123`
- `/user/123-ZURA`
- `/user/123_ZURA`

### Multiple Parameter Constraints

You can define constraints for several route parameters:

```php
Route::get("{lang}/product/{id}", function(string $lang, string $id) {
    // ...
})
    ->whereAlpha("lang")   // Only uppercase and lowercase letters
    ->whereNumber("id");   // Only digits
```

Matches:
- `/en/product/123`
- `/ka/product/456`
- `/test/product/01234`

**Does not match:**
- `/en/product/123abc`
- `/ka/product/abc456`
- `/en123/product/01234`

### whereIn() - Specific Values

The `lang` route parameter must contain one of the provided values:

```php
Route::get("{lang}/products", function(string $lang) {
    // ...
})->whereIn("lang", ["en", "ka", "in"]);
```

Matches:
- `/en/products`
- `/ka/products`
- `/in/products`

**Does not match:**
- `/de/products`
- `/es/products`
- `/test/products`

## Route Parameter Validation With Custom Regex

### Lowercase Letters Only

The `username` parameter must only contain lowercase letters:

```php
Route::get('/user/{username}', function(string $username) {
    // ...
})->where('username', '[a-z]+');
```

### Multiple Parameters with Custom Regex

The `lang` parameter must be exactly 2 letters and `id` must contain at least 4 digits:

```php
Route::get("{lang}/product/{id}", function(string $lang, string $id) {
    // ...
})->where(["lang" => '[a-z]{2}', 'id' => '\d{4,}']);
```

### Encoded Forward Slashes

By default, Laravel allows every character in route parameters except `/`. If you want `/` to be matched in your route parameter, use the following pattern:

```php
Route::get('/search/{search}', function (string $search) {
    return $search;
})->where('search', '.*');
```

This allows the `search` parameter to contain forward slashes, enabling URLs like `/search/category/subcategory`.

## Named Routes

You can assign a name to each route:

```php
Route::get("/about", function() {
    // ...
})->name("about");
```

### Why Use Named Routes?

**Without route names**, URLs are generated like this:

```php
$aboutPageUrl = "/about";
// Use in navigation:
// <a href="USE_IT_HERE">About</a>
```

**Problem:** If this URL is used in multiple places (header navigation, footer navigation, sidebar) and you decide to change it from `/about` to `/about-us`, you must update it everywhere. Missing even one place creates broken URLs.

**With route names**, URLs are generated like this:

```php
public function index()
{
    $aboutPageUrl = route('about');
    
    return view('your-view', ['aboutPageUrl' => $aboutPageUrl]);
}
// <a href="{{ $aboutPageUrl }}">O nas</a>
```

**Benefits:** If you change the route from `/about` to `/about-us`:

```php
Route::get("/about-us", function() {
    // ...
})->name("about");
```

Laravel automatically generates correct URLs everywhere you use `route("about")`, preventing broken links.

## Named Route With Parameter

```php
Route::get("/{lang}/product/{id}", function(string $lang, string $id) {
    // ...
})->name("product.view");
```

### Generating URLs Without Route Name

```php
$productId = 123;
$lang = "en";
$productViewUrl = "/$lang/product/$productId";
```

### Generating URLs With Route Name

```php
$productId = 123;
$lang = "en";
$productViewUrl = route("product.view", ["lang" => $lang, "id" => $productId]);
```

### Benefits of Named Routes with Parameters

While it may seem like more code initially, it provides significant advantages:

**1. URL Pattern Changes Don't Break Code**

If you update the URL pattern:

```php
Route::get("/p/{lang}/{id}", function(string $lang, string $id) {
    // ...
})->name("product.view");
```

You don't need to update URL generation code anywhere in your application.

**2. Easy Route Detection**

You can easily check the current route and execute specific logic for certain pages. With route names, detecting which route is currently active is straightforward.

**3. Simplified Redirects**

Redirecting to a specific route is much easier with route names:

```php
// Define a route with name "profile"
Route::get('/user/profile', function () {
    // ...
})->name('profile');

Route::get("/current-user", function() {
    // Generating Redirects...
    return redirect()->route('profile');
    // or
    return to_route('profile');
});
```

## Route Groups

Route groups allow you to share attributes across multiple routes without defining them on each individual route.

### Route Prefixes

Add a common URI prefix to all routes in the group:

```php
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        // Matches the "/admin/users" URL
    });
});
```

All routes inside this group will be prefixed with `/admin`.

### Route Name Prefixes

Add a common name prefix to all routes in the group:

```php
Route::name('admin.')->group(function () {
    Route::get('/users', function () {
        // Route assigned name "admin.users"
    })->name('users');
});
```

The final route name will be `admin.users` (prefix + individual name).

## Fallback Routes

Typically when a route is not matched, Laravel shows a 404 page. If you want to handle all unmatched routes with custom logic:

```php
Route::fallback(function () {
    // Handle all unmatched routes
});
```

This route will be executed only when no other route matches the request.

## Artisan Command to View Existing Routes

### List All Routes

```bash
php artisan route:list
```

### Verbose Output

```bash
php artisan route:list -v
```

### Very Verbose Output

```bash
php artisan route:list -vv
```

Shows more detailed information about each route.

### Filter by Path

```bash
php artisan route:list --path=api
```

Shows only routes that match the specified path pattern.

### Exclude Vendor Routes

```bash
php artisan route:list --except-vendor
```

Shows only your application routes, excluding routes defined by third-party packages.

### Show Only Vendor Routes

```bash
php artisan route:list --only-vendor
```

Shows only routes defined by third-party packages.

### Combine Multiple Options

```bash
php artisan route:list -vv --except-vendor --path=home
```

You can combine multiple options to filter and display routes exactly as needed.

## Route Caching

Route caching can significantly speed up route registration in production environments.

### Create Cache for All Routes

```bash
php artisan route:cache
```

This command creates a cached file of all routes, improving application performance. After caching, Laravel loads routes from the cache file instead of parsing route files on every request.

**Important:** After making any changes to routes, you must clear the cache and re-cache routes for changes to take effect.

### Clear Cache for All Routes

```bash
php artisan route:clear
```

This removes the route cache. Use this command during development when you're actively changing routes, or before deploying new route changes to production.

## Coding Challenge

### Problem

Create Route which accepts two route parameters. These parameters should be integers. You should calculate the sum of these two parameters and print that sum.

### Solution

```php
Route::get('/sum/{a}/{b}', function(float $a, float $b) {
	echo "Sum: " . ($a + $b);
})->whereNumber(['a', 'b'])
;
```

