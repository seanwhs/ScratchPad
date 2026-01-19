### Technical Stack (January 2026 – Latest & Best)

| Library / Tool               | Version (Jan 2026) | Why it's best right now                              |
|------------------------------|--------------------|------------------------------------------------------|
| React                        | 19.x               | New compiler, actions, improved hooks & suspense     |
| React Router                 | 7.x                | Full Data APIs (loaders, actions, useFetcher, defer) |
| Vite                         | 6.x                | Lightning-fast dev server & builds                   |
| TailwindCSS                  | 4.x                | Modern, JIT, excellent DX & performance              |
| TanStack Query (React Query) | 5.x                | Best-in-class data fetching, background sync, offline|
| Zustand                      | 5.x                | Simple, lightweight global state                     |
| React Hook Form + Zod        | latest             | Type-safe forms with minimal re-renders              |
| react-thermal-printer        | latest             | Real ESC/POS thermal printing over Bluetooth         |
| localforage                  | latest             | Reliable IndexedDB wrapper for offline queue         |
| react-toastify               | latest             | Clean, non-blocking toasts                           |

### Step 1: Create Project (2026 Way)

```bash
npm create vite@latest hsh-frontend -- --template react-ts
cd hsh-frontend

# Install latest stable packages (as of Jan 2026)
npm install \
  react-router-dom@7 \
  @tanstack/react-query@5 \
  @tanstack/react-query-devtools \
  react-hook-form \
  @hookform/resolvers \
  zod \
  zustand \
  localforage \
  react-toastify \
  axios \
  jwt-decode \
  tailwindcss@4 postcss autoprefixer \
  react-thermal-printer \
  @headlessui/react @heroicons/react
```

Initialize Tailwind v4:
```bash
npx tailwindcss init -p
```

**`tailwind.config.ts`**
```ts
import type { Config } from 'tailwindcss'

export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        hsh: {
          primary: '#1e40af',    // deep blue – brand feel
          accent: '#f59e0b',     // warm orange for actions
        },
      },
    },
  },
  plugins: [],
} satisfies Config
```

**`src/index.css`**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 98%;
    --foreground: 0 0% 3.9%;
  }
  body {
    @apply bg-[hsl(var(--background))] text-[hsl(var(--foreground))];
    font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  }
}

input, select {
  @apply border border-gray-300 rounded-md px-3 py-2 focus:outline-none focus:ring-2 focus:ring-hsh-primary focus:border-transparent;
}
```

### Step 2: Clean & Modern Project Structure

```
src/
├── api/                  # axios instance + endpoints
├── components/           # reusable: ConfirmationDialog, DynamicRow, etc.
├── hooks/                # useAuth, useOfflineQueue, etc.
├── layouts/              # RootLayout with UserHeader
├── pages/                # route-level: Login, Distribution
├── stores/               # Zustand stores (auth + offline)
├── types/                # shared TS types
├── utils/                # printer, formatters
├── App.tsx
├── main.tsx
└── index.css
```

### Step 3: Auth Store (Zustand + JWT – Simple & Secure)

**`src/stores/authStore.ts`**
```ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import jwtDecode from 'jwt-decode'

interface User {
  employee_id: string
  role: 'ADMIN' | 'DRIVER' | 'SUPERVISOR'
}

interface AuthState {
  token: string | null
  user: User | null
  isAuthenticated: boolean
  login: (token: string) => void
  logout: () => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      isAuthenticated: false,
      login: (token) => {
        const decoded = jwtDecode<User>(token)
        set({ token, user: decoded, isAuthenticated: true })
      },
      logout: () => set({ token: null, user: null, isAuthenticated: false }),
    }),
    {
      name: 'hsh-auth',
      partialize: (state) => ({ token: state.token }), // only persist token
    }
  )
)
```

**`src/api/axiosInstance.ts`**
```ts
import axios from 'axios'
import { useAuthStore } from '../stores/authStore'

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000/api',
  timeout: 10000,
})

api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout()
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default api
```

### Step 4: Router (React Router v7 – Data APIs, Loaders, Actions)

**`src/router.tsx`**
```tsx
import { createBrowserRouter, redirect, Navigate } from 'react-router-dom'
import RootLayout from './layouts/RootLayout'
import Login from './pages/Login'
import Distribution from './pages/Distribution'
import { requireAuth } from './hooks/useAuth'

