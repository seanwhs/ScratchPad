# Step-by-Step Tutorial: Building the Frontend for HSH LPG Cylinder Logistics System (MVP)

This tutorial guides you through building the React-based frontend for the HSH system, focusing on the MVP features as outlined in the architecture. We'll use React 18.x (or latest stable in Jan 2026), Vite for bundling, Tailwind CSS for styling, and integrate with the Django backend via REST APIs with JWT authentication. The core focus is the distribution workflow: data entry, confirmation, offline queuing, and thermal printing.

**Prerequisites:**
- Node.js 18+ and npm/yarn/pnpm installed.
- The backend server running (from the previous tutorial) at `http://localhost:8000` (adjust as needed).
- Basic familiarity with React, hooks, and REST APIs.
- Bluetooth-enabled device for testing thermal printing (e.g., ESC/POS printer).
- Git for version control (optional).

**Estimated Time:** 4-6 hours for a basic setup, plus testing.

**Final Structure Overview:**
Your project will look like this:
```
hsh_frontend/
├── src/
│   ├── api/
│   │   └── distributionApi.js     # API wrappers
│   ├── components/
│   │   ├── ConfirmationDialog.jsx # Safety confirmation
│   │   ├── DistributionForm.jsx   # Main input form
│   │   ├── DistributionTable.jsx  # Dynamic rows
│   │   ├── DynamicItemRow.jsx     # Single row
│   │   ├── ReceiptPrinter.jsx     # Thermal print
│   │   └── UserHeader.jsx         # User info display
│   ├── hooks/
│   │   └── useOfflineQueue.js     # Offline handling
│   ├── routes/
│   │   └── Distribution.jsx       # Main page
│   ├── App.jsx
│   ├── main.jsx
│   └── index.css                  # Tailwind imports
├── vite.config.js
├── tailwind.config.js
├── package.json
└── ... (other files from Vite)
```

---

## Step 1: Set Up the Vite + React Project
1. Create the project:
   ```
   npm create vite@latest hsh_frontend -- --template react
   cd hsh_frontend
   ```

2. Install core dependencies:
   ```
   npm install react-router-dom axios tailwindcss postcss autoprefixer @tanstack/react-query react-hook-form react-toastify localforage jwt-decode
   ```
   - `react-router-dom`: For routing.
   - `axios`: For API calls.
   - `tailwindcss`: Styling.
   - `@tanstack/react-query`: For data fetching and caching (optional but recommended for API sync).
   - `react-hook-form`: Form handling.
   - `react-toastify`: Notifications (e.g., "Queued offline").
   - `localforage`: Offline storage (IndexedDB wrapper for queue).
   - `jwt-decode`: Decode JWT for user info.

   For thermal printing, add `escpos-buffer` or similar (browser-compatible ESC/POS lib):
   ```
   npm install escpos-buffer
   ```

3. Initialize Tailwind:
   ```
   npx tailwindcss init -p
   ```
   Update `tailwind.config.js`:
   ```js
   /** @type {import('tailwindcss').Config} */
   export default {
     content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
     theme: { extend: {} },
     plugins: [],
   };
   ```

   Update `src/index.css`:
   ```css
   @tailwind base;
   @tailwind components;
   @tailwind utilities;
   ```

4. Clean up defaults: Remove unnecessary files from `src/` (e.g., App.css, logo).

---

## Step 2: Set Up Routing and Authentication
1. Create `src/App.jsx` for basic routing:
   ```jsx
   import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
   import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
   import { ToastContainer } from 'react-toastify';
   import 'react-toastify/dist/ReactToastify.css';
   import DistributionPage from './routes/Distribution.jsx';
   import LoginPage from './routes/Login.jsx'; // Create this later

   const queryClient = new QueryClient();

   function App() {
     return (
       <QueryClientProvider client={queryClient}>
         <Router>
           <Routes>
             <Route path="/login" element={<LoginPage />} />
             <Route path="/distribution" element={<DistributionPage />} />
             <Route path="/" element={<div>Dashboard Placeholder</div>} />
           </Routes>
         </Router>
         <ToastContainer position="top-center" autoClose={3000} />
       </QueryClientProvider>
     );
   }

   export default App;
   ```

