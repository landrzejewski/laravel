## What are Services in Laravel?

**Services** are classes that contain your application's business logic. They are part of a design pattern called **Service Layer Pattern**, which helps separate business logic from controllers and models.

## What are Services used for?

### 1. **Separating business logic from controllers**
Controllers should be "thin" - their main job is to receive requests and return responses, not execute complex logic.

### 2. **Code reusability**
The same logic can be used in different places (controllers, Artisan commands, jobs, events).

### 3. **Easier testing**
Business logic in separate classes is easier to test than code in controllers.

### 4. **Better code management**
Code is better organized and easier to maintain.

### 5. **Single Responsibility Principle**
Each class has one responsibility - services handle logic, controllers handle HTTP.

## When to use Services?

### USE services when:
- Business logic is complex
- The same code is used in multiple places
- Operation requires multiple steps
- You want to easily test code
- Logic goes beyond simple CRUD

### DON'T USE services when:
- Simple CRUD (Create, Read, Update, Delete)
- One-line operation
- Code used only in one place

## Usage Examples

### Example 1: BEFORE using service (BAD PATTERN)

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Models\Order;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use App\Mail\OrderConfirmation;

class OrderController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([
            'user_id' => 'required|exists:users,id',
            'products' => 'required|array',
            'products.*.id' => 'required|exists:products,id',
            'products.*.quantity' => 'required|integer|min:1',
        ]);

        // CONTROLLER DOES TOO MUCH!
        DB::beginTransaction();
        
        try {
            // Calculate total
            $total = 0;
            foreach ($validated['products'] as $product) {
                $productModel = \App\Models\Product::find($product['id']);
                $total += $productModel->price * $product['quantity'];
                
                // Check availability
                if ($productModel->stock < $product['quantity']) {
                    throw new \Exception('Not enough stock');
                }
                
                // Decrease stock
                $productModel->stock -= $product['quantity'];
                $productModel->save();
            }
            
            // Create order
            $order = Order::create([
                'user_id' => $validated['user_id'],
                'total' => $total,
                'status' => 'pending',
            ]);
            
            // Add products to order
            foreach ($validated['products'] as $product) {
                $order->products()->attach($product['id'], [
                    'quantity' => $product['quantity'],
                    'price' => \App\Models\Product::find($product['id'])->price,
                ]);
            }
            
            // Send email
            $user = User::find($validated['user_id']);
            Mail::to($user)->send(new OrderConfirmation($order));
            
            DB::commit();
            
            return response()->json(['order' => $order], 201);
            
        } catch (\Exception $e) {
            DB::rollback();
            return response()->json(['error' => $e->getMessage()], 400);
        }
    }
}
```

### Example 1: AFTER using service (GOOD PATTERN)

**Service: `app/Services/OrderService.php`**

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\Product;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use App\Mail\OrderConfirmation;

class OrderService
{
    /**
     * Create a new order
     */
    public function createOrder(int $userId, array $products): Order
    {
        DB::beginTransaction();
        
        try {
            // Validate availability
            $this->validateProductAvailability($products);
            
            // Calculate total
            $total = $this->calculateTotal($products);
            
            // Create order
            $order = $this->createOrderRecord($userId, $total);
            
            // Add products
            $this->attachProducts($order, $products);
            
            // Update stock
            $this->updateStock($products);
            
            // Send confirmation
            $this->sendConfirmationEmail($order);
            
            DB::commit();
            
            return $order;
            
        } catch (\Exception $e) {
            DB::rollback();
            throw $e;
        }
    }
    
    /**
     * Validate product availability
     */
    protected function validateProductAvailability(array $products): void
    {
        foreach ($products as $product) {
            $productModel = Product::findOrFail($product['id']);
            
            if ($productModel->stock < $product['quantity']) {
                throw new \Exception("Not enough stock for product: {$productModel->name}");
            }
        }
    }
    
    /**
     * Calculate order total
     */
    protected function calculateTotal(array $products): float
    {
        $total = 0;
        
        foreach ($products as $product) {
            $productModel = Product::find($product['id']);
            $total += $productModel->price * $product['quantity'];
        }
        
        return $total;
    }
    
    /**
     * Create order record
     */
    protected function createOrderRecord(int $userId, float $total): Order
    {
        return Order::create([
            'user_id' => $userId,
            'total' => $total,
            'status' => 'pending',
        ]);
    }
    
    /**
     * Attach products to order
     */
    protected function attachProducts(Order $order, array $products): void
    {
        foreach ($products as $product) {
            $productModel = Product::find($product['id']);
            
            $order->products()->attach($product['id'], [
                'quantity' => $product['quantity'],
                'price' => $productModel->price,
            ]);
        }
    }
    
    /**
     * Update product stock
     */
    protected function updateStock(array $products): void
    {
        foreach ($products as $product) {
            $productModel = Product::find($product['id']);
            $productModel->decrement('stock', $product['quantity']);
        }
    }
    
    /**
     * Send confirmation email
     */
    protected function sendConfirmationEmail(Order $order): void
    {
        $user = $order->user;
        Mail::to($user)->send(new OrderConfirmation($order));
    }
}
```

