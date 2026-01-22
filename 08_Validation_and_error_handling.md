## Validation in Laravel

Validating incoming data is one of the fundamental mechanisms in any web application. Laravel offers several different approaches to validate data coming into your application.

### Basic Validation in Controller

The simplest way to validate is using the `validate` method directly in the controller. Let's imagine a situation where we have a form for creating a blog post. In the controller, we can write:

```php
public function store(Request $request)
{
    $validated = $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ]);

    // Code only executes if validation passes
    return redirect('/posts');
}
```

When validation fails, Laravel automatically redirects the user back with the appropriate errors. If the request comes through AJAX (XHR), instead of a redirect, a JSON response will be returned with status code 422 and all validation errors.

We can write validation rules in two ways - as a string with `|` separator or as an array:

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```

### Stopping Validation on First Failure

Sometimes we want validation to stop checking rules for a given field after the first encountered error. The `bail` rule serves this purpose:

```php
$request->validate([
    'title' => 'bail|required|unique:posts|max:255',
    'body' => 'required',
]);
```

In this example, if `unique` fails, the `max` rule won't be checked anymore.

### Validating Nested Fields

When we have a form with nested data (for example, author data inside a post), we can use dot notation:

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'author.name' => 'required',
    'author.description' => 'required',
]);
```

However, if the field name actually contains a dot, we can escape it with a backslash:

```php
$request->validate([
    'v1\.0' => 'required',
]);
```

### Displaying Validation Errors

Laravel automatically makes the `$errors` variable available in all views. This is a `MessageBag` instance that contains all validation errors. In a Blade view, we can display errors like this:

```blade
@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif
```

Or for a specific field, using the `@error` directive:

```blade
<input id="title" type="text" name="title" class="@error('title') is-invalid @enderror"/>

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

### Repopulating Forms

When validation fails, Laravel automatically flashes all input data to the session. This allows us to easily repopulate the form using the `old()` function:

```blade
<input type="text" name="title" value="{{ old('title') }}">
```

If there's no old value, the function returns `null`.

### Optional Fields

Laravel includes the `ConvertEmptyStringsToNull` middleware by default, so optional fields should be marked as `nullable`:

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
    'publish_at' => 'nullable|date',
]);
```

Without the `nullable` modifier, the validator would consider a `null` value as an invalid date.

### Form Request Validation

For more complex validation scenarios, we can create a dedicated Form Request class. We create it with the command:

```bash
php artisan make:request StorePostRequest
```

The Form Request class has two main methods: `authorize()` and `rules()`.

```php
public function authorize(): bool
{
    $comment = Comment::find($this->route('comment'));
    return $comment && $this->user()->can('update', $comment);
}

public function rules(): array
{
    return [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ];
}
```

We use it as a type-hint in the controller method:

```php
public function store(StorePostRequest $request): RedirectResponse
{
    // Validation already passed successfully
    $validated = $request->validated();
    
    // We can get only selected fields
    $validated = $request->safe()->only(['name', 'email']);
    
    return redirect('/posts');
}
```

### Additional Validation

Sometimes we need to perform additional validation after the basic one. We can do this using the `after()` method:

```php
public function after(): array
{
    return [
        function (Validator $validator) {
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add(
                    'field',
                    'Something is wrong with this field!'
                );
            }
        }
    ];
}
```

### Customizing Error Messages

We can override default error messages in three ways:

1. Directly in the validate call:

```php
$validatedData = $request->validate([
    'title' => 'required',
], [
    'title.required' => 'A title is required',
]);
```

2. In the Form Request class:

```php
public function messages(): array
{
    return [
        'title.required' => 'A title is required',
        'body.required' => 'A message is required',
    ];
}
```

3. In the language file `lang/en/validation.php`:

```php
'custom' => [
    'email' => [
        'required' => 'We need to know your email address!',
    ],
],
```

### Customizing Attribute Names

We can change how field names appear in error messages:

```php
public function attributes(): array
{
    return [
        'email' => 'email address',
    ];
}
```

Now instead of "The email field is required" we'll get "The email address field is required".

### Manually Creating Validators

If we don't want to use the `validate()` method, we can create a validator manually:

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
]);

if ($validator->fails()) {
    return redirect('/post/create')
        ->withErrors($validator)
        ->withInput();
}

$validated = $validator->validated();
```

### Named Error Bags

If we have multiple forms on a single page, we can name our error bags:

```php
return redirect('/register')->withErrors($validator, 'login');
```

And in the view reference them:

```blade
{{ $errors->login->first('email') }}
```

### Working With Validated Data

After validation, we have several ways to retrieve the data:

```php
// As an array
$validated = $request->validated();

