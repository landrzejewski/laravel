## 1. SQLite Configuration

**config/database.php** - already configured by default

**.env**
```env
DB_CONNECTION=sqlite
# DB_DATABASE=/absolute/path/to/database.sqlite
```

**Creating the database:**
```bash
touch database/database.sqlite
```

## 2. Migration

```bash
php artisan make:migration create_products_table
```

**database/migrations/xxxx_create_products_table.php**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->text('description')->nullable();
            $table->decimal('price', 10, 2);
            $table->integer('stock')->default(0);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

```bash
php artisan migrate
```

---

## Version 1: Query Builder (Database)

### CREATE - Adding Products

```php
use Illuminate\Support\Facades\DB;

// Single record
DB::table('products')->insert([
    'name' => 'Dell Laptop',
    'description' => 'Business laptop 15.6"',
    'price' => 3499.99,
    'stock' => 10,
    'created_at' => now(),
    'updated_at' => now(),
]);

// Returns ID of new record
$id = DB::table('products')->insertGetId([
    'name' => 'Logitech Mouse',
    'description' => 'Wireless mouse',
    'price' => 89.99,
    'stock' => 50,
    'created_at' => now(),
    'updated_at' => now(),
]);

// Multiple records
DB::table('products')->insert([
    [
        'name' => 'Keyboard',
        'price' => 149.99,
        'stock' => 25,
        'created_at' => now(),
        'updated_at' => now(),
    ],
    [
        'name' => 'Monitor',
        'price' => 899.99,
        'stock' => 15,
        'created_at' => now(),
        'updated_at' => now(),
    ],
]);
```

### READ - Reading Products

```php
// All products
$products = DB::table('products')->get();

// Single product
$product = DB::table('products')->where('id', 1)->first();
$product = DB::table('products')->find(1); // Shortcut

// With conditions
$expensiveProducts = DB::table('products')
    ->where('price', '>', 1000)
    ->get();

// Specific columns
$names = DB::table('products')
    ->select('name', 'price')
    ->get();

// Sorting and limit
$topProducts = DB::table('products')
    ->orderBy('price', 'desc')
    ->limit(5)
    ->get();

// Pagination
$products = DB::table('products')
    ->paginate(15);
```

### UPDATE - Updating Products

```php
// Update single record
DB::table('products')
    ->where('id', 1)
    ->update([
        'price' => 3299.99,
        'stock' => 8,
        'updated_at' => now(),
    ]);

// Update multiple records
DB::table('products')
    ->where('stock', '<', 5)
    ->update([
        'stock' => 10,
        'updated_at' => now(),
    ]);

// Increment/Decrement
DB::table('products')
    ->where('id', 1)
    ->increment('stock', 5); // Increase by 5

DB::table('products')
    ->where('id', 1)
    ->decrement('stock', 2); // Decrease by 2
```

### DELETE - Deleting Products

```php
// Delete specific product
DB::table('products')->where('id', 1)->delete();

// Delete multiple products
DB::table('products')
    ->where('stock', 0)
    ->delete();

// Delete everything (WARNING!)
DB::table('products')->truncate();
```

---

## Version 2: Eloquent ORM

### Model

```bash
php artisan make:model Product
```

**app/Models/Product.php**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    protected $fillable = [
        'name',
        'description',
        'price',
        'stock',
    ];

    protected $casts = [
        'price' => 'decimal:2',
        'stock' => 'integer',
    ];
}
```

### CREATE - Adding Products

```php
use App\Models\Product;

// Method 1: create()
$product = Product::create([
    'name' => 'Dell Laptop',
    'description' => 'Business laptop 15.6"',
    'price' => 3499.99,
    'stock' => 10,
]);

// Method 2: new + save()
$product = new Product();
$product->name = 'Logitech Mouse';
$product->description = 'Wireless mouse';
$product->price = 89.99;
$product->stock = 50;
$product->save();

// Method 3: firstOrCreate() - find or create
$product = Product::firstOrCreate(
    ['name' => 'Keyboard'], // Search condition
    [
        'description' => 'Mechanical keyboard',
        'price' => 299.99,
        'stock' => 20,
    ]
);

// Method 4: updateOrCreate() - update or create
$product = Product::updateOrCreate(
    ['name' => 'Monitor'],
    [
        'description' => '27" 4K Monitor',
        'price' => 1499.99,
        'stock' => 5,
    ]
);
```

### READ - Reading Products

```php
// All products
$products = Product::all();

// Find by ID
$product = Product::find(1);
$product = Product::findOrFail(1); // Throws exception if not found

// First match
$product = Product::where('name', 'Dell Laptop')->first();

// With conditions
$expensiveProducts = Product::where('price', '>', 1000)->get();

// Chaining conditions
$products = Product::where('price', '>', 100)
    ->where('stock', '>', 0)
    ->get();

// Sorting
$products = Product::orderBy('price', 'desc')->get();

// Limit and offset
$products = Product::skip(10)->take(5)->get();

// Pagination
$products = Product::paginate(15);

// Only selected columns
$products = Product::select('name', 'price')->get();

// Counting
$count = Product::count();
$sum = Product::sum('stock');
$avg = Product::avg('price');
```

### UPDATE - Updating Products

```php
// Method 1: Find and update
$product = Product::find(1);
$product->price = 3299.99;
$product->stock = 8;
$product->save();

// Method 2: update() - mass update
$product = Product::find(1);
$product->update([
    'price' => 3299.99,
    'stock' => 8,
]);

// Method 3: Update via query
Product::where('stock', '<', 5)
    ->update(['stock' => 10]);

// Increment/Decrement
$product = Product::find(1);
$product->increment('stock', 5);
$product->decrement('stock', 2);

// With additional update
$product->increment('stock', 5, [
    'price' => 99.99
]);
```

### DELETE - Deleting Products

```php
// Method 1: Find and delete
$product = Product::find(1);
$product->delete();

// Method 2: Delete by ID
Product::destroy(1);
Product::destroy([1, 2, 3]); // Multiple IDs

// Method 3: Delete via query
Product::where('stock', 0)->delete();

// Soft Delete (optional - requires configuration)
// In model add: use SoftDeletes;
// In migration add: $table->softDeletes();
```

---

## Comparison in Practice

### Example Controller with Both Methods

```php
<?php

namespace App\Http\Controllers;

use App\Models\Product;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class ProductController extends Controller
{
    // QUERY BUILDER VERSION
    public function indexDB()
    {
        $products = DB::table('products')
            ->orderBy('created_at', 'desc')
            ->paginate(10);
        
        return view('products.index', compact('products'));
    }

    public function storeDB(Request $request)
    {
        $id = DB::table('products')->insertGetId([
            'name' => $request->name,
            'description' => $request->description,
            'price' => $request->price,
            'stock' => $request->stock,
            'created_at' => now(),
            'updated_at' => now(),
        ]);

        return redirect()->route('products.show', $id);
    }

    // ELOQUENT VERSION
    public function indexEloquent()
    {
        $products = Product::orderBy('created_at', 'desc')
            ->paginate(10);
        
        return view('products.index', compact('products'));
    }

    public function storeEloquent(Request $request)
    {
        $product = Product::create($request->validated());

        return redirect()->route('products.show', $product);
    }
}
```