**Controller: `app/Http/Controllers/OrderController.php`**

```php
<?php

namespace App\Http\Controllers;

use App\Services\OrderService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class OrderController extends Controller
{
    public function __construct(
        protected OrderService $orderService
    ) {}
    
    public function store(Request $request): JsonResponse
    {
        // CONTROLLER IS NOW SIMPLE AND CLEAN!
        $validated = $request->validate([
            'user_id' => 'required|exists:users,id',
            'products' => 'required|array',
            'products.*.id' => 'required|exists:products,id',
            'products.*.quantity' => 'required|integer|min:1',
        ]);
        
        try {
            $order = $this->orderService->createOrder(
                $validated['user_id'],
                $validated['products']
            );
            
            return response()->json(['order' => $order], 201);
            
        } catch (\Exception $e) {
            return response()->json(['error' => $e->getMessage()], 400);
        }
    }
}
```

## Example 2: User Management Service

**Service: `app/Services/UserService.php`**

```php
<?php

namespace App\Services;

use App\Models\User;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Storage;

class UserService
{
    /**
     * Create a new user
     */
    public function createUser(array $data): User
    {
        return User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
            'role' => $data['role'] ?? 'user',
        ]);
    }
    
    /**
     * Update user profile
     */
    public function updateProfile(User $user, array $data): User
    {
        $user->update([
            'name' => $data['name'] ?? $user->name,
            'email' => $data['email'] ?? $user->email,
        ]);
        
        if (isset($data['password'])) {
            $user->update([
                'password' => Hash::make($data['password'])
            ]);
        }
        
        return $user->fresh();
    }
    
    /**
     * Upload user avatar
     */
    public function uploadAvatar(User $user, $file): string
    {
        // Delete old avatar
        if ($user->avatar) {
            Storage::disk('public')->delete($user->avatar);
        }
        
        // Save new one
        $path = $file->store('avatars', 'public');
        
        $user->update(['avatar' => $path]);
        
        return $path;
    }
    
    /**
     * Delete user account
     */
    public function deleteUser(User $user): bool
    {
        // Delete avatar
        if ($user->avatar) {
            Storage::disk('public')->delete($user->avatar);
        }
        
        // Delete related data
        $user->posts()->delete();
        $user->comments()->delete();
        
        // Delete user
        return $user->delete();
    }
}
```

**Controller using the service:**

```php
<?php

namespace App\Http\Controllers;

use App\Services\UserService;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function __construct(
        protected UserService $userService
    ) {}
    
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8',
            'role' => 'nullable|in:user,admin',
        ]);
        
        $user = $this->userService->createUser($validated);
        
        return response()->json(['user' => $user], 201);
    }
    
    public function uploadAvatar(Request $request, User $user)
    {
        $request->validate([
            'avatar' => 'required|image|max:2048',
        ]);
        
        $path = $this->userService->uploadAvatar($user, $request->file('avatar'));
        
        return response()->json(['avatar_path' => $path]);
    }
}
```

## Example 3: Report Service

**Service: `app/Services/ReportService.php`**

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\User;
use Carbon\Carbon;
use Illuminate\Support\Collection;