// Using the ValidatedInput object
$validated = $request->safe()->only(['name', 'email']);
$validated = $request->safe()->except(['name', 'email']);
$validated = $request->safe()->all();

// We can iterate over the data
foreach ($request->safe() as $key => $value) {
    //
}

// Or as a collection
$collection = $request->safe()->collect();
```

### Available Validation Rules

Laravel offers a huge number of built-in rules. Here are a few key categories:

**Boolean fields:**
- `accepted` - field must have value "yes", "on", 1, "1", true, "true"
- `boolean` - field must be castable to boolean
- `declined` - field must have value "no", "off", 0, "0", false, "false"

**Strings:**
- `required` - field must be present and not empty
- `email` - field must be a valid email address
- `max:value` - field cannot be longer than value characters
- `min:value` - field must have at least value characters
- `url` - field must be a valid URL

**Numbers:**
- `integer` - field must be an integer
- `numeric` - field must be numeric
- `between:min,max` - value must be between min and max
- `gt:field` - must be greater than another field
- `gte:field` - must be greater than or equal to another field

**Dates:**
- `date` - field must be a valid date
- `after:date` - date must be after the given date
- `before:date` - date must be before the given date
- `date_format:format` - date must match the given format

**Arrays:**
- `array` - field must be a PHP array
- `distinct` - array cannot have duplicates
- `in:foo,bar` - value must be one of the given values
- `contains:foo,bar` - array must contain all given values

**Files:**
- `file` - field must be a successfully uploaded file
- `image` - file must be an image (jpg, jpeg, png, bmp, gif, webp)
- `mimes:jpg,png` - file must have the given MIME type
- `dimensions` - image must meet dimension requirements

**Database:**
- `exists:table,column` - value must exist in the database
- `unique:table,column` - value must be unique in the database

### Conditional Rules

We can add rules conditionally using the `Rule::when()` method:

```php
use Illuminate\Validation\Rule;

$request->validate([
    'email' => [
        'required',
        Rule::when($request->user()->is_admin, [
            'unique:users',
        ]),
    ],
]);
```

Or using built-in conditional rules:
- `required_if:anotherfield,value` - required if another field has the given value
- `required_unless:anotherfield,value` - required unless another field has the given value
- `required_with:foo,bar` - required if any of the given fields are present
- `required_without:foo,bar` - required if any of the given fields are not present

### Validating Arrays

We can validate arrays using asterisk notation:

```php
$request->validate([
    'photos' => 'required|array',
    'photos.*' => 'required|image',
    'photos.*.description' => 'required|string|max:255',
]);
```

Errors for array elements are flattened to dot notation: `photos.0.description`, `photos.1.description`, etc.

### Validating Files

For files we can use the fluent rule builder:

```php
use Illuminate\Validation\Rules\File;

$request->validate([
    'attachment' => [
        'required',
        File::types(['jpg', 'png'])
            ->max(12 * 1024), // 12MB in kilobytes
    ],
]);
```

For images we can check dimensions:

```php
use Illuminate\Validation\Rule;

$request->validate([
    'avatar' => [
        'required',
        Rule::dimensions()
            ->maxWidth(1000)
            ->maxHeight(500)
            ->ratio(3 / 2),
    ],
]);
```

### Password Validation

Laravel offers a special password rule that allows enforcing complexity:

```php
use Illuminate\Validation\Rules\Password;

$request->validate([
    'password' => ['required', Password::min(8)
        ->letters()
        ->mixedCase()
        ->numbers()
        ->symbols()
        ->uncompromised()], // checks if password hasn't leaked in breaches
]);
```

We can also define a default password policy in `AppServiceProvider`:

```php
Password::defaults(function () {
    return Password::min(8)
        ->mixedCase()
        ->numbers()
        ->symbols()
        ->uncompromised();
});
```

### Custom Validation Rules

We can create our own validation rules in three ways:

**1. Using a rule class:**

```bash
php artisan make:rule Uppercase
```

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    if (strtoupper($value) !== $value) {
        $fail('The :attribute must be uppercase.');
    }
}
```

Usage:

```php
use App\Rules\Uppercase;

$request->validate([
    'name' => ['required', new Uppercase],
]);
```

**2. Using closures:**

```php
$request->validate([
    'name' => [
        'required',
        function ($attribute, $value, $fail) {
            if ($value === 'foo') {
                $fail('The '.$attribute.' is invalid.');
            }
        },
    ],
]);
```

**3. Using fluent rule builder:**

```php
use Illuminate\Validation\Rule;

Rule::forEach(function ($value, $attribute) {
    return [
        Rule::requiredIf($value > 100),
    ];
});
```

