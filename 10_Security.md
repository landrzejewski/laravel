## Step 1: Database Configuration

### 1.1 Modify Users Migration

Edit `database/migrations/xxxx_xx_xx_create_users_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->enum('role', ['admin', 'editor', 'user'])->default('user');
            $table->rememberToken();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

### 1.2 Run Migrations

```bash
php artisan migrate
```

### 1.3 Modify User Model

Edit `app/Models/User.php`:

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }

    // Helper methods for checking roles
    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }

    public function isEditor(): bool
    {
        return $this->role === 'editor';
    }

    public function isUser(): bool
    {
        return $this->role === 'user';
    }
}
```

### 1.4 Seeder for Test Users

Create seeder `database/seeders/UserSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Admin user
        User::create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => Hash::make('password'),
            'role' => 'admin',
        ]);

        // Editor user
        User::create([
            'name' => 'Editor User',
            'email' => 'editor@example.com',
            'password' => Hash::make('password'),
            'role' => 'editor',
        ]);

        // Regular user
        User::create([
            'name' => 'Regular User',
            'email' => 'user@example.com',
            'password' => Hash::make('password'),
            'role' => 'user',
        ]);
    }
}
```

Run the seeder:

```bash
php artisan db:seed --class=UserSeeder
```

## Step 2: Creating Authorization Policy

### 2.1 Generate Policy

```bash
php artisan make:policy UserPolicy
```

Edit `app/Policies/UserPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Models\User;

class UserPolicy
{
    /**
     * Determine if the user can access admin panel.
     */
    public function accessAdmin(User $user): bool
    {
        return $user->isAdmin();
    }

    /**
     * Determine if the user can access editor panel.
     */
    public function accessEditor(User $user): bool
    {
        return $user->isAdmin() || $user->isEditor();
    }

    /**
     * Determine if the user can access user dashboard.
     */
    public function accessDashboard(User $user): bool
    {
        return true; // All authenticated users
    }
}
```

### 2.2 Register Policy

In `app/Providers/AppServiceProvider.php` add:

```php
<?php

namespace App\Providers;

use App\Models\User;
use App\Policies\UserPolicy;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        // Define gates
        Gate::define('access-admin', [UserPolicy::class, 'accessAdmin']);
        Gate::define('access-editor', [UserPolicy::class, 'accessEditor']);
        Gate::define('access-dashboard', [UserPolicy::class, 'accessDashboard']);
    }
}
```

## Step 3: Middleware for Role Checking

### 3.1 Create Middleware

```bash
php artisan make:middleware CheckRole
```

Edit `app/Http/Middleware/CheckRole.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckRole
{
    public function handle(Request $request, Closure $next, string $role): Response
    {
        if (!$request->user()) {
            return redirect()->route('login');
        }

        if ($request->user()->role !== $role && !$request->user()->isAdmin()) {
            abort(403, 'Unauthorized action.');
        }

        return $next($request);
    }
}
```

### 3.2 Register Middleware

In `bootstrap/app.php` add middleware alias:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'role' => \App\Http\Middleware\CheckRole::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

## Step 4: Controllers

### 4.1 Auth Controller

```bash
php artisan make:controller AuthController
```

Edit `app/Http/Controllers/AuthController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\View\View;

class AuthController extends Controller
{
    /**
     * Show login form
     */
    public function showLogin(): View
    {
        return view('auth.login');
    }

    /**
     * Handle login request
     */
    public function login(Request $request): RedirectResponse
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);

        if (Auth::attempt($credentials, $request->filled('remember'))) {
            $request->session()->regenerate();

            // Redirect based on role
            if (Auth::user()->isAdmin()) {
                return redirect()->intended('/admin');
            } elseif (Auth::user()->isEditor()) {
                return redirect()->intended('/editor');
            }

            return redirect()->intended('/dashboard');
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ])->onlyInput('email');
    }

    /**
     * Show register form
     */
    public function showRegister(): View
    {
        return view('auth.register');
    }

    /**
     * Handle register request
     */
    public function register(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'string', 'min:8', 'confirmed'],
        ]);

        $user = \App\Models\User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => $validated['password'],
            'role' => 'user', // Default role
        ]);

        Auth::login($user);

        return redirect('/dashboard');
    }

    /**
     * Handle logout
     */
    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

### 4.2 Dashboard Controllers

```bash
php artisan make:controller AdminController
php artisan make:controller EditorController
php artisan make:controller DashboardController
```

`app/Http/Controllers/AdminController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\View\View;

class AdminController extends Controller
{
    public function index(): View
    {
        return view('admin.index');
    }
}
```

`app/Http/Controllers/EditorController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\View\View;

class EditorController extends Controller
{
    public function index(): View
    {
        return view('editor.index');
    }
}
```

`app/Http/Controllers/DashboardController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\View\View;

