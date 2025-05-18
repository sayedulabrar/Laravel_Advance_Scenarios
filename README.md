
# Laravel Automatic Relationship Loading

Yes! **Here's the simplest way to think about it:**

1. **For "I don't know what relations will be needed"** (e.g., admin dashboards, dynamic UIs):  
   ```php
   $projects = Project::all()->withRelationshipAutoloading(); // Auto-loads ANY accessed relation
   ```

2. **For "I know exactly what's needed"** (e.g., APIs, performance-critical code):  
   ```php
   $projects = Project::with(['client', 'tasks'])->get(); // Explicitly loads ONLY these
   ```
3. **Check loaded relations**
   ```php
   dd($projects->first()->getRelations());
   ```
   
### **Key Points:**
- **Don’t enable `automaticallyEagerLoadRelationships()` globally** if you want precise control.  
- **Use `withRelationshipAutoloading()` per-query** only when lazy-loading is acceptable.  
- **Default to `with([])`** for predictable performance.  




Absolutely! Here's a detailed note on **how to properly implement API versioning in Laravel**, including which files to modify and what to write in each. You can add this to your team’s internal documentation or use it as a setup guide.

---

## 📘 Laravel API Versioning Guide

To ensure stability and backward compatibility for consumers of your API, follow this structured approach to implement versioning in Laravel.

---

### 📁 1. Folder Structure

Organize versioned controllers under appropriately named folders:

```
app/
└── Http/
    └── Controllers/
        └── Api/
            ├── V1/
            │   └── UserController.php
            └── V2/
                └── UserController.php
```

---

### 📄 2. Create Namespaced Controllers

**Example: `UserController` for V1**

```php
// File: app/Http/Controllers/Api/V1/UserController.php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index()
    {
        return response()->json(['message' => 'Users from API v1']);
    }
}
```

---

### 📄 3. Define Versioned Routes

Edit your `routes/api.php` file to register versioned endpoints using route groups and namespaces.

```php
use Illuminate\Support\Facades\Route;

// API V1 Routes
Route::prefix('v1')->namespace('App\Http\Controllers\Api\V1')->group(function () {
    Route::apiResource('users', UserController::class);
});

// API V2 Routes (example)
Route::prefix('v2')->namespace('App\Http\Controllers\Api\V2')->group(function () {
    Route::apiResource('users', UserController::class);
});
```

✅ You can also use Laravel's **`Route::group`** with `as` for route naming:

```php
Route::prefix('v1')
    ->namespace('App\Http\Controllers\Api\V1')
    ->as('api.v1.')
    ->group(function () {
        Route::apiResource('posts', PostController::class);
    });
```

---

---

### 🛠️ 5. Testing Endpoints

Test versioned routes via Postman or curl:

```bash
curl http://localhost:8000/api/v1/users
curl http://localhost:8000/api/v2/users
```






# Browsershot within a Laravel application to generate a PDF report.

Here's a step-by-step explanation and some best practices:

**1. Installation (Likely Already Done):**

First, ensure you have Browsershot installed in your Laravel project. If not, you can install it via Composer:

```bash
composer require spatie/browsershot
```
# Inside your Laravel project directory
npm install puppeteer --save

**2. Importing the Browsershot Class:**

