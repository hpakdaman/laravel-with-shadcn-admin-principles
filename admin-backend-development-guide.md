# Admin Backend Development Guide

This comprehensive guide covers all aspects of building admin modules in this Laravel CMS, including models, controllers, data validation, and best practices.

## Architecture Principles

### 1. Thin Controllers, Fat Models

**Controllers should only handle HTTP concerns** - request validation, response formatting, and redirects. All business logic belongs in models through scopes, static methods, and model methods.

```php
// ✅ Good - Thin controller
public function store(PostData $postData)
{
    $post = Post::createWithCategories($postData);
    return redirect()
        ->route('admin.posts.index')
        ->with('success', 'Post created successfully');
}

// ❌ Bad - Too much logic in controller
public function store(Request $request)
{
    $validated = $request->validate([...]);
    $post = new Post();
    $post->title = $validated['title'];
    $post->slug = Str::slug($validated['title']);
    // ... more business logic that should be in model
}
```

### 2. Namespace Usage

Always define namespaces at the top using `use` statements - never write full namespaces in method bodies.

### 3. Authentication Helpers

Always use the provided helper functions instead of direct `auth()` calls:

```php
// ✅ Use helpers
if (authCheck()) {
    $userId = authId();
    $user = authUser();
}

if (isAdmin()) {
    // Admin-only logic
}

// ❌ Avoid direct auth() calls
if (auth()->check()) { // Don't do this
    $userId = auth()->id();
}
```

## Model Structure

### Basic Model Setup

Always use the project's helper traits for consistent functionality:

```php
<?php

namespace App\Models;

use App\Helpers\Traits\DateTimeScopes\DateTimeScopes;
use App\Helpers\Traits\Ownership\HasOwnership;
use App\Helpers\Traits\ProtectedFillable\ProtectedFillable;
use App\Helpers\Traits\Query\FilteredQueryBuilder;
use App\Helpers\Traits\Query\QueryScopes;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasOwnership, FilteredQueryBuilder, QueryScopes, DateTimeScopes;
    use ProtectedFillable;

    // Base fillable - always available
    protected $fillable = [
        'title',
        'slug',
        'content',
        'status',
        'user_id',
    ];

    // Admin-only fields
    protected $adminFillable = [
        'featured',
        'order',
        'meta_description',
    ];

    // Create-only fields
    protected $createFillable = [
        'title',
        'content',
        'slug',
    ];

    // Update-only fields
    protected $updateFillable = [
        'title',
        'content',
        'status',
        'meta_description',
    ];

    ########################################################
    #################       MUTATORS      #################

    /**
     * Set the post title and automatically generate slug.
     *
     * @param string $value The title value
     * @return void
     */
    public function setTitleAttribute(string $value): void
    {
        $this->attributes['title'] = $value;
        $this->attributes['slug'] = Str::slug($value);
    }

    /**
     * Get the formatted content.
     *
     * @return string
     */
    public function getFormattedContentAttribute(): string
    {
        return nl2br(e($this->content));
    }

    ########################################################
    #################       RELATIONS      #################

    /**
     * Get the user that owns the post.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo<User, Post>
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the categories for the post.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsToMany<Category, Post>
     */
    public function categories(): BelongsToMany
    {
        return $this->belongsToMany(Category::class);
    }

    /**
     * Get the comments for the post.
     *
     * @return \Illuminate\Database\Eloquent\Relations\HasMany<Comment, Post>
     */
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    ########################################################
    #################       SCOPES      #################

    /**
     * Scope a query to only include published posts.
     *
     * @param \Illuminate\Database\Eloquent\Builder<Post> $query
     * @return \Illuminate\Database\Eloquent\Builder<Post>
     */
    public function scopePublished(Builder $query): Builder
    {
        return $query->where('status', 'published');
    }

    /**
     * Scope a query to only include draft posts.
     *
     * @param \Illuminate\Database\Eloquent\Builder<Post> $query
     * @return \Illuminate\Database\Eloquent\Builder<Post>
     */
    public function scopeDraft(Builder $query): Builder
    {
        return $query->where('status', 'draft');
    }

    /**
     * Scope a query to only include posts with featured flag.
     *
     * @param \Illuminate\Database\Eloquent\Builder<Post> $query
     * @return \Illuminate\Database\Eloquent\Builder<Post>
     */
    public function scopeFeatured(Builder $query): Builder
    {
        return $query->where('featured', true);
    }
}
```

### Method Annotations

All model methods should have proper PHPDoc annotations for better IDE support and code documentation:

