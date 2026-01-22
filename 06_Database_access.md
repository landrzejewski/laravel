## Database Configuration and Setup

Laravel provides support for five major database systems: MariaDB 10.3+, MySQL 5.7+, PostgreSQL 10.0+, SQLite 3.26.0+, and SQL Server 2017+. Additionally, MongoDB is supported through the official `mongodb/laravel-mongodb` package.

All database configuration lives in `config/database.php`. Most configuration options are driven by environment variables in your `.env` file. Laravel's default configuration works with Laravel Sail, which is a Docker configuration for local development.

### Working with SQLite

SQLite databases are contained in a single file on your filesystem. Create a new SQLite database using:

```bash
touch database/database.sqlite
```

Then configure your environment variables:

```
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

Foreign key constraints are enabled by default for SQLite connections. To disable them, set `DB_FOREIGN_KEYS=false`.

### Database URL Configuration

Instead of configuring database connections with multiple variables (host, database, username, password), you can use a single database URL:

```
DB_URL=mysql://root:[email protected]/forge?charset=UTF-8
```

The format follows: `driver://username:password@host:port/database?options`

### Read and Write Connections

You can configure separate database connections for SELECT statements versus INSERT, UPDATE, and DELETE statements:

```php
'mysql' => [
    'driver' => 'mysql',
    'read' => [
        'host' => [
            '192.168.1.1',
            '196.168.1.2',
        ],
    ],
    'write' => [
        'host' => [
            '192.168.1.3',
        ],
    ],
    'database' => env('DB_DATABASE', 'laravel'),
    'username' => env('DB_USERNAME', 'root'),
    'password' => env('DB_PASSWORD', ''),
    // ... other options
],
```

The `sticky` option is available to allow immediate reading of records written during the current request cycle. When enabled, if a write operation occurs during the request, any further read operations will use the write connection. This ensures data written during a request can be immediately read back.

## Running Raw SQL Queries

Once your database connection is configured, you can run queries using the `DB` facade. The facade provides methods for each query type: `select`, `update`, `insert`, `delete`, and `statement`.

### Running SELECT Queries

To run a basic SELECT query:

```php
use Illuminate\Support\Facades\DB;

$users = DB::select('select * from users where active = ?', [1]);
```

The first argument is the SQL query, and the second argument contains parameter bindings. Parameter binding protects against SQL injection attacks.

The `select` method returns an array of results. Each result is a PHP `stdClass` object:

```php
foreach ($users as $user) {
    echo $user->name;
}
```

### Getting Scalar Values

When your query returns a single scalar value, use the `scalar` method:

```php
$burgers = DB::scalar(
    "select count(case when food = 'burger' then 1 end) as burgers from menu"
);
```

### Named Bindings

Instead of using `?` placeholders, you can use named bindings:

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```

### INSERT, UPDATE, and DELETE Statements

```php
// Insert
DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);

// Update (returns number of affected rows)
$affected = DB::update(
    'update users set votes = 100 where name = ?',
    ['Anita']
);

// Delete (returns number of affected rows)
$deleted = DB::delete('delete from users');
```

### Running General Statements

For statements that don't return values (like DROP TABLE):

```php
DB::statement('drop table users');
```

### Using Multiple Database Connections

If you've defined multiple connections in `config/database.php`, access them using the `connection` method:

```php
$users = DB::connection('sqlite')->select(/* ... */);
```

Get the raw PDO instance:

```php
$pdo = DB::connection()->getPdo();
```

### Listening for Query Events

You can register a closure to be invoked for each SQL query executed by your application. This is useful for logging or debugging:

```php
use Illuminate\Database\Events\QueryExecuted;

DB::listen(function (QueryExecuted $query) {
    // $query->sql
    // $query->bindings
    // $query->time
    // $query->toRawSql()
});
```

Register this in a service provider's `boot` method.

### Monitoring Query Time

You can be notified when your application spends too much time querying the database:

```php
use Illuminate\Database\Connection;
use Illuminate\Database\Events\QueryExecuted;

DB::whenQueryingForLongerThan(500, function (Connection $connection, QueryExecuted $event) {
    // Notify development team...
});
```

This dispatches a `DatabaseBusy` event when the cumulative query time exceeds the threshold (in milliseconds).

## Database Transactions

Use the `transaction` method to run operations within a database transaction. If an exception is thrown, the transaction automatically rolls back. If the closure executes successfully, it automatically commits:

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');
    DB::delete('delete from posts');
});
```

### Handling Deadlocks

The transaction method accepts an optional second argument defining retry attempts when a deadlock occurs:

```php
DB::transaction(function () {
    DB::update('update users set votes = 1');
    DB::delete('delete from posts');
}, attempts: 5);
```

### Manual Transaction Control

For complete control over rollbacks and commits:

```php
DB::beginTransaction();

// ... perform operations

DB::rollBack(); // or DB::commit();
```

## Query Builder

Laravel's query builder provides a fluent interface for creating and running database queries. It works with all supported database systems and uses PDO parameter binding for SQL injection protection.

### Retrieving All Rows

