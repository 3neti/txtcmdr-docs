# UI/UX Design

## Design Philosophy

Text Commander's interface is designed like a **mobile SMS app** - simple, focused, and familiar. The primary action (sending SMS) is front and center, with management features accessible but not intrusive.

## Authentication

### No Sign-Up Page

**Initial Setup:**
```bash
# Option 1: Artisan command (recommended)
php artisan txtcmdr:create-admin
# Prompts for email and password

# Option 2: Database seeder
php artisan db:seed --class=AdminUserSeeder
# Creates default admin (can change password after login)
```

**Seeder Implementation:**
```php
// database/seeders/AdminUserSeeder.php
namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class AdminUserSeeder extends Seeder
{
    public function run()
    {
        User::firstOrCreate(
            ['email' => 'admin@txtcmdr.local'],
            [
                'name' => 'Admin',
                'password' => Hash::make('password'),
                'role' => 'admin',
            ]
        );
    }
}
```

**Single User System:**
- Only one user account at a time
- Simple password change in settings
- No user management complexity
- Focus on SMS sending, not user administration

### Login Screen

**Minimal and Clean:**
```
┌────────────────────────────────┐
│                                │
│        Text Commander          │
│                                │
│    ┌────────────────────┐     │
│    │ Email              │     │
│    └────────────────────┘     │
│                                │
│    ┌────────────────────┐     │
│    │ Password           │     │
│    └────────────────────┘     │
│                                │
│    [ Login ]                   │
│                                │
└────────────────────────────────┘
```

**Features:**
- Email + password only
- "Remember me" checkbox
- No "Forgot password" (use artisan command to reset)
- Clean, centered design

---

## Main Dashboard: SMS Composer (Primary View)

### Design Principle
**"Send First, Manage Later"**

The dashboard IS the SMS composer - like opening your phone's Messages app. Everything else is secondary.

### Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Text Commander    [Contacts] [Logs] [Settings] [@admin ▼] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Send SMS                                           📤      │
│  ─────────────────────────────────────────────────────     │
│                                                             │
│  To:  [ 0917 123 4567, Health Workers, Juan...    ▼ ]     │
│                                                             │
│  ┌───────────────────────────────────────────────────┐    │
│  │                                                   │    │
│  │  Type your message here...                        │    │
│  │                                                   │    │
│  │                                                   │    │
│  │                                                   │    │
│  │                                                   │    │
│  └───────────────────────────────────────────────────┘    │
│                                                             │
│  160/160 characters (1 SMS)        From: [QUEZON_CITY ▼]  │
│                                                             │
│                    [ Send Now ]  [ Schedule ]              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Features

#### 1. Smart "To:" Field
**Multi-input with chips/tags:**
```
┌────────────────────────────────────────────────┐
│ [× 0917 123 4567] [× Health Workers] [× Juan] │
│ Type number, contact, or group...              │
└────────────────────────────────────────────────┘
```

**Supports:**
- Direct phone numbers: `0917 123 4567`, `+63 917 123 4567`
- Contact names: `Juan`, `Maria`
- Group names: `Health Workers`, `Barangay Leaders`
- Multiple recipients (comma-separated or tagged)

**Autocomplete dropdown:**
```
┌────────────────────────────────┐
│ 📱 0917 123 4567              │
│ 👤 Juan Dela Cruz             │
│    0917 123 4567              │
│ 👥 Health Workers (25)        │
│ 👥 Barangay Leaders (50)      │
└────────────────────────────────┘
```

#### 2. Message Composer
- Large textarea (like mobile SMS)
- Auto-expanding height
- Character counter: `160/160 (1 SMS)`, `320/320 (2 SMS)`
- Visual feedback when exceeding single SMS

#### 3. Sender ID Selector
```
From: [ QUEZON_CITY ▼ ]
      [ TXTCMDR ▼ ]
      [ (Add new...) ]
```

#### 4. Send Actions
- **Send Now** - Primary button, blue, prominent
- **Schedule** - Secondary button, opens datetime picker

---

## Secondary Screens (Accessible via Top Nav)

### Navigation Menu
```
┌─────────────────────────────────────────────────────┐
│  📤 Text Commander  [Contacts] [Logs] [⚙️ Settings] │
└─────────────────────────────────────────────────────┘
```

**Always visible:**
- Logo/Brand (clickable → back to composer)
- Contacts link
- Logs link
- Settings link
- User dropdown (logout, change password)