```php
/**
 * Get paginated items with relationships.
 *
 * @param int $perPage Number of items per page
 * @return \Illuminate\Contracts\Pagination\LengthAwarePaginator
 */
public static function getPaginatedWithRelationships(int $perPage = 15)
{
    return self::buildFilteredQuery()
        ->with(['user', 'categories'])
        ->paginate($perPage)
        ->withQueryString();
}

/**
 * Get statistics for dashboard.
 *
 * @return array{total: int, active: int, inactive: int}
 */
public static function getStats(): array
{
    $stats = self::selectRaw('
        COUNT(*) as total,
        SUM(CASE WHEN status = ? THEN 1 ELSE 0 END) as active,
        SUM(CASE WHEN status != ? THEN 1 ELSE 0 END) as inactive
    ', ['active', 'active'])
    ->first();

    return [
        'total' => (int) $stats->total,
        'active' => (int) $stats->active,
        'inactive' => (int) $stats->inactive,
    ];
}

/**
 * Create with relationships.
 *
 * @param array<string, mixed> $data The model data
 * @param array<string, mixed> $relations The relations data
 * @return self
 * @throws \Throwable
 */
public static function createWithRelations(array $data, array $relations = []): self
{
    return DB::transaction(function () use ($data, $relations) {
        $item = static::create($data);

        if (isset($relations['categories'])) {
            $item->categories()->sync($relations['categories']);
        }

        return $item;
    });
}

/**
 * Set the post title and automatically generate slug.
 *
 * @param string $value The title value
 * @return void
 */
public function setTitleAttribute(string $value): void
{
    $this->attributes['title'] = $value;
    $this->attributes['slug'] = Str::slug($value);
}

/**
 * Get the user that owns the post.
 *
 * @return \Illuminate\Database\Eloquent\Relations\BelongsTo<User, Post>
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

/**
 * Scope a query to only include published posts.
 *
 * @param \Illuminate\Database\Eloquent\Builder<Post> $query
 * @return \Illuminate\Database\Eloquent\Builder<Post>
 */
public function scopePublished(Builder $query): Builder
{
    return $query->where('status', 'published');
}
```

**Annotation Guidelines:**
- Always include `@param` with type and description
- Always include `@return` with type and description
- Use specific types like `int`, `string`, `array<string, mixed>`
- For relationships, include the related models: `BelongsTo<User, Post>`
- For query scopes, include the Builder type: `Builder<Post>`
- Include `@throws` for methods that might throw exceptions
- Use `void` for methods that don't return a value

### Essential Model Methods

```php
/**
 * Get paginated items with relationships.
 */
public static function getPaginatedWithRelationships(int $perPage = 15)
{
    return self::buildFilteredQuery()
        ->with(['user', 'categories'])
        ->paginate($perPage)
        ->withQueryString();
}

/**
 * Get statistics for dashboard.
 */
public static function getStats(): array
{
    $stats = self::selectRaw('
        COUNT(*) as total,
        SUM(CASE WHEN status = ? THEN 1 ELSE 0 END) as active,
        SUM(CASE WHEN status != ? THEN 1 ELSE 0 END) as inactive
    ', ['active', 'active'])
    ->first();

    return [
        'total' => (int) $stats->total,
        'active' => (int) $stats->active,
        'inactive' => (int) $stats->inactive,
    ];
}

/**
 * Create with relationships.
 */
public static function createWithRelations(array $data, array $relations = []): self
{
    return DB::transaction(function () use ($data, $relations) {
        $item = static::create($data);

        if (isset($relations['categories'])) {
            $item->categories()->sync($relations['categories']);
        }

        return $item;
    });
}
```

### FilteredQueryBuilder Configuration

```php
// Custom filters for FilteredQueryBuilder
protected function getCustomFilters(): array
{
    return [
        AllowedFilter::callback('category', function ($query, $value) {
            $query->whereHas('categories', function ($q) use ($value) {
                $q->where('categories.id', $value);
            });
        }),
    ];
}

// Searchable fields
protected function getSearchableFields(): array
{
    return ['title', 'content'];
}

// Default relationships to load
protected function getDefaultRelations(): array
{
    return ['user', 'categories'];
}

// Custom sorts
protected function getCustomSorts(): array
{
    return [
        AllowedCaseInsensitiveSort::field('name'),
        AllowedSort::field('created_at'),
    ];
}
```

## Data Classes for Validation & Transformation

### DTO-Based Validation: The Spatie Laravel Data Approach

DTO = Data Transfer Object. With Spatie Laravel Data, Data class defines your structure, validation rules, and types all in one place.

#### How It Works

When you use a Data class as a controller parameter:
1. Laravel receives the HTTP request
2. Spatie maps request fields into your Data class constructor
3. Validation happens automatically using the attributes
4. If validation fails → automatic Laravel validation response
5. If validation passes → you get a fully validated, typed PHP object

#### Controller Signature Change

**Before (manual validation):**
```php
public function store(Request $request)
{
    $validated = $request->validate([...]);
    // Manual validation logic
}
```

