## Creating and Rendering Views

Views are files responsible for presentation logic in your Laravel application and are stored in the `resources/views` folder.

**Views are typically Blade files.**

**Blade** is Laravel's templating engine that helps you build HTML views efficiently. It allows you to mix HTML with PHP using simple, clean syntax. Blade supports:
- Template inheritance - define a base layout and extend it in other views
- Directives for control flow (loops, conditionals, etc.)
- Reusable components and slots

### Create a View

All Blade views must have the `.blade.php` extension.

You can create a file manually in the `resources/views` directory or use the artisan command:

```bash
php artisan make:view index
```

Open the created file, delete its content, and paste the following code:

```html
<h1>Hello From Laravel Course</h1>
```

Now open `HomeController` and update its `index` method:

```php
public function index()
{
    return view("index");
}
```

When you access the application's home page, you'll see the heading displayed.

### Rendering Views from Nested Folders

Move your `index.blade.php` file into the `resources/views/home` folder. The blade file is now nested inside the `home` folder. Render it by updating the `index` method:

```php
public function index()
{
    return view("home.index");
}
```

Laravel uses dot notation to reference nested views - the dot represents a directory separator.

## Render Views Using View Facade

### Render with `View::make()`

Besides using the `view()` helper function, you can render views using the `View` facade's `make()` method:

```php
public function index()
{
    return View::make("home.index");
}
```

**Important:** Make sure you import the `View` facade correctly:
```php
use Illuminate\Support\Facades\View;
```

### Render First Available View

You can render the first available view from a list of view names:

```php
return View::first(["index", "home.index"]);
```

Laravel will check each view in order and render the first one that exists.

### Check if the View Exists

You can check whether a view exists before attempting to render it:

```php
if (!View::exists("index")) {
    dump("View does not exist");
}
```

This is useful for conditional rendering or debugging.

## Passing Data to Views

### Option 1 - Array as Second Parameter

Pass data as an associative array as the second parameter to the `view()` function:

```php
public function index()
{
    return view("index", [
        "name" => "Zura",
        "surname" => "Sekhniashvili"
    ]);
}
```

### Option 2 - Using with() Method

Chain the `with()` method to pass individual variables:

```php
public function index()
{
    return view("index")
        ->with("name", "Zura")
        ->with("surname", "Sekhniashvili");
}
```

Both options achieve the same result - the variables become available in the view.

### Access Variables in Blade File

In your Blade file, access the variables using double curly braces:

```php
<h1>Hello From Laravel</h1>
<p>My Name is {{ $name }} {{ $surname }}</p>
```

The double curly braces `{{ }}` automatically escape the output to prevent XSS attacks.

When you access http://127.0.0.1:8000/, you'll see:

```
Hello From Laravel
My Name is Zura Sekhniashvili
```

## Sharing Data With All Views

To make a variable available in all views across your application, use the `View::share()` method in a service provider.

Open `AppServiceProvider.php` and add the following to the `boot()` method:

```php
public function boot(): void
{
    // ...
    
    View::share('year', date('Y'));
}
```

This makes the `year` variable available in every view file in your application.

In any Blade file, you can now access this variable:

```php
<p>Global shared variable: {{ $year }}</p>
```

This is useful for data that needs to be available throughout your application, such as site settings, the current year for copyright notices, or user authentication status.

## Challenge

### Problem

1. Create `HelloController` with a `welcome` method
2. Create a `welcome` view using artisan and place it in the `hello` subfolder
3. From `HelloController::welcome`, render the `welcome` view you just created
4. Pass your name and surname to the view and output them in the Blade file
5. Define route `/hello` so that accessing it outputs your name and surname

### Solution

**1. Create new controller**

```bash
php artisan make:controller HelloController
```

**2. Create view**

```bash
php artisan make:view hello.welcome
```

**3. Open the controller and add the welcome method**

```php
public function welcome()
{
    return view('hello.welcome', [
        'name' => 'Zura',
        'surname' => 'Sekhniashvili'
    ]);
}
```

**4. Open the welcome view and output name and surname**

```php
<div>
    {{ $name }} {{ $surname }}
</div>
```

**5. Define the route in routes/web.php**

```php
Route::get('/hello', [\App\Http\Controllers\HelloController::class, 'welcome']);
```

**6. Test the solution**

Access http://localhost:8000/hello and you should see your name and surname displayed.

## Use and Output PHP Function Results

### Using PHP Functions Directly

You can use PHP functions directly in Blade templates:

```php
{{ date('Y') }}
```

