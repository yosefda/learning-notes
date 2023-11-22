# Laravel Getting Started Guide

My notes on basic/typical flow of working with Laravel for beginners.

## 1. Install Laravel

https://kinsta.com/knowledgebase/install-laravel/#how-to-install-laravel-on-macos

## 2. Starting a project

Starting a Laravel project can be done either as new project or working with the existing one.

### New project

Create a new project by running:

```
composer create-project laravel/laravel <project-name>
```

Run `npm run`.

### Existing project

Clone the project from the repo. Then initialise the project by running:

```
composer install
cp env.example .env
php artisan key:generate
```

Run `npm run`.

After that run `php artisan serve` to check that all dependencies installed and project initialised correctly.

Next thing to do is to check connection to database is set correctly:

1. Create the database in mysql
2. Check the database config in `.env`, especially the following

   ```
   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=<database-name>
   DB_USERNAME=root
   DB_PASSWORD=
   ```

3. Run migration dry-run `php artisan migrate --pretend`

## 3. Database entities

Understand the database entities required for the project, including their relationships. This will help to create the correct migration script.

## 4. Create database migration

Migration scripts are stored in `database/migrations` directory.

Run `php artisan make:migration <migration-script-name>`, for example `php artisan make:migration CreateUsersTable`. This will create a new migration script in the directory above.

Then we modify the `up()` and `down()` functions to define migration and rollback:

```php
public function up()
{
    Schema::create('users', function (Blueprint $table) {
        $table->id(),
        $table->string('first_name'),
        $table->string('last_name'),
        $table->unsignedInteger('profession_id')->notNullable(),
        $table->unsignedInteger('credit_balance')
    });

    $table->timestamps();
}

public function down()
{
    Schema::dropIfExists('users');
}
```

After that we run `php artisan migrate`.

## 5. Create model

After we have tables created in MySQL, the next step is to create model to represent entity in the code/app.

Run `php artisan make:model <model-name>`, for example `php artisan make:model User`. This will create a new model `app/Models/User.php`.

Then we modify the model class, for example the `User model` will have:

```php
// List of fields that can be mass assigned
protected $fillable = [
    'first_name',
    'last_name',
    'profession_id',
    'credit_balance'
];

// Optional table name if it doesnt follow the standard table name
protected $table = 'users';

// Optional primary key name if it doesnt use standard id column name
protected $primaryKey = 'id';

// Optional if the table doesnt need created_at and updated_at
public $timestamps = true;
```

## 5. Create factory

Factory for the model is useful if we want to create a mock/test data.

Run `php artisan meke:factory <FactoryName>`. For example `php artisan make:factory UserFactory` will create a new factory `database/factories/UserFactory.php`.

