# 7. React Architecture

## The Central Challenge

React is a library, not a framework. It renders UI. Everything else — data fetching, state management, routing, caching — you bring yourself. This is powerful but requires deliberate decisions about where different kinds of logic live.

The most common failure mode in React codebases is state being in the wrong place, leading to components that are impossible to test, prop drilling that makes refactoring painful, and inconsistent UI behavior when the same data is displayed in multiple places.

This section covers the decisions that prevent those problems.

---

## Component Design

### One responsibility per component

A component should do one thing. If you find yourself naming a component `UserProfileFormWithAvatarAndSettings`, that is a signal it should be split. The guideline from Brad Frost's Atomic Design and from React's own documentation is consistent: small, composable, single-purpose components are easier to reason about, test, and reuse.

### Separate UI from data fetching

Components that fetch their own data are harder to test and harder to reuse. The pattern that works at scale is:

- **Page components** handle data fetching (or delegate to hooks that do)
- **Feature components** receive data as props and render it
- **UI components (atoms)** are pure presentational — no data fetching, minimal local state

```
pages/OrdersPage.tsx       ← fetches data, handles loading/error states
  └── features/orders/
        └── OrderList.tsx  ← receives orders as props, renders the list
              └── ui/
                    └── OrderCard.tsx  ← receives one order, renders a card
```

---

## State Management

React state comes in several kinds. Using the right tool for each kind prevents unnecessary complexity.

### Server state vs. client state

**Server state** is data that lives on the server and is synchronized to the client: user profiles, orders, products. It has a lifecycle (loading, success, error, stale) and needs to stay consistent with the server.

**Client state** is ephemeral UI state: is this modal open, what is the current filter value, is this accordion expanded. It does not need to be synchronized with anything.

The mistake many teams make is managing server state with a client state tool (Context, Zustand, Redux). This requires you to manually handle loading states, caching, refetching, and invalidation — all things that TanStack Query (formerly React Query) handles automatically.

**Rule of thumb:**
- Server state → TanStack Query
- Global client state → Zustand or React Context (for small-to-medium state)
- Local component state → `useState`
- URL-derived state → React Router's `useSearchParams`

### TanStack Query

TanStack Query is the standard choice for server state management in React. It handles caching, background refetching, cache invalidation, loading and error states, and pagination — all without you writing a reducer.

```typescript
// features/orders/hooks/useOrders.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { ordersApi } from '../api/orders.api';
import { CreateOrderDto } from '@project/shared';

export const orderKeys = {
  all: ['orders'] as const,
  byId: (id: string) => ['orders', id] as const,
  byUser: (userId: string) => ['orders', 'user', userId] as const,
};

export function useOrders() {
  return useQuery({
    queryKey: orderKeys.all,
    queryFn: ordersApi.getAll,
    staleTime: 1000 * 60, // Consider data fresh for 1 minute
  });
}

export function useOrder(orderId: string) {
  return useQuery({
    queryKey: orderKeys.byId(orderId),
    queryFn: () => ordersApi.getById(orderId),
    enabled: !!orderId,
  });
}

export function useCreateOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (dto: CreateOrderDto) => ordersApi.create(dto),
    onSuccess: () => {
      // Invalidate the orders list so it refetches after a new order is created
      queryClient.invalidateQueries({ queryKey: orderKeys.all });
    },
  });
}
```

### Zustand for client state

For global client state (theme preference, sidebar open/closed, multi-step form state), Zustand is a lean alternative to Redux that does not require boilerplate:

```typescript
// lib/stores/ui.store.ts
import { create } from 'zustand';

interface UIState {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
  setSidebarOpen: (open: boolean) => void;
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
}));
```

---

## Data Fetching Pattern (API Client)

Centralize all API calls behind a typed API client. Do not scatter `fetch` calls throughout components or hooks.

```typescript
// lib/api-client.ts
import axios, { AxiosError } from 'axios';
import { AppError } from '@project/shared';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true, // Required to send httpOnly cookies
  headers: {
    'Content-Type': 'application/json',
  },
});

// Intercept 401 responses and attempt a token refresh
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as typeof error.config & { _retry?: boolean };

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        await apiClient.post('/auth/refresh');
        return apiClient(originalRequest);
      } catch {
        // Refresh failed — redirect to login
        window.location.href = '/login';
      }
    }

    return Promise.reject(error);
  }
);
```

```typescript
// features/orders/api/orders.api.ts
import { apiClient } from '../../../lib/api-client';
import { Order, CreateOrderDto } from '@project/shared';

interface ApiResponse<T> {
  status: 'success';
  data: T;
}

export const ordersApi = {
  getAll: async (): Promise<Order[]> => {
    const res = await apiClient.get<ApiResponse<Order[]>>('/orders');
    return res.data.data;
  },

  getById: async (id: string): Promise<Order> => {
    const res = await apiClient.get<ApiResponse<Order>>(`/orders/${id}`);
    return res.data.data;
  },

  create: async (dto: CreateOrderDto): Promise<Order> => {
    const res = await apiClient.post<ApiResponse<Order>>('/orders', dto);
    return res.data.data;
  },
};
```

---

## Routing with React Router

Define routes in one place:

```typescript
// router/index.tsx
import { createBrowserRouter } from 'react-router-dom';
import { RootLayout } from '../shared/components/layout/RootLayout';
import { ProtectedRoute } from '../shared/components/ProtectedRoute';
import { DashboardPage } from '../features/dashboard/pages/DashboardPage';
import { OrdersPage } from '../features/orders/pages/OrdersPage';
import { OrderDetailPage } from '../features/orders/pages/OrderDetailPage';
import { LoginPage } from '../features/auth/pages/LoginPage';
import { NotFoundPage } from '../shared/components/NotFoundPage';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      {
        element: <ProtectedRoute />,
        children: [
          { path: 'dashboard', element: <DashboardPage /> },
          { path: 'orders', element: <OrdersPage /> },
          { path: 'orders/:id', element: <OrderDetailPage /> },
        ],
      },
    ],
  },
  { path: '/login', element: <LoginPage /> },
  { path: '*', element: <NotFoundPage /> },
]);
```

```typescript
// shared/components/ProtectedRoute.tsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

export function ProtectedRoute() {
  const { user, isLoading } = useAuth();

  if (isLoading) return <div>Loading...</div>; // Replace with skeleton

  if (!user) return <Navigate to="/login" replace />;

  return <Outlet />;
}
```

---

## Error Boundaries

React error boundaries catch rendering errors and prevent the entire application from crashing. Every major section of the app should be wrapped in one.

```typescript
// shared/components/ErrorBoundary.tsx
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  state = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // Send to error tracking (Sentry, etc.)
    console.error('Uncaught error:', error, info);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <div>Something went wrong. Please refresh the page.</div>;
    }
    return this.props.children;
  }
}
```

---

## Sources

- Facebook/Meta. React Documentation. https://react.dev — Especially "Thinking in React" and "Managing State."
- TanStack Query Documentation. https://tanstack.com/query/latest
- Zustand Documentation. https://github.com/pmndrs/zustand
- React Router Documentation. https://reactrouter.com/en/main
- Frost, Brad. *Atomic Design*. 2016. https://atomicdesign.bradfrost.com — Component hierarchy principles.
- Kent C. Dodds. "Application State Management with React." https://kentcdodds.com/blog/application-state-management-with-react
- Kent C. Dodds. "Colocation." https://kentcdodds.com/blog/colocation — Where state should live.