2. Implement basic JWT auth in `src/api/authApi.js` (create the file):
   ```js
   import axios from 'axios';

   const API_BASE = 'http://localhost:8000/api'; // Adjust to your backend URL

   export const login = async (username, password) => {
     const response = await axios.post(`${API_BASE}/token/`, { username, password });
     localStorage.setItem('accessToken', response.data.access);
     localStorage.setItem('refreshToken', response.data.refresh);
     return response.data;
   };

   // Add interceptors for auth in a separate file if needed
   axios.interceptors.request.use(config => {
     const token = localStorage.getItem('accessToken');
     if (token) config.headers.Authorization = `Bearer ${token}`;
     return config;
   });
   ```

3. Create a simple `src/routes/Login.jsx` for testing:
   ```jsx
   import { useState } from 'react';
   import { useNavigate } from 'react-router-dom';
   import { login } from '../api/authApi';

   function LoginPage() {
     const [username, setUsername] = useState('');
     const [password, setPassword] = useState('');
     const navigate = useNavigate();

     const handleLogin = async () => {
       try {
         await login(username, password);
         navigate('/distribution');
       } catch (error) {
         alert('Login failed');
       }
     };

     return (
       <div className="p-4">
         <input type="text" value={username} onChange={e => setUsername(e.target.value)} placeholder="Username" className="border p-2 mb-2" />
         <input type="password" value={password} onChange={e => setPassword(e.target.value)} placeholder="Password" className="border p-2 mb-2" />
         <button onClick={handleLogin} className="bg-blue-500 text-white p-2">Login</button>
       </div>
     );
   }

   export default LoginPage;
   ```

---

## Step 3: Implement API Wrappers
Create `src/api/distributionApi.js`:
```js
import axios from 'axios';

const API_BASE = 'http://localhost:8000/api';

export const createDistribution = async (payload) => {
  const response = await axios.post(`${API_BASE}/distributions/`, payload);
  return response.data;
};

export const getDistributions = async () => {
  const response = await axios.get(`${API_BASE}/distributions/list/`);
  return response.data;
};

// Add more as needed (e.g., fetch depots, equipment)
export const getDepots = async () => {
  // Assume endpoint /depots/
  const response = await axios.get(`${API_BASE}/depots/`);
  return response.data;
};

export const getEquipment = async () => {
  // Assume /equipment/
  const response = await axios.get(`${API_BASE}/equipment/`);
  return response.data;
};
```