---

### Contacts Screen

**Simplified Management:**

```
┌─────────────────────────────────────────────────────────┐
│  ← Back to SMS                                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Contacts                      [+ Add] [Import CSV]    │
│                                                         │
│  Search: [________________]                             │
│                                                         │
│  Groups                                                 │
│  ├─ 👥 Health Workers (25)         [Send] [Edit]      │
│  ├─ 👥 Barangay Leaders (50)       [Send] [Edit]      │
│  └─ 👥 Volunteers (12)             [Send] [Edit]      │
│                                                         │
│  Recent Contacts                                        │
│  ├─ 👤 Juan Dela Cruz (0917...)   [Send] [Edit]      │
│  ├─ 👤 Maria Santos (0918...)     [Send] [Edit]      │
│  └─ 👤 Pedro Garcia (0919...)     [Send] [Edit]      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Features:**
- Quick "Send" button next to each contact/group
- Simple list view (no complex tables)
- Groups shown prominently with member count
- Search filters everything (groups + contacts)

---

### Logs Screen

**SMS History:**

```
┌─────────────────────────────────────────────────────────┐
│  ← Back to SMS                                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Message Logs                 Filter: [All ▼] [Today]  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │ ✓ Sent to Health Workers (25 recipients)         │ │
│  │   "Please attend the meeting..."                 │ │
│  │   Jan 15, 2024 10:30 AM · From: QUEZON_CITY     │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │ ✓ Sent to +639171234567                          │ │
│  │   "Hello! This is a test message"                │ │
│  │   Jan 15, 2024 09:15 AM · From: TXTCMDR          │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │ ⏱️ Scheduled for Jan 16, 2024 8:00 AM             │ │
│  │   To: Barangay Leaders (50)                       │ │
│  │   "Reminder: Town hall meeting..."                │ │
│  │   [Cancel] [Edit]                                 │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Status Icons:**
- ✓ Delivered
- ⏱️ Scheduled
- 📤 Sending
- ❌ Failed

---

### Settings Screen

**Simple Configuration:**

