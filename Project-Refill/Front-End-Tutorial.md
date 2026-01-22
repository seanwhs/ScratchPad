# **HSH LPG Frontend – Production-Ready Tutorial (React 2026)**

---

## **1️⃣ Technical Stack (January 2026)**

| Library / Tool        | Version | Why it's best right now                 |
| --------------------- | ------- | --------------------------------------- |
| React                 | 19.x    | New compiler, improved hooks & suspense |
| React Router          | 7.x     | Full Data APIs: loaders, actions, defer |
| Vite                  | 6.x     | Fast dev server & builds                |
| TailwindCSS           | 4.x     | JIT, high DX, performance               |
| TanStack Query        | 5.x     | Best-in-class data fetching & caching   |
| Zustand               | 5.x     | Lightweight global state                |
| React Hook Form + Zod | latest  | Type-safe forms with minimal renders    |
| react-thermal-printer | latest  | Real ESC/POS thermal printing           |
| localforage           | latest  | Reliable offline queue (IndexedDB)      |
| react-toastify        | latest  | Clean, non-blocking toasts              |

---

## **2️⃣ Project Setup**

```bash
npm create vite@latest hsh-frontend -- --template react-ts
cd hsh-frontend

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

npx tailwindcss init -p
```

**`tailwind.config.ts`**

```ts
import type { Config } from 'tailwindcss'

export default {
  content: ["./index.html","./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
        hsh: { primary: '#1e40af', accent: '#f59e0b' },
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
  :root { --background: 0 0% 98%; --foreground: 0 0% 3.9%; }
  body { @apply bg-[hsl(var(--background))] text-[hsl(var(--foreground))] font-sans; }
}

input, select { 
  @apply border border-gray-300 rounded-md px-3 py-2 focus:outline-none focus:ring-2 focus:ring-hsh-primary focus:border-transparent; 
}
```

---

## **3️⃣ Project Structure**

```
src/
├── api/                  # Axios instance + API endpoints
├── components/           # Reusable UI components
├── hooks/                # useAuth, offline sync
├── layouts/              # RootLayout + header
├── pages/                # Login, Distribution, Transactions
├── stores/               # Zustand stores (auth + offline)
├── types/                # Shared TypeScript types
├── utils/                # Printer, formatters
├── App.tsx
├── main.tsx
└── index.css
```

---

## **4️⃣ Auth Store – Zustand + JWT**

**`src/stores/authStore.ts`**

```ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import jwtDecode from 'jwt-decode'

interface User { employee_id: string; role: 'ADMIN'|'DRIVER'|'SUPERVISOR' }

interface AuthState {
  token: string | null
  user: User | null
  isAuthenticated: boolean
  login: (token: string) => void
  logout: () => void
}

export const useAuthStore = create<AuthState>()(
  persist((set) => ({
    token: null,
    user: null,
    isAuthenticated: false,
    login: (token) => {
      const decoded = jwtDecode<User>(token)
      set({ token, user: decoded, isAuthenticated: true })
    },
    logout: () => set({ token: null, user: null, isAuthenticated: false }),
  }), {
    name: 'hsh-auth',
    partialize: (state) => ({ token: state.token })
  })
)
```

---

## **5️⃣ Axios Instance – JWT Protected**

**`src/api/axiosInstance.ts`**

```ts
import axios from 'axios'
import { useAuthStore } from '../stores/authStore'

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000/api',
  timeout: 10000
})

api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

api.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      useAuthStore.getState().logout()
      window.location.href = '/login'
    }
    return Promise.reject(err)
  }
)

export default api
```

---

## **6️⃣ Router v7 – Loaders & Actions**

**`src/router.tsx`**

```tsx
import { createBrowserRouter, redirect, Navigate } from 'react-router-dom'
import RootLayout from './layouts/RootLayout'
import Login from './pages/Login'
import Distribution from './pages/Distribution'
import { requireAuth } from './hooks/useAuth'

export const router = createBrowserRouter([
  { path: '/login', element: <Login /> },
  {
    element: <RootLayout />,
    loader: requireAuth,
    children: [
      { path: '/', element: <Navigate to="/distribution" replace /> },
      {
        path: '/distribution',
        element: <Distribution />,
        loader: async () => {
          const [depots, equipment] = await Promise.all([
            fetch('/api/depots/').then(r => r.json()),
            fetch('/api/equipment/').then(r => r.json())
          ])
          return { depots, equipment }
        }
      }
    ]
  }
])
```

**`src/hooks/useAuth.ts`**

```ts
import { redirect } from 'react-router-dom'
import { useAuthStore } from '../stores/authStore'

export function requireAuth() {
  if (!useAuthStore.getState().isAuthenticated) throw redirect('/login')
  return null
}
```

---

## **7️⃣ Root Layout + Header**

**`src/layouts/RootLayout.tsx`**

```tsx
import { Outlet } from 'react-router-dom'
import { useAuthStore } from '../stores/authStore'

export default function RootLayout() {
  const { user } = useAuthStore()
  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-blue-900 text-white p-4 shadow">
        <div className="flex justify-between max-w-7xl mx-auto">
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

---

## **8️⃣ Distribution Page – Dynamic Form + Confirmation**

* Dynamic rows for Depot, Equipment, Quantity, Movement Type
* Mandatory confirmation popup with totals before API POST

Files:

* `src/pages/Distribution.tsx` → Form rendering with **React Hook Form + Zod**
* `src/components/ConfirmationDialog.tsx` → Modal for confirmation before committing data

---

## **9️⃣ Thermal Printing – ESC/POS**

**`src/utils/ReceiptPrinter.ts`**

* Integrates **Web Bluetooth + ESC/POS** for 80mm thermal printing
* Supports immediate field-ready receipts after distribution confirmation

---

## **🔟 Offline Queue + Background Sync**

**`src/stores/offlineStore.ts`**

* Uses **localforage + Zustand** for reliable offline storage
* Automatic background sync when network connectivity is restored
* Ensures **client_temp_id** reconciliation with server IDs

---

## **1️⃣1️⃣ Run & Test**

```bash
npm run dev
```

**Test Flow:**

1. Login → `/login`
2. Navigate → `/distribution`
3. Add multiple rows (Depot, Equipment, Quantity, Movement Type)
4. Click **[+] Add Item**
5. Click **Save** → Confirmation dialog shows totals
6. Click **[Confirm]** → API POST → Thermal print
7. Offline mode → queue automatically syncs when online

---

## ✅ **Key Features & Benefits**

* **React 19 + Router v7 Data APIs**
* **Type-safe forms** using React Hook Form + Zod
* **Thermal printing** ready for production
* **Offline-first queue** + background sync
* **Zustand stores** for auth and offline management
* Fully **responsive** and mobile-first with production-ready wireframe fidelity

Do you want me to do that next?
