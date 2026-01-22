## Create App Layout

**What is a Layout?** A layout is a master template that defines the common structure of your pages (header, footer, navigation). Instead of repeating the same HTML in every view, you create it once in a layout and reuse it across all pages.

**What is Blade?** Blade is Laravel's templating engine that allows you to write cleaner PHP code in your views using special directives (like `@yield`, `@section`, `@extends`). Blade files have the `.blade.php` extension.

Create a new blade file at `resources/views/layouts/app.blade.php` manually or by executing:

```bash
php artisan make:view layouts.app
```

Open the file and add the following content:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@yield('title') | Car Selling Website</title>
  </head>
  <body>
    <header>Your Header</header>
    @yield('content')
    
    <footer>Your Footer</footer>
  </body>
</html>
```

This layout outputs content for two sections:

1. `@yield('title')` - for the page title
2. `@yield('content')` - for the main page content

The `@yield` directive defines a placeholder that child views can fill with content.

### Using the Layout

Open `home/index.blade.php` and use the layout:

```php
@extends('layouts.app')

@section('title', 'Home Page')

@section('content')
    Home Page content goes here
@endsection
```

This:

- Extends the `layouts.app` blade file using `@extends` - tells Laravel to use this layout
- Defines the `title` section with "Home Page" - this will replace `@yield('title')` in the layout
- Defines the `content` section with your HTML content - this will replace `@yield('content')` in the layout

This is how you define proper layouts with template inheritance.

## Output Language, Application Name and CSRF Token in App Layout

### Output `lang`

Open the `layouts.app` view file and modify the `<html>` opening tag:

```php
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
```

**What this does:**

- `app()->getLocale()` retrieves the application's locale from config (e.g., "en_US", "pl_PL")
- `str_replace('_', '-', ...)` converts "en_US" to "en-US" (HTML standard requires hyphens)
- The double curly braces `{{ }}` output the value in Blade templates

### Output Application Name

Change the `<title>` tag content:

```php
<title>@yield('title') | {{ config('app.name', 'Laravel') }}</title>
```

**What this does:**

- `config('app.name', 'Laravel')` retrieves the application name from `config/app.php` file
- The second parameter ('Laravel') is the default value if the config is not set
- This allows you to change your app name in one place (config file) instead of editing every template

### Output CSRF Token

Add this `<meta>` tag just before the `<title>` tag:

```php
<meta name="csrf-token" content="{{ csrf_token() }}">
```

**What is CSRF?** CSRF (Cross-Site Request Forgery) is a security attack where a malicious website tricks your browser into making unwanted requests to your application. The CSRF token is a random value that verifies requests are coming from your application, not a malicious site. This is particularly important for AJAX requests and form submissions.

## Directives - @section, @show, @parent

**Note:** To better understand this section, we'll first create a base layout that our `app` layout will extend.

### Creating a Base Layout

Create `resources/views/layouts/clean.blade.php`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My Application</title>
  </head>
  <body>
    @yield('childContent')
  </body>
</html>
```

This is a minimal layout that only yields `childContent`.

### Defining Sections in Layouts

Now update the `layouts/app.blade.php` view file to extend the clean layout and add `@yield` inside the footer:

```php
@extends('layouts.clean')

@section('childContent')
    @include('layouts.partials.header')
    @yield('content')
    <footer>
        @yield('footerLinks')
    </footer>
@endsection
```

**What this creates:**

- A three-level hierarchy: `clean.blade.php` → `app.blade.php` → `home/index.blade.php`
- The `app` layout extends `clean` layout
- The `app` layout defines what goes into `childContent` section of `clean` layout
- The `app` layout also yields its own sections (`content` and `footerLinks`) for child views

**Create the header partial:** Create `resources/views/layouts/partials/header.blade.php`:

```html
<header>
    <nav>Your Navigation Here</nav>
</header>
```

### Providing Section Content

Open the `home/index.blade.php` view file and define the new `footerLinks` section:

```php
@extends('layouts.app')

@section('content')
    Home Page content goes here
@endsection

@section('footerLinks')
    <a href="#">Link 3</a>
    <a href="#">Link 4</a>
@endsection
```

