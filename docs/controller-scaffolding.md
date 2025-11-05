# Controller Scaffolding

This document provides complete scaffolding for all Laravel controllers, including Inertia controllers for page rendering and API Actions for AJAX requests.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Inertia Controllers](#inertia-controllers)
- [API Actions](#api-actions)
- [Routes](#routes)
- [Middleware](#middleware)

---

## Architecture Overview

### Two Controller Types

**1. Inertia Controllers** - Render Vue pages with SSR data
- Location: `app/Http/Controllers/`
- Return: `Inertia::render('page-name', $props)`
- Used for: Initial page loads, navigation
- Routes: `routes/web.php`

**2. API Actions** - Handle AJAX/fetch requests
- Location: `app/Actions/`
- Return: JSON responses via `ActionRequest`
- Used for: Form submissions, data fetching, mutations
- Routes: `routes/api.php`
- Already documented in: [Backend Services](backend-services.md)

### Rationalization

```
┌─────────────────────────────────────────────────────────────┐
│                     Browser / Vue App                        │
└─────────────────────────────────────────────────────────────┘
                     │                    │
        Initial Load │                    │ AJAX/Fetch
                     ▼                    ▼
         ┌───────────────────┐  ┌──────────────────┐
         │ Inertia Controller│  │   API Actions    │
         │  (web.php)        │  │  (api.php)       │
         │                   │  │                  │
         │ Returns:          │  │ Returns:         │
         │ Inertia::render() │  │ JSON response    │
         └───────────────────┘  └──────────────────┘
```

**Why separate?**

- **Inertia Controllers**: Handle page-level concerns (auth, permissions, shared data)
- **API Actions**: Handle business logic, validation, mutations (already have `asController`)
- **Clean separation**: Page rendering vs data operations

---

## Inertia Controllers

### `app/Http/Controllers/DashboardController.php`

**Route:** `GET /`

**Purpose:** Render SMS composer dashboard with initial data

```php
namespace App\Http\Controllers;

use App\Models\Contact;
use App\Models\Group;
use App\Models\SenderID;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class DashboardController extends Controller
{
    public function __invoke(Request $request): Response
    {
        return Inertia::render('dashboard', [
            'contacts' => Contact::orderBy('created_at', 'desc')
                ->take(10)
                ->get(),
            'groups' => Group::withCount('contacts')
                ->orderBy('name')
                ->get(),
            'senderIds' => SenderID::orderBy('is_default', 'desc')
                ->orderBy('name')
                ->get(),
            // Pre-fill recipient if passed via query string
            'prefilledRecipient' => $request->query('recipient'),
        ]);
    }
}
```

---

### `app/Http/Controllers/Auth/LoginController.php`

**Routes:** 
- `GET /login` - Show login form
- `POST /login` - Handle login

```php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Inertia\Inertia;
use Inertia\Response;

class LoginController extends Controller
{
    /**
     * Show login form
     */
    public function showLoginForm(): Response
    {
        return Inertia::render('auth/login');
    }

    /**
     * Handle login attempt
     */
    public function login(Request $request): RedirectResponse
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $remember = $request->boolean('remember');

        if (Auth::attempt($credentials, $remember)) {
            $request->session()->regenerate();

            return redirect()->intended('/');
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ])->onlyInput('email');
    }

    /**
     * Handle logout
     */
    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/login');
    }
}
```

---

### `app/Http/Controllers/ContactController.php`

**Route:** `GET /contacts`

**Purpose:** Show contacts and groups management page

```php
namespace App\Http\Controllers;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class ContactController extends Controller
{
    public function index(Request $request): Response
    {
        $query = $request->query('search');

        $contacts = Contact::query()
            ->when($query, function ($q) use ($query) {
                $q->where('mobile', 'like', "%{$query}%")
                    ->orWhereRaw("extra_attributes->>'name' ILIKE ?", ["%{$query}%"]);
            })
            ->orderBy('created_at', 'desc')
            ->paginate(20);

        $groups = Group::withCount('contacts')
            ->when($query, function ($q) use ($query) {
                $q->where('name', 'like', "%{$query}%");
            })
            ->orderBy('name')
            ->get();

        return Inertia::render('contacts/index', [
            'contacts' => $contacts,
            'groups' => $groups,
            'search' => $query,
        ]);
    }

    public function edit(int $id): Response
    {
        $contact = Contact::findOrFail($id);

        return Inertia::render('contacts/edit', [
            'contact' => $contact,
            'groups' => Group::orderBy('name')->get(),
        ]);
    }
}
```

---

### `app/Http/Controllers/LogController.php`

**Route:** `GET /logs`

**Purpose:** Show message history with filters

```php
namespace App\Http\Controllers;

use App\Models\ScheduledMessage;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class LogController extends Controller
{
    public function index(Request $request): Response
    {
        $status = $request->query('status', 'all');
        $date = $request->query('date', 'all');

        $query = ScheduledMessage::query()
            ->orderBy('scheduled_at', 'desc');

        // Status filter
        if ($status !== 'all') {
            $query->where('status', $status);
        }

        // Date filter
        if ($date === 'today') {
            $query->whereDate('scheduled_at', today());
        } elseif ($date === 'week') {
            $query->whereBetween('scheduled_at', [now()->startOfWeek(), now()->endOfWeek()]);
        } elseif ($date === 'month') {
            $query->whereBetween('scheduled_at', [now()->startOfMonth(), now()->endOfMonth()]);
        }

        return Inertia::render('logs/index', [
            'messages' => $query->paginate(20),
            'filters' => [
                'status' => $status,
                'date' => $date,
            ],
        ]);
    }
}
```

---

### `app/Http/Controllers/SettingsController.php`

**Route:** `GET /settings`

**Purpose:** Show settings page with account and gateway info

```php
namespace App\Http\Controllers;

use App\Models\SenderID;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Inertia\Inertia;
use Inertia\Response;

class SettingsController extends Controller
{
    public function index(): Response
    {
        return Inertia::render('settings/index', [
            'user' => Auth::user(),
            'senderIds' => SenderID::orderBy('is_default', 'desc')
                ->orderBy('name')
                ->get(),
            'gateway' => [
                'provider' => config('sms.default_driver', 'engagespark'),
                'balance' => $this->getGatewayBalance(),
            ],
        ]);
    }

    /**
     * Get current SMS gateway balance
     */
    protected function getGatewayBalance(): float
    {
        // TODO: Implement actual balance check via engageSPARK API
        // For now, return mock data
        return 1234.56;
    }
}
```

---

## API Actions

**Note:** These are already documented in [Backend Services](backend-services.md). Listed here for completeness.

### SMS Broadcasting
- `POST /api/send` - `SendToMultipleRecipients`
- `POST /api/groups/send` - `SendToMultipleGroups`
- `POST /api/send/schedule` - `ScheduleMessage`

### Group Management
- `GET /api/groups` - `ListGroups`
- `POST /api/groups` - `CreateGroup`
- `GET /api/groups/{id}` - `GetGroup`
- `PUT /api/groups/{id}` - `UpdateGroup`
- `DELETE /api/groups/{id}` - `DeleteGroup`

### Contact Management
- `GET /api/groups/{id}/contacts` - `ListGroupContacts`
- `POST /api/groups/{id}/contacts` - `AddContactToGroup`
- `PUT /api/groups/{group_id}/contacts/{contact_id}` - `UpdateContactInGroup`
- `DELETE /api/groups/{group_id}/contacts/{contact_id}` - `DeleteContactFromGroup`

### Scheduled Messages
- `GET /api/scheduled-messages` - `ListScheduledMessages`
- `PUT /api/scheduled-messages/{id}` - `UpdateScheduledMessage`
- `POST /api/scheduled-messages/{id}/cancel` - `CancelScheduledMessage`

---

## Routes

### `routes/web.php`

```php
use App\Http\Controllers\Auth\LoginController;
use App\Http\Controllers\ContactController;
use App\Http\Controllers\DashboardController;
use App\Http\Controllers\LogController;
use App\Http\Controllers\SettingsController;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Web Routes (Inertia)
|--------------------------------------------------------------------------
|
| These routes return Inertia responses for initial page loads.
| All mutations should use API routes (routes/api.php) instead.
|
*/

// Authentication
Route::get('login', [LoginController::class, 'showLoginForm'])
    ->name('login')
    ->middleware('guest');

Route::post('login', [LoginController::class, 'login'])
    ->middleware('guest');

Route::post('logout', [LoginController::class, 'logout'])
    ->name('logout')
    ->middleware('auth');

// Authenticated Pages
Route::middleware(['auth'])->group(function () {
    // Dashboard (SMS Composer)
    Route::get('/', DashboardController::class)->name('dashboard');

    // Contacts
    Route::get('/contacts', [ContactController::class, 'index'])->name('contacts.index');
    Route::get('/contacts/{id}/edit', [ContactController::class, 'edit'])->name('contacts.edit');

    // Logs
    Route::get('/logs', [LogController::class, 'index'])->name('logs.index');

    // Settings
    Route::get('/settings', [SettingsController::class, 'index'])->name('settings.index');
});
```

### `routes/api.php`

```php
use App\Actions\CancelScheduledMessage;
use App\Actions\Contacts\AddContactToGroup;
use App\Actions\Contacts\DeleteContactFromGroup;
use App\Actions\Contacts\ListGroupContacts;
use App\Actions\Contacts\UpdateContactInGroup;
use App\Actions\Groups\CreateGroup;
use App\Actions\Groups\DeleteGroup;
use App\Actions\Groups\GetGroup;
use App\Actions\Groups\ListGroups;
use App\Actions\Groups\UpdateGroup;
use App\Actions\ListScheduledMessages;
use App\Actions\ScheduleMessage;
use App\Actions\SendToMultipleGroups;
use App\Actions\SendToMultipleRecipients;
use App\Actions\UpdateScheduledMessage;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| API Routes (Actions)
|--------------------------------------------------------------------------
|
| These routes handle AJAX/fetch requests from the Vue frontend.
| All return JSON responses via ActionRequest.
|
*/

Route::middleware(['auth:sanctum'])->group(function () {
    // SMS Broadcasting
    Route::post('/send', SendToMultipleRecipients::class);
    Route::post('/groups/send', SendToMultipleGroups::class);
    Route::post('/send/schedule', ScheduleMessage::class);

    // Group Management
    Route::get('/groups', ListGroups::class);
    Route::post('/groups', CreateGroup::class);
    Route::get('/groups/{id}', GetGroup::class);
    Route::put('/groups/{id}', UpdateGroup::class);
    Route::delete('/groups/{id}', DeleteGroup::class);

    // Contact Management (within groups)
    Route::get('/groups/{id}/contacts', ListGroupContacts::class);
    Route::post('/groups/{id}/contacts', AddContactToGroup::class);
    Route::put('/groups/{group_id}/contacts/{contact_id}', UpdateContactInGroup::class);
    Route::delete('/groups/{group_id}/contacts/{contact_id}', DeleteContactFromGroup::class);

    // Scheduled Messages
    Route::get('/scheduled-messages', ListScheduledMessages::class);
    Route::put('/scheduled-messages/{id}', UpdateScheduledMessage::class);
    Route::post('/scheduled-messages/{id}/cancel', CancelScheduledMessage::class);
});
```

---

## Middleware

### `app/Http/Middleware/HandleInertiaRequests.php`

**Purpose:** Share global data with all Inertia pages

```php
namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    /**
     * The root template that's loaded on the first page visit.
     */
    protected $rootView = 'app';

    /**
     * Determines the current asset version.
     */
    public function version(Request $request): ?string
    {
        return parent::version($request);
    }

    /**
     * Defines the props that are shared by default.
     */
    public function share(Request $request): array
    {
        return array_merge(parent::share($request), [
            // Authentication
            'auth' => [
                'user' => $request->user() ? [
                    'id' => $request->user()->id,
                    'name' => $request->user()->name,
                    'email' => $request->user()->email,
                    'role' => $request->user()->role ?? 'admin',
                ] : null,
            ],

            // Flash messages
            'flash' => [
                'success' => fn () => $request->session()->get('success'),
                'error' => fn () => $request->session()->get('error'),
                'info' => fn () => $request->session()->get('info'),
            ],

            // Config
            'appName' => config('app.name', 'Text Commander'),
        ]);
    }
}
```

**Register in `app/Http/Kernel.php`:**

```php
protected $middlewareGroups = [
    'web' => [
        // ... other middleware
        \App\Http\Middleware\HandleInertiaRequests::class,
    ],
];
```

---

## Additional Controllers (Optional)

### `app/Http/Controllers/GroupController.php`

**Routes:** `GET /groups/{id}`, `GET /groups/{id}/edit`

```php
namespace App\Http\Controllers;

use App\Models\Group;
use Inertia\Inertia;
use Inertia\Response;

class GroupController extends Controller
{
    public function show(int $id): Response
    {
        $group = Group::with('contacts')
            ->withCount('contacts')
            ->findOrFail($id);

        return Inertia::render('groups/show', [
            'group' => $group,
        ]);
    }

    public function edit(int $id): Response
    {
        $group = Group::findOrFail($id);

        return Inertia::render('groups/edit', [
            'group' => $group,
        ]);
    }
}
```

### `app/Http/Controllers/ScheduledMessageController.php`

**Route:** `GET /scheduled-messages/{id}/edit`

```php
namespace App\Http\Controllers;

use App\Models\ScheduledMessage;
use Inertia\Inertia;
use Inertia\Response;

class ScheduledMessageController extends Controller
{
    public function edit(int $id): Response
    {
        $message = ScheduledMessage::findOrFail($id);

        // Only allow editing pending messages
        if (!$message->isEditable()) {
            abort(403, 'Cannot edit this message');
        }

        return Inertia::render('scheduled-messages/edit', [
            'message' => $message,
        ]);
    }
}
```

---

## Testing

### Feature Test: `tests/Feature/DashboardControllerTest.php`

```php
namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class DashboardControllerTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function it_shows_dashboard_to_authenticated_users()
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->get('/');

        $response->assertStatus(200);
        $response->assertInertia(fn ($page) => 
            $page->component('dashboard')
                ->has('contacts')
                ->has('groups')
                ->has('senderIds')
        );
    }

    /** @test */
    public function it_redirects_guests_to_login()
    {
        $response = $this->get('/');

        $response->assertRedirect('/login');
    }
}
```

### Feature Test: `tests/Feature/LoginControllerTest.php`

```php
namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Hash;
use Tests\TestCase;

class LoginControllerTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function it_shows_login_form()
    {
        $response = $this->get('/login');

        $response->assertStatus(200);
        $response->assertInertia(fn ($page) => $page->component('auth/login'));
    }

    /** @test */
    public function it_authenticates_user_with_valid_credentials()
    {
        $user = User::factory()->create([
            'email' => 'admin@txtcmdr.local',
            'password' => Hash::make('password'),
        ]);

        $response = $this->post('/login', [
            'email' => 'admin@txtcmdr.local',
            'password' => 'password',
        ]);

        $response->assertRedirect('/');
        $this->assertAuthenticatedAs($user);
    }

    /** @test */
    public function it_rejects_invalid_credentials()
    {
        User::factory()->create([
            'email' => 'admin@txtcmdr.local',
            'password' => Hash::make('password'),
        ]);

        $response = $this->post('/login', [
            'email' => 'admin@txtcmdr.local',
            'password' => 'wrong-password',
        ]);

        $response->assertSessionHasErrors('email');
        $this->assertGuest();
    }
}
```

---

## Summary

### Inertia Controllers (5 controllers)
1. **DashboardController** - SMS composer with initial data
2. **LoginController** - Authentication (show form, login, logout)
3. **ContactController** - Contacts and groups listing
4. **LogController** - Message history with filters
5. **SettingsController** - Settings page with account info

### API Actions (14 endpoints)
- All use `ActionRequest` with `asController()` method
- Return JSON responses
- Handle validation via `rules()` method
- Already documented in [Backend Services](backend-services.md)

### Key Principles
- ✅ **Inertia for pages**: Initial loads, SSR data
- ✅ **API for mutations**: AJAX/fetch requests
- ✅ **Clean separation**: Page concerns vs business logic
- ✅ **Shared data**: Via `HandleInertiaRequests` middleware
- ✅ **Type-safe**: Both controller types return predictable data structures

---

## Related Documentation

- [Frontend Scaffolding](frontend-scaffolding.md) - Vue pages and components
- [Backend Services](backend-services.md) - API Actions with `asController()`
- [UI/UX Design](ui-ux-design.md) - Design specifications
- [Development Plan](development-plan.md) - Implementation roadmap