Use the `table` method to begin a query, then chain the `get` method to retrieve results:

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();

foreach ($users as $user) {
    echo $user->name;
}
```

The `get` method returns an `Illuminate\Support\Collection` instance containing `stdClass` objects.

### Retrieving a Single Row

Use the `first` method to get a single row:

```php
$user = DB::table('users')->where('name', 'John')->first();
return $user->email;
```

Use `firstOrFail` to throw an exception if no row is found (automatically returns 404 HTTP response):

```php
$user = DB::table('users')->where('name', 'John')->firstOrFail();
```

To extract a single value from a record:

```php
$email = DB::table('users')->where('name', 'John')->value('email');
```

To retrieve a row by its ID:

```php
$user = DB::table('users')->find(3);
```

### Retrieving Column Values

Use `pluck` to get a collection of values from a single column:

```php
$titles = DB::table('users')->pluck('title');

foreach ($titles as $title) {
    echo $title;
}
```

Specify the key for the resulting collection:

```php
$titles = DB::table('users')->pluck('title', 'name');

foreach ($titles as $name => $title) {
    echo $title;
}
```

### Chunking Results

For working with thousands of records, use `chunk` to retrieve results in small chunks:

```php
use Illuminate\Support\Collection;

DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    foreach ($users as $user) {
        // Process the user...
    }
});
```

Stop further chunks by returning `false` from the closure.

If you're updating records while chunking, use `chunkById` instead. This method paginates based on the primary key:

```php
DB::table('users')->where('active', false)
    ->chunkById(100, function (Collection $users) {
        foreach ($users as $user) {
            DB::table('users')
                ->where('id', $user->id)
                ->update(['active' => true]);
        }
    });
```

### Streaming Results Lazily

The `lazy` method executes the query in chunks but returns a LazyCollection, allowing you to interact with results as a single stream:

```php
DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // Process user...
});
```

If updating records while iterating, use `lazyById` or `lazyByIdDesc`.

### Aggregates

The query builder provides aggregate methods:

```php
$users = DB::table('users')->count();
$price = DB::table('orders')->max('price');
$price = DB::table('orders')->min('price');
$price = DB::table('orders')->avg('price');
$total = DB::table('orders')->sum('price');
```

Combine with other clauses:

```php
$price = DB::table('orders')
    ->where('finalized', 1)
    ->avg('price');
```

Check if records exist:

```php
if (DB::table('orders')->where('finalized', 1)->exists()) {
    // Records exist...
}

if (DB::table('orders')->where('finalized', 1)->doesntExist()) {
    // No records...
}
```

## Select Statements

### Specifying Select Clause

Specify custom columns to select:

```php
$users = DB::table('users')
    ->select('name', 'email as user_email')
    ->get();
```

Force distinct results:

```php
$users = DB::table('users')->distinct()->get();
```

Add columns to an existing query:

```php
$query = DB::table('users')->select('name');
$users = $query->addSelect('age')->get();
```

## Raw Expressions

Sometimes you need to insert arbitrary strings into queries. Use `DB::raw()`:

```php
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();
```

**Warning**: Raw statements are injected as strings, so be extremely careful to avoid SQL injection vulnerabilities.

### Raw Methods

Instead of `DB::raw()`, you can use these methods:

**selectRaw**
```php
$orders = DB::table('orders')
    ->selectRaw('price * ? as price_with_tax', [1.0825])
    ->get();
```

**whereRaw / orWhereRaw**
```php
$orders = DB::table('orders')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();
```

**havingRaw / orHavingRaw**
```php
$orders = DB::table('orders')
    ->select('department', DB::raw('SUM(price) as total_sales'))
    ->groupBy('department')
    ->havingRaw('SUM(price) > ?', [2500])
    ->get();
```

**orderByRaw**
```php
$orders = DB::table('orders')
    ->orderByRaw('updated_at - created_at DESC')
    ->get();
```

**groupByRaw**
```php
$orders = DB::table('orders')
    ->select('city', 'state')
    ->groupByRaw('city, state')
    ->get();
```

## Joins

### Inner Join

```php
$users = DB::table('users')
    ->join('contacts', 'users.id', '=', 'contacts.user_id')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'contacts.phone', 'orders.price')
    ->get();
```

### Left Join / Right Join

```php
$users = DB::table('users')
    ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
    ->get();

$users = DB::table('users')
    ->rightJoin('posts', 'users.id', '=', 'posts.user_id')
    ->get();
```

### Cross Join

Cross joins generate a cartesian product:

```php
$sizes = DB::table('sizes')
    ->crossJoin('colors')
    ->get();
```

### Advanced Join Clauses

Pass a closure to specify more complex join conditions:

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')
             ->orOn(/* ... */);
    })
    ->get();
```

Use where clauses in joins:

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')
             ->where('contacts.user_id', '>', 5);
    })
    ->get();
```

### Subquery Joins

Join a query to a subquery:

```php
$latestPosts = DB::table('posts')
    ->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
    ->where('is_published', true)
    ->groupBy('user_id');

$users = DB::table('users')
    ->joinSub($latestPosts, 'latest_posts', function (JoinClause $join) {
        $join->on('users.id', '=', 'latest_posts.user_id');
    })->get();