When you open the application in the browser, you'll see **only Link 3 and Link 4** in the footer.

### Using @show for Default Content

**The Problem:** Currently, if a child view doesn't define `footerLinks`, the footer will be empty.

**The Solution:** Use `@show` to provide default content.

If you want **Link 1 and Link 2** to always be displayed in the footer by default, open `layouts/app.blade.php` and replace `@yield('footerLinks')` with:

```php
@section('footerLinks')
    <a href="#">Link 1</a>
    <a href="#">Link 2</a>
@show
```

**What is `@show`?** The `@show` directive:

1. Defines a section with default content
2. Immediately renders that content
3. Allows child views to override it completely

This is equivalent to writing:

```php
@section('footerLinks')
    <a href="#">Link 1</a>
    <a href="#">Link 2</a>
@endsection
@yield('footerLinks')
```

Now:

- If you **remove** the `footerLinks` section from `home/index.blade.php`, it will render **Link 1 and Link 2** (default)
- If you **keep** the section in `home/index.blade.php`, it will **override completely** and render only **Link 3 and Link 4**

### Using @parent to Append Content

**The Problem:** With `@show`, child views completely replace the default content. What if you want to keep the default content AND add more?

**The Solution:** Use `@parent` to include the parent's content.

To render all links together (both from parent and child), update the `footerLinks` section in `home/index.blade.php`:

```php
@section('footerLinks')
    @parent
    <a href="#">Link 3</a>
    <a href="#">Link 4</a>
@endsection
```

**What is `@parent`?** The `@parent` directive:

1. Includes the content from the parent layout's section
2. Then appends the new content after it
3. This will render **all 4 links** in the footer (Link 1, Link 2, Link 3, Link 4)

**Summary:**

- `@yield('name')` - just a placeholder, no default content
- `@section('name')...@show` - placeholder with default content that can be completely overridden
- `@parent` - includes parent section content before adding child content

## More on Directives

### @hasSection

Check if a section has been defined in a child view:

```php
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>
    
    <div class="clearfix"></div>
@endif
```

**When to use:** When you want to conditionally render HTML only if a child view provides content for a specific section.

### @sectionMissing

Execute code when a section is NOT defined:

```php
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```

**When to use:** When you want to show default content only if the child view doesn't provide its own.

### @checked

Conditionally add the `checked` attribute to checkboxes:

```php
<input type="checkbox" name="terms" @checked(old('terms')) />
```

**What this does:**

- If the expression inside `@checked()` is `true`, it outputs `checked="checked"`
- Useful for preserving checkbox state after form validation errors
- `old('terms')` retrieves the previously submitted value

### @disabled

```php
<input type="checkbox" name="terms" @disabled($user->isBanned()) />
```

**What this does:**

- Conditionally adds `disabled="disabled"` attribute
- Useful for disabling form fields based on conditions

### @readonly

```php
<input type="text" name="email" @readonly($user->email_verified) />
```

**What this does:**

- Conditionally adds `readonly="readonly"` attribute
- Useful for fields that should be viewable but not editable

### @required

```php
<input type="text" name="email" @required(!$user->hasEmail()) />
```

**What this does:**

- Conditionally adds `required="required"` attribute
- Useful for dynamic form validation requirements

### @selected

Conditionally select an option in a dropdown:

```php
<select name="year">
    @foreach($years as $year)
        <option value="{{ $year }}" @selected($year == date('Y'))>
            {{ $year }}
        </option>
    @endforeach
</select>
```

**What this does:**

- Adds `selected="selected"` to the option that matches the condition
- In this example, it automatically selects the current year
- `date('Y')` returns the current year (e.g., 2024)

### @env

Execute code based on the application environment:

```php
@env('staging')
    // The application is running in "staging"...
    <div class="debug-toolbar">Debug Mode Active</div>
@endenv

@env(['staging', 'production'])
    // The application is running in "staging" or "production"...
    <script src="analytics.js"></script>
@endenv
```

**When to use:** When you need to show/hide content based on where your application is running (local development, staging server, production server).

### @production

Execute code only in production environment:

```php
@production
    // Production specific content...
    <script src="https://analytics.google.com/tracking.js"></script>
@endproduction
```