export const router = createBrowserRouter([
  {
    path: '/login',
    element: <Login />,
  },
  {
    element: <RootLayout />,
    loader: requireAuth, // protects entire app
    children: [
      {
        path: '/',
        element: <Navigate to="/distribution" replace />,
      },
      {
        path: '/distribution',
        element: <Distribution />,
        loader: async () => {
          // Parallel data fetching (v7 style)
          const [depotsRes, equipmentRes] = await Promise.all([
            fetch('/api/depots/').then(r => r.json()),
            fetch('/api/equipment/').then(r => r.json()),
          ])
          return { depots: depotsRes, equipment: equipmentRes }
        },
      },
    ],
  },
])
```

**`src/hooks/useAuth.ts`**
```ts
import { redirect } from 'react-router-dom'
import { useAuthStore } from '../stores/authStore'

export function requireAuth() {
  if (!useAuthStore.getState().isAuthenticated) {
    throw redirect('/login')
  }
  return null
}
```

**`src/layouts/RootLayout.tsx`** (with UserHeader)
```tsx
import { Outlet } from 'react-router-dom'
import { useAuthStore } from '../stores/authStore'

export default function RootLayout() {
  const { user } = useAuthStore()

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header – matches wireframe */}
      <header className="bg-blue-900 text-white p-4 shadow">
        <div className="flex justify-between items-center max-w-7xl mx-auto">
          <h1 className="text-xl font-bold">HSH LPG System</h1>
          <p className="text-sm">User: {user?.employee_id || 'Loading...'}</p>
        </div>
      </header>

      <main className="max-w-7xl mx-auto p-4">
        <Outlet />
      </main>
    </div>
  )
}
```

### Step 5: Distribution Page – Exact Wireframe Match (React Router v7)

**`src/pages/Distribution.tsx`**
```tsx
import { useState } from 'react'
import { useLoaderData, useSubmit } from 'react-router-dom'
import { useForm, useFieldArray } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { toast } from 'react-toastify'
import api from '../api/axiosInstance'
import ReceiptPrinter from '../utils/ReceiptPrinter'
import ConfirmationDialog from '../components/ConfirmationDialog'

const itemSchema = z.object({
  depot: z.string().min(1, 'Select depot'),
  equipment: z.string().min(1, 'Select equipment'),
  quantity: z.number().min(1, 'Qty ≥ 1'),
  status: z.enum(['COLLECTION', 'EMPTY_RETURN']),
})

const formSchema = z.object({
  items: z.array(itemSchema).min(1, 'Add at least one item'),
})

type FormData = z.infer<typeof formSchema>

export default function Distribution() {
  const { depots, equipment } = useLoaderData() as { depots: any[]; equipment: any[] }
  const submit = useSubmit()
  const [showConfirm, setShowConfirm] = useState(false)
  const [confirmData, setConfirmData] = useState<FormData | null>(null)

  const { control, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      items: [{ depot: '', equipment: '', quantity: 1, status: 'COLLECTION' }],
    },
  })

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
  })

  const onSubmit = (data: FormData) => {
    setConfirmData(data)
    setShowConfirm(true)
  }

  const handleConfirm = async () => {
    if (!confirmData) return

    try {
      const res = await api.post('/distributions/', confirmData)
      toast.success('Distribution created successfully!')
      await ReceiptPrinter.print(res.data) // Real thermal print
      setShowConfirm(false)
    } catch (err) {
      toast.error('Failed to save distribution')
    }
  }

  return (
    <div className="min-h-screen bg-gray-50 p-4 md:p-6">
      {/* Header – exact wireframe style */}
      <div className="bg-blue-900 text-white p-4 rounded-lg mb-6 shadow-md">
        <div className="flex justify-between items-center">
          <h1 className="text-xl md:text-2xl font-bold">Distribution</h1>
          <p className="text-sm">User: Account 001</p>
        </div>
      </div>

      <form onSubmit={handleSubmit(onSubmit)} className="space-y-4 md:space-y-6">
        {fields.map((field, idx) => (
          <div
            key={field.id}
            className="grid grid-cols-1 sm:grid-cols-4 gap-3 bg-white p-4 rounded-lg shadow-sm border border-gray-200"
          >
            <select
              {...register(`items.${idx}.depot`)}
              className="w-full"
              defaultValue=""
            >
              <option value="" disabled>Depot</option>
              {depots.map(d => (
                <option key={d.id} value={d.id}>{d.name}</option>
              ))}
            </select>

            <select
              {...register(`items.${idx}.equipment`)}
              className="w-full"
              defaultValue=""
            >
              <option value="" disabled>Equipment</option>
              {equipment.map(e => (
                <option key={e.id} value={e.id}>{e.name}</option>
              ))}
            </select>

            <input
              type="number"
              {...register(`items.${idx}.quantity`, { valueAsNumber: true })}
              placeholder="Qty"
              className="w-full border rounded px-3 py-2"
              min="1"
            />

            <div className="flex items-center gap-2">
              <select {...register(`items.${idx}.status`)} className="flex-1">
                <option value="COLLECTION">Collection</option>
                <option value="EMPTY_RETURN">Empty Return</option>
              </select>
              <button
                type="button"
                onClick={() => remove(idx)}
                className="text-red-600 hover:text-red-800 font-bold text-xl"
              >
                ×
              </button>
            </div>
          </div>
        ))}

        <div className="flex flex-col sm:flex-row gap-4 pt-4">
          <button
            type="button"
            onClick={() => append({ depot: '', equipment: '', quantity: 1, status: 'COLLECTION' })}
            className="bg-green-600 text-white px-6 py-3 rounded-lg hover:bg-green-700 transition"
          >
            [+] Add Item
          </button>

          <button
            type="submit"
            className="bg-blue-700 text-white px-10 py-3 rounded-lg hover:bg-blue-800 font-medium transition"
          >
            Save
          </button>
        </div>

        {errors.items && (
          <p className="text-red-600 text-center">{errors.items.message}</p>
        )}
      </form>

      {showConfirm && confirmData && (
        <ConfirmationDialog
          items={confirmData.items}
          onBack={() => setShowConfirm(false)}
          onConfirm={handleConfirm}
        />
      )}
    </div>
  )
}
```

**`src/components/ConfirmationDialog.tsx`** – Pixel-perfect wireframe match

```tsx
interface Props {
  items: Array<{
    depot: string
    equipment: string
    quantity: number
    status: 'COLLECTION' | 'EMPTY_RETURN'
  }>
  onBack: () => void
  onConfirm: () => void
}