```

### Lateral Joins

Lateral joins (PostgreSQL, MySQL 8.0.14+, SQL Server) evaluate for each row and can reference columns outside the subquery:

```php
$latestPosts = DB::table('posts')
    ->select('id as post_id', 'title as post_title', 'created_at as post_created_at')
    ->whereColumn('user_id', 'users.id')
    ->orderBy('created_at', 'desc')
    ->limit(3);

$users = DB::table('users')
    ->joinLateral($latestPosts, 'latest_posts')
    ->get();
```

## Unions

Combine two or more queries:

```php
$first = DB::table('users')->whereNull('first_name');

$users = DB::table('users')
    ->whereNull('last_name')
    ->union($first)
    ->get();
```

Use `unionAll` to keep duplicate results.

## Where Clauses

### Basic Where Clauses

The most basic `where` call requires three arguments: column name, operator, and value:

```php
$users = DB::table('users')
    ->where('votes', '=', 100)
    ->where('age', '>', 35)
    ->get();
```

For equality checks, you can omit the operator:

```php
$users = DB::table('users')->where('votes', 100)->get();
```

You can use any operator supported by your database:

```php
$users = DB::table('users')->where('votes', '>=', 100)->get();
$users = DB::table('users')->where('votes', '<>', 100)->get();
$users = DB::table('users')->where('name', 'like', 'T%')->get();
```

Pass an associative array for multiple conditions:

```php
$users = DB::table('users')->where([
    'first_name' => 'Jane',
    'last_name' => 'Doe',
])->get();
```

Pass an array of conditions:

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['subscribed', '<>', '1'],
])->get();
```

### Or Where Clauses

Chain where clauses with `orWhere`:

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'John')
    ->get();
```

Group "or" conditions in parentheses by passing a closure:

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere(function (Builder $query) {
        $query->where('name', 'Abigail')
              ->where('votes', '>', 50);
    })
    ->get();
```

This produces: `select * from users where votes > 100 or (name = 'Abigail' and votes > 50)`

### Where Not Clauses

Negate a group of constraints:

```php
$products = DB::table('products')
    ->whereNot(function (Builder $query) {
        $query->where('clearance', true)
              ->orWhere('price', '<', 10);
    })
    ->get();
```

### Where Any / All / None

Apply the same constraints to multiple columns:

**whereAny** - any column matches:
```php
$users = DB::table('users')
    ->where('active', true)
    ->whereAny([
        'name',
        'email',
        'phone',
    ], 'like', 'Example%')
    ->get();
```

**whereAll** - all columns match:
```php
$posts = DB::table('posts')
    ->where('published', true)
    ->whereAll([
        'title',
        'content',
    ], 'like', '%Laravel%')
    ->get();
```

**whereNone** - no columns match:
```php
$albums = DB::table('albums')
    ->where('published', true)
    ->whereNone([
        'title',
        'lyrics',
        'tags',
    ], 'like', '%explicit%')
    ->get();
```

### JSON Where Clauses

For databases supporting JSON column types (MariaDB 10.3+, MySQL 8.0+, PostgreSQL 12.0+, SQL Server 2017+, SQLite 3.39.0+):

```php
$users = DB::table('users')
    ->where('preferences->dining->meal', 'salad')
    ->get();

$users = DB::table('users')
    ->whereIn('preferences->dining->meal', ['pasta', 'salad', 'sandwiches'])
    ->get();
```

Query JSON arrays:

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', 'en')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', 'en')
    ->get();
```

Check for JSON keys:

```php
$users = DB::table('users')
    ->whereJsonContainsKey('preferences->dietary_requirements')
    ->get();
```

Query JSON array length:

```php
$users = DB::table('users')
    ->whereJsonLength('options->languages', 0)
    ->get();

$users = DB::table('users')
    ->whereJsonLength('options->languages', '>', 1)
    ->get();
```

### Additional Where Clauses

**whereLike / orWhereLike / whereNotLike / orWhereNotLike**

Database-agnostic pattern matching:

```php
$users = DB::table('users')
    ->whereLike('name', '%John%')
    ->get();

// Case-sensitive search
$users = DB::table('users')
    ->whereLike('name', '%John%', caseSensitive: true)
    ->get();
```

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

```php
$users = DB::table('users')
    ->whereIn('id', [1, 2, 3])
    ->get();

$users = DB::table('users')
    ->whereNotIn('id', [1, 2, 3])
    ->get();
```

You can pass a query as the second argument:

```php
$activeUsers = DB::table('users')->select('id')->where('is_active', 1);

$users = DB::table('comments')
    ->whereIn('user_id', $activeUsers)
    ->get();
```

**whereBetween / whereNotBetween**

```php
$users = DB::table('users')
    ->whereBetween('votes', [1, 100])
    ->get();

$users = DB::table('users')
    ->whereNotBetween('votes', [1, 100])
    ->get();
```

**whereBetweenColumns / whereNotBetweenColumns**

Check if a column's value is between two other columns in the same row:

```php
$patients = DB::table('patients')
    ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

