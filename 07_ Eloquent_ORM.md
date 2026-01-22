# Laravel Eloquent ORM

Eloquent is Laravel's elegant object-relational mapper (ORM) that makes database interaction enjoyable and productive. Each database table has a corresponding "Model" that is used to interact with that table.

## Getting Started with Eloquent

### Generating Models

Create a model using the Artisan command:

```bash
php artisan make:model Flight
```

Generate a model with additional resources:

```bash
# With migration
php artisan make:model Flight -m

# With factory
php artisan make:model Flight -f

# With seeder
php artisan make:model Flight -s

# With controller
php artisan make:model Flight -c

# With migration, factory, seeder, and controller
php artisan make:model Flight -mfsc

# All resources (migration, factory, seeder, policy, controller, and form requests)
php artisan make:model Flight -a

# Pivot model
php artisan make:model Member --pivot
```

### Model Conventions

Models are placed in `app/Models` and extend `Illuminate\Database\Eloquent\Model`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    // ...
}
```

**Table Names**: By convention, the "snake case", plural name of the class is used as the table name. `Flight` → `flights`, `AirTrafficController` → `air_traffic_controllers`.

Override the table name:

```php
protected $table = 'my_flights';
```

**Primary Keys**: Eloquent assumes each table has a primary key named `id`.

Override the primary key:

```php
protected $primaryKey = 'flight_id';
```

For non-incrementing or non-numeric keys:

```php
public $incrementing = false;
protected $keyType = 'string';
```

**UUIDs and ULIDs**:

Use UUIDs:

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;

class Article extends Model
{
    use HasUuids;
}

$article = Article::create(['title' => 'Traveling to Europe']);
$article->id; // "8f8e8478-9035-4d23-b9a7-62f4d2612ce5"
```

Use ULIDs (26 characters):

```php
use Illuminate\Database\Eloquent\Concerns\HasUlids;

class Article extends Model
{
    use HasUlids;
}

$article = Article::create(['title' => 'Traveling to Asia']);
$article->id; // "01gd4d3tgrrfqeda94gdbtdk5c"
```

**Timestamps**: By default, Eloquent expects `created_at` and `updated_at` columns.

Disable timestamps:

```php
public $timestamps = false;
```

Customize timestamp format:

```php
protected $dateFormat = 'U';
```

Customize column names:

```php
public const CREATED_AT = 'creation_date';
public const UPDATED_AT = 'updated_date';
```

**Database Connections**: Specify a different connection:

```php
protected $connection = 'mysql';
```

**Default Attribute Values**:

```php
protected $attributes = [
    'options' => '[]',
    'delayed' => false,
];
```

## Retrieving Models

### Basic Queries

```php
use App\Models\Flight;

// Get all records
foreach (Flight::all() as $flight) {
    echo $flight->name;
}

// With constraints
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

### Refreshing Models

```php
$flight = Flight::where('number', 'FR 900')->first();

// Get fresh instance (doesn't affect original)
$freshFlight = $flight->fresh();

// Refresh existing instance
$flight->refresh();
```

### Collections

Eloquent returns `Illuminate\Database\Eloquent\Collection` instances:

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});

foreach ($flights as $flight) {
    echo $flight->name;
}
```

### Chunking Results

Process large datasets efficiently:

```php
Flight::chunk(200, function ($flights) {
    foreach ($flights as $flight) {
        // Process flight...
    }
});

// When updating records while chunking
Flight::where('departed', true)
    ->chunkById(200, function ($flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

### Lazy Collections

```php
foreach (Flight::lazy() as $flight) {
    // ...
}

// When updating records
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

### Cursors

Minimize memory usage:

```php
foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}

// With collection methods
$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

### Advanced Subqueries

Select with subqueries:

```php
use App\Models\Destination;
use App\Models\Flight;

return Destination::addSelect(['last_flight' => Flight::select('name')
    ->whereColumn('destination_id', 'destinations.id')
    ->orderByDesc('arrived_at')
    ->limit(1)
])->get();
```

Order by subqueries:

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

## Retrieving Single Models

```php
// By primary key
$flight = Flight::find(1);

// First matching record
$flight = Flight::where('active', 1)->first();

// Alternative syntax
$flight = Flight::firstWhere('active', 1);

// Execute closure if not found
$flight = Flight::findOr(1, function () {
    // ...
});