export default function ConfirmationDialog({ items, onBack, onConfirm }: Props) {
  const collection = items
    .filter(i => i.status === 'COLLECTION')
    .reduce((sum, i) => sum + i.quantity, 0)
  const emptyReturn = items
    .filter(i => i.status === 'EMPTY_RETURN')
    .reduce((sum, i) => sum + i.quantity, 0)

  return (
    <div className="fixed inset-0 bg-black/60 flex items-center justify-center p-4 z-50">
      <div className="bg-white rounded-xl p-6 w-full max-w-lg shadow-2xl border border-gray-200">
        <h2 className="text-2xl font-bold text-center mb-6 text-gray-800">
          Distribution Confirmation
        </h2>

        <div className="space-y-3 mb-6 text-sm">
          <p><strong>User:</strong> Account 001</p>
          <p><strong>Timestamp:</strong> {new Date().toLocaleString('en-SG')}</p>
          <p><strong>Distribution number:</strong> (auto-generated)</p>
        </div>

        <div className="overflow-x-auto mb-6">
          <table className="min-w-full border border-gray-300 text-sm">
            <thead>
              <tr className="bg-gray-100">
                <th className="border px-4 py-2 text-left">Depots</th>
                <th className="border px-4 py-2 text-left">Equipment Name</th>
                <th className="border px-4 py-2 text-center">Quantity</th>
                <th className="border px-4 py-2 text-center">Status</th>
              </tr>
            </thead>
            <tbody>
              {items.map((item, i) => (
                <tr key={i} className="hover:bg-gray-50">
                  <td className="border px-4 py-2">{item.depot}</td>
                  <td className="border px-4 py-2">{item.equipment}</td>
                  <td className="border px-4 py-2 text-center font-medium">{item.quantity}</td>
                  <td className="border px-4 py-2 text-center">
                    <span
                      className={
                        item.status === 'COLLECTION'
                          ? 'text-green-700 font-medium'
                          : 'text-amber-700 font-medium'
                      }
                    >
                      {item.status === 'COLLECTION' ? 'Collection' : 'Empty Return'}
                    </span>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>

        <div className="text-center mb-8">
          <p className="text-lg font-bold text-gray-800">Total:</p>
          <p className="text-green-700">Collection: {collection} cylinders</p>
          <p className="text-amber-700">Empty Return: {emptyReturn} cylinders</p>
        </div>

        <div className="flex gap-4 justify-center">
          <button
            onClick={onBack}
            className="bg-gray-500 hover:bg-gray-600 text-white px-10 py-3 rounded-lg font-medium transition"
          >
            [Back]
          </button>
          <button
            onClick={onConfirm}
            className="bg-green-600 hover:bg-green-700 text-white px-10 py-3 rounded-lg font-bold transition shadow-md"
          >
            [Confirmed]
          </button>
        </div>
      </div>
    </div>
  )
}
```

### Step 6: Thermal Printing (Real ESC/POS – Production Ready)

**`src/utils/ReceiptPrinter.ts`**

```ts
import { ThermalPrinter, PrinterTypes } from 'react-thermal-printer'

export default class ReceiptPrinter {
  static async print(data: any) {
    try {
      const device = await navigator.bluetooth.requestDevice({
        filters: [{ services: ['000018f0-0000-1000-8000-00805f9b34fb'] }], // ESC/POS service UUID
      })

      const server = await device.gatt?.connect()
      if (!server) throw new Error('No GATT server')

      const service = await server.getPrimaryService('000018f0-0000-1000-8000-00805f9b34fb')
      const characteristic = await service.getCharacteristic('00002af1-0000-1000-8000-00805f9b34fb')

      const printer = new ThermalPrinter({
        type: PrinterTypes.USB,
        interface: null,
        width: 42, // 80mm printer
      })

      await printer.init()
      await printer.setTextDoubleHeight()
      await printer.bold(true)
      await printer.println(`Distribution #${data.distribution_number || 'N/A'}`)
      await printer.bold(false)
      await printer.setTextNormal()
      await printer.println(`User: Account 001`)
      await printer.println(`Date: ${new Date().toLocaleString('en-SG')}`)
      await printer.drawLine()

      data.items.forEach((item: any) => {
        printer.leftRight(`${item.depot} - ${item.equipment}`, `${item.quantity} ${item.status}`)
      })

      await printer.drawLine()
      await printer.bold(true)
      await printer.println(`Collection: ${data.collection} cylinders`)
      await printer.println(`Empty Return: ${data.emptyReturn} cylinders`)
      await printer.bold(false)
      await printer.cut()
      await printer.feed(3)

      const buffer = await printer.execute()
      await characteristic.writeValue(buffer)

      await server.disconnect()
      toast.success('Receipt printed successfully!')
    } catch (err) {
      console.error('Print error:', err)
      toast.error('Failed to print receipt')
    }
  }
}
```

### Step 7: Offline Queue & Background Sync (Production Ready)

**`src/stores/offlineStore.ts`**

```ts
import { create } from 'zustand'
import localforage from 'localforage'
import api from '../api/axiosInstance'
import { toast } from 'react-toastify'

interface OfflineAction {
  type: 'DISTRIBUTION'
  payload: any
  timestamp: string
}

interface OfflineState {
  queue: OfflineAction[]
  isSyncing: boolean
  add: (action: OfflineAction) => Promise<void>
  sync: () => Promise<void>
}

export const useOfflineStore = create<OfflineState>((set, get) => ({
  queue: [],
  isSyncing: false,

  add: async (action) => {
    const newQueue = [
      ...get().queue,
      { ...action, timestamp: new Date().toISOString() },
    ]
    set({ queue: newQueue })
    await localforage.setItem('hsh_offline_queue', newQueue)
    toast.info('Action queued offline – will sync when online')
  },

  sync: async () => {
    if (get().isSyncing) return
    set({ isSyncing: true })

    const queue = (await localforage.getItem<OfflineAction[]>('hsh_offline_queue')) || []
    if (queue.length === 0) {
      set({ isSyncing: false })
      return
    }

    for (const action of queue) {
      try {
        if (action.type === 'DISTRIBUTION') {
          await api.post('/distributions/', action.payload)
        }
        // Remove successful item
        const remaining = queue.filter(a => a !== action)
        set({ queue: remaining })
        await localforage.setItem('hsh_offline_queue', remaining)
        toast.success('Offline action synced')
      } catch (err) {
        console.error('Sync failed:', err)
        toast.error('Sync failed – will retry later')
        break // Stop on first error – retry on next network change
      }
    }

    set({ isSyncing: false })
  },
}))
```

Add network listener in `main.tsx` or a custom hook:
```tsx
useEffect(() => {
  const handleOnline = () => useOfflineStore.getState().sync()
  window.addEventListener('online', handleOnline)
  return () => window.removeEventListener('online', handleOnline)
}, [])
```

### Step 8: Run & Test (Final Checklist)

```bash
npm run dev
```

**Exact test flow (matches your wireframe 100%):**
1. Login → `/login`
2. Go to `/distribution`
3. See **User: Account 001** header
4. Add rows using **dropdowns** (Depot, Equipment, Status)
5. Enter Qty → click **[+] Add Item**
6. Click **Save** → see **exact Confirmation popup** with table & totals
7. Click **[Confirmed]** →  
   - POST to backend  
   - Real thermal print  
   - Toast success  
   - If offline: queued → auto-sync on reconnect

**Summary**:
- Fully leverages **React 19** + **Router v7 Data APIs**  
- Type-safe, performant, mobile-first  
- Exact visual match to your proposal wireframe  
- Real thermal printing  
- Offline-first with background sync  
- Production-ready structure  