Modify `definition()` to define the values to be created. It will use [FakerPHP](https://fakerphp.github.io/), for example:

```php
public function definition()
{
    return [
        'first_name' => fake()->firstName(),
        'last_name' => fake()->lastName(),
        'profession_id' => fake()->numberBetween(1, 3),
        'credit_balance' => fake()->numberBetween(0, 1000)
    ];
}
```

## 6. Create seeder

Seeder is another way to create test data. This step is useful for manual testing of endpoint during development.

Run `php artisan make:seeder <seeder-name>`. For example `php artisan make:seeder UserSeeder` will create a new seeder `database/seeders/UserSeeder.php`.

Modify `run()` to define records to be created, for example:

```php
public function run()
{
    User::create([
        'id' => 1,
        'first_name' => 'Barry',
        'last_name' => 'Allen',
        'profession_id' => 1,
        'credit_balance' => 0
    ]);

    User::create([
            'id' => 2,
            'first_name' => 'Tony',
            'last_name' => 'Stark',
            'profession_id' => 2,
            'credit_balance' => 100
        ]);
}
```

Then run `php artisan db:seeder --class=<seeder-name>`, for example `php artisan db:seeder --class=UserSeeder` to create `Users` records in database.

Optionally we can run all seeders with a single call. For this we need to modify `database/seeders/DatabaseSeeder.php` to call all the seeders:

```php
public function run()
{
    $this->call([
        ProfessionSeeder::class,
        UserSeeder::class
    ]);
}
```

After that we can simply run `php artisan db:seeder` to create all the seeded records.

The seeders can be ran multiple times, but we have to remember to `truncate` the tables before running them.

## 7. Controller

The step is to create controller for the entity.

For REST API, the best practice is to put the controllers in `app/Http/Controllers/Api` directory.

Best to create a base `API Controller` that can be extended by other REST API controllers. For example this base controller can define the methods to format response and error:

1. Run `php artisan make:controller Api/ApiController`, this will create a new controller `app/Http/Controllers/Api/ApiController.php`
2. Modify the class to add response and error functions

   ```php
   class ApiController extends Controller
   {
       protected function sendResponse($message, $httpCode, $data)
       {
           return response()->json([
               'success' => true,
               'message' => $message,
               'data' => $data
           ], $httpCode);
       }

       protected function sendError($message, $httpCode, $error)
       {
           return response()->json([
               'success' => false,
               'message' => $message,
               'error' => $error
           ], $httpCode);
       }
   }
   ```

After that we can create controller for the other entity, for example `php artisan make:controller Api/UserController`. This will create another controller `app/Http/Controllers/Api/UserController.php`.

Because this is an REST API controller, we want it to extend `ApiController`. So we need to modify the class:

```php
class UserController extends ApiController
{
    //...
}
```

## 8. Add routes

Routes for REST API are defined in `routes/api.php`.

For example we want to add routes for `POST /users` and `GET /users`, we need to add:

```php
Route::post('/users', [UserController::class, 'create']);
Route::get('/users', [UserController::class, 'index']);
```

`create` and `index` are the handler methods that we have to implement in `UserController`.

We can run `php artisan serve` and check the endpoints are working. If everything is defined correct the endpoints should be accessible and return `HTTP 200`. The url will be `http://127.0.0.1:8000/api/users`.

## 9. Implement controller

The previous steps are basically the prep for this step. Implementing a method in a controller is basically a repeat of the following steps:

#### Implement controller logic

Make sure we use the model and validator classes

```php
use App\Models\User;
use Validator;
```

Example for `/GET users`

```php
public function index()
{
    public function index()
    {
        $users = User::join('professions', 'users.profession_id', '=', 'professions.id')
            ->get();

        return $this->sendResponse('Professions found', 200, $users);
    }
}
```

Example for `/POST users`

```php
public function create(Request $request)
    {
        $validator = Validator::make(
            $request->all(),
            [
                'first_name' => ['required', 'string', 'max:255'],
                'last_name' => ['required', 'string', 'max:255'],
                'profession_id' => ['required', 'numeric', 'exists:professions,id'],
                'credit_balance' => ['required', 'numeric']
            ]
        );

        if ($validator->fails()) {
            return $this->sendError(
                'Validation error',
                400,
                $validator->errors()
            );
        }

        $input = $request->all();
        $user = User::create($input);

        return $this->sendResponse(
            'User created',
            200,
            $user
        );
    }
```

#### Try it on browser

Use Postman to access the endpoint and examine the response and http code.

#### Add feature test for it

It is a good practice to create the feature after we verify that endpoint working with manual test before continuing.

Run `php artisan make:test <test-name>` to create a new feature test. For example `php artisan make:test UserTest` will create a new test `tests/Feature/UserTest.php`.

Then add the test scenario by adding more `test_<test_name>()`, for example:

```php
class UserTest extends TestCase
{
    use RefreshDatabase;

    /**
     * A basic feature test example.
     *
     * @return void
     */
    public function test_get_users_returns_data()
    {
        $this->seed([
            ProfessionSeeder::class,
            UserSeeder::class
        ]);

         $response = $this->get('/api/users');
         $response->assertStatus(200)
            ->assertJson([
                'success' => true,
                'message' => 'User(s) found',
                'data' => [
                    [
                        'id' => 1,
                        'first_name' => 'Barry',
                        'last_name' => 'Allen',
                        'profession_id' => 1,
                        'credit_balance' => 0,
                        'description' => 'Carpentry'
                    ]
                ]
        ]);

    }

    public function test_post_users_success()
    {
        $response = $this->post('/api/users', [
            'first_name' => 'Tony',
            'last_name' => 'Stark',
            'profession_id' => 3,
            'credit_balance' => 0
        ]);

        $response->assertStatus(200)
            ->assertJson([
                'success' => true,
                'message' => 'User created',
                'data' => [
                    'id' => 1,
                        'first_name' => 'Barry',
                        'last_name' => 'Allen',
                        'profession_id' => 1,
                        'credit_balance' => 0,
                        'description' => 'Carpentry'
                ]
            ]);

        $this->assertDatabaseHas('professions', [
            'first_name' => 'Barry',
            'last_name' => 'Allen',
            'profession_id' => 1,
            'credit_balance' => 0,
            'description' => 'Carpentry'
        ]);

        // To assert if a particular record is no longer in db
        // $this->assertDatabaseMissing('users', [
        //     'id' => 2,
        //      'first_name' => 'Tony',
        //      'last_name' => 'Stark',
        // ]);
    }
}
```

#### Repeat