// Throw exception if not found (returns 404)
$flight = Flight::findOrFail(1);
$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

### Retrieving or Creating Models

```php
// Retrieve or create
$flight = Flight::firstOrCreate([
    'name' => 'London to Paris'
]);

// With additional attributes
$flight = Flight::firstOrCreate(
    ['name' => 'London to Paris'],
    ['delayed' => 1, 'arrival_time' => '11:30']
);

// Retrieve or instantiate (not saved)
$flight = Flight::firstOrNew([
    'name' => 'London to Paris'
]);

// Save manually
$flight->save();
```

### Aggregates

```php
$count = Flight::where('active', 1)->count();
$max = Flight::where('active', 1)->max('price');
$min = Flight::where('active', 1)->min('price');
$avg = Flight::where('active', 1)->avg('price');
$sum = Flight::where('active', 1)->sum('price');
```

## Inserting and Updating Models

### Inserts

```php
$flight = new Flight;
$flight->name = $request->name;
$flight->save();

// Or use create (requires fillable/guarded)
$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

### Updates

```php
$flight = Flight::find(1);
$flight->name = 'Paris to London';
$flight->save();

// Update or create
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);

// Check if model was just created
if ($flight->wasRecentlyCreated) {
    // New record was inserted
}
```

### Mass Updates

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

### Examining Attribute Changes

```php
$user = User::create([
    'first_name' => 'Taylor',
    'last_name' => 'Otwell',
    'title' => 'Developer',
]);

$user->title = 'Painter';

$user->isDirty(); // true
$user->isDirty('title'); // true
$user->isDirty('first_name'); // false

$user->isClean(); // false
$user->isClean('first_name'); // true

$user->save();

$user->isDirty(); // false
$user->isClean(); // true

// Check what changed after save
$user->wasChanged(); // true
$user->wasChanged('title'); // true

// Get original values
$user->getOriginal('name'); // Returns original value
$user->getOriginal(); // Returns all original values

// Get changes made during last save
$user->getChanges(); // ['name' => 'Jack', 'email' => '[email protected]']

// Get previous values before last save
$user->getPrevious(); // ['name' => 'John', 'email' => '[email protected]']
```

### Mass Assignment

Define fillable attributes:

```php
class Flight extends Model
{
    protected $fillable = ['name'];
}
```

Or guard specific attributes:

```php
protected $guarded = [];
```

For JSON columns:

```php
protected $fillable = [
    'options->enabled',
];
```

### Upserts

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

## Deleting Models

```php
$flight = Flight::find(1);
$flight->delete();

// Delete by primary key(s)
Flight::destroy(1);
Flight::destroy(1, 2, 3);
Flight::destroy([1, 2, 3]);

// Delete all matching records
$deleted = Flight::where('active', 0)->delete();
```

### Soft Deleting

Enable soft deletes:

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Flight extends Model
{
    use SoftDeletes;
}
```

Add `deleted_at` column to table:

```php
Schema::table('flights', function (Blueprint $table) {
    $table->softDeletes();
});
```

Using soft deletes:

```php
$flight->delete(); // Sets deleted_at

// Check if soft deleted
if ($flight->trashed()) {
    // ...
}

// Restore soft deleted model
$flight->restore();

// Restore multiple models
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();

// Permanently delete
$flight->forceDelete();
```

### Querying Soft Deleted Models

```php
// Include soft deleted
$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();

// Only soft deleted
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

## Pruning Models

Automatically delete old models:

```php
use Illuminate\Database\Eloquent\Prunable;

class Flight extends Model
{
    use Prunable;

    public function prunable(): Builder
    {
        return static::where('created_at', '<=', now()->minus(months: 1));
    }

    // Optional: run before deleting
    protected function pruning(): void
    {
        // Delete related resources
    }
}
```

Schedule the pruning command:

```php
Schedule::command('model:prune')->daily();
```

For mass deletion (more efficient, no events):

```php
use Illuminate\Database\Eloquent\MassPrunable;

class Flight extends Model
{
    use MassPrunable;
}
```

## Replicating Models

```php
$shipping = Address::create([
    'type' => 'shipping',
    'line_1' => '123 Example Street',
    'city' => 'Victorville',
    'state' => 'CA',
    'postcode' => '90001',
]);

$billing = $shipping->replicate()->fill([
    'type' => 'billing'
]);

$billing->save();