**When to use:** For production-only features like analytics, ads, or optimization scripts.

### @session

Check if a session variable exists and display it:

```php
@session('successMessage')
    <div class="p-4 bg-green-100">
        {{ $successMessage }}
    </div>
@endsession
```

**What this does:**

- Checks if session variable 'successMessage' exists
- If it exists, the variable `$successMessage` automatically contains the session value
- Useful for displaying flash messages after form submissions

**Example in controller:**

```php
return redirect('/')->with('successMessage', 'Profile updated successfully!');
```

## Creating & Rendering Components

### What is a Component

Components are reusable pieces of UI that can be included in your Blade views. They encapsulate HTML markup, styles, and logic into a single unit, making it easier to manage and reuse code.

**Real-world analogy:** Think of components like LEGO blocks. Instead of building everything from scratch each time, you create reusable blocks (components) that you can use in different places.

**Benefits:**

- **Reusability:** Write once, use everywhere
- **Maintainability:** Change the component in one place, updates everywhere
- **Organization:** Keep related code together
- **Flexibility:** Accept data through attributes and slots

Components can accept data as attributes and include slots for flexible content insertion.

Laravel provides two types of components:

- **Class-based components** - have a PHP class and a blade file (more powerful, good for complex logic)
- **Anonymous components** - only have a blade file (simpler, good for simple UI elements)

### Create & Use a Basic Component

Create a new file at `resources/views/components/card.blade.php`:

```html
<div class="card">Card Component</div>
```

Use it in your blade files with the `<x-` prefix:

```html
<x-card />
```

**Naming convention:**

- File name: `card.blade.php` (lowercase, kebab-case)
- Usage: `<x-card />` (lowercase, with `x-` prefix and self-closing tag)

### Nested Components

You can create components in nested folders for better organization.

Create: `resources/views/components/admin/card.blade.php`

```html
<div class="card">Admin Card Component</div>
```

Use it with dot notation:

```html
<x-admin.card />
```

**Naming convention for nested:**

- Folder path: `admin/card.blade.php`
- Usage: `<x-admin.card />` (dots represent folder separators)

### Create Anonymous Component Using Artisan

```bash
# Generates file in resources/views/components/button.blade.php
php artisan make:component Button --view

# Generates file in resources/views/components/admin/button.blade.php
php artisan make:component Admin.Button --view
```

The `--view` flag indicates that you want to generate only a blade file (anonymous component), without a PHP class.

**Note:** It doesn't matter if you provide the name in uppercase or lowercase - Laravel generates the file in lowercase (button.blade.php, not Button.blade.php).

## Component Slots

### What is a Component Slot

A **component slot** is a placeholder where you can insert custom content when using the component. It's like a "hole" in your component where different content can be inserted each time you use it.

**Real-world analogy:** Think of a picture frame (component) with a space for a photo (slot). The frame stays the same, but you can put different photos in it.

There are two types of slots:

- **Default slot** - unnamed slot for the main content (only one per component)
- **Named slots** - multiple named sections for different content areas (as many as you need)

### Default Slots

Currently, the `card` component always renders the same content: "Card Component". Let's make it flexible by using a slot:

```html
<div class="card">
    {{ $slot }}
</div>
```

**What is `$slot`?** `$slot` is a special variable that contains everything between the opening and closing tags of your component.

Use it in your blade file:

```html
<x-card>
    Custom card Content
</x-card>
```

**What happens:** The content "Custom card Content" will be rendered in place of `{{ $slot }}`.

**Result:**

```html
<div class="card">
    Custom card Content
</div>
```

### Named Slots

**The limitation:** There can be only one default `$slot`, but you often need multiple content areas (header, body, footer).

**The solution:** Use named slots.

Update the `card` component to have multiple content areas:

```html
<div class="card">
    <div class="card-header">{{ $title }}</div>
    {{ $slot }}
    <div class="card-footer">{{ $footer }}</div>
</div>
```

**What changed:**

- `{{ $title }}` - named slot for header content
- `{{ $slot }}` - default slot for main content
- `{{ $footer }}` - named slot for footer content

To provide content for named slots, use the `<x-slot>` tag with colon syntax:

