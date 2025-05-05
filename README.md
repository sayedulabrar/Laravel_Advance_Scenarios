Here is the entire Laravel SMS-sending process written in **Markdown format** — ideal for documentation, GitHub README, or team sharing.

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
Absolutely! Here's the **full `handle()` method** for your `SendSmsJob`, using Laravel's `RateLimiter` to **throttle SMS sending to 100 messages per minute**.

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

## ⚙️ 6. .env and Config Setup

### `.env`

```env
SMS_API_KEY=your-real-api-key
QUEUE_CONNECTION=redis
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

Let me know if you want this as a downloadable `.md` file!