(Note: You'll need to add corresponding views/serializers in backend for depots/equipment if not already done.)

---

## Step 4: Implement Offline Queue Hook
Create `src/hooks/useOfflineQueue.js`:
```js
import { useEffect, useState } from 'react';
import localforage from 'localforage';
import { toast } from 'react-toastify';
import { createDistribution } from '../api/distributionApi';

const QUEUE_KEY = 'offlineQueue';

export const useOfflineQueue = () => {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  const addToQueue = async (item) => {
    const queue = (await localforage.getItem(QUEUE_KEY)) || [];
    queue.push(item);
    await localforage.setItem(QUEUE_KEY, queue);
    toast('Added to offline queue');
  };

  const syncQueue = async () => {
    const queue = (await localforage.getItem(QUEUE_KEY)) || [];
    for (const item of queue) {
      try {
        if (item.type === 'distribution') {
          await createDistribution(item.payload);
        }
        // Remove from queue
        const newQueue = queue.filter(q => q !== item);
        await localforage.setItem(QUEUE_KEY, newQueue);
        toast('Synced offline item');
      } catch (error) {
        console.error('Sync failed:', error);
        break; // Stop if error, retry later
      }
    }
  };

  useEffect(() => {
    if (isOnline) syncQueue();
  }, [isOnline]);

  return { isOnline, addToQueue };
};
```

This hook queues distributions offline and auto-syncs when online.

---

## Step 5: Build Core Components
1. **UserHeader.jsx** (simple user display):
   ```jsx
   import { useEffect, useState } from 'react';
   import jwtDecode from 'jwt-decode';

   function UserHeader({ user }) {
     const [decodedUser, setDecodedUser] = useState(null);

     useEffect(() => {
       const token = localStorage.getItem('accessToken');
       if (token) setDecodedUser(jwtDecode(token));
     }, []);

     return (
       <div className="p-4 bg-blue-600 text-white">
         User: {decodedUser?.username || 'Guest'}
       </div>
     );
   }

   export default UserHeader;
   ```

2. **DynamicItemRow.jsx** (single row input):
   ```jsx
   import { useFormContext } from 'react-hook-form';

   function DynamicItemRow({ index, onRemove, depots, equipment }) {
     const { register } = useFormContext();

     return (
       <div className="flex gap-2 mb-2">
         <select {...register(`items.${index}.depot`)} className="border p-2 flex-1">
           <option>Select Depot</option>
           {depots.map(d => <option key={d.id} value={d.id}>{d.name}</option>)}
         </select>
         <select {...register(`items.${index}.equipment`)} className="border p-2 flex-1">
           <option>Select Equipment</option>
           {equipment.map(e => <option key={e.id} value={e.id}>{e.name}</option>)}
         </select>
         <input type="number" {...register(`items.${index}.quantity`)} placeholder="Qty" className="border p-2 w-20" />
         <select {...register(`items.${index}.status`)} className="border p-2">
           <option>Collection</option>
           <option>Empty Return</option>
         </select>
         <button onClick={() => onRemove(index)} className="bg-red-500 text-white p-2">X</button>
       </div>
     );
   }

   export default DynamicItemRow;
   ```

3. **DistributionTable.jsx** (list of rows):
   ```jsx
   import { useFieldArray, useForm } from 'react-hook-form';
   import DynamicItemRow from './DynamicItemRow';

   function DistributionTable({ depots, equipment, onSubmit }) {
     const { control, handleSubmit } = useForm({ defaultValues: { items: [] } });
     const { fields, append, remove } = useFieldArray({ control, name: 'items' });

     const handleAdd = () => append({ depot: '', equipment: '', quantity: 0, status: 'Collection' });

     return (
       <form onSubmit={handleSubmit(onSubmit)}>
         {fields.map((field, index) => (
           <DynamicItemRow key={field.id} index={index} onRemove={remove} depots={depots} equipment={equipment} />
         ))}
         <button type="button" onClick={handleAdd} className="bg-green-500 text-white p-2 mb-4">Add Row</button>
         <button type="submit" className="bg-blue-500 text-white p-2">Save</button>
       </form>
     );
   }

   export default DistributionTable;
   ```

4. **ConfirmationDialog.jsx** (safety net):
   ```jsx
   function ConfirmationDialog({ items, onBack, onConfirm }) {
     const totals = {
       collection: items.reduce((sum, i) => i.status === 'Collection' ? sum + parseInt(i.quantity) : sum, 0),
       return: items.reduce((sum, i) => i.status === 'Empty Return' ? sum + parseInt(i.quantity) : sum, 0),
     };

     return (
       <div className="fixed inset-0 bg-black/50 flex items-center justify-center p-4">
         <div className="bg-white p-6 rounded-lg max-w-md w-full">
           <h2 className="text-lg font-bold mb-4">Confirm Distribution</h2>
           <ul className="mb-4">
             {items.map((item, i) => (
               <li key={i}>{item.depot} - {item.equipment} - {item.quantity} ({item.status})</li>
             ))}
           </ul>
           <p>Collection: {totals.collection}</p>
           <p>Empty Return: {totals.return}</p>
           <div className="flex gap-4 mt-4">
             <button onClick={onBack} className="bg-gray-300 p-2 flex-1">Back</button>
             <button onClick={onConfirm} className="bg-green-600 text-white p-2 flex-1">Confirm</button>
           </div>
         </div>
       </div>
     );
   }

   export default ConfirmationDialog;
   ```

5. **ReceiptPrinter.jsx** (thermal print via Web Bluetooth):
   ```jsx
   import escpos from 'escpos-buffer'; // Adjust based on lib

   async function printReceipt(data) {
     // Request Bluetooth device
     const device = await navigator.bluetooth.requestDevice({ filters: [{ services: ['000018f0-0000-1000-8000-00805f9b34fb'] }] }); // ESC/POS service
     const server = await device.gatt.connect();
     const service = await server.getPrimaryService('000018f0-0000-1000-8000-00805f9b34fb');
     const characteristic = await service.getCharacteristic('00002af1-0000-1000-8000-00805f9b34fb');

     // Build ESC/POS commands
     const buffer = new escpos.Buffer();
     buffer.text(`Distribution: ${data.distribution_number}\n`);
     buffer.text(`Collection: ${data.total_collection}\n`);
     buffer.text(`Return: ${data.total_return}\n`);
     buffer.cut();

     await characteristic.writeValue(buffer.get());
     server.disconnect();
   }

   export default printReceipt;
   ```

   (Note: Web Bluetooth for printers; test on mobile. Fallback to alert if not supported.)

6. **DistributionForm.jsx** (wrapper if needed; can combine into table).

---

## Step 6: Assemble the Main Page
Create `src/routes/Distribution.jsx`:
```jsx
import { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { toast } from 'react-toastify';
import { useOfflineQueue } from '../hooks/useOfflineQueue';
import { createDistribution } from '../api/distributionApi';
import { getDepots, getEquipment } from '../api/distributionApi'; // Adjust
import DistributionTable from '../components/DistributionTable';
import ConfirmationDialog from '../components/ConfirmationDialog';
import printReceipt from '../components/ReceiptPrinter';
import UserHeader from '../components/UserHeader';

function DistributionPage() {
  const [items, setItems] = useState([]);
  const [showConfirm, setShowConfirm] = useState(false);
  const [depots, setDepots] = useState([]);
  const [equipment, setEquipment] = useState([]);
  const { isOnline, addToQueue } = useOfflineQueue();
  const navigate = useNavigate();

  useEffect(() => {
    // Fetch master data
    getDepots().then(setDepots);
    getEquipment().then(setEquipment);
  }, []);

  const handleSubmit = (data) => {
    setItems(data.items);
    setShowConfirm(true);
  };

  const handleConfirm = async () => {
    const payload = { items }; // Add user if needed
    try {
      if (!isOnline) {
        addToQueue({ type: 'distribution', payload });
        toast('Queued for sync');
        navigate('/');
        return;
      }
      const response = await createDistribution(payload);
      await printReceipt(response);
      toast('Distribution created');
      navigate('/');
    } catch (error) {
      toast.error('Error');
    }
    setShowConfirm(false);
  };

  return (
    <div className="p-4">
      <UserHeader />
      <DistributionTable depots={depots} equipment={equipment} onSubmit={handleSubmit} />
      {showConfirm && <ConfirmationDialog items={items} onBack={() => setShowConfirm(false)} onConfirm={handleConfirm} />}
    </div>
  );
}

export default DistributionPage;
```

---

## Step 7: Testing and Running
1. Run the app:
   ```
   npm run dev
   ```
   Access at `http://localhost:5173` (Vite default). Test on mobile via network IP.

2. Test flow:
   - Login → Add rows → Save → Confirm → Check backend DB/audit.
   - Simulate offline: Disconnect network, submit, reconnect to sync.

3. Add validations: Use `react-hook-form` resolvers for positive quantities.

4. UI Polish: Add mobile responsiveness (Tailwind classes like `sm:`, `md:`).

---

## Step 8: Polish and Next Steps
- **Error Handling:** Global error boundaries.
- **PIN for Confirm:** Add input in dialog for extra safety.
- **Master Data Caching:** Use React Query for depots/equipment.
- **Dark Mode:** Tailwind `dark:` classes.
- **Deployment:** Build with `npm run build`, serve via Nginx or Vercel.
- **Integration Testing:** Use Cypress for E2E tests (e.g., offline scenarios).

This frontend emphasizes safety with confirmation and offline support. Test on actual devices in Singapore's variable connectivity—focus on UX speed for field users. If issues, check React docs or reach out!
