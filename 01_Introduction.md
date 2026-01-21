## Create Laravel Project

There are several ways to create a Laravel project locally.

### Install using Composer

```bash
composer create-project laravel/laravel [YOUR_PROJECT_NAME]
```

### Install with Laravel Installer

Install the Laravel installer globally and use it to create new projects (the installer uses Composer internally):

```bash
# Install Laravel Installer globally
composer global require laravel/installer

# Create new project with Laravel Installer
laravel new [YOUR_PROJECT_NAME]
```

### Install with Sail

Laravel Sail provides a Docker-based local development environment that can be set up with simple commands.

```bash
curl -s https://laravel.build/[YOUR_PROJECT_NAME] | bash
```

### Using Laravel Herd

You can create a new project through Laravel Herd's interface. Navigate to the Sites menu item and add a new site.

## Start the Project

To start the Laravel project, navigate to the project's root directory in your terminal and execute:

```bash
php artisan serve
```

When Laravel starts successfully, you'll see the address http://127.0.0.1:8000 in the output. Open this URL in your browser to see your new Laravel installation.

Na Windows należy dodać `variables_order = "GPCS"` do pliku php.ini.

## Introduction to Artisan

### What is Artisan?

Artisan is Laravel's command-line tool that helps you perform various development tasks such as generating code, managing migrations, clearing caches, and more.

### Get Information About the Project

```bash
php artisan about
```

### Get the List of All Artisan Commands

```bash
php artisan list
```

### Get the List of Commands in a Specific Namespace

```bash
php artisan make help
```

This returns all available commands in the `make` namespace.

### Get Details of a Specific Command

```bash
php artisan make:controller --help
```

This displays detailed information about the command, including available options and arguments.

## Artisan Challenge

### Challenge 1

Execute an artisan command to get the list of artisan commands in the `queue` namespace.

### Challenge 2

Execute an artisan command to get all details of the `migrate:refresh` command.

## Directory Structure

### Explore All Files and Folders in Root Directory

Laravel provides a default folder and file structure. Each folder serves a specific purpose in the application architecture.

The `app` folder is where most of your application code lives. Everything inside the `app` folder is autoloaded, meaning you can create new folders and place classes there without additional configuration.

For example:
- You can create an `app/Enum` folder to store PHP enums
- Certain artisan commands automatically create folders. For instance:

```bash
php artisan make:job UserJob
```

This creates an `app/Jobs` folder and places the new file inside.

## Define a Basic Route With Basic View File

**Important:** Whenever you define a route, ensure the Route class is properly imported at the top of the file. If not imported, add:

```php
use Illuminate\Support\Facades\Route;
```

Open `routes/web.php` and add the following code:

```php
// routes/web.php
Route::get("/about", function() {
    return view("about");
});
```

Create `resources/views/about.blade.php` and add HTML content:

```html
<h1>Hello, This course is on Laravel</h1>
<p>Lorem ipsum dolor sit amet consectetur adipisicing elit. Odit, nulla quisquam! Expedita porro quos, omnis nulla reprehenderit sequi quas fugiat ullam itaque repudiandae, eaque sapiente totam minus. Quasi, dignissimos tempore.</p>
```

Access the URL http://127.0.0.1:8000/about in your browser to see the result.

## Request Lifecycle

In Laravel, the request starts at `public/index.php`. This file includes `vendor/autoload.php`, then creates the application in `bootstrap/app.php`. The application loads configuration files and initiates service providers.

**Service Providers** prepare and configure the application, handling tasks like registering services, event listeners, middleware, and routes. They ensure both core and custom features are set up correctly, making the application ready to function efficiently.

The request is then passed to the Router.

The **Router** takes the request, finds the corresponding function to execute, and runs it. Optionally, middleware may be registered on the route. Middleware executes code before the request reaches the action and can potentially block execution. After processing, the controller sends the response back to the user. If middleware is registered, the response passes through the middleware first before being returned to the user.

## Configuration

Laravel comes with several configuration files out of the box. Many configuration options are also available in the `.env` file.

If you don't need to modify these configuration files, you can delete them. The default files exist in the `vendor` folder in Laravel's core and will be used automatically.

To create configuration files for different services, use the artisan command:

```bash
php artisan config:publish
```

This displays a prompt where you type the configuration file name (e.g., `app` or `broadcasting`).

If you know exactly which configuration file you want to publish, execute:

```bash
php artisan config:publish [FILE_NAME]
```

Example:
```bash
php artisan config:publish app
```

### Customizing Configuration Files

When you publish a configuration file (e.g., `cors`), you don't need to keep all options in your local file. You can customize only the specific settings you need and delete the rest.

For example, if you only want to change `supports_credentials` to `true`, your `config/cors.php` can look like:

```php
<?php

return [
    'supports_credentials' => true,
];
```

All other configuration options will use the default values from Laravel's core.

## Debugging Functions

### dump()

Prints variable information and shows the location from which it was called:

```php
$person = [
    "name" => "Zura",
    "email" => "zura@example.com",
];
dump($person);
```

### dd()

Prints the same output as `dump()`, but stops script execution immediately:

```php
$person = [
    "name" => "Zura",
    "email" => "zura@example.com",
];
dd($person);
```

The "dd" stands for "dump and die" - it dumps the variable and terminates the script execution.