class DashboardController extends Controller
{
    public function index(): View
    {
        return view('dashboard.index');
    }
}
```

## Step 5: Routes

Edit `routes/web.php`:

```php
<?php

use App\Http\Controllers\AuthController;
use App\Http\Controllers\AdminController;
use App\Http\Controllers\EditorController;
use App\Http\Controllers\DashboardController;
use Illuminate\Support\Facades\Route;

// Public routes
Route::get('/', function () {
    return view('welcome');
})->name('home');

// Guest routes (only for non-authenticated users)
Route::middleware('guest')->group(function () {
    Route::get('/login', [AuthController::class, 'showLogin'])->name('login');
    Route::post('/login', [AuthController::class, 'login']);
    Route::get('/register', [AuthController::class, 'showRegister'])->name('register');
    Route::post('/register', [AuthController::class, 'register']);
});

// Authenticated routes
Route::middleware('auth')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout'])->name('logout');

    // User dashboard - accessible by all authenticated users
    Route::get('/dashboard', [DashboardController::class, 'index'])
        ->name('dashboard');

    // Editor panel - accessible by editors and admins
    Route::middleware('can:access-editor')->group(function () {
        Route::get('/editor', [EditorController::class, 'index'])->name('editor.index');
    });

    // Admin panel - accessible only by admins
    Route::middleware('can:access-admin')->group(function () {
        Route::get('/admin', [AdminController::class, 'index'])->name('admin.index');
    });
});
```

## Step 6: Blade Templates

### 6.1 Layout Template

Create `resources/views/layouts/app.blade.php`:

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@yield('title', 'Laravel Auth')</title>
</head>
<body>
    <nav>
        <a href="{{ route('home') }}">Home</a>
        
        @auth
            <a href="{{ route('dashboard') }}">Dashboard</a>
            
            @can('access-editor')
                <a href="{{ route('editor.index') }}">Editor Panel</a>
            @endcan
            
            @can('access-admin')
                <a href="{{ route('admin.index') }}">Admin Panel</a>
            @endcan
            
            <span>Welcome, {{ Auth::user()->name }} ({{ Auth::user()->role }})</span>
            
            <form method="POST" action="{{ route('logout') }}" style="display: inline;">
                @csrf
                <button type="submit">Logout</button>
            </form>
        @else
            <a href="{{ route('login') }}">Login</a>
            <a href="{{ route('register') }}">Register</a>
        @endauth
    </nav>

    <hr>

    <main>
        @if ($errors->any())
            <div style="color: red;">
                <ul>
                    @foreach ($errors->all() as $error)
                        <li>{{ $error }}</li>
                    @endforeach
                </ul>
            </div>
        @endif

        @if (session('success'))
            <div style="color: green;">
                {{ session('success') }}
            </div>
        @endif

        @yield('content')
    </main>
</body>
</html>
```

### 6.2 Welcome Page

Create `resources/views/welcome.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Welcome')

@section('content')
    <h1>Welcome to Laravel Auth System</h1>
    
    <p>This is a demonstration of Laravel authentication and authorization with role-based access control.</p>

    @guest
        <p>Please <a href="{{ route('login') }}">login</a> or <a href="{{ route('register') }}">register</a> to continue.</p>
    @endguest

    @auth
        <p>You are logged in as <strong>{{ Auth::user()->role }}</strong></p>
        <p>Go to your <a href="{{ route('dashboard') }}">dashboard</a></p>
    @endauth

    <h2>Test Credentials</h2>
    <ul>
        <li>Admin: admin@example.com / password</li>
        <li>Editor: editor@example.com / password</li>
        <li>User: user@example.com / password</li>
    </ul>
@endsection
```

### 6.3 Login Page

Create `resources/views/auth/login.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Login')

@section('content')
    <h1>Login</h1>

    <form method="POST" action="{{ route('login') }}">
        @csrf

        <div>
            <label for="email">Email:</label>
            <input 
                type="email" 
                id="email" 
                name="email" 
                value="{{ old('email') }}" 
                required 
                autofocus
            >
            @error('email')
                <span style="color: red;">{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label for="password">Password:</label>
            <input 
                type="password" 
                id="password" 
                name="password" 
                required
            >
            @error('password')
                <span style="color: red;">{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label>
                <input type="checkbox" name="remember">
                Remember me
            </label>
        </div>

        <div>
            <button type="submit">Login</button>
        </div>
    </form>

    <p>Don't have an account? <a href="{{ route('register') }}">Register here</a></p>
@endsection
```

### 6.4 Register Page