**After (automatic validation):**
```php
public function store(AdvertisementData $data)
{
    // $data is already validated and typed!
    Advertisement::create($data->toArray());
}
```

### Data Class Structure

Create Data classes in `app/Data/[Model]Data.php`:

```php
<?php

namespace App\Data;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Required;
use Spatie\LaravelData\Attributes\Validation\StringType;
use Spatie\LaravelData\Attributes\Validation\Max;
use Spatie\LaravelData\Attributes\Validation\ArrayType;
use Spatie\LaravelData\Attributes\Validation\Nullable;
use Spatie\LaravelData\Attributes\Validation\BooleanType;

class AdvertisementData extends Data
{
    public function __construct(
        #[Required, StringType, Max(255)]
        public string $name,

        #[Required, StringType, Max(255)]
        public string $description,

        #[Nullable, ArrayType]
        public ?array $advertisements,
    ) {}

    public static function rules(\Spatie\LaravelData\Support\Validation\ValidationContext $context = null): array
    {
        return [
            'advertisements.*' => ['integer', 'exists:advertisements,id'],
        ];
    }

    public static function fromModel(Advertisement $advertisement): self
    {
        return new self(
            name: $advertisement->name,
            description: $advertisement->description,
            advertisements: $advertisement->advertisements->pluck('id')->all(),
        );
    }
}
```

### Validation Attributes

Every validation rule lives in the Data class using attributes:

```php
public function __construct(
    #[Required, StringType, Max(255)]
    public string $name,

    #[Nullable, BooleanType]
    public ?bool $is_active,

    #[ArrayType, Nullable]
    public ?array $advertisements,
) {}
```

### Arrays and Nested Data

For arrays of integers:
```php
#[ArrayType, Nullable]
public ?array $advertisements,
```

Then add array validation rules in the `rules()` method:
```php
public static function rules(\Spatie\LaravelData\Support\Validation\ValidationContext $context = null): array
{
    return [
        'advertisements.*' => ['integer', 'exists:advertisements,id'],
    ];
}
```

For arrays of objects (Data objects):
```php
#[DataCollectionOf(TagData::class)]
public DataCollection $tags,
```

### Controller Implementation

**Clean controller with automatic validation:**

```php
public function store(AdvertisementData $data): RedirectResponse
{
    // Zero manual validation - already done!
    $advertisement = Advertisement::create($data->toArray());

    return redirect()
        ->route('admin.advertisements.index')
        ->with('success', 'Advertisement created');
}

public function update(AdvertisementData $data, Advertisement $advertisement): RedirectResponse
{
    $advertisement->update($data->toArray());

    return back()->with('success', 'Advertisement updated');
}
```

### Handling Partial Updates: Separate DTOs

**Problem:** You have a full DTO with required fields, but need to update only some fields (like status).

**Solution:** Create separate DTOs for different update scenarios.

#### Example: AdZone with Template Management

```php
// Full DTO for complete operations
class AdZoneData extends Data
{
    public function __construct(
        #[Required, StringType, Max(255)]
        public string $name,

        #[Required, StringType, Max(50)]
        public string $size,

        #[Required, StringType, Max(50)]
        public string $type,

        #[Nullable, BooleanType]
        public ?bool $is_active,

        #[Nullable, ArrayType]
        public ?array $advertisements,
    ) {}
}

// Separate DTO for status-only updates
class AdZoneStatusData extends Data
{
    public function __construct(
        #[Required, BooleanType]
        public bool $is_active,
    ) {}
}
```

#### Controller with Multiple DTOs

```php
public function update(AdZoneData $adZoneData, AdZone $adZone): RedirectResponse
{
    // For template-managed zones, redirect to status update
    if ($adZone->is_template_managed) {
        return redirect()
            ->route('admin.ad-zones.update-status', $adZone)
            ->with('error', 'Use status update endpoint for template-managed zones');
    }

    // For non-template zones, allow full updates
    $adZone->updateWithRelations(
        $adZoneData->toArray(),
        ['advertisements' => $adZoneData->advertisements ?? []]
    );

    return redirect()
        ->route('admin.ad-zones.index')
        ->with('success', 'Ad Zone updated successfully');
}

public function updateStatus(AdZoneStatusData $statusData, AdZone $adZone): RedirectResponse
{
    $adZone->update(['is_active' => $statusData->is_active]);

    return redirect()
        ->route('admin.ad-zones.index')
        ->with('success', 'Ad Zone status updated successfully');
}
```

### Simple Transformation Pattern

Use Laravel Data's built-in `collect()` method for clean transformations:

```php
// Controller - one line transformation
public function index(Request $request): Response
{
    $users = User::getPaginated(getPerPage());

    return Inertia::render('admin/users/index', [
        'users' => UserData::collect($users),
    ]);
}

// For editing - use fromModel()
public function edit(User $user): Response
{
    return Inertia::render('admin/users/edit', [
        'user' => UserData::fromModel($user),
    ]);
}
```