```html
<x-card>
    <x-slot:title>Card Header</x-slot>
    Card Content
    <x-slot:footer>Card footer</x-slot>
</x-card>
```

Alternative syntax with `name` attribute:

```html
<x-card>
    <x-slot name="title">Card title 1</x-slot>
    Card Content 1
    <x-slot name="footer">Card footer 1</x-slot>
</x-card>
```

**Both syntaxes are equivalent:**

- `<x-slot:title>` - shorthand, cleaner
- `<x-slot name="title">` - explicit, more readable

The name you provide becomes a variable inside the component (`$title`, `$footer`).

### Check if the Slot is Empty

You can check if a slot is empty and render alternative content:

```php
@if ($slot->isEmpty())
    <p>Please provide some content</p>
@else
    {{ $slot }}
@endif
```

**When to use:** When you want to show a default message or placeholder if no content is provided.

## Class Based Components

**Anonymous vs Class-based:**

- **Anonymous:** Just a blade file, simple UI
- **Class-based:** PHP class + blade file, can have logic, methods, computed properties

To create a class-based component, execute the `make:component` artisan command **without** the `--view` flag:

```bash
php artisan make:component SearchForm
```

This generates **two files**:

1. `app/View/Components/SearchForm.php` - the component class (logic)
2. `resources/views/components/search-form.blade.php` - the blade template (HTML)

**Important naming:**

- Command: `SearchForm` (PascalCase)
- PHP class: `SearchForm.php` (PascalCase)
- Blade file: `search-form.blade.php` (kebab-case)
- Usage: `<x-search-form />` (kebab-case)

### Generated Component Class

```php
<?php

namespace App\View\Components;

use Closure;
use Illuminate\Contracts\View\View;
use Illuminate\View\Component;

class SearchForm extends Component
{
    /**
     * Create a new component instance.
     */
    public function __construct()
    {
        //
    }

    /**
     * Get the view / contents that represent the component.
     */
    public function render(): View|Closure|string
    {
        return view('components.search-form');
    }
}
```

**Understanding the class:**

- `__construct()` - called when component is created, accepts properties
- `render()` - specifies which blade file to use

### Benefits of Class-Based Components

Class-based components allow you to:

**1. Override the default blade file location** by updating the `render` method:

```php
public function render(): View|Closure|string
{
    return view('components.forms.search'); // Custom path
}
```

**2. Add complex data preparation logic** in the `__construct` method:

```php
public function __construct()
{
    // Fetch data from database
    $this->categories = Category::all();
    
    // Perform calculations
    $this->totalItems = Product::count();
}
```

**When to use:** When your component needs to fetch data, perform calculations, or execute logic before rendering.

**3. Use public properties and methods** in the component's blade file:

```php
public function test()
{
    return 'Something';
}

public function formatPrice($price)
{
    return number_format($price, 2) . ' PLN';
}
```

In the blade file:

```php
<div>{{ $test() }}</div>
<div>{{ $formatPrice(100) }}</div>
```

**Output:**

```html
<div>Something</div>
<div>100.00 PLN</div>
```

## Attributes for Class Based Components

**What are component attributes?** Attributes are like function parameters - they allow you to pass data into your component to make it dynamic.

To pass properties to a `SearchForm` component (e.g., `action` and `method` for a form), define these properties in the `__construct` method:

```php
public function __construct(
    public string $action,
    public string $method
) {
    //
}
```

**What this does:**

- Accepts two parameters: `$action` and `$method`
- The `public` keyword makes them automatically available in the blade file
- No need to manually assign them to properties

In `search-form.blade.php`, use these properties:

```php
<form
    action="{{ $action }}"
    method="{{ $method }}"
    class="find-a-car-form card flex p-medium"
>
    @csrf
    <!-- Form fields here -->
</form>
```

Pass attributes when using the component:

```php
<x-search-form action="/search" method="GET" />
```

**Result:**

```html
<form action="/search" method="GET" class="find-a-car-form card flex p-medium">
    ...
</form>
```

### Default Values

You can provide default values for attributes so they're optional:

```php
public function __construct(
    public string $action = '/search',
    public string $method = 'GET'
) {
    //
}
```

Now the component can be used **without** attributes:

```php
<x-search-form />
```

This will use the defaults: `action="/search"` and `method="GET"`.

You can also override just one:

```php
<x-search-form action="/custom-search" />
```

This uses `action="/custom-search"` and `method="GET"` (default).

## Attributes for Anonymous Components

For anonymous components (without a PHP class), use the `@props` directive to accept attributes.

Open `components/card.blade.php`:

```php
@props(['color', 'bgColor'])

<div class="card">
    ...
</div>
```

**What `@props` does:**

- Defines which attributes the component accepts
- Makes them available as variables (`$color`, `$bgColor`)
- Attribute names in kebab-case become camelCase variables

### Using Properties

```php
@props(['color', 'bgColor'])

<div class="card card-text-{{ $color }} card-bg-{{ $bgColor }}">
    ...
</div>
```

Pass attributes to the component (use kebab-case):

```php
<x-card color='yellow' bg-color='blue' />
```

**Naming conversion:**

- HTML attribute: `bg-color` (kebab-case)
- PHP variable: `$bgColor` (camelCase)

This renders:

```html
<div class="card card-text-yellow card-bg-blue">
    ...
</div>
```

### Default Values

Provide default values in the `@props` directive:

```php
@props(['color' => 'white', 'bgColor' => 'gray'])

<div class="card card-text-{{ $color }} card-bg-{{ $bgColor }}">
    ...
</div>
```

Now you can use the component without attributes:

```php
<x-card />
```

**Result:**

```html
<div class="card card-text-white card-bg-gray">
    ...
</div>
```

Or override specific attributes:

```php
<x-card color="red" />
```

**Result:**

```html
<div class="card card-text-red card-bg-gray">
    ...
</div>
```

### Passing Variables as Attributes

**Static values** (hardcoded strings):

```php
<x-card color="yellow" bg-color="blue" />
```

**Dynamic values** (PHP variables):

If you have values stored in variables:

```php
@php
    $color = 'yellow';
    $bgColor = 'blue';
@endphp

<x-card :color="$color" :bg-color="$bgColor" />
```

**Important:** The colon (`:`) before the attribute name tells Laravel to evaluate the value as PHP code, not treat it as a string.

**Without colon:**

```php
<x-card color="$color" />  <!-- Passes the literal string "$color" -->
```

**With colon:**

```php
<x-card :color="$color" />  <!-- Passes the value of $color variable -->
```

### Shorthand Syntax

When the attribute name matches the variable name exactly, use shorthand syntax:

```php
@php
    $color = 'yellow';
    $bgColor = 'blue';
@endphp

<x-card :$color :$bgColor />
```

**This is equivalent to:**

```php
<x-card :color="$color" :bg-color="$bgColor" />
```

**When to use:**

- Shorthand: When variable name matches attribute name
- Full syntax: When you need to pass expressions or transform values

## Additional Attributes to Components

**What are additional attributes?** These are HTML attributes you pass to a component that aren't explicitly defined in `@props` or `__construct`. Laravel collects them automatically in the `$attributes` variable.

### Passing Additional Attributes

```php
@php
    $color = 'red';
    $bgColor = 'blue';
@endphp

<x-card :$color :$bgColor lang="en" class="card-rounded">
    ...
</x-card>
```

**In this example:**

- `color` and `bgColor` - defined props (handled explicitly)
- `lang` and `class` - additional attributes (not in `@props`)

### Rendering Additional Attributes

**What is `$attributes`?** `$attributes` is a special variable that contains all the HTML attributes passed to your component that weren't explicitly defined as props.

To render additional attributes on the root HTML element, use the `$attributes` variable:

```php
@props(['color', 'bgColor'])

<div{{ $attributes }} class="card card-text-{{ $color }} card-bg-{{ $bgColor }}">
    ...
</div>
```

**Note the syntax:** `<div{{ $attributes }}>` - no space before `{{`

This renders:

```html
<div lang="en" class="card-rounded" class="card card-text-red card-bg-blue">
    ...
</div>
```

**Problem:** Notice the `class` attribute appears twice:

- Once from `$attributes` (`class="card-rounded"`)
- Once defined directly (`class="card card-text-red card-bg-blue"`)

