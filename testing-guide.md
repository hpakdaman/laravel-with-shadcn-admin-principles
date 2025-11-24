# Testing Guide

This guide covers best practices for writing tests in your Laravel application, with special focus on testing models, components, and functionality.

## Table of Contents
1. [Philosophy](#philosophy)
2. [Testing Models](#testing-models)
3. [Testing Components](#testing-components)
4. [Testing API Endpoints](#testing-api-endpoints)
5. [Testing Filters and Scopes](#testing-filters-and-scopes)
6. [Common Testing Patterns](#common-testing-patterns)
7. [Best Practices](#best-practices)

## Philosophy

### Why We Test
- **Reliability**: Ensure code works as expected
- **Regression Prevention**: Catch breaking changes early
- **Documentation**: Tests serve as living documentation
- **Refactoring Confidence**: Make changes with assurance

### What Makes a Good Test
1. **Specific**: Tests one thing at a time
2. **Clear**: Test name describes what it validates
3. **Independent**: Tests don't rely on each other
4. **Repeatable**: Same result every time
5. **Fast**: Quick to run

## Testing Models

### Using Sushi for Test Data

For model testing, we use [Sushi](https://github.com/calebporzio/sushi) to create in-memory SQLite databases with test data. This allows testing Eloquent models without needing a full database setup.

See [tests/Models/readme.md](../tests/Models/readme.md) for detailed Sushi usage.

### Basic Model Test Structure

```php
<?php

use Tests\Models\TestModel;

// Test basic functionality
it('can create a record', function () {
    $model = TestModel::create([
        'name' => 'Test Item',
        'status' => 'active',
    ]);

    expect($model)->toBeInstanceOf(TestModel::class);
    expect($model->name)->toBe('Test Item');
    expect($model->status)->toBe('active');
});

// Test scopes
it('can filter by scope correctly', function () {
    // Create test data
    TestModel::create(['name' => 'Active Item', 'status' => 'active']);
    TestModel::create(['name' => 'Inactive Item', 'status' => 'inactive']);

    // Test scope
    $activeItems = TestModel::active()->get();

    // Verify the scope works
    expect($activeItems)->toHaveCount(1);
    expect($activeItems->first()->name)->toBe('Active Item');

    // Verify all returned items match the criteria
    $activeItems->each(function ($item) {
        expect($item->status)->toBe('active');
    });
});

// Test relationships
it('can load related models', function () {
    $user = TestUser::create(['name' => 'John']);
    $role = TestRole::create(['name' => 'Admin']);

    $user->role()->associate($role);
    $user->save();

    $loadedUser = TestUser::with('role')->first();

    expect($loadedUser->role)->toBeInstanceOf(TestRole::class);
    expect($loadedUser->role->name)->toBe('Admin');
});
```

### Testing Scopes and Query Methods

**❌ Bad Test - Only checks return type:**
```php
it('uses today scope', function () {
    $results = TestModel::today()->get();
    expect($results)->toBeInstanceOf(Collection::class);
});
```

**✅ Good Test - Verifies actual functionality:**
```php
it('uses today scope correctly', function () {
    // Setup test data
    $today = now()->toDateString();
    TestModel::create(['name' => 'Today Item', 'created_at' => now()]);
    TestModel::create(['name' => 'Yesterday Item', 'created_at' => now()->subDay()]);

    // Execute scope
    $results = TestModel::today()->get();

    // Verify all results are from today
    $results->each(function ($item) use ($today) {
        expect($item->created_at->toDateString())->toBe($today);
    });

    // Verify expected items are included
    expect($results)->toHaveCount(1);
    expect($results->first()->name)->toBe('Today Item');
});
```

### Testing Model Methods

```php
it('can calculate total correctly', function () {
    $item = TestModel::create([
        'price' => 100,
        'tax' => 20,
    ]);

    $total = $item->calculateTotal();

    expect($total)->toBe(120);
});

it('validates required fields', function () {
    expect(fn() => TestModel::create([]))
        ->toThrow('Database query failed');
});
```

## Testing Components

### Testing React Components

For testing React components, ensure you're actually testing the component's behavior, not just that it renders.

**❌ Bad Test - Only checks if it renders:**
```php
it('renders button', function () {
    render(<Button>Click me</Button>);
    expect(true)->toBeTrue(); // Doesn't test anything!
});
```

**✅ Good Test - Verifies behavior:**
```php
it('triggers onClick when clicked', function () {
    const handleClick = jest.fn();

    render(
        <Button onClick={handleClick}>
            Click me
        </Button>
    );

    fireEvent.click(screen.getByRole('button', { name: 'Click me' }));

    expect(handleClick).toHaveBeenCalledTimes(1);
});

it('shows loading state when disabled', function () {
    render(
        <Button disabled loading>
            Submit
        </Button>
    );

    expect(screen.getByRole('button')).toBeDisabled();
    expect(screen.getByText('Submitting...')).toBeInTheDocument();
});
```

### Testing Filter Components

```php
it('updates search query after debounce', function () {
    const onSearchChange = jest.fn();

    render(
        <SearchInput
            config={{ key: 'search', label: 'Search', type: 'search' }}
            value=""
            onChange={onSearchChange}
        />
    );

    // Type in search
    fireEvent.change(screen.getByPlaceholderText('Search...'), {
        target: { value: 'test query' }
    });

    // Should not call immediately (debounce)
    expect(onSearchChange).not.toHaveBeenCalled();

    // Wait for debounce
    act(() => {
        jest.advanceTimersByTime(500);
    });

    // Should call after debounce
    expect(onSearchChange).toHaveBeenCalledWith('test query');
});
```

## Testing API Endpoints

### Basic Endpoint Test

```php
it('can fetch users list', function () {
    // Create test data
    User::factory()->create(['name' => 'John Doe']);

    // Make request
    $response = $this->get('/api/users');

    // Assert response
    $response->assertStatus(200)
             ->assertJsonStructure([
                 'data' => [
                     '*' => ['id', 'name', 'email', 'created_at']
                 ]
             ]);

    // Verify data
    $response->assertJsonFragment([
        'name' => 'John Doe'
    ]);
});

it('can create new user', function () {
    $userData = [
        'name' => 'Jane Doe',
        'email' => 'jane@example.com',
        'password' => 'password123',
    ];

    $response = $this->post('/api/users', $userData);

    $response->assertStatus(201);

    $this->assertDatabaseHas('users', [
        'name' => 'Jane Doe',
        'email' => 'jane@example.com'
    ]);
});
```

### Testing with Authentication

```php
it('requires authentication to access protected endpoint', function () {
    $response = $this->get('/api/profile');

    $response->assertStatus(401);
});

it('returns user profile when authenticated', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
                     ->get('/api/profile');

    $response->assertStatus(200)
             ->assertJsonFragment([
                 'id' => $user->id,
                 'name' => $user->name
             ]);
});
```

## Testing Filters and Scopes

### Comprehensive Filter Testing

```php
it('applies multiple filters correctly', function () {
    // Setup varied test data
    User::factory()->create([
        'name' => 'John Admin',
        'role' => 'admin',
        'status' => 'active',
        'created_at' => now()
    ]);

    User::factory()->create([
        'name' => 'Jane User',
        'role' => 'user',
        'status' => 'inactive',
        'created_at' => now()->subDays(5)
    ]);

    // Apply filters
    $results = User::active()
                   ->role('admin')
                   ->createdAfter(now()->subDays(1))
                   ->get();

    // Verify all filters applied
    expect($results)->toHaveCount(1);
    expect($results->first()->name)->toBe('John Admin');

    // Verify each filter condition
    $results->each(function ($user) {
        expect($user->status)->toBe('active');
        expect($user->role)->toBe('admin');
        expect($user->created_at->greaterThan(now()->subDays(1)))->toBeTrue();
    });
});

it('handles empty results gracefully', function () {
    $results = User::where('name', 'NonExistent')->get();

    expect($results)->toHaveCount(0);
    expect($results->isEmpty())->toBeTrue();
});

it('combines scopes correctly', function () {
    User::factory()->count(10)->create(['status' => 'active']);
    User::factory()->count(5)->create(['status' => 'inactive']);

    $activeUsers = User::active()->latest()->take(5)->get();

    expect($activeUsers)->toHaveCount(5);
    $activeUsers->each(function ($user) {
        expect($user->status)->toBe('active');
    });
});
```

## Testing Date/Time Functionality

### Testing Date Ranges

```php
it('correctly filters by date range', function () {
    $baseDate = now();

    // Create records across different dates
    $records = [
        ['date' => $baseDate->copy()->subDays(5), 'name' => 'Five days ago'],
        ['date' => $baseDate->copy()->subDays(3), 'name' => 'Three days ago'],
        ['date' => $baseDate->copy()->subDays(1), 'name' => 'Yesterday'],
        ['date' => $baseDate, 'name' => 'Today'],
        ['date' => $baseDate->copy()->addDays(1), 'name' => 'Tomorrow'],
    ];

    foreach ($records as $record) {
        TestModel::create($record);
    }

    // Test date range
    $start = $baseDate->copy()->subDays(2);
    $end = $baseDate->copy()->addDays(1);

    $results = TestModel::dateBetween($start, $end)->get();

    // Verify only records within range
    expect($results)->toHaveCount(3);
    expect($results->pluck('name')->toArray())->toEqual([
        'Yesterday',
        'Today',
        'Tomorrow'
    ]);

    // Verify dates are actually in range
    $results->each(function ($record) use ($start, $end) {
        expect($record->date->greaterThanOrEqualTo($start))->toBeTrue();
        expect($record->date->lessThanOrEqualTo($end))->toBeTrue();
    });
});
```

## Common Testing Patterns

### Data Factory Pattern

```php
// Create a factory
class UserFactory extends Factory
{
    public function definition()
    {
        return [
            'name' => fake()->name,
            'email' => fake()->unique()->safeEmail,
            'status' => 'active',
        ];
    }

    public function inactive()
    {
        return $this->state(fn () => ['status' => 'inactive']);
    }

    public function admin()
    {
        return $this->state(fn () => ['role' => 'admin']);
    }
}

// Use in tests
it('handles inactive users', function () {
    $user = User::factory()->inactive()->create();

    expect($user->status)->toBe('inactive');
});
```

### Setup and Teardown

```php
beforeEach(function () {
    // Runs before each test
    $this->user = User::factory()->create();
});

afterEach(function () {
    // Runs after each test
    // Cleanup if needed
});

beforeAll(function () {
    // Runs once before all tests in file
});

afterAll(function () {
    // Runs once after all tests in file
});
```

### Custom Assertions

```php
// Create custom assertion in TestCase.php
protected function assertUserIsActive($user)
{
    $this->assertTrue($user->isActive(), 'User should be active');
}

// Use in test
it('marks user as active after registration', function () {
    $user = User::factory()->inactive()->create();
    $user->activate();

    $this->assertUserIsActive($user);
});
```

## Best Practices

### DO ✅
1. **Test one thing per test** - Keep tests focused
2. **Use descriptive test names** - Explain what is being tested
3. **Arrange, Act, Assert** - Follow this pattern
4. **Test edge cases** - Empty data, null values, boundaries
5. **Use factories** - For test data creation
6. **Verify both positive and negative cases** - What works and what doesn't
7. **Keep tests fast** - Use in-memory databases when possible
8. **Mock external dependencies** - APIs, file systems
9. **Test error conditions** - Invalid input, failures
10. **Use meaningful assertions** - Verify actual behavior

### DON'T ❌
1. **Don't test implementation details** - Test behavior, not code
2. **Don't rely on test order** - Tests should be independent
3. **Don't ignore test failures** - Fix or understand before continuing
4. **Don't use hardcoded IDs** - Use factories or fixtures
5. **Don't test framework features** - Laravel's tests cover this
6. **Don't create unnecessary abstractions** - Keep tests simple
7. **Don't skip edge cases** - They often reveal bugs
8. **Don't use live databases** in unit tests
9. **Don't ignore performance** - Tests should run quickly
10. **Don't write vague assertions** - Be specific about expectations

### Example of a Complete Test

```php
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase);

it('can search users with multiple criteria', function () {
    // Arrange
    $activeAdmin = User::factory()->create([
        'name' => 'Admin User',
        'email' => 'admin@example.com',
        'role' => 'admin',
        'status' => 'active',
        'created_at' => now()
    ]);

    $inactiveUser = User::factory()->create([
        'name' => 'Inactive User',
        'email' => 'inactive@example.com',
        'role' => 'user',
        'status' => 'inactive',
        'created_at' => now()->subDays(10)
    ]);

    $activeUser = User::factory()->create([
        'name' => 'Regular User',
        'email' => 'user@example.com',
        'role' => 'user',
        'status' => 'active',
        'created_at' => now()->subDays(5)
    ]);

    // Act
    $results = User::active()
                   ->search('User')
                   ->createdAfter(now()->subDays(7))
                   ->get();

    // Assert
    expect($results)->toHaveCount(2);

    // Verify correct users returned
    $returnedNames = $results->pluck('name')->toArray();
    expect($returnedNames)->toContain('Admin User', 'Regular User');
    expect($returnedNames)->not->toContain('Inactive User');

    // Verify all conditions met
    $results->each(function ($user) {
        expect($user->status)->toBe('active');
        expect($user->name)->toContain('User');
        expect($user->created_at->greaterThan(now()->subDays(7)))->toBeTrue();
    });
});
```

## Running Tests

```bash
# Run all tests
./vendor/bin/pest

# Run specific test file
./vendor/bin/pest tests/Unit/UserTest.php

# Run specific test
./vendor/bin/pest tests/Unit/UserTest.php --filter="can create user"

# Run with coverage
./vendor/bin/pest --coverage

# Run in parallel (faster)
./vendor/bin/pest --parallel
```

## Resources

- [Laravel Testing Documentation](https://laravel.com/docs/testing)
- [Pest Testing Framework](https://pestphp.com/)
- [Sushi Package](https://github.com/calebporzio/sushi)
- [Testing Laravel Applications](https://laraveldaily.com/testing-laravel-applications/)

Remember: Good tests are your safety net. They give you confidence to make changes and prevent regressions!