## Error Handling in Laravel

Laravel 12 introduced a new way of managing exceptions. Instead of the `App\Exceptions\Handler` class, we now configure everything in the `bootstrap/app.php` file using the `withExceptions()` method.

### Configuration

The `APP_DEBUG` environment variable in the `.env` file controls how much error information is shown to the user. During development it should be set to `true`, but in production ALWAYS to `false`, to avoid exposing sensitive information.

### Reporting Exceptions

Exception reporting is used for logging them or sending them to external services like Sentry or Flare.

We can register a custom callback for specific exception types:

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->report(function (InvalidOrderException $e) {
        // Custom reporting logic
    });
})
```

If we want to stop propagation to the default logging stack, we use `stop()`:

```php
$exceptions->report(function (InvalidOrderException $e) {
    // ...
})->stop();
```

Or return `false`:

```php
$exceptions->report(function (InvalidOrderException $e) {
    return false;
});
```

### Global Log Context

Laravel automatically adds the current user's ID to every log entry. We can add our own global context:

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->context(fn () => [
        'foo' => 'bar',
    ]);
})
```

### Per-Exception Context

We can also define context specific to a given exception:

```php
class InvalidOrderException extends Exception
{
    public function context(): array
    {
        return ['order_id' => $this->orderId];
    }
}
```

### The `report()` Helper

Sometimes we want to report an exception but continue processing the request:

```php
public function isValid(string $value): bool
{
    try {
        // Validate the value...
    } catch (Throwable $e) {
        report($e);
        return false;
    }
}
```

### Deduplicating Reported Exceptions

If we use `report()` in many places, we might accidentally report the same exception multiple times. We can prevent this:

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->dontReportDuplicates();
})
```

### Exception Log Levels

We can set the log level for specific exception types:

```php
use PDOException;
use Psr\Log\LogLevel;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);
})
```

### Ignoring Exceptions

We can completely ignore some exceptions:

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);
})
```

Or by implementing the `ShouldntReport` interface:

```php
use Illuminate\Contracts\Debug\ShouldntReport;

class PodcastProcessingException extends Exception implements ShouldntReport
{
    //
}
```

### Rendering Exceptions

We can customize how exceptions are presented to the user:

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });
})
```

For APIs we can return JSON:

```php
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (NotFoundHttpException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => 'Record not found.'
            ], 404);
        }
    });
})
```

### Controlling JSON Rendering

We can control when Laravel should return a JSON response:

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
        if ($request->is('admin/*')) {
            return true;
        }
        return $request->expectsJson();
    });
})
```

### Customizing the Entire Response

In rare cases we might want to modify the entire HTTP response:

```php
use Symfony\Component\HttpFoundation\Response;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->respond(function (Response $response) {
        if ($response->getStatusCode() === 419) {
            return back()->with([
                'message' => 'The page expired, please try again.',
            ]);
        }
        return $response;
    });
})
```

### Reportable and Renderable Exceptions

Instead of defining logic in `bootstrap/app.php`, we can add `report()` and `render()` methods directly to the exception class:

```php
class InvalidOrderException extends Exception
{
    public function report(): void
    {
        // Custom reporting logic
    }

    public function render(Request $request): Response
    {
        return response(/* ... */);
    }
}
```

If we want to use default rendering in some cases, we return `false`:

```php
public function render(Request $request): Response|bool
{
    if (/** condition */) {
        return response(/* ... */);
    }
    return false;
}
```

### Throttling Reported Exceptions

If the application reports a lot of exceptions, we can sample them randomly:

```php
use Illuminate\Support\Lottery;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->throttle(function (Throwable $e) {
        return Lottery::odds(1, 1000); // 1 in 1000
    });
})
```

Or use rate limiting:

```php
use Illuminate\Cache\RateLimiting\Limit;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->throttle(function (Throwable $e) {
        if ($e instanceof BroadcastException) {
            return Limit::perMinute(300);
        }
    });
})
```

### HTTP Exceptions

We can throw HTTP exceptions using the `abort()` helper:

```php
abort(404);
abort(403, 'Unauthorized action.');
```

### Custom Error Pages

Laravel allows creating custom error pages for different HTTP status codes. Just create a view at `resources/views/errors/404.blade.php`. The view will receive an `$exception` variable:

```blade
<h2>{{ $exception->getMessage() }}</h2>
```

We can also define fallback pages for series of error codes by creating `4xx.blade.php` or `5xx.blade.php`. Remember that fallbacks don't affect 404, 500, and 503 - for these you need to create dedicated views.

We can publish Laravel's default error templates:

```bash
php artisan vendor:publish --tag=laravel-errors
```