### Key Benefits

1. **No duplicate validation** - All rules live in one place
2. **No manual validation** - Automatic validation happens before controller runs
3. **Type safety** - Values are cast correctly
4. **Clean controllers** - No more `$request->validate()` or custom validators
5. **Consistent API** - Same pattern for all data operations
6. **Better testing** - Validation logic is isolated and testable

### What to Avoid

❌ **Don't do this - manual validation in controller:**
```php
public function store(Request $request)
{
    $validated = AdvertisementData::validate($request->all());
    Advertisement::create($validated);
}
```

❌ **Don't do this - duplicate validation rules:**
```php
// In Data class
#[Required, StringType, Max(255)]
public string $name;

// AND also in controller
$request->validate(['name' => 'required|string|max:255']);
```

❌ **Don't do this - trying to validate partial data with full DTO:**
```php
// This will fail because name, size, type are required
public function updateStatus(Request $request, AdZone $adZone)
{
    $data = AdZoneData::validate($request->only('is_active')); // FAILS!
    $adZone->update(['is_active' => $data->is_active]);
}
```

✅ **Do this - clean, automatic validation:**
```php
public function store(AdvertisementData $data): RedirectResponse
{
    Advertisement::create($data->toArray());
    return redirect()->route('admin.advertisements.index');
}

// For partial updates, use dedicated DTO
public function updateStatus(AdZoneStatusData $statusData, AdZone $adZone): RedirectResponse
{
    $adZone->update(['is_active' => $statusData->is_active]);
    return redirect()->route('admin.ad-zones.index');
}
```

## Controller Patterns

### Standard Controller Structure

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Data\PostData;
use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Inertia\Inertia;
use Inertia\Response;
use function App\Helpers\getPerPage;

class PostController extends Controller
{
    public function index(Request $request): Response
    {
        $posts = Post::getPaginatedWithRelationships(
            perPage: getPerPage()
        );

        return Inertia::render('admin/posts/index', [
            'posts' => PostData::collect($posts),
            'filters' => $request->only(['search', 'status', 'category']),
            'stats' => Post::getStats(),
            'categories' => Category::orderBy('name')->get(['id', 'name']),
        ]);
    }

    public function create(): Response
    {
        return Inertia::render('admin/posts/create', [
            'categories' => Category::orderBy('name')->get(['id', 'name']),
        ]);
    }

    public function store(PostData $postData): RedirectResponse
    {
        $post = Post::createWithRelations(
            $postData->all(),
            ['categories' => $postData->categories ?? []]
        );

        return redirect()
            ->route('admin.posts.index')
            ->with('success', 'Post created successfully');
    }

    public function edit(Post $post): Response
    {
        $post->load(['categories']);

        return Inertia::render('admin/posts/Edit', [
            'post' => PostData::from($post),
            'categories' => Category::orderBy('name')->get(['id', 'name']),
        ]);
    }

    public function update(PostData $postData, Post $post): RedirectResponse
    {
        $post->updateWithRelations(
            $postData->all(),
            ['categories' => $postData->categories ?? []]
        );

        return back()->with('success', 'Post updated successfully');
    }

    public function destroy(Post $post): RedirectResponse
    {
        $post->delete();

        return redirect()
            ->route('admin.posts.index')
            ->with('success', 'Post deleted successfully');
    }
}
```

### Special Controller Features

#### Status Management

```php
public function toggleStatus(Post $post): RedirectResponse
{
    // define this method in the model
    $post->toggleStatus();

    return back()->with('success', 'Status updated')
}
```

#### File Uploads

```php
public function uploadFile(Request $request, Post $post): RedirectResponse
{
    $request->validate([
        'file' => ['required', 'image', 'max:2048'],
    ]);

    $post->handleFileUpload($request->file('file'));
    $post->save();

    return back()->with('success', 'File uploaded successfully');
}
```

## Route Configuration

### Standard Route Structure (Or you can use resource routes)

```php
// routes/web.php
Route::middleware(['auth', 'verified'])->prefix('admin')->name('admin.')->group(function () {

    Route::prefix('posts')->name('posts.')->group(function () {
        Route::get('/', [PostController::class, 'index'])->name('index');
        Route::get('/create', [PostController::class, 'create'])->name('create');
        Route::post('/', [PostController::class, 'store'])->name('store');
        Route::get('/{post}/edit', [PostController::class, 'edit'])->name('edit');
        Route::put('/{post}', [PostController::class, 'update'])->name('update');
        Route::delete('/{post}', [PostController::class, 'destroy'])->name('destroy');

        // Custom routes
        Route::patch('{post}/toggle-status', [PostController::class, 'toggleStatus'])
            ->name('toggle-status');
    });

});

