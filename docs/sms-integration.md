# SMS Integration

Text Commander uses a two-layer architecture for SMS delivery powered by two Laravel packages.

## Core Packages

1. **lbhurtado/sms** - SMS abstraction layer with driver support
2. **lbhurtado/engagespark** - engageSPARK platform integration

For detailed architecture documentation, see [Package Architecture](package-architecture.md).

## SMS Facade Usage

The primary way to send SMS in Text Commander is through the SMS facade.

### Basic Usage

```php
use LBHurtado\SMS\Facades\SMS;

SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Your message')
    ->send();
```

### With Airtime Topup

```php
SMS::channel('engagespark')
    ->from('QUEZON_CITY')
    ->to('+639171234567')
    ->content('Thank you! Here\'s P25 load.')
    ->send()
    ->topup(25);
```

### Broadcast to Multiple Recipients

```php
$recipients = ['+639171234567', '+639181234567', '+639191234567'];

foreach ($recipients as $mobile) {
    SMS::channel('engagespark')
        ->from('QUEZON_CITY')
        ->to($mobile)
        ->content('Emergency alert')
        ->send();
}
```

## Driver Architecture

The SMS facade supports multiple drivers:

- **engagespark** - Primary driver (branded sender IDs)
- **nexmo** - Fallback driver (Vonage/Nexmo)
- **null** - Testing driver (no actual sending)

### Switch Drivers at Runtime

```php
// Use engageSPARK
SMS::channel('engagespark')->from('TXTCMDR')->to($mobile)->content($message)->send();

// Fall back to Nexmo if needed
SMS::channel('nexmo')->from('+639171111111')->to($mobile)->content($message)->send();
```

## Integration with engageSPARK

engageSPARK provides:
- **Branded Sender IDs** - Pre-approved sender names (e.g., "QUEZON_CITY")
- **Airtime Topups** - Incentivize recipients with mobile load
- **Delivery Reports** - Real-time status via webhooks
- **Two-way SMS** - Handle incoming replies

### Configuration

```dotenv
ENGAGESPARK_API_KEY=your_api_key_here
ENGAGESPARK_ORGANIZATION_ID=your_org_id
ENGAGESPARK_SENDER_ID=TXTCMDR
ENGAGESPARK_SMS_WEBHOOK=https://yourapp.com/webhooks/engagespark/sms
ENGAGESPARK_AIRTIME_WEBHOOK=https://yourapp.com/webhooks/engagespark/airtime
```

## Queue Jobs

### SendSMSJob

Queued job for sending individual SMS messages.

**Location:** `app/Jobs/SendSMSJob.php`

**Properties:**
- `$mobile` - Recipient phone number
- `$message` - SMS content
- `$senderId` - Branded sender ID

**Usage:**
```php
SendSMSJob::dispatch(
    mobile: '+639171234567',
    message: 'Your message',
    senderId: 'TXTCMDR'
);
```

**Implementation:**
```php
namespace App\Jobs;

use LBHurtado\SMS\Facades\SMS;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendSMSJob implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public function __construct(
        public string $mobile,
        public string $message,
        public string $senderId
    ) {}

    public function handle()
    {
        SMS::channel('engagespark')
            ->from($this->senderId)
            ->to($this->mobile)
            ->content($this->message)
            ->send();
    }
}
```

### BroadcastToGroupJob

Queued job for broadcasting to all contacts in a group.

**Location:** `app/Jobs/BroadcastToGroupJob.php`

**Properties:**
- `$groupId` - Group ID
- `$message` - SMS content
- `$senderId` - Branded sender ID

**Implementation:**
```php
namespace App\Jobs;

use App\Models\Group;
use LBHurtado\SMS\Facades\SMS;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class BroadcastToGroupJob implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public function __construct(
        public int $groupId,
        public string $message,
        public string $senderId
    ) {}

    public function handle()
    {
        $group = Group::with('contacts')->findOrFail($this->groupId);

        foreach ($group->contacts as $contact) {
            SendSMSJob::dispatch(
                $contact->mobile,
                $this->message,
                $this->senderId
            );
        }
    }
}
```

## Event Listeners

### LogSMSSent

Logs SMS sending events for analytics and debugging.

**Location:** `app/Listeners/LogSMSSent.php`

**Listens To:** `LBHurtado\EngageSpark\Events\MessageSent`

**Implementation:**
```php
namespace App\Listeners;

use LBHurtado\EngageSpark\Events\MessageSent;
use Illuminate\Support\Facades\Log;

class LogSMSSent
{
    public function handle(MessageSent $event)
    {
        Log::info('SMS sent via engageSPARK', [
            'recipient' => $event->recipient,
            'sender' => $event->sender,
            'message' => $event->message,
            'timestamp' => now(),
        ]);
    }
}
```

### UpdateDeliveryStatus

Updates message delivery status in database.

**Location:** `app/Listeners/UpdateDeliveryStatus.php`

**Listens To:** Webhook events

## Webhook Handling

Handle delivery reports from engageSPARK.

### HandleEngageSparkWebhook Action

**Location:** `app/Actions/HandleEngageSparkWebhook.php`

**Endpoints:**
- `POST /webhooks/engagespark/sms` - SMS delivery status
- `POST /webhooks/engagespark/airtime` - Airtime transfer status

**Payload Example:**
```json
{
  "organizationId": "12345",
  "recipientPhoneNumber": "+639171234567",
  "status": "delivered",
  "messageId": "msg_abc123",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Status Values:**
- `queued` - Message queued for delivery
- `sent` - Message sent to carrier
- `delivered` - Confirmed delivery
- `failed` - Delivery failed

## Testing

Use the `null` driver for testing without sending real SMS:

```php
// In tests
config(['sms.default' => 'null']);

SMS::channel('null')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Test message')
    ->send(); // No SMS sent
```

## Related Documentation

- [Package Architecture](package-architecture.md) - Detailed package structure
- [Quick Start](quick-start.md) - 5-minute setup guide
- [Backend Services](backend-services.md) - Actions and Controllers
- [API Documentation](api-documentation.md) - HTTP API endpoints