This outputs the current year in the browser.

### Using Functions with Variables

You can apply PHP functions to existing variables:

```php
<p>{{ $name . ' ' . strtoupper($surname) }}</p>
```

This outputs:
```
Zura SEKHNIASHVILI
```

### Using Constants and Static Functions

You can use PHP constants and static class methods:

```html
<p>{{ Illuminate\Support\Str::after("Hello World", "Hello ") }}</p>
<p>{{ Illuminate\Foundation\Application::VERSION }}</p>
<p>{{ PHP_VERSION }}</p>
```

The output:
```html
World
12.x
8.3.0
```

## Displaying Unescaped Data

### Default Escaped Output

When you pass HTML as a string:

```php
return view("hello", [
    "name" => "Zura",
    "surname" => "Sekhniashvili",
    "job" => "<b>Developer</b>",
]);
```

And render it using double curly braces:

```html
<p>Your Job: {{ $job }}</p>
```

The HTML is escaped and rendered as plain text. The browser displays:
```
Your Job: <b>Developer</b>
```

The `<b>` tags are visible as text, not rendered as HTML.

### Rendering Unescaped HTML

If you want the HTML to be rendered (not escaped), use `{!! !!}` syntax:

```html
<p>Your Job: {!! $job !!}</p>
```

The output will render the HTML:
```
Your Job: Developer
```

The word "Developer" appears bold because the `<b>` tag is interpreted as HTML.

**Important:** Only use `{!! !!}` with trusted content. Using it with user-provided data can expose your application to XSS (Cross-Site Scripting) attacks.

## Blade and JavaScript Frameworks

### Escaping Blade Syntax

Sometimes you need to output Blade-like syntax (e.g., `{{ name }}`) for JavaScript frameworks like Vue.js or Alpine.js that use the same syntax. If you write:

```html
<p>{{ name }}</p>
```

Blade will try to evaluate it and you'll get an error:
```
Undefined constant "name"
```

To tell Blade to ignore the expression, add an `@` symbol in front:

```html
<p>@{{ name }}</p>
```

This outputs `{{ name }}` in the browser, which can then be processed by your JavaScript framework.

### Escaping Blade Directives

Similarly, if you want to output a Blade directive name (like `@foreach`) in the browser, writing it directly will cause Blade to try to process it. Add an extra `@` symbol:

```html
@@foreach
```

This outputs `@foreach` in the browser.

### The @verbatim Directive

If you have a large section with multiple expressions that shouldn't be processed by Blade, wrap it with the `@verbatim` directive:

```html
@verbatim
<div>
    Name: {{ name }}
    Age: {{ age }}
    Job: {{ job }}
    Hobbies: {{ hobbies}}
    @if
</div>
@endverbatim
```

Everything inside the `@verbatim` block is output exactly as written. The browser will display:

```html
Name: {{ name }}
Age: {{ age }}
Job: {{ job }}
Hobbies: {{ hobbies}}
@if
```

This is particularly useful when you have Vue.js or Alpine.js components with many template expressions.

## Rendering JSON

Add a `hobbies` variable as an array:

```php
return view("hello", [
    "name" => "Zura",
    "surname" => "Sekhniashvili",
    "job" => "<b>Developer</b>",
    "hobbies" => ["Tennis", "Fishing"]
]);
```

### Proper Way to Output JSON

The correct way to output data as JSON in Blade is using `Js::from()`:

```html
<script>
    const hobbies = {{ Js::from($hobbies) }}
</script>
```

If you check the page source, the result is:

```html
<script>
    const hobbies = JSON.parse('[\u0022Tennis\u0022,\u0022Fishing\u0022]')
</script>
```

The `Js::from()` method properly escapes the data and wraps it in `JSON.parse()`, ensuring safe and correct conversion from PHP arrays/objects to JavaScript. This prevents XSS vulnerabilities and handles special characters correctly.

## What are Blade Directives?

Blade directives are special keywords prefixed with an "@" symbol used within Laravel's Blade templating engine to simplify common tasks like template inheritance, conditional rendering, loops, and more. They convert these shorthand notations into plain PHP code, enhancing readability and maintainability.

Examples include:
- `@if` for conditionals
- `@foreach` for loops
- `@extends` for layout inheritance

By abstracting common patterns, Blade directives streamline template creation and reduce boilerplate code, allowing developers to focus more on the content and logic of their views.

## Comments

### Single Line Comment

```html
{{-- Single line comment --}}
<button>Click me</button>
```

### Multi-line Comment