// IMPORTANT: After adding new routes, always run:
// php artisan route:clear
// php artisan route:cache
```

## Helper Functions

### Pagination Helper

Available in `app/Helpers/CommonHelpers.php`:

```php
// In controller
public function index(Request $request): Response
{
    $posts = Post::getPaginatedWithRelationships(
        perPage: getPerPage()
    );

    return Inertia::render('admin/posts/index', [
        'posts' => PostData::collect($posts),
        'filters' => $request->only(['search', 'status', 'category']),
        'stats' => Post::getStats(),
        'perPageOptions' => getAllowedPerPage(),
    ]);
}
```

**Available functions:**
- `getPerPage()` - Gets `per_page` from request, validates allowed [10, 25, 50, 100], or returns default
- `getDefaultPerPage($default = 25)` - Returns setting value or provided default
- `getAllowedPerPage()` - Returns allowed options [10, 25, 50, 100]

## Helper Traits Usage

### 1. FilteredQueryBuilder Trait

**Key Features:**
- Automatic search in title/name/email fields
- Quick date filters (today, thisWeek, lastMonth, etc.)
- User relationship filtering (auto-detected)
- Status field filtering (auto-detected)

**URL Examples:**
```
?search=keyword                          // Search
?filter[user]=1                         // By user ID
?filter[status]=active                  // By status
?filter[thisMonth]=1                    // This month's records
?sort=-created_at                       // Sort by date desc
```

### 2. QueryScopes Trait

**Usage:**
```php
Model::mine()->get();                    // Current user's records
Model::key(1)->first();                  // Find by ID
Model::exclude([1,2])->get();            // Exclude IDs
Model::active()->get();                  // Where status = active
Model::notNull('field')->get();          // Where field is not null
Model::shallowSearch('term')->get();     // Search in default fields
```

### 3. DateTimeScopes Trait

**Usage:**
```php
Model::ofToday()->get();                 // Today's records
Model::ofLastWeek()->get();              // Last week
Model::ofLast30Days()->get();            // Last 30 days
Model::thisMonth()->get();               // This month to date
Model::DateBetween('2024-01-01', '2024-12-31')->get();
```

### 4. ProtectedFillable Trait

**How it works:**
1. Base `$fillable` - always available
2. `$createFillable` - only when creating
3. `$updateFillable` - only when updating
4. `$adminFillable` - merged if user is admin

### 5. HasOwnership Trait

**Features:**
- Auto-filters records by current user
- Admin users see all records
- Configurable owner field

**Usage:**
```php
// Automatic filtering - users only see their own records
$posts = Post::all();

// Admin bypass
$allPosts = Post::asAdmin()->get();

// Ownership checks
$post->isOwnedBy($user);
$post->canBeAccessedBy($user);
```

## Package Integration

### Key Packages

1. **spatie/laravel-permission** - Role-based permissions
2. **spatie/laravel-medialibrary** - Media management
3. **spatie/laravel-query-builder** - Used by FilteredQueryBuilder
4. **spatie/laravel-data** - Type-safe data transfer and validation
5. **cybercog/laravel-ban** - User banning functionality
6. **beyondcode/laravel-comments** - Comment system
7. **ralphjsmit/laravel-seo** - SEO management

### MediaLibrary Integration

```php
// In model
use Spatie\MediaLibrary\HasMedia;
use Spatie\MediaLibrary\InteractsWithMedia;

class Post extends Model implements HasMedia
{
    use InteractsWithMedia;

    ########################################################
    #################       MUTATORS      #################

    /**
     * Get the featured image URL with fallback.
     *
     * @return string
     */
    public function getFeaturedImageUrlAttribute(): string
    {
        return $this->getFeaturedImageUrl();
    }

    ########################################################
    #################       RELATIONS      #################

    /**
     * Get the user that owns the post.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo<User, Post>
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the categories for the post.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsToMany<Category, Post>
     */
    public function categories(): BelongsToMany
    {
        return $this->belongsToMany(Category::class);
    }

    ########################################################
    #################       SCOPES      #################

    /**
     * Scope a query to only include posts with featured images.
     *
     * @param \Illuminate\Database\Eloquent\Builder<Post> $query
     * @return \Illuminate\Database\Eloquent\Builder<Post>
     */
    public function scopeWithFeaturedImage(Builder $query): Builder
    {
        return $query->whereHas('media', function ($q) {
            $q->where('collection_name', 'featured_image');
        });
    }

    /**
     * Media library methods
     */

    /**
     * Register media collections for the model.
     *
     * @return void
     */
    public function registerMediaCollections(): void
    {
        $this->addMediaCollection('featured_image')
            ->singleFile()
            ->acceptsMimeTypes(['image/jpeg', 'image/png', 'image/webp'])
            ->useDisk('public');
    }