This creates invalid HTML with duplicate `class` attributes.

### Merging CSS Classes

To merge the `class` attribute from the component usage with classes defined in the component:

```php
@props(['color', 'bgColor'])

<div {{ $attributes->class("card card-text-$color card-bg-$bgColor") }}>
    ...
</div>
```

**What `->class()` does:**

- Takes the `class` from `$attributes`
- Merges it with the classes you specify
- Removes duplicates
- Outputs a single `class` attribute

This renders:

```html
<div class="card card-text-red card-bg-blue card-rounded" lang="en">
    ...
</div>
```

**Now there's only one `class` attribute** with all classes merged together, and other attributes (`lang`) are preserved.

### Merging with Default Attributes

To provide default values for attributes that may not be passed:

```php
@props(['color', 'bgColor'])

<div {{
    $attributes
        ->class("card card-text-$color card-bg-$bgColor")
        ->merge(['lang' => 'en'])
}}>
    ...
</div>
```

**What `->merge()` does:**

- Provides default values for attributes
- If the attribute is already in `$attributes`, it uses that value
- If not, it uses the default value

Now, if you use the component **without** the `lang` attribute:

```html
<x-card color="red" bg-color="blue" class="card-rounded" />
```

It renders with the default `lang="en"`:

```html
<div lang="en" class="card card-text-red card-bg-blue card-rounded">
    ...
</div>
```

**But if you pass `lang` explicitly:**

```html
<x-card color="red" bg-color="blue" class="card-rounded" lang="pl" />
```

It uses your value:

```html
<div lang="pl" class="card card-text-red card-bg-blue card-rounded">
    ...
</div>
```

## Component Slot Attributes

You can add HTML attributes to named slots, just like regular HTML elements:

```html
<x-card>
    <x-slot:title class="card-header-blue">Card Header</x-slot>
    Card Content
    <x-slot:footer>Card footer</x-slot>
</x-card>
```

**What this does:** Passes a `class` attribute to the `title` slot that can be used in the component template.

### Accessing Slot Attributes in the Component

First, add `title` and `footer` to the `@props` directive so Laravel knows they're slots, then access slot attributes using `$title->attributes`:

```php
@props(['color', 'bgColor', 'title', 'footer'])

<div class="card">
    <div {{ $title->attributes->class('card-header') }}>
        {{ $title }}
    </div>
    {{ $slot }}
    <div class="card-footer">{{ $footer }}</div>
</div>
```

**Understanding slot attributes:**

- `$title` - the slot content
- `$title->attributes` - the attributes passed to that slot
- `$title->attributes->class('card-header')` - merges slot classes with default classes

**Result:**

```html
<div class="card">
    <div class="card-header card-header-blue">
        Card Header
    </div>
    Card Content
    <div class="card-footer">Card footer</div>
</div>
```

**Key point:** The `$title->attributes` object works the same as `$attributes`, so you can use `->class()`, `->merge()`, and other methods.

## Reserved Keywords

The following keywords are reserved for Blade's internal use and **cannot** be defined as public properties or method names in your components:

- `data`
- `render`
- `resolveView`
- `shouldRender`
- `view`
- `withAttributes`
- `withName`

**Why this matters:** If you try to use these names, Laravel will throw an error or behave unexpectedly because these names are already used internally by the component system.

**Instead of:**

```php
public function render() // ❌ Reserved
{
    return 'something';
}
```

**Use:**

```php
public function renderContent() // ✅ Safe
{
    return 'something';
}
```

## Inline Component Views

**What are inline components?** Inline components are class-based components that don't have a separate `*.blade.php` file. Instead, the HTML content is returned directly from the component class's `render()` method as a string.

**When to use:**

- Very small components (1-5 lines of HTML)
- Components that are only used in one place
- When you want to keep everything in one file

```php
public function render(): View|Closure|string
{
    return <<<'blade'
<div {{ $attributes }}>
    {{ $slot }}
</div>
blade;
}
```

**Understanding the syntax:**