**whereNull / whereNotNull**

```php
$users = DB::table('users')
    ->whereNull('updated_at')
    ->get();

$users = DB::table('users')
    ->whereNotNull('updated_at')
    ->get();
```

**whereDate / whereMonth / whereDay / whereYear / whereTime**

```php
$users = DB::table('users')
    ->whereDate('created_at', '2016-12-31')
    ->get();

$users = DB::table('users')
    ->whereMonth('created_at', '12')
    ->get();

$users = DB::table('users')
    ->whereDay('created_at', '31')
    ->get();

$users = DB::table('users')
    ->whereYear('created_at', '2016')
    ->get();

$users = DB::table('users')
    ->whereTime('created_at', '=', '11:20:45')
    ->get();
```

**wherePast / whereFuture / whereToday**

```php
$invoices = DB::table('invoices')
    ->wherePast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereFuture('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereToday('due_at')
    ->get();
```

**whereColumn / orWhereColumn**

Verify that two columns are equal:

```php
$users = DB::table('users')
    ->whereColumn('first_name', 'last_name')
    ->get();

$users = DB::table('users')
    ->whereColumn('updated_at', '>', 'created_at')
    ->get();
```

### Logical Grouping

Group several where clauses within parentheses for proper logical grouping:

```php
$users = DB::table('users')
    ->where('name', '=', 'John')
    ->where(function (Builder $query) {
        $query->where('votes', '>', 100)
              ->orWhere('title', '=', 'Admin');
    })
    ->get();
```

This produces: `select * from users where name = 'John' and (votes > 100 or title = 'Admin')`

Always group `orWhere` calls to avoid unexpected behavior with global scopes.

## Advanced Where Clauses

### Where Exists

```php
$users = DB::table('users')
    ->whereExists(function (Builder $query) {
        $query->select(DB::raw(1))
              ->from('orders')
              ->whereColumn('orders.user_id', 'users.id');
    })
    ->get();
```

Or pass a query object:

```php
$orders = DB::table('orders')
    ->select(DB::raw(1))
    ->whereColumn('orders.user_id', 'users.id');

$users = DB::table('users')
    ->whereExists($orders)
    ->get();
```

### Subquery Where Clauses

Compare a subquery result to a value:

```php
use App\Models\User;

$users = User::where(function (Builder $query) {
    $query->select('type')
        ->from('membership')
        ->whereColumn('membership.user_id', 'users.id')
        ->orderByDesc('membership.start_date')
        ->limit(1);
}, 'Pro')->get();
```

Compare a column to a subquery:

```php
use App\Models\Income;

$incomes = Income::where('amount', '<', function (Builder $query) {
    $query->selectRaw('avg(i.amount)')->from('incomes as i');
})->get();
```

### Full Text Where Clauses

For columns with full text indexes (MariaDB, MySQL, PostgreSQL):

```php
$users = DB::table('users')
    ->whereFullText('bio', 'web developer')
    ->get();
```

## Ordering, Grouping, Limit and Offset

### Ordering

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->get();

$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->orderBy('email', 'asc')
    ->get();

// Sort direction is ascending by default
$users = DB::table('users')
    ->orderByDesc('verified_at')
    ->get();

// Sort by JSON column value
$corporations = DB::table('corporations')
    ->where('country', 'US')
    ->orderBy('location->state')
    ->get();
```

### Latest and Oldest

```php
$user = DB::table('users')
    ->latest() // Orders by created_at DESC
    ->first();

$user = DB::table('users')
    ->latest('updated_at') // Specify column
    ->first();

$user = DB::table('users')
    ->oldest()
    ->first();
```

### Random Ordering

```php
$randomUser = DB::table('users')
    ->inRandomOrder()
    ->first();
```

### Removing Orderings

```php
$query = DB::table('users')->orderBy('name');
$unorderedUsers = $query->reorder()->get();

// Or apply new ordering
$usersOrderedByEmail = $query->reorder('email', 'desc')->get();
```

### Grouping

```php
$users = DB::table('users')
    ->groupBy('account_id')
    ->having('account_id', '>', 100)
    ->get();

$report = DB::table('orders')
    ->selectRaw('count(id) as number_of_orders, customer_id')
    ->groupBy('customer_id')
    ->havingBetween('number_of_orders', [5, 15])
    ->get();
```

### Limit and Offset

```php
$users = DB::table('users')
    ->offset(10)
    ->limit(5)
    ->get();
```

## Conditional Clauses

Apply query clauses based on conditions using the `when` method:

```php
$role = $request->input('role');

$users = DB::table('users')
    ->when($role, function (Builder $query, string $role) {
        $query->where('role_id', $role);
    })
    ->get();
```

The closure only executes when the first argument is truthy.

Provide a third argument for when the condition is false:

```php
$sortByVotes = $request->boolean('sort_by_votes');

$users = DB::table('users')
    ->when($sortByVotes, function (Builder $query, bool $sortByVotes) {
        $query->orderBy('votes');
    }, function (Builder $query) {
        $query->orderBy('name');
    })
    ->get();
