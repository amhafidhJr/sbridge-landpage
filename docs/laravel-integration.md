# SMS Bridge Gateway - Laravel Integration Guide

## Overview
SMS Bridge Gateway allows your Laravel backend to send SMS messages through an Android device running the SMS Bridge App.  
This guide provides a complete setup: database models, controllers, routes, and integration steps.

---

## Requirements
- Laravel 8+  
- PHP 8.0+  
- MySQL/PostgreSQL database  
- Android device with SMS Bridge App installed  

---

## Step 1: Database Models

### 1.1 SMS Model
Create a model `Sms`:

```php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Sms extends Model
{
    protected $fillable = [
        'sms_status',
        'message', 
        'error_message',
        'sent_at',
        'delivered_at',
        'user_phone_number_id'
    ];

    const STATUS_QUEUED = 'QUEUED';
    const STATUS_PENDING = 'PENDING';
    const STATUS_SENT = 'SENT';
    const STATUS_DELIVERED = 'DELIVERED';
    const STATUS_FAILED = 'FAILED';

    public static function getValidStatuses(): array
    {
        return [
            self::STATUS_QUEUED,
            self::STATUS_PENDING,
            self::STATUS_SENT,
            self::STATUS_DELIVERED,
            self::STATUS_FAILED,
        ];
    }

    public function recipientNumber()
    {
        return $this->belongsTo(UserPhoneNumber::class, 'user_phone_number_id');
    }
}
```

### 1.2 UserPhoneNumber Model
Create a model `UserPhoneNumber`:

```php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class UserPhoneNumber extends Model
{
    protected $fillable = ['phone_number'];

    public function sms()
    {
        return $this->hasMany(Sms::class, 'user_phone_number_id');
    }
}
```

---

## Step 2: Controller

Create a controller `SmsBridgeController`:

```php
<?php
namespace App\Http\Controllers;

use App\Models\Sms;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class SmsBridgeController extends Controller
{
    public function getPendingSms()
    {
        $pendingSms = Sms::with('recipientNumber')
            ->whereIn('sms_status', ['QUEUED','FAILED'])
            ->orderBy('id','asc')
            ->take(50)
            ->get();

        return response()->json($pendingSms->map(fn($sms) => [
            'id' => $sms->id,
            'userId' => $sms->user_phone_number_id,
            'recipientNumber' => $sms->recipientNumber->phone_number ?? null,
            'message' => $sms->message,
            'status' => $sms->sms_status,
            'sentAt' => $sms->sent_at,
            'deliveredAt' => $sms->delivered_at,
        ]));
    }

    public function updateStatus(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'message_id' => 'required|string',
            'status' => 'required|string|in:' . implode(',', Sms::getValidStatuses())
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Invalid request data',
                'errors' => $validator->errors()
            ], 422);
        }

        $sms = Sms::find($request->message_id);
        if (!$sms) return response()->json(['success'=>false,'message'=>'SMS record not found'],404);

        $sms->sms_status = $request->status;
        if ($request->status === Sms::STATUS_SENT) $sms->sent_at = now();
        if ($request->status === Sms::STATUS_DELIVERED) $sms->delivered_at = now();
        if ($request->status === Sms::STATUS_FAILED && $request->has('error_message')) $sms->error_message = $request->error_message;

        $sms->save();

        return response()->json([
            'success' => true,
            'message' => 'Status updated successfully',
            'data' => [
                'message_id'=>$sms->id,
                'status'=>$sms->sms_status,
                'sent_at'=>$sms->sent_at,
                'delivered_at'=>$sms->delivered_at
            ]
        ]);
    }

    public function testConnection()
    {
        return response()->json(['status'=>'ok','message'=>'Connection successful']);
    }
}
```

---

## Step 3: Routes

Add API routes in `routes/web.php` or `routes/api.php`:

```php
use App\Http\Controllers\SmsBridgeController;

Route::prefix('api/v1/sms-bridge')->group(function () {
    Route::get('/pending-sms', [SmsBridgeController::class, 'getPendingSms']);
    Route::post('/update-status', [SmsBridgeController::class, 'updateStatus']);
});

Route::get('/test-connection', [SmsBridgeController::class, 'testConnection']);
```

---

## Step 4: Integration Notes

* The Android app calls `/pending-sms` to fetch messages with status `QUEUED` or `FAILED`.
* After sending, the app updates status using `/update-status`.
* Limit messages fetched per request (e.g., 50) to prevent overloading the device.
* Use a constant `BASE_URL` in the Android app for API calls.
* Handle `FAILED` messages carefully and log errors.

---

## Optional Enhancements

* Add authentication or API tokens for secure endpoints.
* Use queues or cron jobs for large-scale message sending.
* Log all requests for monitoring and debugging.
* Extend models with more metadata (e.g., sender name, message type).

---

Your Laravel backend is now fully integrated with the SMS Bridge App. You can fetch pending messages, update delivery status, and manage SMS sending seamlessly.