In your `ReportController.php` (or wherever you're generating the report), make sure you import the `Browsershot` class at the top of your file:

```php
use Spatie\Browsershot\Browsershot;
```

**3. Preparing Your Data (`$payload`):**

The image shows you've created a `$payload` array containing the data you want to display in your report. This is excellent\! Browsershot will render a Blade view, and you're correctly passing this data to the view.

```php
$payload = [
    'name' => 'The Boring Company',
    'address' => 'Pine Rd. Arcadia, Texas US',
    'phone' => '+1 999-868-7777',
    'email' => 'xyz@example.com',
    'tax_rate' => 12.99,
    'items' => $sales, // Assuming $sales is defined elsewhere
];
```

**4. Rendering the Blade View (`view('report', ...)`):**

You're using Laravel's `view()` helper to load your `report.blade.php` view and passing the `$payload` array to it. This is the standard way to pass data to your views in Laravel.

```php
$template = view('report', ['payload' => $payload])->render();
```

The `->render()` method converts the Blade view into an HTML string, which Browsershot will then process.

**5. Using Browsershot (`Browsershot::html($template)`):**

This is the core of using Browsershot. You're calling the static `html()` method on the `Browsershot` class and providing the rendered HTML content (`$template`) as an argument. This tells Browsershot what content to render as a PDF.

```php
Browsershot::html($template)
```

**6. Saving the PDF (`->save(...)`)**:

The `->save()` method tells Browsershot where to save the generated PDF file. You're using Laravel's `storage_path()` helper to save the PDF within your storage directory. This is a good practice for organizing generated files.

```php
    ->save(storage_path('/app/reports/example.pdf'));
```

**Putting It All Together (Your Controller Code):**

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Spatie\Browsershot\Browsershot;
use Illuminate\Support\Facades\DB; // Assuming you're using the DB facade for $sales

class ReportController extends Controller
{
    public function generateReport()
    {
        // Fetch your sales data (example)
        $sales = DB::table('orders')
            ->orderBy('total_quantity_sold', 'desc')
            ->get();

        $payload = [
            'name' => 'The Boring Company',
            'address' => 'Pine Rd. Arcadia, Texas US',
            'phone' => '+1 999-868-7777',
            'email' => 'xyz@example.com',
            'tax_rate' => 12.99,
            'items' => $sales,
        ];

        $template = view('report', ['payload' => $payload])->render();

        Browsershot::html($template)
            ->save(storage_path('/app/reports/example.pdf'));

        return 'PDF report generated successfully!'; // Or redirect to the file
    }
}
```

**Key Considerations and Best Practices:**

  * **Your Blade View (`report.blade.php`):** This view is crucial. It will receive the `$payload` data and should be structured with HTML and CSS to present the report as you desire. Remember that Browsershot essentially renders this HTML in a headless browser.

  * **CSS Styling:** You can use regular CSS within your `report.blade.php` to style the PDF. Browsershot will interpret most standard CSS. You might need to consider using print-specific stylesheets (`<link rel="stylesheet" type="text/css" media="print" href="print.css">`) for optimal PDF output.

  * **Assets (CSS, Images):** If your Blade view relies on external CSS or image assets, you might need to configure Browsershot to correctly locate these assets. You can use the `setPath()` method if necessary. For example:

    ```php
    Browsershot::html($template)
        ->setPath(public_path('css')) // If your CSS is in the public/css directory
        ->save(storage_path('/app/reports/example.pdf'));
    ```

    For images, ensure the paths in your Blade view are correct (e.g., using `asset('images/logo.png')` in Laravel).

  * **Customization:** Browsershot offers many other customization options. You can set paper size, orientation, margins, add headers and footers, and more. Refer to the official Browsershot documentation for a full list of available methods. For example:

    ```php
    Browsershot::html($template)
        ->setPaperSize('A4')
        ->showBackground()
        ->setPaperOrientation('landscape')
        ->setMargin(10, 10, 15, 10) // top, right, bottom, left
        // ->setHeaderHtml('<h1>Report Header</h1>')
        // ->setFooterHtml('<span>Page number <span class="pageNumber"></span></span>')
        ->save(storage_path('/app/reports/example.pdf'));
    ```

  * **Error Handling:** In a production environment, you should add error handling (e.g., using `try...catch` blocks) to gracefully handle potential issues during PDF generation.

  * **Performance:** Generating PDFs can be resource-intensive. Be mindful of the complexity of your reports and consider background processing (e.g., using Laravel Queues) for large or frequently generated reports to avoid blocking user requests.

---

# 📡 Laravel Bulk SMS Sending (Queue + API)

This guide shows how to send SMS messages to many users (e.g., 10,000) in Laravel using queues and an SMS API.

---

## 🧑‍💻 1. Blade View (`resources/views/send-sms.blade.php`)

```blade
<!-- Simple form to input and send SMS to all users -->
<form action="{{ route('sms.send') }}" method="POST">
    @csrf
    <textarea name="message" rows="4" required placeholder="Enter SMS (max 160 characters)"></textarea>
    <button type="submit">Send SMS</button>
</form>

@if(session('success'))
    <p>{{ session('success') }}</p>
@endif
```

---

## 🌐 2. Routes (`routes/web.php`)

```php
use App\Http\Controllers\SmsController;

// Show the form
Route::get('/send-sms', fn() => view('send-sms'));

// Handle SMS sending
Route::post('/send-sms', [SmsController::class, 'send'])->name('sms.send');
```

---

## 🧾 3. Controller (`app/Http/Controllers/SmsController.php`)

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Jobs\SendSmsJob;
use App\Models\User;

class SmsController extends Controller
{
    public function send(Request $request)
    {
        $request->validate([
            'message' => 'required|string|max:160',
        ]);

        $users = User::whereNotNull('phone')->select('phone')->get();

        foreach ($users as $user) {
            SendSmsJob::dispatch($user->phone, $request->message);
        }

        return back()->with('success', 'SMS has been queued for sending!');
    }
}
```

---

## 🧵 4. Job (`app/Jobs/SendSmsJob.php`)

```php
namespace App\Jobs;

use App\Services\SmsService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SendSmsJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $phone;
    protected $message;

    public function __construct($phone, $message)
    {
        $this->phone = $phone;
        $this->message = $message;
    }

    public function handle()
    {
        SmsService::send($this->phone, $this->message);
    }
}
```
If the API had rate limiter to handle it here is the **full `handle()` method** for your `SendSmsJob`, using Laravel's `RateLimiter` to **throttle SMS sending to 100 messages per minute**.

> ✅ This version ensures you **don't exceed your SMS provider's rate limit**, and Laravel automatically handles retrying.

---

## 📄 `app/Jobs/SendSmsJob.php` – `handle()` Method (Full Code with Throttling)

```php
namespace App\Jobs;

use App\Services\SmsService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\RateLimiter;

class SendSmsJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $phone;
    protected $message;

    public function __construct($phone, $message)
    {
        $this->phone = $phone;
        $this->message = $message;
    }

    public function handle()
    {
        $rateKey = 'sms-sending'; // Unique key for rate limiting

        if (RateLimiter::tooManyAttempts($rateKey, 100)) {
            // Too many attempts: release the job to try again in 10 seconds
            return $this->release(10);
        }

        // Mark one usage of the rate limiter with a 60-second decay
        RateLimiter::hit($rateKey, 60);

        // Send the SMS via service
        SmsService::send($this->phone, $this->message);
    }
}
```

---

### 🧠 How It Works

* ✅ **Limits to 100 SMS sends per minute** globally (for the whole queue).
* ✅ If the job is too early (over the limit), it **reschedules itself** after 10 seconds.
* 🔄 This avoids hammering your SMS API and ensures smooth, consistent delivery.

---


---

Let me know if you'd like to combine this with batching logic or make it user-specific (per-phone rate limits).

---

## 📞 5. SMS API Service (`app/Services/SmsService.php`)

```php
namespace App\Services;

use Illuminate\Support\Facades\Http;

class SmsService
{
    public static function send($phone, $message)
    {
        $response = Http::post('https://api.smsprovider.com/send', [
            'to'      => $phone,
            'message' => $message,
            'api_key' => config('services.sms.api_key'),
        ]);

        if (!$response->successful()) {
            \Log::error('SMS failed for ' . $phone, $response->json());
        }
    }
}
```

---


If you want to handle **throttling in Laravel using Redis**, you’ll need to **complete a few setup steps** beyond what Laravel gives you out of the box.

Here’s a clear checklist tailored just for your use case:

---

## ✅ To Use Throttling with Redis in Laravel (for queue jobs):

### 1. **Install a Redis PHP client**

Laravel won’t work with Redis unless you install one of these:

#### ✔️ Easiest for development (Predis):

```bash
composer require predis/predis
```

> Laravel will auto-detect it.

#### OR (for production, faster performance):

```bash
sudo apt install php-redis  # Linux
brew install php-redis      # macOS
```

> Then ensure it’s enabled in `php.ini`.

---

### 2. **Install and Run Redis Server (if local)**

If Redis is not yet installed:

```bash
# Ubuntu/Debian
sudo apt install redis-server

# macOS
brew install redis
```

Then start the server:

```bash
redis-server
```

> Or use a hosted Redis like [Upstash](https://upstash.com/) or [Redis Cloud](https://redis.com/redis-cloud/) for production.

---

### 3. **Configure Laravel to Use Redis**

In your `.env` file:

```env
QUEUE_CONNECTION=redis
```

And make sure `config/queue.php` has:

```php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => 90,
    'block_for' => null,
],
```

> Laravel already includes the config — you just enable it via `.env`.

---

### 4. **Run the Queue Worker**

```bash
php artisan queue:work redis
```

> This processes queued jobs **with Redis**, allowing rate-limited logic like:

```php
if (RateLimiter::tooManyAttempts('sms-sending', 100)) {
    return $this->release(10);
}
```


---


## ⚙️ 6. .env and Config Setup

### `.env`

```env
SMS_API_KEY=your-real-api-key
QUEUE_CONNECTION=database //if not using throttle handler
```

### `config/services.php`

```php
'sms' => [
    'api_key' => env('SMS_API_KEY'),
],
```

---

## 🛠️ 7. Queue Setup

```bash
php artisan queue:table
php artisan migrate
php artisan queue:work
```

---

## ✅ Summary

| Step         | Description                      |
| ------------ | -------------------------------- |
| UI           | Form input for the message       |
| Controller   | Validates and queues SMS jobs    |
| Job          | Each job sends a message via API |
| Service      | Wraps SMS provider API logic     |
| Queue Worker | Runs jobs in background          |

---

## 🧩 Optional Enhancements
* [ ] Monitor queues using Laravel Horizon
* [ ] Add job retry/backoff for failed SMS
* [ ] Track delivery status if API supports it

---

# Simplified implementation using `Mail::queue()` for sending welcome emails, which is perfect for most standard use cases:

### 1. Create the mail class (if not already exists):
```bash
php artisan make:mail WelcomeMail --markdown=mail.welcome
```

### 2. Set up the mail class (`app/Mail/WelcomeMail.php`):
```php
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class WelcomeMail extends Mailable
{
    use Queueable, SerializesModels;

    public $user;

    public function __construct($user)
    {
        $this->user = $user;
    }

    public function build()
    {
        return $this->markdown('mail.welcome')
                    ->subject('Welcome to ' . config('app.name'));
    }
}
```

### 3. Create the markdown template (`resources/views/mail/welcome.blade.php`):
```blade
@component('mail::message')
# Welcome {{ $user->name }}!

Thank you for joining {{ config('app.name') }}. We're excited to have you on board!

@component('mail::button', ['url' => url('/dashboard')])
Access Your Dashboard
@endcomponent

Thanks,<br>
The {{ config('app.name') }} Team
@endcomponent
```

### 4. In your registration controller:
```php
use App\Mail\WelcomeMail;
use Illuminate\Support\Facades\Mail;

protected function registered(Request $request, $user)
{
    // Queue the welcome email
    Mail::to($user->email)->queue(new WelcomeMail($user));
    
    // If you want a small delay to prevent queue flooding:
    // Mail::to($user->email)->later(now()->addMinutes(5), new WelcomeMail($user));
}
```

### 5. Configure your queue in `.env`:
```env
QUEUE_CONNECTION=database # or redis, sync for local testing
```

### 6. Run the migrations for the jobs table (if using database queue):
```bash
php artisan queue:table
php artisan migrate
```

### 7. Start your queue worker:
```bash
php artisan queue:work
```

# Minify CSS,JS

## Installation

Minify for Laravel requires PHP 7.2 or higher. This particular version supports Laravel 8.x, 9.x, 10.x, 11.x, and 12.x.

To get the latest version, simply require the project using [Composer](https://getcomposer.org):

```sh
composer require fahlisaputra/laravel-minify
```
## Configuration
Minify for Laravel supports optional configuration. To get started, you'll need to publish all vendor assets:

```sh
php artisan vendor:publish --provider="Fahlisaputra\Minify\MinifyServiceProvider"
```

This will create a config/minify.php file in your app that you can modify to set your configuration. Also, make sure you check for changes to the original config file in this package between releases.

## Register the Middleware (Laravel 11 or newer)
In order Minify for Laravel can intercept your request to minify and obfuscate, you need to add the Minify middleware to the `bootstrap/app.php` file:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        \Fahlisaputra\Minify\Middleware\MinifyHtml::class,
        \Fahlisaputra\Minify\Middleware\MinifyCss::class,
        \Fahlisaputra\Minify\Middleware\MinifyJavascript::class,
    ]);
})
```
## Usage

This is how you can use Minify for Laravel in your project.

### Minify Asset Files
You must set `true` on `assets_enabled` in the `config/minify.php` file to minify your asset files. 
If the option set to `false` the route will not registered from service provider. For example:

```php
"assets_enabled" => env("MINIFY_ASSETS_ENABLED", true),
```

You can minify your asset files by using the `minify()` helper function. This function will minify your asset files and return the minify designed route. For example:

```html
<link rel="stylesheet" href="{{ minify('/css/test.css') }}">
```

```html
<script src="{{ minify('/js/test.js') }}"></script>
```

You can modify the assets storage directory path by setting `assets_path` in the `config/minify.php` file. By default, the assets storage directory path is `resources`.
**So you can just copy and paste the JS,css files in resources/css or resources/js**   For example:

```php
"assets_storage" => env("MINIFY_ASSETS_STORAGE", 'resources'),
```

In order to minimize the security risk, the root storage directory is hidden from the public. For the example, you set the `assets_storage` to `storage/app/private/assets` and you want to access the file `test.css` in the `storage/app/private/assets/test.css`. You can use the `minify()` helper function like this:

```html
<link rel="stylesheet" href="{{ minify('test.css') }}">
```

The `minify()` helper function will automatically search the file in the `storage/app/private/assets` directory. The result on the browser will be like this:

```html
<link rel="stylesheet" href="{minify_route_path}/test.css">
```

The `/storage/app/private/assets` directory is hidden from the public.