```

## Insert Statements

Insert a single record:

```php
DB::table('users')->insert([
    'email' => '[email protected]',
    'votes' => 0
]);
```

Insert multiple records:

```php
DB::table('users')->insert([
    ['email' => '[email protected]', 'votes' => 0],
    ['email' => '[email protected]', 'votes' => 0],
]);
```

### Auto-Incrementing IDs

Get the auto-incremented ID after inserting:

```php
$id = DB::table('users')->insertGetId(
    ['email' => '[email protected]', 'votes' => 0]
);
```

### Upserts

Insert records that don't exist, update records that do:

```php
DB::table('flights')->upsert(
    [
        ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
        ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
    ],
    ['departure', 'destination'], // Unique columns
    ['price'] // Columns to update if record exists
);
```

## Update Statements

Update existing records:

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);
```

### Update or Insert

Update if exists, insert if doesn't:

```php
DB::table('users')
    ->updateOrInsert(
        ['email' => '[email protected]', 'name' => 'John'],
        ['votes' => '2']
    );
```

You can customize based on whether the record exists:

```php
DB::table('users')->updateOrInsert(
    ['user_id' => $user_id],
    fn ($exists) => $exists ? [
        'name' => $data['name'],
        'email' => $data['email'],
    ] : [
        'name' => $data['name'],
        'email' => $data['email'],
        'marketable' => true,
    ],
);
```

### Updating JSON Columns

Use `->` syntax (MariaDB 10.3+, MySQL 5.7+, PostgreSQL 9.5+):

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['options->enabled' => true]);
```

### Increment and Decrement

```php
DB::table('users')->increment('votes');
DB::table('users')->increment('votes', 5);
DB::table('users')->decrement('votes');
DB::table('users')->decrement('votes', 5);

// Update additional columns
DB::table('users')->increment('votes', 1, ['name' => 'John']);

// Increment multiple columns
DB::table('users')->incrementEach([
    'votes' => 5,
    'balance' => 100,
]);
```

## Delete Statements

```php
$deleted = DB::table('users')->delete();

$deleted = DB::table('users')->where('votes', '>', 100)->delete();
```

## Pessimistic Locking

Execute SELECT statements with locks to prevent concurrent modifications:

```php
// Shared lock
DB::table('users')
    ->where('votes', '>', 100)
    ->sharedLock()
    ->get();

// Lock for update
DB::table('users')
    ->where('votes', '>', 100)
    ->lockForUpdate()
    ->get();
```

It's recommended to wrap pessimistic locks in transactions:

```php
DB::transaction(function () {
    $sender = DB::table('users')
        ->lockForUpdate()
        ->find(1);

    $receiver = DB::table('users')
        ->lockForUpdate()
        ->find(2);

    if ($sender->balance < 100) {
        throw new RuntimeException('Balance too low.');
    }

    DB::table('users')
        ->where('id', $sender->id)
        ->update(['balance' => $sender->balance - 100]);

    DB::table('users')
        ->where('id', $receiver->id)
        ->update(['balance' => $receiver->balance + 100]);
});
```

## Debugging

Dump query information and stop execution:

```php
DB::table('users')->where('votes', '>', 100)->dd();
```

Dump without stopping:

```php
DB::table('users')->where('votes', '>', 100)->dump();
```

Dump with bindings substituted:

```php
DB::table('users')->where('votes', '>', 100)->dumpRawSql();
DB::table('users')->where('votes', '>', 100)->ddRawSql();
```

## Pagination

Laravel provides easy pagination integrated with the query builder and Eloquent ORM. By default, pagination HTML is compatible with Tailwind CSS, but Bootstrap support is also available.

### Paginating Query Builder Results

Use the `paginate` method, which automatically handles setting the limit and offset based on the current page:

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->paginate(15);
```

The current page is detected from the `page` query string parameter.

### Simple Pagination

If you only need "Next" and "Previous" links without showing the total number of pages:

```php
$users = DB::table('users')->simplePaginate(15);
```

This is more efficient as it doesn't count total records.

### Paginating Eloquent Results

```php
use App\Models\User;

$users = User::paginate(15);
```

With constraints:

```php
$users = User::where('votes', '>', 100)->paginate(15);
```

Or use simple pagination:

```php
$users = User::where('votes', '>', 100)->simplePaginate(15);
```

### Cursor Pagination

Cursor pagination provides the most efficient database performance for large datasets and infinite scrolling UIs. Instead of page numbers, it uses an encoded cursor string:

```php
$users = DB::table('users')->orderBy('id')->cursorPaginate(15);
```

Cursor pagination requires an "order by" clause and works best when:
- You have large datasets
- You need better performance than offset pagination
- You only need Next/Previous navigation

It has limitations:
- Can't generate page number links
- Requires ordering by unique column(s)
- Query expressions in "order by" must be aliased and in the select clause

### Multiple Paginators Per Page

If you need multiple paginators on one page, pass a custom page name:

```php
$users = User::where('votes', '>', 100)->paginate(
    $perPage = 15, $columns = ['*'], $pageName = 'users'
);
```

### Customizing Pagination URLs