```html
{{--
    Multi
    line
    comment
--}}
<button>Click me</button>
```

Blade comments are not included in the rendered HTML output, unlike HTML comments (`<!-- -->`).

## Conditionals - @if, @unless, @isset, @empty, @auth, @guest

### @if Directive

Basic if statement:

```php
@if (count($cars) > 1)
    <p>There is more than 1 car</p>
@endif
```

If-else statement:

```php
@if (count($cars) > 0)
    <p>There is at least one car</p>
@else
    <p>There are no cars</p>
@endif
```

If-elseif-else statement:

```php
@if (count($cars) > 1)
    <p>There is more than one car</p>
@elseif (count($cars) === 1)
    <p>There is exactly one car</p>
@else
    <p>There are no cars</p>
@endif
```

### @unless - Opposite of @if

The `@unless` directive is the opposite of `@if`:

```php
@unless (false)
    <p>unless</p>
@endunless
```

This is equivalent to:

```php
@if (true)
    <p>unless</p>
@endif
```

### @isset, @empty

Check if a variable is set:

```php
@isset($cars)
    <p>isset</p>
@endisset
```

Check if a variable is empty:

```php
@empty($cars)
    <p>empty</p>
@endempty
```

### @auth, @guest

Check if a user is authenticated:

```php
@auth
    <p>User is authenticated</p>
@endauth
```

Check if a user is a guest (not authenticated):

```php
@guest
    <p>User is guest</p>
@endguest
```

## Switch Statement - @switch

```php
@switch($country)
    @case('ge')
        <p>Georgia</p>
        @break
    @case('uk')
        <p>United Kingdom</p>
        @break
    @case('us')
        <p>USA</p>
        @break
    @default
        <p>Unknown Country</p>
@endswitch
```

The `@switch` directive provides a cleaner alternative to multiple `@if`/`@elseif` statements when checking a single variable against multiple values.

## Loops - @for, @foreach, @forelse, @while

### @for

```php
@for ($i = 0; $i < 5; $i++)
   <p>{{ $i + 1 }}</p>
@endfor
```

This outputs numbers from 1 to 5.

### @foreach

```php
@foreach ($cars as $car)
    <p>Model: {{ $car->model }}</p>
@endforeach
```

### @forelse

The `@forelse` directive is like `@foreach` but includes an `@empty` clause for when the array is empty:

```php
@forelse ($cars as $car)
    <p>Model: {{ $car->model }}</p>
@empty
    <p>There are no cars</p>
@endforelse
```

This is cleaner than wrapping `@foreach` in an `@if` statement to check if the array has items.

### @while

```php
@while (false)
@endwhile
```

## @continue and @break

### @continue

Skip the current iteration:

```php
@foreach ([1, 2, 3, 4, 5] as $n)
    @if ($n == 2)
         @continue
    @endif
    <p>{{$n}}</p>
@endforeach
```

Shorthand version:

```php
@foreach ([1, 2, 3, 4, 5] as $n)
    @continue($n == 2)
    <p>{{$n}}</p>
@endforeach
```

This outputs: 1, 3, 4, 5 (skipping 2).

### @break

Exit the loop entirely:

```php
@foreach ([1, 2, 3, 4, 5] as $n)
    <p>{{$n}}</p>
    @if ($n == 4)
         @break
    @endif
@endforeach
```

Shorthand version:

```php
@foreach ([1, 2, 3, 4, 5] as $n)
    <p>{{$n}}</p>
    @break($n == 4)
@endforeach
```

This outputs: 1, 2, 3, 4 (stopping at 4).

## The Loop Variable

Inside `@foreach` and `@forelse` loops, Laravel provides a special `$loop` variable with useful information:

```php
@foreach ($cars as $c)
     {{ dd($loop) }}
@endforeach
```

### Available Loop Properties

```php
@foreach ($cars as $c)
     <p>index: {{ $loop->index }}</p>
    <p>iteration: {{ $loop->iteration }}</p>
    <p>remaining: {{ $loop->remaining }}</p>
    <p>count: {{ $loop->count }}</p>
    <p>first: {{ $loop->first }}</p>
    <p>last: {{ $loop->last }}</p>
    <p>even: {{ $loop->even }}</p>
    <p>odd: {{ $loop->odd }}</p>
    <p>depth: {{ $loop->depth }}</p>
    <p>parent: {{ $loop->parent }}</p>

    <h4>Start of inner loop</h4>
    @foreach([1, 2, 3] as $n)
        <p>{{ $loop->depth }}</p>
        <p>{{ $loop->parent->depth }}</p>
    @endforeach
    <h4>End of inner loop</h4>
@endforeach
```