- `<<<'blade'` - starts a heredoc string (PHP's way of defining multi-line strings)
- `blade` - just a label (could be anything, but 'blade' is conventional)
- Everything between the heredocs is the HTML template
- The closing `blade;` must be on its own line with no indentation

Everything else about inline components works the same as regular components:

1. You can use them the same way: `<x-component-name />`
2. You can pass attributes and define them in `__construct`
3. You can merge classes or attributes with `->class()` and `->merge()` methods
4. You can define slots with `{{ $slot }}`
5. You can use class component methods

The only difference is that the HTML is defined in the PHP class instead of a separate blade file.

**Example of a complete inline component:**

```php
<?php

namespace App\View\Components;

use Closure;
use Illuminate\Contracts\View\View;
use Illuminate\View\Component;

class Alert extends Component
{
    public function __construct(
        public string $type = 'info'
    ) {}

    public function render(): View|Closure|string
    {
        return <<<'blade'
<div {{ $attributes->class("alert alert-$type") }}>
    {{ $slot }}
</div>
blade;
    }
}
```

**Usage:**

```php
<x-alert type="success" class="mb-4">
    Your profile has been updated!
</x-alert>
```

## Layouts Using Components

**Traditional approach vs Component approach:**

- **Traditional:** Uses `@extends`, `@section`, `@yield` (template inheritance)
- **Component-based:** Uses components with slots (more flexible, easier to understand)

This section shows how to recreate the same layout system using components instead of traditional Blade directives.

### Create BaseLayout and AppLayout Components

**Step 1:** Generate the components using artisan:

```bash
php artisan make:component BaseLayout
php artisan make:component AppLayout
```

This creates:

- `app/View/Components/BaseLayout.php` and `resources/views/components/base-layout.blade.php`
- `app/View/Components/AppLayout.php` and `resources/views/components/app-layout.blade.php`

**Step 2:** Delete the automatically created blade files (we'll use custom paths):

```bash
rm resources/views/components/base-layout.blade.php
rm resources/views/components/app-layout.blade.php
```

**Step 3:** Rename your existing layout:

```bash
mv resources/views/layouts/clean.blade.php resources/views/layouts/base.blade.php
```

**Step 4:** Update the component classes to point to the correct blade files:

In `app/View/Components/BaseLayout.php`:

```php
public function render(): View|Closure|string
{
    return view('layouts.base');
}
```

In `app/View/Components/AppLayout.php`:

```php
public function render(): View|Closure|string
{
    return view('layouts.app');
}
```

**Step 5:** Move the header partial to components folder:

```bash
mv resources/views/layouts/partials/header.blade.php resources/views/components/layouts/header.blade.php
```

Now you can use it as: `<x-layouts.header />`

### Final Version of `base.blade.php`

This is the most basic layout with minimal HTML structure:

```php
@props(['bodyClass' => '', 'title' => ''])

<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>{{ $title }} | {{ config('app.name', 'Laravel') }}</title>

    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link
        href="https://fonts.googleapis.com/css2?family=Ubuntu:ital,wght@0,300;0,400;0,500;0,700;1,300;1,400;1,500;1,700&display=swap"
        rel="stylesheet"
    />
    <link
        rel="stylesheet"
        href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css"
        integrity="sha512-1ycn6IcaQQ40/MKBW2W4Rhis/DbILU74C1vSrLJxCq57o941Ym01SwNsOMqvEBFlcgUa6xLiPY/NS5R+E6ztJQ=="
        crossorigin="anonymous"
        referrerpolicy="no-referrer"
    />

    <link rel="stylesheet" href="css/app.css" />
</head>
<body @if($bodyClass)class="{{ $bodyClass }}"@endif>

{{ $slot }}

<script
    src="https://cdnjs.cloudflare.com/ajax/libs/scrollReveal.js/4.0.9/scrollreveal.js"
    integrity="sha512-XJgPMFq31Ren4pKVQgeD+0JTDzn0IwS1802sc+QTZckE6rny7AN2HLReq6Yamwpd2hFe5nJJGZLvPStWFv5Kww=="
    crossorigin="anonymous"
    referrerpolicy="no-referrer"
></script>
<script src="js/app.js"></script>
</body>
</html>
```

**What changed:**

- `@props(['bodyClass' => '', 'title' => ''])` - accepts two props with defaults
- `{{ $title }}` - uses prop instead of `@yield('title')`
- `@if($bodyClass)class="{{ $bodyClass }}"@endif` - conditionally adds body class
- `{{ $slot }}` - uses slot instead of `@yield('childContent')`

### Final Version of `app.blade.php`

This is the application layout that includes header and footer:

```php
@props(['title' => '', 'footerLinks' => ''])

<x-base-layout :$title>
    <x-layouts.header />
    {{ $slot }}
    <footer>
        <a href="#">Link 1</a>
        <a href="#">Link 2</a>
        {{ $footerLinks }}
    </footer>
</x-base-layout>
```

**What this does:**

- Accepts `title` and `footerLinks` props
- Passes `title` to `BaseLayout` component using `:$title` (shorthand)
- Includes header component
- Renders main content via `{{ $slot }}`
- Renders footer with default links + additional `footerLinks`

**Component nesting:** `BaseLayout` (HTML structure) → `AppLayout` (header/footer) → `index.blade.php` (page content)

### Final Version of `index.blade.php`

This is a regular page using the component-based layout:

```php
<x-app-layout title="Home Page">
    @php
        $color = 'red';
        $bgColor = 'blue';
    @endphp
    <x-card :$color :$bgColor class="card-rounded">
        <x-slot:title class="card-header-blue">Card title 1</x-slot>
        Card Content 1
        <x-slot:footer>Card footer 1</x-slot>
    </x-card>
    <x-test-component class="card">
        Lorem ipsum dolor sit amet
    </x-test-component>
    <!-- Home Slider -->
    <section class="home-slider">
        ...
    </section>
    <!--/ Home Slider -->
    <main>
        ...
    </main>
    <x-slot:footerLinks>
        <a href="#">Link 3</a>
        <a href="#">Link 4</a>
    </x-slot:footerLinks>
</x-app-layout>
```

**What's happening:**

- `<x-app-layout title="Home Page">` - uses AppLayout with page title
- Main content goes in default slot (between opening/closing tags)
- `<x-slot:footerLinks>` - provides additional footer links
- All components and content are wrapped in the layout

### Final Version of `login.blade.php`

This page uses BaseLayout directly, skipping AppLayout (no header/footer):

```php
<x-base-layout title="Login" bodyClass="page-login">
    <main>
        <form action="/login" method="POST">
            @csrf
            <!-- Login form fields -->
        </form>
    </main>
</x-base-layout>
```

**Why skip AppLayout?** Login pages typically don't need the site header/footer - they need a minimal layout. This demonstrates the flexibility of component-based layouts.

**Summary of component-based layouts:**

**Advantages:**

- More flexible than `@extends/@section`
- Easier to understand (components are familiar)
- Can pass props directly
- Can compose layouts in different ways
- Better IDE support

**When to use:**

- New Laravel projects → Use components
- Existing projects with `@extends/@section` → Either is fine, no need to migrate unless you want to

This demonstrates using the component-based layout system where layouts are components that accept props and slots, providing a more flexible and modern alternative to traditional template inheritance with `@extends` and `@section`.

## Adding Custom CSS in Laravel

### 1. **Simple Approach – `public` Directory**

Create a CSS file in `public/css/`:

```bash
public/css/custom.css
```

Include it in your Blade template:

```html
<link rel="stylesheet" href="{{ asset('css/custom.css') }}">
```

- Best for small or very simple projects
- No bundling, versioning, or optimization

---

### 2. **Vite (Laravel 9+, Recommended)**

Place your CSS file in `resources/css/`:

```bash
resources/css/app.css
```

Make sure your `vite.config.js` contains:

```javascript
export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```

Include it in your Blade template:

```html
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

Run Vite:

```bash
npm run dev    # development
npm run build  # production
```

---

### 3. **Multiple CSS Files with Vite**

You can create additional CSS files, e.g.:

```bash
resources/css/custom.css
```

Then register them in `vite.config.js`:

```javascript
input: [
    'resources/css/app.css',
    'resources/css/custom.css',
    'resources/js/app.js'
]
```

And include them in Blade:

```html
@vite(['resources/css/custom.css'])
```

💡 **Best practice**:
For larger projects, import custom styles inside `app.css`:

```css
@import './custom.css';
```