    /**
     * Register media conversions for the model.
     *
     * @param \Spatie\MediaLibrary\MediaCollections\Models\Media|null $media
     * @return void
     */
    public function registerMediaConversions($media = null): void
    {
        $this->addMediaConversion('thumb')
            ->width(300)
            ->height(200)
            ->sharpen(10)
            ->performOnCollections('featured_image');
    }

    /**
     * Get the featured image URL with conversion.
     *
     * @param string $conversion The conversion name
     * @return string
     */
    public function getFeaturedImageUrl(string $conversion = 'thumb'): string
    {
        return $this->getFirstMediaUrl('featured_image', $conversion)
            ?? asset('images/placeholder.jpg');
    }
}

// Handle upload in controller
public function store(PostData $postData, Request $request): RedirectResponse
{
    $post = Post::create($postData->toArray());

    if ($request->hasFile('featured_image')) {
        $post->addMediaFromRequest('featured_image')
            ->toMediaCollection('featured_image');
    }

    return redirect()->route('admin.posts.index');
}
```

## Security Practices

### Validation with Laravel Data

```php
use Spatie\LaravelData\Attributes\Validation\Required;
use Spatie\LaravelData\Attributes\Validation\String;
use Spatie\LaravelData\Attributes\Validation\Max;
use Spatie\LaravelData\Attributes\Validation\Image;
use Spatie\LaravelData\Attributes\Validation\File;

class PostData extends Data
{
    public function __construct(
        #[Required, String, Max(255)]
        public string $title,

        #[Required, String]
        public string $content,

        #[Image, File('max:4096')]
        public ?UploadedFile $image = null,

        public array $categories = [],
    ) {}
}
```

### Authorization with Permissions and Ownership

```php
// In controller - automatic ownership check with HasOwnership
public function update(PostData $postData, Post $post): RedirectResponse
{
    // HasOwnership ensures users can only update their own records
    // Admins bypass this automatically
    $this->authorize('update', $post);

    $post->update($postData->toArray());

    return redirect()->back()->with('success', 'Post updated');
}

// Policy combining permissions and ownership
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $post->canBeAccessedBy($user) &&
               $user->can('edit posts');
    }
}

// Middleware protection
Route::middleware(['auth', 'verified', 'permission:manage posts'])
    ->prefix('admin/posts')
    ->group(function () {
        // All post management routes
    });
```

## Clean Models: Move Transformations to Data Layer

Keep models focused only on data logic, not presentation transformations:

### Clean Model Example

```php
// ✅ Clean model - no presentation logic
class User extends Model
{
    use HasOwnership, FilteredQueryBuilder, QueryScopes, DateTimeScopes;
    use ProtectedFillable;

    // Base fillable - always available
    protected $fillable = [
        'name',
        'email',
        'password',
        'status',
    ]; 
    ########################################################
    #################       MUTATORS      #################

    /**
     * Hash the password when setting it.
     *
     * @param string $value The password value
     * @return void
     */
    public function setPasswordAttribute(string $value): void
    {
        $this->attributes['password'] = bcrypt($value);
    }

    /**
     * Get the user's full name.
     *
     * @return string
     */
    public function getFullNameAttribute(): string
    {
        return trim($this->first_name . ' ' . $this->last_name);
    }

    ########################################################
    #################       RELATIONS      #################

    /**
     * Get the roles assigned to the user.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsToMany<Role, User>
     */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }

    /**
     * Get the posts created by the user.
     *
     * @return \Illuminate\Database\Eloquent\Relations\HasMany<Post, User>
     */
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    /**
     * Get the media associated with the user.
     *
     * @return \Illuminate\Database\Eloquent\Relations\MorphMany<Media, User>
     */
    public function media(): MorphMany
    {
        return $this->morphMany(Media::class, 'model');
    }

    ########################################################
    #################       SCOPES      #################

    /**
     * Scope a query to only include active users.
     *
     * @param \Illuminate\Database\Eloquent\Builder<User> $query
     * @return \Illuminate\Database\Eloquent\Builder<User>
     */
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', 'active');
    }

    /**
     * Scope a query to only include admin users.
     *
     * @param \Illuminate\Database\Eloquent\Builder<User> $query
     * @return \Illuminate\Database\Eloquent\Builder<User>
     */
    public function scopeAdmins(Builder $query): Builder
    {
        return $query->where('is_admin', true);
    }

 

    /**
     * Get paginated users with relationships.
     *
     * @param int $perPage Number of items per page
     * @return \Illuminate\Contracts\Pagination\LengthAwarePaginator
     */
    public static function getPaginated(int $perPage = 15): LengthAwarePaginator
    {
        return self::buildFilteredQuery()
            ->with(['roles', 'media'])
            ->paginate($perPage);
    }
}

