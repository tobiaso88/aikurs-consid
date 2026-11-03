### Laravel Version

11.x

### PHP Version

8.4

### Database Driver & Version

pgsql 17

### Description

# Type Mismatch in Polymorphic Relationships When Using PostgreSQL

## Description
When using polymorphic relationships with PostgreSQL, there's a type mismatch issue between the parent model's ID (integer) and the morphed model's foreign key (string). This occurs because the `getKeyType()` method always returns 'int' for regular incrementing IDs, while the morphed table's foreign key is defined as a string.

## Current Behavior
The query fails with the following PostgreSQL error:
```sql
SQLSTATE[42883]: Undefined function: 7 ERROR: operator does not exist: character varying = integer
LINE 1: ...om "integrations" where "integrations"."owner_id" in (1) and...
^
HINT: No operator matches the given name and argument types. You might need to add explicit type casts.
```

## Database Schema
### Integrations Table
```php
Schema::create('integrations', function (Blueprint $table) {
    $table->id();
    $table->string('owner_type');
    $table->string('owner_id'); // Defined as string
    $table->string('provider');
    $table->timestamps();
});
```

### Users Table
```php
Schema::create('users', function (Blueprint $table) {
    $table->id(); // Integer type
    // ... other columns
});
```

## Model Relationships
### User Model
```php
public function integrations(): MorphMany
{
    return $this->morphMany(Integration::class, 'owner');
}
```

### Integration Model
```php
public function owner(): MorphTo
{
    return $this->morphTo('owner');
}
```

## Expected Behavior
The polymorphic relationship should handle the type casting between the integer primary key of the parent model and the string foreign key in the morphed table when using PostgreSQL.

## Possible Solution
Consider implementing automatic type casting for morphed foreign keys based on the column type, rather than relying solely on the parent model's `getKeyType()` method.

## Additional Context
- This issue specifically affects PostgreSQL due to its strict type checking
- MySQL handles this scenario without errors due to implicit type conversion
- The `getKeyType()` method currently only returns 'string' for UUID keys, defaulting to 'int' for all other cases

## Tags
- Bug
- PostgreSQL
- Eloquent
- Polymorphic Relationships

### Steps To Reproduce

## Steps to Reproduce

1. Create the database tables:
```php
// Create users table
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamps();
});

// Create integrations table
Schema::create('integrations', function (Blueprint $table) {
    $table->id();
    $table->string('owner_type');
    $table->string('owner_id');
    $table->string('provider');
    $table->timestamps();
});
```

2. Set up the models:
```php
// User.php
class User extends Model
{
    public function integrations(): MorphMany
    {
        return $this->morphMany(Integration::class, 'owner');
    }
}

// Integration.php
class Integration extends Model
{
    public function owner(): MorphTo
    {
        return $this->morphTo('owner');
    }
}
```

3. Create a test user:
```php
$user = User::create([
    'name' => 'Test User',
    'email' => 'test@example.com'
]);
```

4. Try to query the integrations for the user:
```php
// This will trigger the error
$integrations = Integration::where('owner_id', $user->id)
    ->where('owner_type', User::class)
    ->get();

// Or alternatively, this will also trigger the error
$userIntegrations = $user->integrations()->get();
```

5. Observe the PostgreSQL error:
```
SQLSTATE[42883]: Undefined function: 7 ERROR: operator does not exist: character varying = integer
LINE 1: ...om "integrations" where "integrations"."owner_id" in (1) and...
^
HINT: No operator matches the given name and argument types. You might need to add explicit type casts.
```

### Environment:
- Laravel Framework (latest version)
- PostgreSQL database
- PHP 8.x

The error occurs specifically with PostgreSQL due to its strict type checking between string and integer comparisons. The same code works in MySQL due to implicit type conversion.