// Exclude attributes from replication
$flight = $flight->replicate([
    'last_flown',
    'last_pilot_id'
]);
```

## Query Scopes

### Global Scopes

Create a global scope:

```bash
php artisan make:scope AncientScope
```

Define the scope:

```php
namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class AncientScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('created_at', '<', now()->minus(years: 2000));
    }
}
```

Apply to model:

```php
use App\Models\Scopes\AncientScope;
use Illuminate\Database\Eloquent\Attributes\ScopedBy;

#[ScopedBy([AncientScope::class])]
class User extends Model
{
    //
}
```

Or manually:

```php
protected static function booted(): void
{
    static::addGlobalScope(new AncientScope);
}
```

Anonymous global scopes:

```php
protected static function booted(): void
{
    static::addGlobalScope('ancient', function (Builder $builder) {
        $builder->where('created_at', '<', now()->minus(years: 2000));
    });
}
```

Remove global scopes:

```php
User::withoutGlobalScope(AncientScope::class)->get();
User::withoutGlobalScope('ancient')->get();
User::withoutGlobalScopes()->get();
```

### Local Scopes

Define scopes with the `Scope` attribute:

```php
use Illuminate\Database\Eloquent\Attributes\Scope;

class User extends Model
{
    #[Scope]
    protected function popular(Builder $query): void
    {
        $query->where('votes', '>', 100);
    }

    #[Scope]
    protected function active(Builder $query): void
    {
        $query->where('active', 1);
    }
}
```

Use scopes:

```php
$users = User::popular()->active()->orderBy('created_at')->get();
```

Dynamic scopes (with parameters):

```php
#[Scope]
protected function ofType(Builder $query, string $type): void
{
    $query->where('type', $type);
}

// Usage
$users = User::ofType('admin')->get();
```

### Pending Attributes

Create models with scope constraints:

```php
#[Scope]
protected function draft(Builder $query): void
{
    $query->withAttributes([
        'hidden' => true,
    ]);
}

$draft = Post::draft()->create(['title' => 'In Progress']);
$draft->hidden; // true
```

## Comparing Models

```php
if ($post->is($anotherPost)) {
    // Same primary key, table, and connection
}

if ($post->isNot($anotherPost)) {
    // Different models
}

// Compare related models without loading
if ($post->author()->is($user)) {
    // ...
}
```

## Events

Eloquent models dispatch events: `retrieved`, `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `trashed`, `forceDeleting`, `forceDeleted`, `restoring`, `restored`, `replicating`.

### Using Event Classes

```php
class User extends Authenticatable
{
    protected $dispatchesEvents = [
        'saved' => UserSaved::class,
        'deleted' => UserDeleted::class,
    ];
}
```

### Using Closures

```php
protected static function booted(): void
{
    static::created(function (User $user) {
        // ...
    });
}

// Queue event listeners
use function Illuminate\Events\queueable;

static::created(queueable(function (User $user) {
    // ...
}));
```

### Observers

Create an observer:

```bash
php artisan make:observer UserObserver --model=User
```

Define observer methods:

```php
class UserObserver
{
    public function created(User $user): void
    {
        // ...
    }

    public function updated(User $user): void
    {
        // ...
    }

    public function deleted(User $user): void
    {
        // ...
    }
}
```

Register the observer:

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

Or manually:

```php
User::observe(UserObserver::class);
```

### Muting Events

```php
$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();
    return User::find(2);
});

// Save without events
$user->saveQuietly();
$user->deleteQuietly();
$user->restoreQuietly();
```

## Eloquent Relationships

### One to One

Define the relationship:

```php
class User extends Model
{
    public function phone(): HasOne
    {
        return $this->hasOne(Phone::class);
    }
}

// Access
$phone = User::find(1)->phone;
```

Define the inverse:

```php
class Phone extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Custom keys:

```php
return $this->hasOne(Phone::class, 'foreign_key');
return $this->hasOne(Phone::class, 'foreign_key', 'local_key');

return $this->belongsTo(User::class, 'foreign_key');
return $this->belongsTo(User::class, 'foreign_key', 'owner_key');
```

Default models:

```php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault([
        'name' => 'Guest Author',
    ]);
}
```

### One to Many

```php
class Post extends Model
{
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }
}

// Access
$comments = Post::find(1)->comments;

// With constraints
$comment = Post::find(1)->comments()
    ->where('title', 'foo')
    ->first();
