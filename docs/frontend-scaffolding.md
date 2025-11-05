# Frontend Scaffolding

This document provides complete scaffolding for all Vue 3 + Inertia.js pages, components, TypeScript types, and composables for Text Commander.

---

## Table of Contents

- [Directory Structure](#directory-structure)
- [TypeScript Types](#typescript-types)
- [Layouts](#layouts)
- [Pages](#pages)
- [Components](#components)
- [Composables](#composables)
- [API Client](#api-client)
- [Configuration](#configuration)

---

## Directory Structure

```
resources/
├── js/
│   ├── pages/
│   │   ├── auth/
│   │   │   └── login.vue
│   │   ├── dashboard.vue
│   │   ├── contacts/
│   │   │   ├── index.vue
│   │   │   └── edit.vue
│   │   ├── logs/
│   │   │   └── index.vue
│   │   ├── blacklist/
│   │   │   └── index.vue
│   │   └── settings/
│   │       └── index.vue
│   ├── components/
│   │   ├── layout/
│   │   │   ├── app-layout.vue
│   │   │   └── top-nav.vue
│   │   ├── sms-composer/
│   │   │   ├── recipient-input.vue
│   │   │   ├── message-textarea.vue
│   │   │   └── sender-select.vue
│   │   ├── shared/
│   │   │   ├── button.vue
│   │   │   ├── input.vue
│   │   │   ├── card.vue
│   │   │   └── schedule-modal.vue
│   │   ├── contacts/
│   │   │   ├── contact-card.vue
│   │   │   └── group-card.vue
│   │   └── blacklist/
│   │       └── add-blacklist-modal.vue
│   ├── composables/
│   │   ├── use-contacts.ts
│   │   ├── use-sms.ts
│   │   ├── use-autocomplete.ts
│   │   └── use-toast.ts
│   ├── types/
│   │   ├── models.ts
│   │   ├── api.ts
│   │   └── forms.ts
│   ├── lib/
│   │   ├── api.ts
│   │   ├── formatters.ts
│   │   └── validators.ts
│   └── app.ts
└── css/
    └── app.css
```

---

## TypeScript Types

### `resources/js/types/models.ts`

```typescript
export interface Contact {
  id: number
  mobile: string
  mobile_e164: string
  country: string
  name?: string
  bank_account?: string
  bank_code?: string
  account_number?: string
  tags: string[]
  extra_attributes: Record<string, any>
  created_at: string
  updated_at: string
}

export interface Group {
  id: number
  name: string
  description?: string
  contacts_count: number
  created_at: string
  updated_at: string
}

export interface ScheduledMessage {
  id: number
  message: string
  sender_id: string
  recipient_type: 'numbers' | 'group' | 'mixed'
  recipient_data: {
    numbers: string[]
    groups: Array<{
      id: number
      name: string
      count: number
    }>
  }
  scheduled_at: string
  sent_at?: string
  status: 'pending' | 'processing' | 'sent' | 'failed' | 'cancelled'
  total_recipients: number
  sent_count: number
  failed_count: number
  errors?: Array<{
    number: string
    error: string
  }>
  created_at: string
  updated_at: string
}

export interface User {
  id: number
  name: string
  email: string
  role: string
  created_at: string
  updated_at: string
}

export interface SenderID {
  id: number
  name: string
  is_default: boolean
}

export interface SMSLog {
  id: number
  mobile: string
  message: string
  sender_id: string
  status: 'sent' | 'failed' | 'pending'
  sent_at?: string
  error?: string
  created_at: string
}

export interface BlacklistedNumber {
  id: number
  mobile: string
  reason: string
  added_by: string | null
  blocked_at: string
  created_at: string
  updated_at: string
}
```

### `resources/js/types/api.ts`

```typescript
import { Contact, Group, ScheduledMessage, SMSLog } from './models'

export interface SendSMSRequest {
  recipients: string | string[]
  message: string
  sender_id?: string
}

export interface SendSMSResponse {
  status: 'queued'
  count: number
  recipients: string[]
  invalid_count: number
}

export interface ScheduleSMSRequest {
  recipients: string | string[]
  message: string
  scheduled_at: string
  sender_id?: string
}

export interface ScheduleSMSResponse {
  id: number
  status: 'scheduled'
  scheduled_at: string
  recipient_count: number
  message: string
}

---

## API Client {#api-client}

### `resources/js/lib/api.ts`

Centralized API client using Axios for making HTTP requests to the backend.

```typescript
import axios, { AxiosInstance } from 'axios'
import { SendSMSRequest, SendSMSResponse, ScheduleSMSRequest } from '@/types/api'

class APIClient {
  private client: AxiosInstance

  constructor() {
    this.client = axios.create({
      baseURL: '/api',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    })

    // Add auth token from meta tag
    const token = document.head.querySelector('meta[name="api-token"]')?.getAttribute('content')
    if (token) {
      this.client.defaults.headers.common['Authorization'] = `Bearer ${token}`
    }
  }

  // SMS Methods
  async sendSMS(data: SendSMSRequest): Promise<SendSMSResponse> {
    const response = await this.client.post('/sms/send', data)
    return response.data
  }

  async scheduleSMS(data: ScheduleSMSRequest) {
    const response = await this.client.post('/sms/schedule', data)
    return response.data
  }

  // Group Methods
  async getGroups() {
    const response = await this.client.get('/groups')
    return response.data
  }

  async getGroup(id: number) {
    const response = await this.client.get(`/groups/${id}`)
    return response.data
  }

  // Contact Methods
  async getContacts(params?: { search?: string; page?: number }) {
    const response = await this.client.get('/contacts', { params })
    return response.data
  }

  async addContactToGroup(contactId: number, groupId: number) {
    const response = await this.client.post(`/contacts/${contactId}/groups`, { group_id: groupId })
    return response.data
  }
}

export const api = new APIClient()
```

export interface PaginatedResponse<T> {
  data: T[]
  links: {
    first: string
    last: string
    prev: string | null
    next: string | null
  }
  meta: {
    current_page: number
    from: number
    last_page: number
    path: string
    per_page: number
    to: number
    total: number
  }
}

export interface ApiError {
  message: string
  errors?: Record<string, string[]>
}
```

### `resources/js/types/forms.ts`

```typescript
export interface SMSComposerForm {
  recipients: string[]
  message: string
  senderId: string
}

export interface ContactForm {
  mobile: string
  name?: string
  tags: string[]
}

export interface GroupForm {
  name: string
  description?: string
}

export interface LoginForm {
  email: string
  password: string
  remember: boolean
}

export interface ScheduleForm {
  date: string
  time: string
}
```

---

## Layouts

### `resources/js/components/layout/app-layout.vue`

```vue
<template>
  <div class="min-h-screen bg-gray-50">
    <TopNav />
    
    <main class="max-w-7xl mx-auto py-6 sm:px-6 lg:px-8">
      <slot />
    </main>
  </div>
</template>

<script setup lang="ts">
import TopNav from './TopNav.vue'
</script>
```

### `resources/js/components/layout/top-nav.vue`

```vue
<template>
  <nav class="bg-white shadow-sm border-b border-gray-200">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="flex justify-between h-16">
        <!-- Logo / Brand -->
        <div class="flex items-center">
          <Link href="/" class="flex items-center space-x-2">
            <span class="text-2xl">📤</span>
            <span class="text-xl font-bold text-gray-900">Text Commander</span>
          </Link>
        </div>

        <!-- Navigation Links -->
        <div class="hidden sm:flex sm:items-center sm:space-x-8">
          <Link
            href="/contacts"
            :class="navLinkClass('/contacts')"
          >
            Contacts
          </Link>
          <Link
            href="/logs"
            :class="navLinkClass('/logs')"
          >
            Logs
          </Link>
          <Link
            href="/settings"
            :class="navLinkClass('/settings')"
          >
            Settings
          </Link>

          <!-- User Dropdown -->
          <Menu as="div" class="relative">
            <MenuButton class="flex items-center space-x-2 text-gray-700 hover:text-gray-900">
              <span>{{ user.name }}</span>
              <ChevronDownIcon class="w-4 h-4" />
            </MenuButton>
            <transition
              enter-active-class="transition duration-100 ease-out"
              enter-from-class="transform scale-95 opacity-0"
              enter-to-class="transform scale-100 opacity-100"
              leave-active-class="transition duration-75 ease-in"
              leave-from-class="transform scale-100 opacity-100"
              leave-to-class="transform scale-95 opacity-0"
            >
              <MenuItems class="absolute right-0 mt-2 w-48 origin-top-right bg-white rounded-lg shadow-lg ring-1 ring-black ring-opacity-5 focus:outline-none">
                <div class="py-1">
                  <MenuItem v-slot="{ active }">
                    <Link
                      href="/settings"
                      :class="[active ? 'bg-gray-100' : '', 'block px-4 py-2 text-sm text-gray-700']"
                    >
                      Change Password
                    </Link>
                  </MenuItem>
                  <MenuItem v-slot="{ active }">
                    <Link
                      href="/logout"
                      method="post"
                      as="button"
                      :class="[active ? 'bg-gray-100' : '', 'block w-full text-left px-4 py-2 text-sm text-gray-700']"
                    >
                      Logout
                    </Link>
                  </MenuItem>
                </div>
              </MenuItems>
            </transition>
          </Menu>
        </div>

        <!-- Mobile Menu Button -->
        <div class="flex items-center sm:hidden">
          <button
            @click="mobileMenuOpen = !mobileMenuOpen"
            class="inline-flex items-center justify-center p-2 rounded-md text-gray-700 hover:bg-gray-100"
          >
            <Bars3Icon v-if="!mobileMenuOpen" class="w-6 h-6" />
            <XMarkIcon v-else class="w-6 h-6" />
          </button>
        </div>
      </div>
    </div>

    <!-- Mobile Menu -->
    <div v-if="mobileMenuOpen" class="sm:hidden border-t border-gray-200">
      <div class="pt-2 pb-3 space-y-1">
        <Link
          href="/contacts"
          :class="mobileLinkClass('/contacts')"
        >
          Contacts
        </Link>
        <Link
          href="/logs"
          :class="mobileLinkClass('/logs')"
        >
          Logs
        </Link>
        <Link
          href="/settings"
          :class="mobileLinkClass('/settings')"
        >
          Settings
        </Link>
        <Link
          href="/logout"
          method="post"
          as="button"
          class="block w-full text-left px-4 py-2 text-base font-medium text-gray-700 hover:bg-gray-100"
        >
          Logout
        </Link>
      </div>
    </div>
  </nav>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { Link, usePage } from '@inertiajs/vue3'
import { Menu, MenuButton, MenuItems, MenuItem } from '@headlessui/vue'
import { ChevronDownIcon, Bars3Icon, XMarkIcon } from '@heroicons/vue/24/outline'
import type { User } from '@/types/models'

const page = usePage()
const user = computed(() => page.props.auth.user as User)

const mobileMenuOpen = ref(false)

const navLinkClass = (path: string) => {
  const isActive = page.url.startsWith(path)
  return [
    'text-sm font-medium transition-colors',
    isActive
      ? 'text-blue-600'
      : 'text-gray-700 hover:text-gray-900'
  ]
}

const mobileLinkClass = (path: string) => {
  const isActive = page.url.startsWith(path)
  return [
    'block px-4 py-2 text-base font-medium',
    isActive
      ? 'bg-blue-50 text-blue-600'
      : 'text-gray-700 hover:bg-gray-100'
  ]
}
</script>
```

---

## Pages

### `resources/js/pages/auth/login.vue`

```vue
<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-50 py-12 px-4 sm:px-6 lg:px-8">
    <div class="max-w-md w-full space-y-8">
      <div>
        <div class="flex justify-center">
          <span class="text-6xl">📤</span>
        </div>
        <h2 class="mt-6 text-center text-3xl font-bold text-gray-900">
          Text Commander
        </h2>
        <p class="mt-2 text-center text-sm text-gray-600">
          SMS Broadcasting System
        </p>
      </div>

      <form @submit.prevent="submit" class="mt-8 space-y-6">
        <div class="rounded-md shadow-sm space-y-4">
          <div>
            <label for="email" class="block text-sm font-medium text-gray-700 mb-2">
              Email
            </label>
            <input
              id="email"
              v-model="form.email"
              type="email"
              required
              class="appearance-none relative block w-full px-3 py-2 border border-gray-300 placeholder-gray-500 text-gray-900 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
              placeholder="admin@txtcmdr.local"
            />
            <div v-if="form.errors.email" class="mt-1 text-sm text-red-600">
              {{ form.errors.email }}
            </div>
          </div>

          <div>
            <label for="password" class="block text-sm font-medium text-gray-700 mb-2">
              Password
            </label>
            <input
              id="password"
              v-model="form.password"
              type="password"
              required
              class="appearance-none relative block w-full px-3 py-2 border border-gray-300 placeholder-gray-500 text-gray-900 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
              placeholder="••••••••"
            />
            <div v-if="form.errors.password" class="mt-1 text-sm text-red-600">
              {{ form.errors.password }}
            </div>
          </div>
        </div>

        <div class="flex items-center">
          <input
            id="remember"
            v-model="form.remember"
            type="checkbox"
            class="h-4 w-4 text-blue-600 focus:ring-blue-500 border-gray-300 rounded"
          />
          <label for="remember" class="ml-2 block text-sm text-gray-900">
            Remember me
          </label>
        </div>

        <div>
          <button
            type="submit"
            :disabled="form.processing"
            class="group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-lg text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {{ form.processing ? 'Logging in...' : 'Login' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useForm } from '@inertiajs/vue3'
import type { LoginForm } from '@/types/forms'

const form = useForm<LoginForm>({
  email: '',
  password: '',
  remember: false
})

const submit = () => {
  form.post('/login')
}
</script>
```

### `resources/js/pages/dashboard.vue`

```vue
<template>
  <AppLayout>
    <div class="max-w-4xl mx-auto">
      <h1 class="text-2xl font-bold text-gray-900 mb-6">📤 Send SMS</h1>

      <Card>
        <form @submit.prevent="sendNow">
          <!-- To: Recipients -->
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-700 mb-2">
              To:
            </label>
            <RecipientInput
              v-model="form.recipients"
              :contacts="contacts"
              :groups="groups"
            />
          </div>

          <!-- Message -->
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-700 mb-2">
              Message:
            </label>
            <MessageTextarea v-model="form.message" />
          </div>

          <!-- Footer: Character Count + Sender ID -->
          <div class="mb-6 flex items-center justify-between">
            <div class="text-sm text-gray-600">
              {{ characterCount }}/160 ({{ smsCount }} SMS)
            </div>
            <SenderSelect v-model="form.senderId" :senders="senderIds" />
          </div>

          <!-- Actions -->
          <div class="flex gap-3">
            <Button
              type="submit"
              :disabled="!canSend || form.processing"
              variant="primary"
              class="flex-1"
            >
              {{ form.processing ? 'Sending...' : 'Send Now' }}
            </Button>
            <Button
              type="button"
              @click="openScheduleModal"
              :disabled="!canSend"
              variant="secondary"
              class="flex-1"
            >
              Schedule
            </Button>
          </div>
        </form>
      </Card>
    </div>

    <!-- Schedule Modal -->
    <ScheduleModal
      :show="showScheduleModal"
      :recipients="form.recipients"
      :message="form.message"
      :sender-id="form.senderId"
      :recipient-count="recipientCount"
      @close="showScheduleModal = false"
      @success="handleScheduleSuccess"
    />
  </AppLayout>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { useForm } from '@inertiajs/vue3'
import AppLayout from '@/components/layout/app-layout.vue'
import Card from '@/components/shared/card.vue'
import Button from '@/components/shared/button.vue'
import RecipientInput from '@/components/sms-composer/recipient-input.vue'
import MessageTextarea from '@/components/sms-composer/message-textarea.vue'
import SenderSelect from '@/components/sms-composer/sender-select.vue'
import ScheduleModal from '@/components/shared/schedule-modal.vue'
import { useToast } from '@/composables/use-toast'
import type { Contact, Group, SenderID } from '@/types/models'
import type { SMSComposerForm } from '@/types/forms'

interface Props {
  contacts: Contact[]
  groups: Group[]
  senderIds: SenderID[]
}

const props = defineProps<Props>()
const { showSuccess } = useToast()

const form = useForm<SMSComposerForm>({
  recipients: [],
  message: '',
  senderId: props.senderIds.find(s => s.is_default)?.name || 'TXTCMDR'
})

const showScheduleModal = ref(false)

const characterCount = computed(() => form.message.length)
const smsCount = computed(() => Math.ceil(characterCount.value / 160) || 1)
const recipientCount = computed(() => form.recipients.length)
const canSend = computed(() => form.recipients.length > 0 && form.message.trim().length > 0)

const sendNow = () => {
  form.post('/api/send', {
    onSuccess: () => {
      showSuccess('✓ Message sent!')
      form.reset()
    }
  })
}

const openScheduleModal = () => {
  showScheduleModal.value = true
}

const handleScheduleSuccess = () => {
  showSuccess('✓ Message scheduled!')
  form.reset()
  showScheduleModal.value = false
}
</script>
```

### `resources/js/pages/contacts/index.vue`

```vue
<template>
  <AppLayout>
    <div class="max-w-6xl mx-auto">
      <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold text-gray-900">Contacts</h1>
        
        <div class="flex gap-3">
          <Button @click="showImportModal = true" variant="secondary">
            Import CSV
          </Button>
          <Button @click="showAddModal = true" variant="primary">
            + Add Contact
          </Button>
        </div>
      </div>

      <!-- Search -->
      <div class="mb-6">
        <Input
          v-model="searchQuery"
          type="search"
          placeholder="Search contacts and groups..."
          class="max-w-md"
        />
      </div>

      <!-- Groups -->
      <div class="mb-8">
        <h2 class="text-lg font-semibold text-gray-900 mb-4">Groups</h2>
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          <GroupCard
            v-for="group in filteredGroups"
            :key="group.id"
            :group="group"
            @send="handleSendToGroup"
            @edit="handleEditGroup"
          />
        </div>
      </div>

      <!-- Recent Contacts -->
      <div>
        <h2 class="text-lg font-semibold text-gray-900 mb-4">Recent Contacts</h2>
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          <ContactCard
            v-for="contact in filteredContacts"
            :key="contact.id"
            :contact="contact"
            @send="handleSendToContact"
            @edit="handleEditContact"
          />
        </div>
      </div>
    </div>
  </AppLayout>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { router } from '@inertiajs/vue3'
import AppLayout from '@/components/layout/app-layout.vue'
import Button from '@/components/shared/button.vue'
import Input from '@/components/shared/input.vue'
import GroupCard from '@/components/contacts/group-card.vue'
import ContactCard from '@/components/contacts/contact-card.vue'
import type { Contact, Group } from '@/types/models'

interface Props {
  contacts: Contact[]
  groups: Group[]
}

const props = defineProps<Props>()

const searchQuery = ref('')
const showAddModal = ref(false)
const showImportModal = ref(false)

const filteredGroups = computed(() => {
  if (!searchQuery.value) return props.groups
  return props.groups.filter(g => 
    g.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
})

const filteredContacts = computed(() => {
  if (!searchQuery.value) return props.contacts
  return props.contacts.filter(c => 
    c.name?.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
    c.mobile.includes(searchQuery.value)
  )
})

const handleSendToGroup = (group: Group) => {
  router.visit('/', {
    data: { recipient: group.name }
  })
}

const handleSendToContact = (contact: Contact) => {
  router.visit('/', {
    data: { recipient: contact.mobile }
  })
}

const handleEditGroup = (group: Group) => {
  router.visit(`/groups/${group.id}/edit`)
}

const handleEditContact = (contact: Contact) => {
  router.visit(`/contacts/${contact.id}/edit`)
}
</script>
```

### `resources/js/pages/logs/index.vue`

```vue
<template>
  <AppLayout>
    <div class="max-w-6xl mx-auto">
      <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold text-gray-900">Message Logs</h1>
        
        <div class="flex gap-2">
          <select
            v-model="statusFilter"
            class="px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
          >
            <option value="all">All</option>
            <option value="pending">Scheduled</option>
            <option value="sent">Sent</option>
            <option value="failed">Failed</option>
            <option value="cancelled">Cancelled</option>
          </select>

          <select
            v-model="dateFilter"
            class="px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
          >
            <option value="all">All Time</option>
            <option value="today">Today</option>
            <option value="week">This Week</option>
            <option value="month">This Month</option>
          </select>
        </div>
      </div>

      <!-- Messages List -->
      <div class="space-y-4">
        <Card
          v-for="message in messages.data"
          :key="message.id"
          class="hover:shadow-md transition-shadow"
        >
          <div class="flex items-start justify-between">
            <div class="flex-1">
              <!-- Status Badge + Recipient Summary -->
              <div class="flex items-center gap-2 mb-2">
                <span
                  :class="statusBadgeClass(message.status)"
                  class="px-2 py-1 text-xs font-medium rounded"
                >
                  {{ statusLabel(message.status) }}
                </span>
                <span class="text-sm text-gray-600">
                  {{ message.total_recipients }} recipient(s)
                </span>
              </div>
              
              <!-- Message Content -->
              <p class="text-gray-900 mb-2">{{ message.message }}</p>
              
              <!-- Timestamp + Sender ID -->
              <div class="text-sm text-gray-600">
                <span v-if="message.status === 'pending'">
                  ⏱️ Scheduled for {{ formatDateTime(message.scheduled_at) }}
                </span>
                <span v-else-if="message.sent_at">
                  ✓ Sent {{ formatDateTime(message.sent_at) }}
                </span>
                <span class="ml-3">From: {{ message.sender_id }}</span>
              </div>
            </div>

            <!-- Actions for Scheduled Messages -->
            <div v-if="message.status === 'pending'" class="flex gap-2 ml-4">
              <Button
                @click="editScheduledMessage(message.id)"
                variant="secondary"
                size="sm"
              >
                Edit
              </Button>
              <Button
                @click="cancelScheduledMessage(message.id)"
                variant="danger"
                size="sm"
              >
                Cancel
              </Button>
            </div>
          </div>
        </Card>
      </div>

      <!-- Pagination -->
      <div v-if="messages.meta.last_page > 1" class="mt-6 flex justify-center">
        <nav class="flex gap-2">
          <Button
            v-for="page in paginationRange"
            :key="page"
            @click="goToPage(page)"
            :variant="page === messages.meta.current_page ? 'primary' : 'secondary'"
            size="sm"
          >
            {{ page }}
          </Button>
        </nav>
      </div>
    </div>
  </AppLayout>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { router } from '@inertiajs/vue3'
import AppLayout from '@/components/layout/app-layout.vue'
import Card from '@/components/shared/card.vue'
import Button from '@/components/shared/button.vue'
import { useToast } from '@/composables/use-toast'
import type { ScheduledMessage } from '@/types/models'
import type { PaginatedResponse } from '@/types/api'
import dayjs from 'dayjs'

interface Props {
  messages: PaginatedResponse<ScheduledMessage>
}

const props = defineProps<Props>()
const { showSuccess, showError } = useToast()

const statusFilter = ref('all')
const dateFilter = ref('all')

const paginationRange = computed(() => {
  const current = props.messages.meta.current_page
  const last = props.messages.meta.last_page
  const delta = 2
  const range = []
  
  for (let i = Math.max(2, current - delta); i <= Math.min(last - 1, current + delta); i++) {
    range.push(i)
  }
  
  if (current - delta > 2) range.unshift('...')
  if (current + delta < last - 1) range.push('...')
  
  range.unshift(1)
  if (last !== 1) range.push(last)
  
  return range
})

watch([statusFilter, dateFilter], () => {
  router.get('/logs', {
    status: statusFilter.value,
    date: dateFilter.value
  }, {
    preserveState: true
  })
})

const statusBadgeClass = (status: string) => {
  const classes = {
    pending: 'bg-yellow-100 text-yellow-800',
    processing: 'bg-blue-100 text-blue-800',
    sent: 'bg-green-100 text-green-800',
    failed: 'bg-red-100 text-red-800',
    cancelled: 'bg-gray-100 text-gray-800'
  }
  return classes[status] || 'bg-gray-100 text-gray-800'
}

const statusLabel = (status: string) => {
  const labels = {
    pending: '⏱️ Scheduled',
    processing: '📤 Sending',
    sent: '✓ Sent',
    failed: '❌ Failed',
    cancelled: '🚫 Cancelled'
  }
  return labels[status] || status
}

const formatDateTime = (dateTime: string) => {
  return dayjs(dateTime).format('MMM D, YYYY [at] h:mm A')
}

const goToPage = (page: number | string) => {
  if (typeof page === 'number') {
    router.visit(`/logs?page=${page}`)
  }
}

const editScheduledMessage = (id: number) => {
  router.visit(`/scheduled-messages/${id}/edit`)
}

const cancelScheduledMessage = async (id: number) => {
  if (!confirm('Cancel this scheduled message?')) return

  try {
    await axios.post(`/api/scheduled-messages/${id}/cancel`)
    showSuccess('Message cancelled')
    router.reload()
  } catch (error) {
    showError('Failed to cancel message')
  }
}
</script>
```

### `resources/js/pages/settings/index.vue`

```vue
<template>
  <AppLayout>
    <div class="max-w-4xl mx-auto">
      <h1 class="text-2xl font-bold text-gray-900 mb-6">Settings</h1>

      <!-- Account Settings -->
      <Card class="mb-6">
        <h2 class="text-lg font-semibold text-gray-900 mb-4">Account</h2>
        
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">
              Email
            </label>
            <p class="text-gray-900">{{ user.email }}</p>
          </div>

          <div>
            <Button @click="showChangePasswordModal = true" variant="secondary">
              Change Password
            </Button>
          </div>
        </div>
      </Card>

      <!-- Sender IDs -->
      <Card class="mb-6">
        <div class="flex items-center justify-between mb-4">
          <h2 class="text-lg font-semibold text-gray-900">Sender IDs</h2>
          <Button @click="showAddSenderModal = true" variant="secondary" size="sm">
            + Add New
          </Button>
        </div>
        
        <div class="space-y-2">
          <div
            v-for="sender in senderIds"
            :key="sender.id"
            class="flex items-center justify-between py-2 px-3 border border-gray-200 rounded-lg"
          >
            <div class="flex items-center gap-2">
              <span class="font-medium text-gray-900">{{ sender.name }}</span>
              <span v-if="sender.is_default" class="text-xs bg-blue-100 text-blue-800 px-2 py-0.5 rounded">
                Default
              </span>
            </div>
            <Button
              v-if="!sender.is_default"
              @click="setDefaultSender(sender.id)"
              variant="secondary"
              size="sm"
            >
              Set as Default
            </Button>
          </div>
        </div>
      </Card>

      <!-- SMS Gateway -->
      <Card>
        <h2 class="text-lg font-semibold text-gray-900 mb-4">SMS Gateway</h2>
        
        <div class="space-y-3">
          <div class="flex items-center justify-between">
            <span class="text-sm text-gray-700">Provider:</span>
            <span class="font-medium text-gray-900">{{ gateway.provider }}</span>
          </div>
          <div class="flex items-center justify-between">
            <span class="text-sm text-gray-700">Status:</span>
            <span class="flex items-center gap-1">
              <span class="w-2 h-2 bg-green-500 rounded-full"></span>
              <span class="text-sm font-medium text-green-700">Connected</span>
            </span>
          </div>
          <div class="flex items-center justify-between">
            <span class="text-sm text-gray-700">Balance:</span>
            <span class="font-medium text-gray-900">₱{{ gateway.balance.toFixed(2) }}</span>
          </div>
        </div>
      </Card>
    </div>
  </AppLayout>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { router } from '@inertiajs/vue3'
import AppLayout from '@/components/layout/app-layout.vue'
import Card from '@/components/shared/card.vue'
import Button from '@/components/shared/button.vue'
import { useToast } from '@/composables/use-toast'
import type { User, SenderID } from '@/types/models'

interface Props {
  user: User
  senderIds: SenderID[]
  gateway: {
    provider: string
    balance: number
  }
}

const props = defineProps<Props>()
const { showSuccess } = useToast()

const showChangePasswordModal = ref(false)
const showAddSenderModal = ref(false)

const setDefaultSender = async (id: number) => {
  try {
    await axios.post(`/api/sender-ids/${id}/set-default`)
    showSuccess('Default sender updated')
    router.reload()
  } catch (error) {
    console.error(error)
  }
}
</script>
```

---

## Components

### `resources/js/components/shared/button.vue`

```vue
<template>
  <button
    :type="type"
    :disabled="disabled"
    :class="classes"
  >
    <slot />
  </button>
</template>

<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  type?: 'button' | 'submit' | 'reset'
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  type: 'button',
  variant: 'primary',
  size: 'md',
  disabled: false
})

const classes = computed(() => {
  const base = 'font-semibold rounded-lg transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed'
  
  const variants = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500',
    secondary: 'bg-white text-blue-600 border-2 border-blue-600 hover:bg-blue-50 focus:ring-blue-500',
    danger: 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500'
  }
  
  const sizes = {
    sm: 'px-3 py-1 text-sm',
    md: 'px-6 py-3 text-base',
    lg: 'px-8 py-4 text-lg'
  }
  
  return [base, variants[props.variant], sizes[props.size]]
})
</script>
```

### `resources/js/components/shared/input.vue`

```vue
<template>
  <input
    :type="type"
    :value="modelValue"
    @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    :placeholder="placeholder"
    :disabled="disabled"
    :class="classes"
  />
</template>

<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  modelValue: string
  type?: string
  placeholder?: string
  disabled?: boolean
  error?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  type: 'text',
  placeholder: '',
  disabled: false,
  error: false
})

defineEmits<{
  'update:modelValue': [value: string]
}>()

const classes = computed(() => {
  const base = 'w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2 transition-colors'
  const error = props.error
    ? 'border-red-300 focus:ring-red-500 focus:border-red-500'
    : 'border-gray-300 focus:ring-blue-500 focus:border-blue-500'
  
  return [base, error]
})
</script>
```

### `resources/js/components/shared/card.vue`

```vue
<template>
  <div class="bg-white border border-gray-200 rounded-lg p-6 shadow-sm">
    <slot />
  </div>
</template>

<script setup lang="ts"></script>
```

---

## Composables

### `resources/js/composables/use-toast.ts`

```typescript
import { ref } from 'vue'

interface Toast {
  id: number
  message: string
  type: 'success' | 'error' | 'info'
}

const toasts = ref<Toast[]>([])
let nextId = 0

export function useToast() {
  const showToast = (message: string, type: Toast['type'] = 'info') => {
    const id = nextId++
    toasts.value.push({ id, message, type })
    
    setTimeout(() => {
      toasts.value = toasts.value.filter(t => t.id !== id)
    }, 3000)
  }

  const showSuccess = (message: string) => showToast(message, 'success')
  const showError = (message: string) => showToast(message, 'error')
  const showInfo = (message: string) => showToast(message, 'info')

  return {
    toasts,
    showToast,
    showSuccess,
    showError,
    showInfo
  }
}
```

### `resources/js/composables/use-sms.ts`

```typescript
import { ref } from 'vue'
import axios from 'axios'
import type { SendSMSRequest, SendSMSResponse, ScheduleSMSRequest, ScheduleSMSResponse } from '@/types/api'

export function useSMS() {
  const loading = ref(false)
  const error = ref<string | null>(null)

  const sendSMS = async (data: SendSMSRequest): Promise<SendSMSResponse> => {
    loading.value = true
    error.value = null
    
    try {
      const response = await axios.post<SendSMSResponse>('/api/send', data)
      return response.data
    } catch (err: any) {
      error.value = err.response?.data?.message || 'Failed to send SMS'
      throw err
    } finally {
      loading.value = false
    }
  }

  const scheduleSMS = async (data: ScheduleSMSRequest): Promise<ScheduleSMSResponse> => {
    loading.value = true
    error.value = null
    
    try {
      const response = await axios.post<ScheduleSMSResponse>('/api/send/schedule', data)
      return response.data
    } catch (err: any) {
      error.value = err.response?.data?.message || 'Failed to schedule SMS'
      throw err
    } finally {
      loading.value = false
    }
  }

  return {
    loading,
    error,
    sendSMS,
    scheduleSMS
  }
}
```

### `resources/js/composables/use-contacts.ts`

```typescript
import { ref, computed } from 'vue'
import axios from 'axios'
import type { Contact, Group } from '@/types/models'

export function useContacts() {
  const contacts = ref<Contact[]>([])
  const groups = ref<Group[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  const fetchContacts = async () => {
    loading.value = true
    error.value = null
    
    try {
      const response = await axios.get<Contact[]>('/api/contacts')
      contacts.value = response.data
    } catch (err: any) {
      error.value = err.response?.data?.message || 'Failed to fetch contacts'
      throw err
    } finally {
      loading.value = false
    }
  }

  const fetchGroups = async () => {
    loading.value = true
    error.value = null
    
    try {
      const response = await axios.get<Group[]>('/api/groups')
      groups.value = response.data
    } catch (err: any) {
      error.value = err.response?.data?.message || 'Failed to fetch groups'
      throw err
    } finally {
      loading.value = false
    }
  }

  const searchOptions = computed(() => {
    const contactOptions = contacts.value.map(c => ({
      label: c.name || c.mobile,
      value: c.mobile,
      type: 'contact' as const,
      icon: '👤'
    }))

    const groupOptions = groups.value.map(g => ({
      label: `${g.name} (${g.contacts_count})`,
      value: g.name,
      type: 'group' as const,
      icon: '👥'
    }))

    return [...groupOptions, ...contactOptions]
  })

  return {
    contacts,
    groups,
    loading,
    error,
    searchOptions,
    fetchContacts,
    fetchGroups
  }
}
```

### `resources/js/composables/use-autocomplete.ts`

```typescript
import { ref, computed } from 'vue'

export interface AutocompleteOption {
  label: string
  value: string
  type: 'contact' | 'group' | 'number'
  icon?: string
}

export function useAutocomplete(options: AutocompleteOption[]) {
  const query = ref('')
  const selected = ref<string[]>([])
  const isOpen = ref(false)

  const filteredOptions = computed(() => {
    if (!query.value) return options

    const lowerQuery = query.value.toLowerCase()
    return options.filter(option =>
      option.label.toLowerCase().includes(lowerQuery) ||
      option.value.toLowerCase().includes(lowerQuery)
    )
  })

  const addSelection = (value: string) => {
    if (!selected.value.includes(value)) {
      selected.value.push(value)
    }
    query.value = ''
    isOpen.value = false
  }

  const removeSelection = (value: string) => {
    selected.value = selected.value.filter(v => v !== value)
  }

  const clearAll = () => {
    selected.value = []
  }

  return {
    query,
    selected,
    isOpen,
    filteredOptions,
    addSelection,
    removeSelection,
    clearAll
  }
}
```

---

## Blacklist Management Pages

### `resources/js/pages/blacklist/index.vue`

```vue
<template>
  <app-layout>
    <div class="max-w-6xl mx-auto p-6">
      <div class="flex justify-between items-center mb-6">
        <h1 class="text-2xl font-bold text-gray-900">🚫 Blacklisted Numbers</h1>
        <button
          @click="showAddModal = true"
          class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        >
          Add Number
        </button>
      </div>

      <!-- Search & Filter -->
      <div class="mb-6 flex gap-4">
        <input
          v-model="filters.search"
          @input="search"
          type="text"
          placeholder="Search mobile number..."
          class="flex-1 px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
        />
        <select
          v-model="filters.reason"
          @change="search"
          class="px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
        >
          <option value="">All Reasons</option>
          <option value="opt-out">Opt-out</option>
          <option value="complaint">Complaint</option>
          <option value="invalid">Invalid</option>
          <option value="manual">Manual</option>
        </select>
      </div>

      <!-- Table -->
      <div class="bg-white rounded-lg shadow overflow-hidden">
        <table class="min-w-full divide-y divide-gray-200">
          <thead class="bg-gray-50">
            <tr>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Mobile Number
              </th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Reason
              </th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Added By
              </th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Blocked At
              </th>
              <th class="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase">
                Actions
              </th>
            </tr>
          </thead>
          <tbody class="bg-white divide-y divide-gray-200">
            <tr v-for="item in blacklisted.data" :key="item.id">
              <td class="px-6 py-4 whitespace-nowrap font-mono text-sm">
                {{ item.mobile }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {{ item.reason }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {{ item.added_by || '—' }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {{ formatDate(item.blocked_at) }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-right text-sm">
                <button
                  @click="removeNumber(item.mobile)"
                  class="text-red-600 hover:text-red-900"
                >
                  Remove
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagination -->
      <div class="mt-4 flex justify-between items-center">
        <div class="text-sm text-gray-700">
          Showing {{ blacklisted.from }} to {{ blacklisted.to }} of {{ blacklisted.total }}
        </div>
        <div class="flex gap-2">
          <button
            v-for="link in blacklisted.links"
            :key="link.label"
            @click="goToPage(link.url)"
            :disabled="!link.url"
            :class="[
              'px-3 py-1 rounded',
              link.active ? 'bg-blue-600 text-white' : 'bg-white border hover:bg-gray-50',
              !link.url && 'opacity-50 cursor-not-allowed'
            ]"
            v-html="link.label"
          />
        </div>
      </div>
    </div>

    <!-- Add Modal -->
    <add-blacklist-modal
      :show="showAddModal"
      @close="showAddModal = false"
      @success="handleAdded"
    />
  </app-layout>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { router } from '@inertiajs/vue3'
import AppLayout from '@/layouts/app-layout.vue'
import AddBlacklistModal from '@/components/blacklist/add-blacklist-modal.vue'
import dayjs from 'dayjs'
import type { BlacklistedNumber } from '@/types/models'

interface Props {
  blacklisted: {
    data: BlacklistedNumber[]
    from: number
    to: number
    total: number
    links: Array<{ label: string; url: string | null; active: boolean }>
  }
}

const props = defineProps<Props>()

const showAddModal = ref(false)

const filters = reactive({
  search: '',
  reason: '',
})

const search = () => {
  router.get('/blacklist', filters, { preserveState: true })
}

const removeNumber = async (mobile: string) => {
  if (!confirm(`Remove ${mobile} from blacklist?`)) return

  try {
    await axios.delete('/api/blacklist', { data: { mobile } })
    router.reload()
  } catch (error) {
    alert('Failed to remove number')
  }
}

const handleAdded = () => {
  showAddModal.value = false
  router.reload()
}

const goToPage = (url: string | null) => {
  if (url) router.visit(url)
}

const formatDate = (date: string) => {
  return dayjs(date).format('MMM D, YYYY h:mm A')
}
</script>
```

---

### `resources/js/components/blacklist/add-blacklist-modal.vue`

```vue
<template>
  <TransitionRoot :show="show" as="template">
    <Dialog as="div" class="relative z-50" @close="close">
      <TransitionChild
        enter="ease-out duration-300"
        enter-from="opacity-0"
        enter-to="opacity-100"
        leave="ease-in duration-200"
        leave-from="opacity-100"
        leave-to="opacity-0"
      >
        <div class="fixed inset-0 bg-black bg-opacity-25" />
      </TransitionChild>

      <div class="fixed inset-0 overflow-y-auto">
        <div class="flex min-h-full items-center justify-center p-4">
          <TransitionChild
            enter="ease-out duration-300"
            enter-from="opacity-0 scale-95"
            enter-to="opacity-100 scale-100"
            leave="ease-in duration-200"
            leave-from="opacity-100 scale-100"
            leave-to="opacity-0 scale-95"
          >
            <DialogPanel class="w-full max-w-md bg-white rounded-lg p-6 shadow-xl">
              <DialogTitle class="text-lg font-bold mb-4">
                Add to Blacklist
              </DialogTitle>

              <form @submit.prevent="submit">
                <!-- Mobile Number -->
                <div class="mb-4">
                  <label class="block text-sm font-medium text-gray-700 mb-2">
                    Mobile Number
                  </label>
                  <input
                    v-model="form.mobile"
                    type="text"
                    required
                    placeholder="0917 123 4567"
                    class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  />
                </div>

                <!-- Reason -->
                <div class="mb-6">
                  <label class="block text-sm font-medium text-gray-700 mb-2">
                    Reason
                  </label>
                  <select
                    v-model="form.reason"
                    class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  >
                    <option value="opt-out">User opted out</option>
                    <option value="complaint">Complaint received</option>
                    <option value="invalid">Invalid number</option>
                    <option value="manual">Manual addition</option>
                  </select>
                </div>

                <!-- Actions -->
                <div class="flex gap-3">
                  <button
                    type="button"
                    @click="close"
                    class="flex-1 px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50"
                  >
                    Cancel
                  </button>
                  <button
                    type="submit"
                    :disabled="processing"
                    class="flex-1 px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 disabled:opacity-50"
                  >
                    {{ processing ? 'Adding...' : 'Add to Blacklist' }}
                  </button>
                </div>
              </form>
            </DialogPanel>
          </TransitionChild>
        </div>
      </div>
    </Dialog>
  </TransitionRoot>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { Dialog, DialogPanel, DialogTitle, TransitionRoot, TransitionChild } from '@headlessui/vue'
import axios from 'axios'

interface Props {
  show: boolean
}

const props = defineProps<Props>()
const emit = defineEmits(['close', 'success'])

const processing = ref(false)

const form = reactive({
  mobile: '',
  reason: 'opt-out',
})

const submit = async () => {
  processing.value = true

  try {
    await axios.post('/api/blacklist', form)
    emit('success')
    form.mobile = ''
    form.reason = 'opt-out'
  } catch (error) {
    alert('Failed to add number to blacklist')
  } finally {
    processing.value = false
  }
}

const close = () => {
  emit('close')
}
</script>
```

---

## Configuration

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path aliases */
    "baseUrl": ".",
    "paths": {
      "@/*": ["resources/js/*"]
    }
  },
  "include": ["resources/js/**/*.ts", "resources/js/**/*.d.ts", "resources/js/**/*.vue"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import laravel from 'laravel-vite-plugin'
import { fileURLToPath } from 'url'

export default defineConfig({
  plugins: [
    laravel({
      input: ['resources/css/app.css', 'resources/js/app.ts'],
      refresh: true,
    }),
    vue({
      template: {
        transformAssetUrls: {
          base: null,
          includeAbsolute: false,
        },
      },
    }),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./resources/js', import.meta.url)),
    },
  },
})
```

### `resources/js/app.ts`

```typescript
import './bootstrap'
import '../css/app.css'

import { createApp, h, DefineComponent } from 'vue'
import { createInertiaApp } from '@inertiajs/vue3'
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers'
import { ZiggyVue } from '../../vendor/tightenco/ziggy/dist/vue.m'

const appName = import.meta.env.VITE_APP_NAME || 'Text Commander'

createInertiaApp({
  title: (title) => `${title} - ${appName}`,
  resolve: (name) =>
    resolvePageComponent(
      `./pages/${name}.vue`,
      import.meta.glob<DefineComponent>('./pages/**/*.vue')
    ),
  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .use(ZiggyVue)
      .mount(el)
  },
  progress: {
    color: '#2563EB',
  },
})
```

---

## Related Documentation

- [UI/UX Design](ui-ux-design.md) - Design specifications and mockups
- [Backend Services](backend-services.md) - API Actions and endpoints
- [Scheduled Messaging](scheduled-messaging.md) - Scheduling feature implementation
- [Development Plan](development-plan.md) - Implementation roadmap