Customize the pagination URL path:

```php
$users = User::paginate(15);
$users->withPath('/admin/users');
```

Append query string values:

```php
$users = User::paginate(15);
$users->appends(['sort' => 'votes']);
```

Append all current query strings:

```php
$users = User::paginate(15)->withQueryString();
```

Append hash fragments:

```php
$users = User::paginate(15)->fragment('users');
```

### Displaying Pagination Results

Paginator instances are iterators, so you can loop through results and display pagination links:

```blade
<div class="container">
    @foreach ($users as $user)
        {{ $user->name }}
    @endforeach
</div>

{{ $users->links() }}
```

### Adjusting Link Window

Control how many additional links are displayed on each side:

```blade
{{ $users->onEachSide(5)->links() }}
```

### Converting to JSON

When returned from a route, paginators automatically convert to JSON:

```php
Route::get('/users', function () {
    return User::paginate();
});
```

JSON includes meta information like `total`, `current_page`, `last_page`, and the data in a `data` array.

### Customizing Pagination Views

Publish the pagination views:

```bash
php artisan vendor:publish --tag=laravel-pagination
```

Views will be in `resources/views/vendor/pagination`. Edit `tailwind.blade.php` to customize.

Set default views in a service provider:

```php
use Illuminate\Pagination\Paginator;

Paginator::defaultView('view-name');
Paginator::defaultSimpleView('view-name');
```

### Using Bootstrap

In your `AppServiceProvider`:

```php
use Illuminate\Pagination\Paginator;

public function boot(): void
{
    Paginator::useBootstrapFive();
    // or Paginator::useBootstrapFour();
}
```

### Paginator Methods

Paginator and LengthAwarePaginator instances provide these methods:

```php
$paginator->count() // Items on current page
$paginator->currentPage() // Current page number
$paginator->firstItem() // First item number
$paginator->hasPages() // Enough items for multiple pages
$paginator->hasMorePages() // More items in data store
$paginator->items() // Items for current page
$paginator->lastItem() // Last item number
$paginator->lastPage() // Last page number
$paginator->nextPageUrl() // URL for next page
$paginator->onFirstPage() // On first page
$paginator->perPage() // Items per page
$paginator->previousPageUrl() // URL for previous page
$paginator->total() // Total matching items
$paginator->url($page) // URL for given page
```

## Migrations

Migrations are like version control for your database. They allow you to define and share your application's database schema definition.

### Generating Migrations

Use the `make:migration` Artisan command:

```bash
php artisan make:migration create_flights_table
```

The new migration will be in `database/migrations`. Each filename contains a timestamp that determines execution order.

Laravel will attempt to guess the table name from the migration name. If you specify:
- `create_flights_table` - Laravel pre-fills code to create the `flights` table
- `add_status_to_flights_table` - Laravel pre-fills code to modify the `flights` table

For custom paths:

```bash
php artisan make:migration create_flights_table --path=database/migrations/custom
```

### Migration Structure

A migration contains two methods: `up` and `down`. The `up` method adds tables, columns, or indexes. The `down` method reverses the operations:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('flights', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('airline');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::drop('flights');
    }
};
```

### Running Migrations

Execute all outstanding migrations:

```bash
php artisan migrate
```

Check migration status:

```bash
php artisan migrate:status
```

Preview SQL without executing:

```bash
php artisan migrate --pretend
```

### Isolated Migration Execution

When deploying across multiple servers, prevent simultaneous migrations:

```bash
php artisan migrate --isolated
```

This uses an atomic lock via your cache driver. Requires `memcached`, `redis`, `dynamodb`, `database`, `file`, or `array` cache driver.

### Forcing Production Migrations

Destructive operations require confirmation in production. Force without prompt:

```bash
php artisan migrate --force
```

### Rolling Back Migrations

Roll back the last batch:

```bash
php artisan migrate:rollback
```

Roll back a specific number of migrations:

```bash
php artisan migrate:rollback --step=5
```

Roll back a specific batch:

```bash
php artisan migrate:rollback --batch=3
```

Roll back all migrations:

```bash
php artisan migrate:reset
```

### Refresh and Fresh

Roll back all and re-migrate:

```bash
php artisan migrate:refresh
php artisan migrate:refresh --seed
```

Refresh the last 5 migrations:

```bash
php artisan migrate:refresh --step=5
```

Drop all tables and re-migrate:

```bash
php artisan migrate:fresh
php artisan migrate:fresh --seed
```

### Squashing Migrations

Consolidate many migrations into a single SQL file:

```bash
php artisan schema:dump

# Dump and delete existing migrations
php artisan schema:dump --prune
```

Schema files are stored in `database/schema`. Laravel executes the schema file first, then runs any remaining migrations.

## Creating Tables

Use `Schema::create`:

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email');
    $table->timestamps();
});
```

### Checking Table Existence

```php
if (Schema::hasTable('users')) {
    // Table exists
}

if (Schema::hasColumn('users', 'email')) {
    // Column exists
}

if (Schema::hasIndex('users', ['email'], 'unique')) {
    // Index exists
}
```

### Table Options