// Transformation logic moves to Data class
class UserData extends Data
{
    public function __construct(
        public int $id,
        public string $name,
        public string $email,
        public string $avatar_url,
        public readonly string $full_name,
    ) {}

    public static function from(User $user): self
    {
        return new self(
            id: $user->id,
            name: $user->name,
            email: $user->email,
            avatar_url: $user->getAvatarUrl('thumb'),
            full_name: trim($user->first_name . ' ' . $user->last_name),
        );
    }
}
```

## Query Scopes Best Practices

### Divide Complex Queries into Reusable Scopes

Always break down complex queries into smaller, reusable scopes. This makes your code more maintainable, testable, and easier to understand.

#### Example: Advertisement Model

Instead of writing complex queries repeatedly, create individual scopes for each condition:

```php
// ❌ BAD - Complex inline query
public function scopeCurrentlyActive($query)
{
    return $query->where('is_active', true)
        ->where(function ($query) {
            $query->whereNull('start_date')
                ->orWhere('start_date', '<=', now());
        })
        ->where(function ($query) {
            $query->whereNull('end_date')
                ->orWhere('end_date', '>=', now());
        });
}

// ✅ GOOD - Break into reusable scopes
/**
 * Scope a query to only include active advertisements.
 *
 * @param \Illuminate\Database\Eloquent\Builder $query
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function scopeActive($query)
{
    return $query->where('is_active', true);
}

/**
 * Scope a query to only include advertisements that have started.
 *
 * @param \Illuminate\Database\Eloquent\Builder $query
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function scopeStarted($query)
{
    return $query->where(function ($query) {
        $query->whereNull('start_date')
            ->orWhere('start_date', '<=', now());
    });
}

/**
 * Scope a query to only include advertisements that have not ended.
 *
 * @param \Illuminate\Database\Eloquent\Builder $query
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function scopeNotEnded($query)
{
    return $query->where(function ($query) {
        $query->whereNull('end_date')
            ->orWhere('end_date', '>=', now());
    });
}

/**
 * Scope a query to only include advertisements that are currently active.
 * This is a convenience scope that combines active, started, and notEnded scopes.
 *
 * @param \Illuminate\Database\Eloquent\Builder $query
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function scopeCurrentlyActive($query)
{
    return $query->active()->started()->notEnded();
}
```

#### Benefits of This Approach

1. **Reusability**: Each scope can be used independently
   ```php
   // Get all active ads (regardless of dates)
   Advertisement::active()->get();
   
   // Get all ads that have started (including inactive ones)
   Advertisement::started()->get();
   
   // Get all ads that haven't ended (including inactive ones)
   Advertisement::notEnded()->get();
   
   // Get fully active ads
   Advertisement::currentlyActive()->get();
   ```

2. **Composability**: Scopes can be combined in different ways
   ```php
   // Active ads that haven't ended
   Advertisement::active()->notEnded()->get();
   
   // Started ads with specific criteria
   Advertisement::started()->where('featured', true)->get();
   ```

3. **Testability**: Each scope can be tested independently
   ```php
   public function test_active_scope()
   {
       $activeAd = Advertisement::factory()->create(['is_active' => true]);
       $inactiveAd = Advertisement::factory()->create(['is_active' => false]);
       
       $ads = Advertisement::active()->get();
       
       $this->assertTrue($ads->contains($activeAd));
       $this->assertFalse($ads->contains($inactiveAd));
   }
   ```

4. **Readability**: The intent of each query is clear
   ```php
   // Clear intent
   Advertisement::active()->started()->notEnded()->get();
   
   // vs unclear intent
   Advertisement::where('is_active', true)
     ->where(function ($query) {
         $query->whereNull('start_date')
             ->orWhere('start_date', '<=', now());
     })
     ->where(function ($query) {
         $query->whereNull('end_date')
             ->orWhere('end_date', '>=', now());
     })
     ->get();
   ```

#### Guidelines for Creating Scopes

1. **Single Responsibility**: Each scope should handle one specific condition
2. **Descriptive Names**: Use clear, verb-based names (`active`, `started`, `notEnded`)
3. **Chainable**: Design scopes to work well together
4. **Documentation**: Always document what each scope does
5. **Consistency**: Follow naming conventions across your models

## Testing Standards

### Controller Tests

```php
<?php

use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;

class PostControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_index_displays_posts()
    {
        Post::factory()->count(5)->create();

        $response = $this->get('/admin/posts');

        $response->assertOk();
        $response->assertInertiaProp('posts');
    }

    public function test_can_create_post()
    {
        $data = [
            'title' => 'Test Post',
            'content' => 'Test content',
            'status' => 'active',
        ];

        $response = $this->post('/admin/posts', $data);

        $response->assertRedirect('/admin/posts');
        $this->assertDatabaseHas('posts', $data);
    }

    public function test_can_update_post()
    {
        $post = Post::factory()->create();

        $response = $this->put("/admin/posts/{$post->id}", [
            'title' => 'Updated Title',
        ]);

        $response->assertRedirect("/admin/posts/{$post->id}");
        $this->assertDatabaseHas('posts', [
            'id' => $post->id,
            'title' => 'Updated Title',
        ]);
    }

    public function test_can_delete_post()
    {
        $post = Post::factory()->create();

        $response = $this->delete("/admin/posts/{$post->id}");

        $response->assertRedirect('/admin/posts');
        $this->assertDatabaseMissing('posts', ['id' => $post->id]);
    }
}
```

## Quick Reference

### Module Development Checklist

1. **Model Setup**
   - [ ] Use all helper traits (HasOwnership, FilteredQueryBuilder, QueryScopes, DateTimeScopes)
   - [ ] Define fillable arrays fillable  and use ProtectedFillable (adminFillable, createFillable, updateFillable) of needed
   - [ ] Create the three modal segments: MUTATORS, RELATIONS, and SCOPES
   - [ ] Add proper PHPDoc annotations to all methods with @param, @return, and @throws tags
   - [ ] create required scopes
   - [ ] create required relations
   - [ ] create required mutatures (if needed)
   - [ ] Add getStats() method (if is mandatory)
   - [ ] Add getPaginatedWithRelationships()
   - [ ] Configure custom filters and sorts (if needed)

2. **Data Objects & Transformations**
   - [ ] Create Data objects: `php artisan make:data PostData`
   - [ ] Use validation attributes instead of form requests
   - [ ] Add from() method for transformations
   - [ ] Move ALL transformations from models to Data layer
   - [ ] Use built-in collect() method for bulk transformations
   - [ ] Generate TypeScript types: `php artisan typescript:transform`

3. **Controller**
   - [ ] Use Data objects for automatic validation
   - [ ] Pass all necessary data to frontend
   - [ ] Include filters in response
   - [ ] Handle file uploads securely

4. **Routes**
   - [ ] Use consistent naming
   - [ ] Apply auth and permission middleware
   - [ ] Clear route cache after changes: `php artisan route:cache`

### Common Patterns

```php
// Model with all traits
class Model extends Model
{
    use HasOwnership, FilteredQueryBuilder, QueryScopes, DateTimeScopes, ProtectedFillable;

    // Base fillable - always available
    protected $fillable = [
        'name',
        'description',
        'status',
        'user_id',
    ];

    // Admin-only fields
    protected $adminFillable = [
        'featured',
        'order',
    ];

    ########################################################
    #################       MUTATORS      #################

    /**
     * Set the model name and automatically generate slug.
     *
     * @param string $value The name value
     * @return void
     */
    public function setNameAttribute(string $value): void
    {
        $this->attributes['name'] = $value;
        $this->attributes['slug'] = Str::slug($value);
    }

    ########################################################
    #################       RELATIONS      #################

    /**
     * Get the user that owns the model.
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo<User, Model>
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    ########################################################
    #################       SCOPES      #################

    /**
     * Scope a query to only include active models.
     *
     * @param \Illuminate\Database\Eloquent\Builder<Model> $query
     * @return \Illuminate\Database\Eloquent\Builder<Model>
     */
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', 'active');
    }

    /**
     * Scope a query to only include featured models.
     *
     * @param \Illuminate\Database\Eloquent\Builder<Model> $query
     * @return \Illuminate\Database\Eloquent\Builder<Model>
     */
    public function scopeFeatured(Builder $query): Builder
    {
        return $query->where('featured', true);
    }
}

// Data object for validation
class PostData extends Data
{
    public function __construct(
        #[Required, String, Max(255)]
        public string $title,
        #[Required, String]
        public string $content,
    ) {}
}

// Controller with Data validation
public function store(PostData $postData): RedirectResponse
{
    // Auto-validated!
    $post = Model::create($postData->toArray());
    return redirect()->route('admin.models.index');
}

// Ownership-aware operations (automatic with HasOwnership)
$myModels = Model::all();  // Only current user's models
$allModels = Model::asAdmin()->get();  // All models for admins

// Date filtering
Model::ofLast30Days()->active()->get();
Model::thisMonth()->desc()->paginate(15);
```

## Performance Best Practices

### Query Optimization

```php
// ✅ CORRECT - Eager loading
$items = Item::with(['categories', 'user'])->paginate(10);

// ✅ CORRECT - Select only needed columns
$items = Item::select(['id', 'name', 'status'])->paginate(10);

// ❌ WRONG - N+1 queries
foreach ($items as $item) {
    echo $item->category->name; // N+1 query
}
```

This guide provides the foundation for building consistent, maintainable admin modules that integrate seamlessly with the existing codebase.