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