**Loop variable properties:**
- `index` - 0-based index of the current iteration
- `iteration` - 1-based index of the current iteration
- `remaining` - Number of iterations remaining
- `count` - Total number of items in the array
- `first` - Boolean indicating if this is the first iteration
- `last` - Boolean indicating if this is the last iteration
- `even` - Boolean indicating if this is an even iteration
- `odd` - Boolean indicating if this is an odd iteration
- `depth` - Nesting level of the current loop
- `parent` - Access to parent loop variable in nested loops

## Conditional Classes & Styles

### Conditional CSS Classes

Apply CSS classes conditionally using the `@class` directive:

```php
<div @class([
    'my-css-class',
    'georgia' => $country === 'ge'
])>
Lorem ipsum, dolor sit amet consectetur adipisicing elit.
</div>
```

The class `my-css-class` is always applied, while `georgia` is only applied if `$country === 'ge'`.

### Conditional Inline Styles

Apply inline styles conditionally using the `@style` directive:

```php
<div @style([
    'color: green',
    'background-color: cyan' => $country === 'ge'
])>
Lorem ipsum, dolor sit amet consectetur adipisicing elit.
</div>
```

The color green is always applied, while the cyan background is only applied if `$country === 'ge'`.

## Include Sub Views

Include a sub-view located at `shared/button.blade.php`:

```php
@include('shared.button')
```

### Pass Additional Data to Sub View

```php
@include('shared.button', ['color' => 'brown'])
```

The included view will have access to all variables from the parent view, plus any additional variables passed in the array.

## Conditionally Rendering Sub Views

### @includeIf

The `@include` directive will fail if the view doesn't exist, but `@includeIf` will simply ignore it:

```php
@includeIf('shared.search_form', ['year' => 2019])
```

### @includeWhen and @includeUnless

Conditionally include views based on a boolean condition:

```php
@includeWhen($searchKeyword, 'shared.search_results', ['year' => 2019])
@includeUnless(!$searchKeyword, 'shared.search_results', ['year' => 2019])
```

- `@includeWhen` includes the view if the condition is true
- `@includeUnless` includes the view if the condition is false

### @includeFirst

Include the first view that exists from a list:

```php
@includeFirst(['admin.button', 'button'], ['color' => 'red'])
```

Laravel will check for `admin.button` first, and if it doesn't exist, it will use `button`.

## Using Sub View Inside Loop - @each

If you want to iterate over an array and render a sub-view for each element, you could do:

```php
@foreach ($cars as $car)
    @include('car.view', ['car' => $car])
@endforeach
```

The sub-view file `car.view.blade.php` receives the `$car` variable.

### @each Directive

The same result can be achieved using the `@each` directive:

```php
@each('car.view', $cars, 'car')
```

**Parameters:**
1. The view name to render
2. The array to iterate over
3. The variable name that each array element will be assigned to in the view

### Empty View

You can pass a fourth argument - a view to render if the array is empty:

```php
@each('car.view', $cars, 'car', 'car.empty')
```

If `$cars` is empty, `car.empty.blade.php` will be rendered instead.

## Raw PHP

### Declare a Variable

```php
@php
    $city = 'Tbilisi'
@endphp
```

Equivalent in PHP:

```php
<?php
    $city = 'Tbilisi';
?>
```

### Import Classes

```php
@use("Illuminate\Support\Str")
```

Equivalent in PHP:

```php
<?php
    use Illuminate\Support\Str;
?>
```

**Benefit of `@use` directive:** Even if you use it multiple times, if the class is already imported it will not throw an error, unlike the standard PHP `use` statement.

## Challenge

### Problem

Create a sub-view `alert.blade.php` which will take two variables `$message` and `$color` and use them in the blade file. Use the `@style` directive to add color to the alert element.

Include that view into another blade file and pass these two variables.

The goal is to create a second blade file, include it in another blade file, and provide variables to it.

### Solution

**1. Create `alert.blade.php` in the `shared` folder**

```bash
php artisan make:view shared.alert
```

**2. Open the `alert` view and define HTML code**

```php
<div @style(['background-color: '.$color])>
    {{ $message }}
</div>
```

**3. Open another blade file and include the `alert` view**

```php
@include('shared.alert', ['message' => 'Your account was created', 'color' => 'green'])
```

This demonstrates how to create reusable components that accept parameters, making your views more modular and maintainable.