```

Inverse:

```php
class Comment extends Model
{
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}

$comment = Comment::find(1);
return $comment->post->title;
```

Query belongs to relationships:

```php
$posts = Post::where('user_id', $user->id)->get();

// More convenient
$posts = Post::whereBelongsTo($user)->get();

// With collection
$users = User::where('vip', true)->get();
$posts = Post::whereBelongsTo($users)->get();
```

### Has One of Many

```php
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}

public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}

public function largestOrder(): HasOne
{
    return $this->hasOne(Order::class)->ofMany('price', 'max');
}
```

Convert has many to has one:

```php
public function orders(): HasMany
{
    return $this->hasMany(Order::class);
}

public function largestOrder(): HasOne
{
    return $this->orders()->one()->ofMany('price', 'max');
}
```

### Has One Through

```php
class Mechanic extends Model
{
    public function carOwner(): HasOneThrough
    {
        return $this->hasOneThrough(Owner::class, Car::class);
    }
}

// Or using existing relationships
return $this->through('cars')->has('owner');
return $this->throughCars()->hasOwner();
```

### Has Many Through

```php
class Application extends Model
{
    public function deployments(): HasManyThrough
    {
        return $this->hasManyThrough(Deployment::class, Environment::class);
    }
}

// Or using existing relationships
return $this->through('environments')->has('deployments');
return $this->throughEnvironments()->hasDeployments();
```

### Many to Many

Define the relationship:

```php
class User extends Model
{
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}

// Access
$roles = User::find(1)->roles;

// Custom table name and keys
return $this->belongsToMany(Role::class, 'role_user');
return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```

Inverse:

```php
class Role extends Model
{
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class);
    }
}
```

### Pivot Table Columns

```php
foreach ($user->roles as $role) {
    echo $role->pivot->created_at;
}

// Specify extra pivot columns
return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');

// Auto-maintain timestamps
return $this->belongsToMany(Role::class)->withTimestamps();

// Custom pivot name
return $this->belongsToMany(Podcast::class)
    ->as('subscription')
    ->withTimestamps();

// Access
echo $podcast->subscription->created_at;
```

Filter by pivot:

```php
return $this->belongsToMany(Role::class)
    ->wherePivot('approved', 1);

return $this->belongsToMany(Role::class)
    ->wherePivotIn('priority', [1, 2]);

return $this->belongsToMany(Podcast::class)
    ->as('subscriptions')
    ->wherePivotBetween('created_at', ['2020-01-01', '2020-12-31']);
```

Order by pivot:

```php
return $this->belongsToMany(Badge::class)
    ->where('rank', 'gold')
    ->orderByPivot('created_at', 'desc');
```

### Custom Pivot Models

```php
class Role extends Model
{
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class)->using(RoleUser::class);
    }
}

use Illuminate\Database\Eloquent\Relations\Pivot;

