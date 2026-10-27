## How to Work With Me

### Don't assume, ask

When I ask a question or give a task, do not fill gaps with your own assumptions. If anything is unclear or could be read more than one way (names, fields, types, validation rules, behavior, edge cases, where something goes), stop and ask me clarifying questions before writing code.

### Workflow for every new feature

Build every feature in this order. Each step is its own stop: do one step, show me what you did, ask any clarifying questions for the next step, and **wait for my confirmation before moving on**. Never do several steps at once.

1. **Domain model:** define the domain model's structure in `app/DomainModel` (fields, types, `fromArray()`, `toArray()`). See [Domain Model](#domain-model).
2. **Action:** define the action class in `app/Actions` that holds the business logic. See [Actions](#actions).
3. **Action tests:** test the action with PHPUnit and run the tests. Show me the results.
4. **Controller and API route:** define the Form Request, the controller and its route in `routes/api.php`. See [Validation](#validation) and [API Routes](#api-routes).
5. **Controller and route tests:** write PHPUnit feature tests that call the API endpoint, and run them. Show me the results.

Run tests with:

```sh
php artisan test
```

## Domain Model

The main domain model lives in `app/DomainModel` (namespace `App\DomainModel`).

A domain model is a plain PHP class:

- No Eloquent, no framework base class, no database access. Just a plain PHP class.
- Its fields are typed and set through the constructor.
- It has a static `fromArray(array $data): self` method that takes an associative array and maps its keys onto the class fields.
- It has a `toArray(): array` method that does the reverse: it turns the fields back into an associative array with the same keys `fromArray()` accepts, so `Order::fromArray($order->toArray())` gives an equal object.

### Code shape

```php
<?php

namespace App\DomainModel;

final class Order
{
    public function __construct(
        public readonly int $id,
        public readonly string $customerName,
        public readonly float $total,
        public readonly ?string $note = null,
    ) {}

    /**
     * @param  array{id: int, customer_name: string, total: float, note?: string|null}  $data
     */
    public static function fromArray(array $data): self
    {
        return new self(
            id: (int) $data['id'],
            customerName: (string) $data['customer_name'],
            total: (float) $data['total'],
            note: $data['note'] ?? null,
        );
    }

    /**
     * @return array{id: int, customer_name: string, total: float, note: string|null}
     */
    public function toArray(): array
    {
        return [
            'id' => $this->id,
            'customer_name' => $this->customerName,
            'total' => $this->total,
            'note' => $this->note,
        ];
    }
}
```

Usage:

```php
$order = Order::fromArray([
    'id' => 1,
    'customer_name' => 'Jane',
    'total' => 99.5,
]);

$data = $order->toArray();
// ['id' => 1, 'customer_name' => 'Jane', 'total' => 99.5, 'note' => null]
```

When you add a new domain model, put it in `app/DomainModel` and follow this shape.

## Repositories

Actions read and save domain models through **repositories**. Each domain model has a repository interface, and for now each interface is implemented by an **in-memory repository**.

- Interfaces: `app/Repositories/Contracts` (namespace `App\Repositories\Contracts`), e.g. `ProductRepository`
- In-memory classes: `app/Repositories/InMemory` (namespace `App\Repositories\InMemory`), e.g. `ProductInMemoryRepository`
- Binding: `app/Providers/RepositoryServiceProvider.php`

Rules:

- Repository methods take and return **domain models**, never arrays and never Eloquent models.
- Actions type-hint the **interface** in their constructor, never the in-memory class.
- The id always comes with the data (from the request or the caller). Repositories never generate ids; `save()` stores the product under its own id.
- The base methods are `all()`, `find()`, `save()` and `delete()`. Add other methods only when a feature needs them.

### In-memory data only lives for one request

PHP starts fresh on every HTTP request, so the in-memory repository's data is lost when the request ends. A product saved by one API call is **not** there in the next API call. Within one request, and within one PHPUnit test, the data is kept. Use the in-memory repository for tests and logic that runs inside one request, not as storage that must survive between API calls.

### Interface shape

```php
<?php

namespace App\Repositories\Contracts;

use App\DomainModel\Product;

interface ProductRepository
{
    /**
     * @return list<Product>
     */
    public function all(): array;

    public function find(int $id): ?Product;

    public function save(Product $product): Product;

    public function delete(int $id): void;
}
```

### In-memory repository shape

Products are stored as `Product` objects in an array keyed by id. Starting data is passed through the constructor as `Product` objects. The repository never receives raw arrays.

```php
<?php

namespace App\Repositories\InMemory;

use App\DomainModel\Product;
use App\Repositories\Contracts\ProductRepository;

final class ProductInMemoryRepository implements ProductRepository
{
    /**
     * @var array<int, Product>
     */
    private array $products = [];

    /**
     * @param  list<Product>  $products
     */
    public function __construct(array $products = [])
    {
        foreach ($products as $product) {
            $this->products[$product->id] = $product;
        }
    }

    public function all(): array
    {
        return array_values($this->products);
    }

    public function find(int $id): ?Product
    {
        return $this->products[$id] ?? null;
    }

    public function save(Product $product): Product
    {
        $this->products[$product->id] = $product;

        return $product;
    }

    public function delete(int $id): void
    {
        unset($this->products[$id]);
    }
}
```

### Seed data file

The starting data for the running app lives in `database/data/<model>s.php`, e.g. `database/data/products.php`. It returns a list of plain arrays in the shape `fromArray()` accepts, with no logic:

```php
<?php

return [
    ['id' => 1, 'name' => 'Keyboard', 'price' => 49.99],
    ['id' => 2, 'name' => 'Mouse', 'price' => 19.99],
];
```

### Binding in `RepositoryServiceProvider`

Create the provider once with:

```sh
php artisan make:provider RepositoryServiceProvider
```

This also registers it in `bootstrap/providers.php`.

Bind every repository with **`singleton()`, not `bind()`**. `bind()` would give each action its own new, empty repository; `singleton()` makes everything in the same request or test share one store. The provider turns the seed arrays into domain models with `fromArray()` and passes them to the constructor:

```php
<?php

namespace App\Providers;

use App\DomainModel\Product;
use App\Repositories\Contracts\ProductRepository;
use App\Repositories\InMemory\ProductInMemoryRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(ProductRepository::class, fn () => new ProductInMemoryRepository(
            array_map(Product::fromArray(...), require database_path('data/products.php')),
        ));
    }
}
```

### Passing data in tests

Tests never rely on the seed file. Each test passes in exactly the products it needs.

Action unit tests create the repository directly and pass it to the action:

```php
$repository = new ProductInMemoryRepository([
    Product::fromArray(['id' => 1, 'name' => 'Keyboard', 'price' => 49.99]),
]);

$action = new UpdateProductPriceAction($repository);
```

Controller and API feature tests replace the shared repository in the container for that test:

```php
$this->app->instance(ProductRepository::class, new ProductInMemoryRepository([
    Product::fromArray(['id' => 1, 'name' => 'Keyboard', 'price' => 49.99]),
]));

$this->getJson('/api/products/1')->assertOk();
```

## Actions

All business logic lives in **Action classes** in `app/Actions` (namespace `App\Actions`). Controllers, Form Requests and domain models contain no business logic. A controller validates (through its Form Request), calls an action, and returns a response.

An action class:

- Does one thing, and is named after it with an `Action` suffix: `CreateOrderAction`, `CancelOrderAction`.
- Receives **all of its dependencies through the constructor** (other actions, services, Laravel contracts). Laravel's service container builds the action and fills these in automatically.
- Has one public `handle()` method. The data the action works on is passed to `handle()`, and is either a **domain model** or **plain data** (scalars or an array).
- Never receives the `Request` object. Dependencies go in the constructor, data goes in `handle()`.

### Creating an action

```sh
php artisan make:class Actions/CreateOrderAction
```

This creates `app/Actions/CreateOrderAction.php` with a namespace and an empty constructor. Then add the constructor dependencies and the `handle()` method so it matches the shape below.

### Action shape (domain model as input)

```php
<?php

namespace App\Actions;

use App\DomainModel\Order;
use Illuminate\Contracts\Events\Dispatcher;
use Illuminate\Database\ConnectionInterface;

class CreateOrderAction
{
    public function __construct(
        private readonly ConnectionInterface $db,
        private readonly Dispatcher $events,
    ) {}

    public function handle(Order $order): Order
    {
        return $this->db->transaction(function () use ($order) {
            // business logic goes here

            return $order;
        });
    }
}
```

### Action shape (plain data as input)

```php
<?php

namespace App\Actions;

use Illuminate\Database\ConnectionInterface;

class CancelOrderAction
{
    public function __construct(
        private readonly ConnectionInterface $db,
    ) {}

    public function handle(int $orderId, ?string $reason = null): void
    {
        // business logic goes here
    }
}
```

### Calling an action from a controller

Type-hint the action in the controller method next to the Form Request. Laravel builds it and passes it in:

```php
public function store(StoreOrderRequest $request, CreateOrderAction $createOrder): JsonResponse
{
    $order = $createOrder->handle(Order::fromArray($request->validated()));

    return response()->json($order->toArray(), 201);
}
```

## API Routes

API endpoints go in `routes/api.php`. Laravel automatically puts the `/api` prefix and the `api` middleware group on every route in that file.

### Activating `routes/api.php`

New Laravel projects (11+) don't have API routes turned on. Check that:

1. `routes/api.php` exists, and
2. `bootstrap/app.php` registers it inside `withRouting(...)`:

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    api: __DIR__.'/../routes/api.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

If either one is missing, run:

```sh
php artisan install:api
```

This creates `routes/api.php`, adds the `api:` line to `bootstrap/app.php`, installs Laravel Sanctum, and runs the migration for the `personal_access_tokens` table.

Check it worked with:

```sh
php artisan route:list --path=api
```

## Validation

Validation is done in the controller using **Form Requests**. Do not call `$request->validate()` or `Validator::make()` inside controller methods, and do not validate inside domain models.

- Create one Form Request per action: `php artisan make:request StoreOrderRequest`
- Form Requests live in `app/Http/Requests`.
- Type-hint the Form Request in the controller method. Laravel validates the request before the method runs. If validation fails, an API request gets a `422` JSON response automatically.
- Pass `$request->validated()` into the domain model's `fromArray()`. Never pass `$request->all()`.
- Every Form Request includes a `toDomainModel()` method that returns the matching domain model, built with `fromArray($this->validated())`. It is optional, so it is **always added commented out**. Uncomment it only when the controller needs it, and then call `$request->toDomainModel()` in the controller instead of calling `fromArray()` there.

### Form Request shape

```php
<?php

namespace App\Http\Requests;

// use App\DomainModel\Order;
use Illuminate\Foundation\Http\FormRequest;

class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'id' => ['required', 'integer'],
            'customer_name' => ['required', 'string', 'max:255'],
            'total' => ['required', 'numeric', 'min:0'],
            'note' => ['nullable', 'string'],
        ];
    }

    // /**
    //  * Build the domain model from the validated data.
    //  */
    // public function toDomainModel(): Order
    // {
    //     return Order::fromArray($this->validated());
    // }
}
```

### Controller shape

```php
<?php

namespace App\Http\Controllers;

use App\Actions\CreateOrderAction;
use App\DomainModel\Order;
use App\Http\Requests\StoreOrderRequest;
use Illuminate\Http\JsonResponse;

class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, CreateOrderAction $createOrder): JsonResponse
    {
        $order = $createOrder->handle(Order::fromArray($request->validated()));

        return response()->json($order->toArray(), 201);
    }
}
```

If `toDomainModel()` is uncommented in the Form Request, the controller uses it instead:

```php
$order = $createOrder->handle($request->toDomainModel());
```

### Route shape (`routes/api.php`)

```php
use App\Http\Controllers\OrderController;
use Illuminate\Support\Facades\Route;

Route::post('/orders', [OrderController::class, 'store']);
```