Specify storage engine (MariaDB/MySQL):

```php
Schema::create('users', function (Blueprint $table) {
    $table->engine('InnoDB');
    // ...
});
```

Character set and collation:

```php
Schema::create('users', function (Blueprint $table) {
    $table->charset('utf8mb4');
    $table->collation('utf8mb4_unicode_ci');
    // ...
});
```

Temporary tables:

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->temporary();
    // ...
});
```

Table comments (MariaDB/MySQL/PostgreSQL):

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');
    // ...
});
```

## Updating Tables

Use `Schema::table` to modify existing tables:

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

## Renaming and Dropping Tables

Rename a table:

```php
Schema::rename($from, $to);
```

Drop a table:

```php
Schema::drop('users');
Schema::dropIfExists('users');
```

## Working with Columns

### Creating Columns

Add columns to existing tables:

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

### Available Column Types

Laravel provides many column type methods. Here are key examples:

**Numeric Types:**
```php
$table->id(); // Auto-incrementing BIGINT primary key
$table->bigInteger('votes');
$table->integer('votes');
$table->smallInteger('votes');
$table->decimal('amount', total: 8, places: 2);
$table->double('amount');
$table->float('amount', precision: 53);
```

**String Types:**
```php
$table->string('name', length: 100); // VARCHAR
$table->char('name', length: 100); // CHAR
$table->text('description'); // TEXT
$table->mediumText('description'); // MEDIUMTEXT
$table->longText('description'); // LONGTEXT
```

**Date/Time Types:**
```php
$table->date('created_at');
$table->dateTime('created_at', precision: 0);
$table->time('sunrise', precision: 0);
$table->timestamp('added_at', precision: 0);
$table->timestamps(precision: 0); // created_at and updated_at
$table->softDeletes('deleted_at', precision: 0);
```

**Boolean:**
```php
$table->boolean('confirmed');
```

**JSON:**
```php
$table->json('options');
$table->jsonb('options'); // JSONB for PostgreSQL
```

**UUID/ULID:**
```php
$table->uuid('id');
$table->ulid('id');
```

**Foreign Keys:**
```php
$table->foreignId('user_id');
$table->foreignUuid('user_id');
$table->foreignUlid('user_id');
```

**Special Types:**
```php
$table->enum('difficulty', ['easy', 'hard']);
$table->ipAddress('visitor');
$table->macAddress('device');
$table->rememberToken(); // For authentication
$table->binary('photo');
```

### Column Modifiers

Modify column behavior by chaining methods:

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('email')->nullable();
    $table->integer('votes')->default(0);
    $table->timestamp('created_at')->useCurrent();
    $table->string('name')->after('email'); // MariaDB/MySQL only
    $table->integer('order')->unsigned();
    $table->text('bio')->comment('User biography');
});
```

Common modifiers:
- `->nullable()` - Allow NULL values
- `->default($value)` - Set default value
- `->unsigned()` - Make integer unsigned
- `->after('column')` - Place after another column (MySQL/MariaDB)
- `->first()` - Place first in table (MySQL/MariaDB)
- `->comment('text')` - Add column comment
- `->useCurrent()` - Use CURRENT_TIMESTAMP for timestamps
- `->autoIncrement()` - Set as auto-incrementing

### Default Expressions

Use `Expression` instances for database functions:

```php
use Illuminate\Database\Query\Expression;

Schema::create('flights', function (Blueprint $table) {
    $table->id();
    $table->json('movies')->default(new Expression('(JSON_ARRAY())'));
    $table->timestamps();
});
```

### Column Order

When using MySQL/MariaDB, add columns after existing ones:

```php
$table->after('password', function (Blueprint $table) {
    $table->string('address_line1');
    $table->string('address_line2');
    $table->string('city');
});
```

### Modifying Columns

Change column type or attributes using the `change` method:

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 50)->change();
});
```

**Important**: You must explicitly include all modifiers you want to keep:

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
});
```

### Renaming Columns

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('from', 'to');
});
```

### Dropping Columns

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn('votes');
});

// Drop multiple columns
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn(['votes', 'avatar', 'location']);
});
```

Convenient methods for common drops:

```php
$table->dropRememberToken();
$table->dropSoftDeletes();
$table->dropTimestamps();
$table->dropMorphs('morphable');
```

## Indexes

### Creating Indexes

Add a unique index to a column:

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('email')->unique();
});
```

Or create the index after defining the column:

```php
$table->unique('email');
```

Compound (composite) index:

```php
$table->index(['account_id', 'created_at']);
```

Specify custom index name:

```php
$table->unique('email', 'unique_email');
```

### Available Index Types

```php
$table->primary('id'); // Primary key
$table->primary(['id', 'parent_id']); // Composite primary key
$table->unique('email'); // Unique index
$table->index('state'); // Basic index
$table->fullText('body'); // Full text index (MariaDB/MySQL/PostgreSQL)
$table->spatialIndex('location'); // Spatial index (except SQLite)
```

### Renaming Indexes

```php
$table->renameIndex('from', 'to');
```

### Dropping Indexes

By default, Laravel generates index names based on table, columns, and type:

```php
$table->dropPrimary('users_id_primary');
$table->dropUnique('users_email_unique');
$table->dropIndex('geo_state_index');
$table->dropFullText('posts_body_fulltext');
$table->dropSpatialIndex('geo_location_spatialindex');
```

Or pass column array to drop by conventional name:

```php
Schema::table('geo', function (Blueprint $table) {
    $table->dropIndex(['state']); // Drops 'geo_state_index'
});
```

### Foreign Key Constraints

Create foreign key constraints to enforce referential integrity:

```php
Schema::table('posts', function (Blueprint $table) {
    $table->unsignedBigInteger('user_id');
    $table->foreign('user_id')->references('id')->on('users');
});
```

Or use the convenient `foreignId` method:

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained();
});
```

Specify table name if it doesn't match conventions:

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained(
        table: 'users', indexName: 'posts_user_id'
    );
});
```

Specify on delete and on update actions:

```php
$table->foreignId('user_id')
    ->constrained()
    ->onUpdate('cascade')
    ->onDelete('cascade');
```

Or use expressive methods:

```php
$table->foreignId('user_id')
    ->constrained()
    ->cascadeOnUpdate()
    ->cascadeOnDelete();
```

Available actions:
- `->cascadeOnUpdate()` / `->cascadeOnDelete()`
- `->restrictOnUpdate()` / `->restrictOnDelete()`
- `->nullOnUpdate()` / `->nullOnDelete()`
- `->noActionOnUpdate()` / `->noActionOnDelete()`

### Dropping Foreign Keys

```php
$table->dropForeign('posts_user_id_foreign');

// Or by column array
$table->dropForeign(['user_id']);
```

### Toggling Foreign Key Constraints

```php
Schema::enableForeignKeyConstraints();
Schema::disableForeignKeyConstraints();

Schema::withoutForeignKeyConstraints(function () {
    // Constraints disabled within this closure
});
```

**Note**: SQLite disables foreign key constraints by default. Enable them in your database configuration.

## Database Seeding

Laravel includes the ability to seed your database with data using seed classes. All seeders are stored in `database/seeders`.

### Writing Seeders

Generate a seeder:

```bash
php artisan make:seeder UserSeeder
```

A seeder contains one method: `run`. This is called when executing `db:seed`:

```php
namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        DB::table('users')->insert([
            'name' => Str::random(10),
            'email' => Str::random(10).'@example.com',
            'password' => Hash::make('password'),
        ]);
    }
}
```

You can type-hint dependencies in the `run` method signature - they'll be resolved via the service container.

### Using Model Factories

Instead of manually specifying attributes, use model factories to generate large amounts of data:

```php
use App\Models\User;

public function run(): void
{
    User::factory()
        ->count(50)
        ->hasPosts(1)
        ->create();
}
```

### Calling Additional Seeders

Use the `call` method to execute additional seed classes:

```php
public function run(): void
{
    $this->call([
        UserSeeder::class,
        PostSeeder::class,
        CommentSeeder::class,
    ]);
}
```

This allows you to break up seeding into multiple files.

### Muting Model Events

Prevent models from dispatching events during seeding:

```php
use Illuminate\Database\Console\Seeds\WithoutModelEvents;

class DatabaseSeeder extends Seeder
{
    use WithoutModelEvents;

    public function run(): void
    {
        $this->call([
            UserSeeder::class,
        ]);
    }
}
```

### Running Seeders

Seed your database:

```bash
php artisan db:seed
```

Run a specific seeder:

```bash
php artisan db:seed --class=UserSeeder
```

Combine with migrations:

```bash
php artisan migrate:fresh --seed
```

Specify a seeder with migrations:

```bash
php artisan migrate:fresh --seed --seeder=UserSeeder
```

Force seeding in production:

```bash
php artisan db:seed --force
```

## Practical Workflow Example

Here's how these features work together in a typical Laravel application:

**1. Create a migration for a new table:**

```bash
php artisan make:migration create_products_table
```

**2. Define the table structure:**

```php
public function up(): void
{
    Schema::create('products', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->text('description')->nullable();
        $table->decimal('price', total: 8, places: 2);
        $table->integer('stock')->default(0);
        $table->foreignId('category_id')->constrained()->cascadeOnDelete();
        $table->timestamps();
    });
}
```

**3. Run the migration:**

```bash
php artisan migrate
```

**4. Create a seeder:**

```bash
php artisan make:seeder ProductSeeder
```

**5. Use the query builder in the seeder:**

```php
public function run(): void
{
    DB::table('products')->insert([
        'name' => 'Sample Product',
        'description' => 'A great product',
        'price' => 99.99,
        'stock' => 10,
        'category_id' => 1,
        'created_at' => now(),
        'updated_at' => now(),
    ]);
}
```

**6. Query products with pagination:**

```php
$products = DB::table('products')
    ->where('stock', '>', 0)
    ->orderBy('name')
    ->paginate(15);
```

**7. Update product stock:**

```php
DB::table('products')
    ->where('id', $id)
    ->decrement('stock', $quantity);
```