class RoleUser extends Pivot
{
    // Custom pivot logic
}
```

## Polymorphic Relationships

### One to One (Polymorphic)

```php
class Image extends Model
{
    public function imageable(): MorphTo
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

class User extends Model
{
    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

// Usage
$post = Post::find(1);
$image = $post->image;

$image = Image::find(1);
$imageable = $image->imageable; // Returns Post or User
```

### One to Many (Polymorphic)

```php
class Comment extends Model
{
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model
{
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Usage
$post = Post::find(1);
foreach ($post->comments as $comment) {
    // ...
}

$comment = Comment::find(1);
$commentable = $comment->commentable; // Returns Post or Video
```

### Many to Many (Polymorphic)

```php
class Post extends Model
{
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

class Tag extends Model
{
    public function posts(): MorphToMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    public function videos(): MorphToMany
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}

// Usage
$post = Post::find(1);
foreach ($post->tags as $tag) {
    // ...
}

$tag = Tag::find(1);
foreach ($tag->posts as $post) {
    // ...
}
```

### Custom Polymorphic Types

Map polymorphic types to simple strings:

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

## Querying Relationships

```php
$user = User::find(1);

// Query the relationship
$user->posts()->where('active', 1)->get();

// Relationship existence
$posts = Post::has('comments')->get();
$posts = Post::has('comments', '>=', 3)->get();

// Nested relationships
$posts = Post::has('comments.images')->get();

// With constraints
$posts = Post::whereHas('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();

// Relationship absence
$posts = Post::doesntHave('comments')->get();

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

### Counting Related Models

```php
$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}

// With constraints
$posts = Post::withCount([
    'votes',
    'comments' => function (Builder $query) {
        $query->where('content', 'like', 'code%');
    }
])->get();

// With aliases
$posts = Post::withCount([
    'comments',
    'comments as pending_comments_count' => function (Builder $query) {
        $query->where('approved', false);
    },
])->get();

// Deferred counting
$book = Book::first();
$book->loadCount('genres');
```

### Other Aggregates

```php
$posts = Post::withSum('comments', 'votes')->get();
echo $posts[0]->comments_sum_votes;

$posts = Post::withAvg('comments', 'votes')->get();
$posts = Post::withMin('comments', 'votes')->get();
$posts = Post::withMax('comments', 'votes')->get();
$posts = Post::withExists('comments')->get();

// Deferred
$post = Post::first();
$post->loadSum('comments', 'votes');
```

## Eager Loading

Solve N+1 query problems:

```php
// N+1 problem
$books = Book::all();
foreach ($books as $book) {
    echo $book->author->name; // Query for each book
}

// Solution: eager loading
$books = Book::with('author')->get();
foreach ($books as $book) {
    echo $book->author->name; // No additional queries
}

// Multiple relationships
$books = Book::with(['author', 'publisher'])->get();

// Nested relationships
$books = Book::with('author.contacts')->get();
```

### Constraining Eager Loads

```php
$users = User::with(['posts' => function (Builder $query) {
    $query->where('title', 'like', '%code%');
}])->get();

// With ordering
$users = User::with(['posts' => function (Builder $query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```

### Lazy Eager Loading

```php
$books = Book::all();

if ($someCondition) {
    $books->load('author', 'publisher');
}

// With constraints
$books->load(['author' => function (Builder $query) {
    $query->orderBy('published_date', 'asc');
}]);
```

### Preventing Lazy Loading

```php
// In AppServiceProvider
use Illuminate\Database\Eloquent\Model;

Model::preventLazyLoading(! $this->app->isProduction());
```

## Inserting and Updating Related Models

### The Save Method

```php
$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);
$post->comments()->save($comment);

// Save multiple
$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

### The Create Method

```php
$post = Post::find(1);

$comment = $post->comments()->create([
    'message' => 'A new comment.',
]);

// Create multiple
$post->comments()->createMany([
    ['message' => 'A new comment.'],
    ['message' => 'Another new comment.'],
]);
```

### Updating Belongs To Relationships

```php
$account = Account::find(10);

$user->account()->associate($account);
$user->save();

// Dissociate
$user->account()->dissociate();
$user->save();
```

### Many to Many Relationships

Attach/detach:

```php
$user = User::find(1);

// Attach
$user->roles()->attach($roleId);
$user->roles()->attach($roleId, ['expires' => $expires]);

// Detach
$user->roles()->detach($roleId);
$user->roles()->detach([1, 2, 3]);
$user->roles()->detach(); // Detach all

// Sync (detach all not provided, attach provided)
$user->roles()->sync([1, 2, 3]);

// Sync with pivot data
$user->roles()->sync([
    1 => ['expires' => true],
    2 => ['expires' => false],
]);

// Sync without detaching
$user->roles()->syncWithoutDetaching([1, 2, 3]);

// Toggle
$user->roles()->toggle([1, 2, 3]);

// Update existing pivot record
$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

### Touching Parent Timestamps

Update parent's `updated_at` when child changes:

```php
class Comment extends Model
{
    protected $touches = ['post'];

    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}
```

## Practical Examples

### Blog System

```php
// Models
class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }

    #[Scope]
    protected function published(Builder $query): void
    {
        $query->where('published', true);
    }
}

class Comment extends Model
{
    protected $touches = ['post'];

    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}

// Usage
$user = User::find(1);

// Get all published posts with comments and tags
$posts = $user->posts()
    ->published()
    ->with(['comments', 'tags'])
    ->withCount('comments')
    ->get();

// Create new post with tags
$post = $user->posts()->create([
    'title' => 'My New Post',
    'content' => 'Post content...',
    'published' => true,
]);

$post->tags()->attach([1, 2, 3]);

// Add comment to post
$post->comments()->create([
    'body' => 'Great post!',
    'user_id' => $commenterId,
]);
```