```
┌─────────────────────────────────────────────────────────┐
│  ← Back to SMS                                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Settings                                               │
│                                                         │
│  Account                                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Email: admin@txtcmdr.local                      │   │
│  │ [Change Password]                               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Sender IDs                                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │ • QUEZON_CITY (default)                         │   │
│  │ • TXTCMDR                                        │   │
│  │ [+ Add New Sender ID]                           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  SMS Gateway                                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Provider: engageSPARK                           │   │
│  │ Status: ✓ Connected                             │   │
│  │ Balance: ₱1,234.56                              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Design System

### Color Scheme

**Primary Colors:**
```
Primary Blue:   #2563EB (buttons, links)
Dark Blue:      #1E40AF (headings)
Light Blue:     #DBEAFE (backgrounds, highlights)
White:          #FFFFFF (main background)
Gray:           #6B7280 (secondary text)
```

**Status Colors:**
```
Success:  #10B981 (✓ delivered)
Warning:  #F59E0B (⏱️ scheduled)
Error:    #EF4444 (❌ failed)
Info:     #3B82F6 (📤 sending)
```

### Typography

**Font Family:**
- System fonts: `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto`
- Optimized for readability

**Sizes:**
```
Heading:  24px, Bold
Body:     16px, Regular
Small:    14px, Regular
Caption:  12px, Regular
```

### Components

#### Buttons

**Primary:**
```css
background: #2563EB
color: white
padding: 12px 24px
border-radius: 8px
font-weight: 600
```

**Secondary:**
```css
background: white
color: #2563EB
border: 2px solid #2563EB
padding: 12px 24px
border-radius: 8px
```

#### Input Fields
```css
border: 2px solid #E5E7EB
border-radius: 8px
padding: 12px
focus: border-color #2563EB
```

#### Cards/Containers
```css
background: white
border: 1px solid #E5E7EB
border-radius: 12px
padding: 20px
box-shadow: 0 1px 3px rgba(0,0,0,0.1)
```

---

## Mobile Responsiveness

### Breakpoints
```
Mobile:  < 640px
Tablet:  640px - 1024px
Desktop: > 1024px
```

### Mobile Layout

**Stack vertically:**
```
┌─────────────────────┐
│ Text Commander  ☰  │
├─────────────────────┤
│                     │
│ To: [           ]   │
│                     │
│ ┌─────────────────┐ │
│ │ Message...      │ │
│ │                 │ │
│ │                 │ │
│ └─────────────────┘ │
│                     │
│ From: [QUEZON... ▼] │
│                     │
│ [   Send Now   ]    │
│ [   Schedule   ]    │
│                     │
└─────────────────────┘
```

**Bottom navigation on mobile:**
```
┌─────────────────────┐
│                     │
│   (Main Content)    │
│                     │
├─────────────────────┤
│ 📤 Send | 👥 | 📋 │
└─────────────────────┘
```

---

## User Flows

### Flow 1: Send SMS to Group
```
1. Land on Dashboard (SMS Composer)
2. Click "To:" field
3. Type "Health" → Autocomplete shows "Health Workers"
4. Select "Health Workers (25)"
5. Type message
6. Click "Send Now"
7. Toast notification: "✓ Sending to 25 recipients"
8. Stay on composer (cleared for next message)
```

### Flow 2: Schedule Broadcast
```
1. Land on Dashboard
2. Add recipients (group or numbers)
3. Type message
4. Click "Schedule"
5. Datetime picker modal opens
6. Select date/time
7. Click "Schedule"
8. Toast: "✓ Scheduled for Jan 16, 8:00 AM"
9. Can view in Logs screen
```

### Flow 3: Quick Send to Recent Contact
```
1. Go to Contacts screen
2. See "Juan Dela Cruz" in recent
3. Click [Send] button next to name
4. Returns to composer with Juan pre-filled
5. Type message
6. Send
```

---

## Technical Stack

### Frontend
- **Framework:** Vue 3 with Inertia.js
- **UI Library:** Tailwind CSS
- **Icons:** Heroicons
- **Forms:** Vuelidate for validation
- **Autocomplete:** vue3-select or custom component

### Components Structure
```
resources/js/
├── Pages/
│   ├── Auth/
│   │   └── Login.vue
│   ├── Dashboard.vue        # SMS Composer
│   ├── Contacts/
│   │   ├── Index.vue
│   │   └── Edit.vue
│   ├── Logs/
│   │   └── Index.vue
│   └── Settings/
│       └── Index.vue
├── Components/
│   ├── Layout/
│   │   ├── AppLayout.vue
│   │   └── TopNav.vue
│   ├── SMSComposer/
│   │   ├── RecipientInput.vue
│   │   ├── MessageTextarea.vue
│   │   └── SenderSelect.vue
│   └── Shared/
│       ├── Button.vue
│       ├── Input.vue
│       └── Card.vue
└── Composables/
    ├── useContacts.js
    ├── useSMS.js
    └── useAutocomplete.js
```

---

## Key Design Decisions

### 1. ✅ No Sign-Up
- Single admin user created via seeder/command
- Eliminates user management complexity
- Fast deployment for government/NGO use

### 2. ✅ Composer-First Dashboard
- Primary action (send SMS) is immediately accessible
- No extra clicks to reach sending interface
- Feels like a messaging app, not a CRM

### 3. ✅ Smart Recipient Input
- Accepts numbers, contacts, and groups
- No mental model of "what type am I adding?"
- Autocomplete makes it fast

### 4. ✅ Minimal Navigation
- Only 3-4 links in top nav
- Everything else is within those sections
- Reduces cognitive load

### 5. ✅ Mobile-First
- Most government workers use mobile devices
- Touch-friendly tap targets (44px minimum)
- Responsive from day one

---

## Accessibility

- **ARIA labels** on all interactive elements
- **Keyboard navigation** for all actions
- **Focus indicators** clearly visible
- **Color contrast** WCAG AA compliant
- **Screen reader** friendly structure

---

## Future Enhancements

### Phase 2
- 📊 Dashboard statistics (messages sent today, this week)
- 📅 Calendar view for scheduled messages
- 📝 Message templates
- 🏷️ Contact tags/labels
- 📤 Batch import improvements

### Phase 3
- 📱 PWA support (install as mobile app)
- 🔔 Push notifications for delivery status
- 📊 Analytics dashboard
- 👥 Multi-user support (if needed)

---

## Related Documentation

- [API Documentation](api-documentation.md) - Backend endpoints
- [Backend Services](backend-services.md) - Actions and logic
- [Contact Package](contact-package.md) - Contact management
- [Development Plan](development-plan.md) - Implementation roadmap