Create `resources/views/auth/register.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Register')

@section('content')
    <h1>Register</h1>

    <form method="POST" action="{{ route('register') }}">
        @csrf

        <div>
            <label for="name">Name:</label>
            <input 
                type="text" 
                id="name" 
                name="name" 
                value="{{ old('name') }}" 
                required 
                autofocus
            >
            @error('name')
                <span style="color: red;">{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label for="email">Email:</label>
            <input 
                type="email" 
                id="email" 
                name="email" 
                value="{{ old('email') }}" 
                required
            >
            @error('email')
                <span style="color: red;">{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label for="password">Password:</label>
            <input 
                type="password" 
                id="password" 
                name="password" 
                required
            >
            @error('password')
                <span style="color: red;">{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label for="password_confirmation">Confirm Password:</label>
            <input 
                type="password" 
                id="password_confirmation" 
                name="password_confirmation" 
                required
            >
        </div>

        <div>
            <button type="submit">Register</button>
        </div>
    </form>

    <p>Already have an account? <a href="{{ route('login') }}">Login here</a></p>
@endsection
```

### 6.5 Dashboard Pages

Create `resources/views/dashboard/index.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'User Dashboard')

@section('content')
    <h1>User Dashboard</h1>

    <p>Welcome to your dashboard, {{ Auth::user()->name }}!</p>

    <div>
        <h2>Your Account Information</h2>
        <ul>
            <li><strong>Name:</strong> {{ Auth::user()->name }}</li>
            <li><strong>Email:</strong> {{ Auth::user()->email }}</li>
            <li><strong>Role:</strong> {{ Auth::user()->role }}</li>
            <li><strong>Member since:</strong> {{ Auth::user()->created_at->format('Y-m-d') }}</li>
        </ul>
    </div>

    <div>
        <h2>Available Sections</h2>
        <ul>
            <li>✓ User Dashboard (current page)</li>
            
            @can('access-editor')
                <li>✓ <a href="{{ route('editor.index') }}">Editor Panel</a></li>
            @else
                <li>✗ Editor Panel (requires editor role)</li>
            @endcan
            
            @can('access-admin')
                <li>✓ <a href="{{ route('admin.index') }}">Admin Panel</a></li>
            @else
                <li>✗ Admin Panel (requires admin role)</li>
            @endcan
        </ul>
    </div>
@endsection
```

Create `resources/views/editor/index.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Editor Panel')

@section('content')
    <h1>Editor Panel</h1>

    <p>Welcome to the editor panel, {{ Auth::user()->name }}!</p>

    <div>
        <h2>Editor Features</h2>
        <p>This section is accessible only to users with 'editor' or 'admin' role.</p>
        
        <ul>
            <li>Create and edit content</li>
            <li>Manage articles</li>
            <li>Review submissions</li>
            <li>Publish content</li>
        </ul>
    </div>

    @can('access-admin')
        <div>
            <p>As an admin, you also have access to the <a href="{{ route('admin.index') }}">Admin Panel</a></p>
        </div>
    @endcan
@endsection
```

Create `resources/views/admin/index.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Admin Panel')

@section('content')
    <h1>Admin Panel</h1>

    <p>Welcome to the admin panel, {{ Auth::user()->name }}!</p>

    <div>
        <h2>Admin Features</h2>
        <p>This section is accessible only to users with 'admin' role.</p>
        
        <ul>
            <li>Manage users</li>
            <li>Configure system settings</li>
            <li>View analytics</li>
            <li>Access all system features</li>
        </ul>
    </div>

    <div>
        <h2>Quick Links</h2>
        <ul>
            <li><a href="{{ route('dashboard') }}">User Dashboard</a></li>
            <li><a href="{{ route('editor.index') }}">Editor Panel</a></li>
        </ul>
    </div>
@endsection
```

## Step 7: Testing

### 7.1 Start Server

```bash
php artisan serve
```

### 7.2 Test the Application

1. Open `http://localhost:8000`
2. Register a new user or login with test credentials:
   - **Admin**: admin@example.com / password
   - **Editor**: editor@example.com / password
   - **User**: user@example.com / password

### 7.3 Check Access Control

- **User** has access only to `/dashboard`
- **Editor** has access to `/dashboard` and `/editor`
- **Admin** has access to all sections: `/dashboard`, `/editor`, `/admin`

## Additional Features

### Checking Authorization in Controllers

```php
public function someMethod()
{
    // Using Gate
    if (Gate::allows('access-admin')) {
        // User is admin
    }

    // Using Gate facade authorize method
    Gate::authorize('access-admin'); // Throws 403 if not authorized

    // Using User model can method
    if (auth()->user()->can('access-admin')) {
        // User is authorized
    }
}
```

### Checking in Blade Templates

```blade
@can('access-admin')
    <!-- Content for admins -->
@elsecan('access-editor')
    <!-- Content for editors -->
@else
    <!-- Content for regular users -->
@endcan

@cannot('access-admin')
    <!-- Content for non-admins -->
@endcannot
```

### Middleware in Routes

```php
// Using gate
Route::get('/admin', [AdminController::class, 'index'])
    ->middleware('can:access-admin');

// Using custom role middleware
Route::get('/admin', [AdminController::class, 'index'])
    ->middleware('role:admin');
```