class ReportService
{
    /**
     * Get sales report for date range
     */
    public function getSalesReport(Carbon $startDate, Carbon $endDate): array
    {
        $orders = Order::whereBetween('created_at', [$startDate, $endDate])->get();
        
        return [
            'total_orders' => $orders->count(),
            'total_revenue' => $orders->sum('total'),
            'average_order_value' => $orders->avg('total'),
            'orders_by_status' => $this->getOrdersByStatus($orders),
            'daily_sales' => $this->getDailySales($orders),
        ];
    }
    
    /**
     * Get user statistics
     */
    public function getUserStatistics(): array
    {
        return [
            'total_users' => User::count(),
            'active_users' => User::where('last_login_at', '>', now()->subDays(30))->count(),
            'new_users_this_month' => User::whereMonth('created_at', now()->month)->count(),
            'users_by_role' => User::groupBy('role')->selectRaw('role, count(*) as count')->pluck('count', 'role'),
        ];
    }
    
    /**
     * Get orders grouped by status
     */
    protected function getOrdersByStatus(Collection $orders): array
    {
        return $orders->groupBy('status')
            ->map(fn($group) => $group->count())
            ->toArray();
    }
    
    /**
     * Get daily sales
     */
    protected function getDailySales(Collection $orders): array
    {
        return $orders->groupBy(fn($order) => $order->created_at->format('Y-m-d'))
            ->map(fn($group) => $group->sum('total'))
            ->toArray();
    }
}
```

## Directory Structure for Services

```
app/
├── Services/
│   ├── OrderService.php
│   ├── UserService.php
│   ├── ReportService.php
│   ├── PaymentService.php
│   └── NotificationService.php
```

## Dependency Injection in Services

### Automatic Injection in Controllers

```php
<?php

namespace App\Http\Controllers;

use App\Services\OrderService;
use App\Services\PaymentService;

class CheckoutController extends Controller
{
    // Laravel automatically injects services through the constructor
    public function __construct(
        protected OrderService $orderService,
        protected PaymentService $paymentService
    ) {}
    
    public function process(Request $request)
    {
        $order = $this->orderService->createOrder(
            $request->user()->id,
            $request->products
        );
        
        $payment = $this->paymentService->processPayment(
            $order,
            $request->payment_method
        );
        
        return response()->json([
            'order' => $order,
            'payment' => $payment,
        ]);
    }
}
```

### Using Services in Other Places

**In Artisan Commands:**

```php
<?php

namespace App\Console\Commands;

use App\Services\ReportService;
use Illuminate\Console\Command;

class GenerateDailyReport extends Command
{
    protected $signature = 'report:daily';
    
    public function handle(ReportService $reportService)
    {
        $report = $reportService->getSalesReport(
            now()->subDay(),
            now()
        );
        
        $this->info("Daily report generated!");
        $this->table(
            ['Metric', 'Value'],
            [
                ['Total Orders', $report['total_orders']],
                ['Total Revenue', $report['total_revenue']],
            ]
        );
    }
}
```

**In Jobs:**

```php
<?php

namespace App\Jobs;

use App\Services\NotificationService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, Queueable;
    
    public function __construct(
        protected int $userId
    ) {}
    
    public function handle(NotificationService $notificationService)
    {
        $notificationService->sendWelcomeEmail($this->userId);
    }
}
```

## Testing Services

```php
<?php

namespace Tests\Unit\Services;

use App\Services\OrderService;
use App\Models\User;
use App\Models\Product;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class OrderServiceTest extends TestCase
{
    use RefreshDatabase;
    
    protected OrderService $orderService;
    
    protected function setUp(): void
    {
        parent::setUp();
        $this->orderService = new OrderService();
    }
    
    public function test_can_create_order()
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['price' => 100, 'stock' => 10]);
        
        $order = $this->orderService->createOrder($user->id, [
            ['id' => $product->id, 'quantity' => 2]
        ]);
        
        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'total' => 200,
        ]);
    }
    
    public function test_throws_exception_when_out_of_stock()
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['stock' => 1]);
        
        $this->expectException(\Exception::class);
        
        $this->orderService->createOrder($user->id, [
            ['id' => $product->id, 'quantity' => 5]
        ]);
    }
}
```
