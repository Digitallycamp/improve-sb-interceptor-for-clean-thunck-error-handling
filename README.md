# improve-sb-interceptor-for-clean-thunck-error-handling


```batch

import axios, { type AxiosRequestConfig } from 'axios';
import { tokenStore } from './tokenStore';
import { getRefreshPromise } from './refreshManager';

// . Define a standard error structure
export interface ApiError {
  message: string;
  code?: string | number;
  errors?: Record<string, string[]>; // For validation errors
}

interface CustomAxiosRequestConfig extends AxiosRequestConfig {
  _retry?: boolean;
}

// ---- Auth failure handler (decoupled) ----
let onAuthFailure: (() => void) | null = null;

export const setAuthFailureHandler = (handler: () => void) => {
  onAuthFailure = handler;
};

// ---- API clients ----
const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,
});

const refreshClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,
});

// STOP REFRESH ON AUTH ROUTES FE AND BE
const FRONTEND_AUTH_ROUTES = [
  `/${import.meta.env.VITE_ADMIN_LOGIN_ROUTE}`,
  `/${import.meta.env.VITE_ADMIN_FORGOT_PASSWORD_ROUTE}`,
  `/${import.meta.env.VITE_ADMIN_RESET_PASSWORD_ROUTE}`,
];

const isFrontendAuthRoute = () => {
  const path = window.location.pathname;
  return FRONTEND_AUTH_ROUTES.some((route) => path.includes(route));
};
// ---- AUTH ROUTES (backend endpoints) ----
const AUTH_ROUTES = [
  '/admin/auth/sign-in',
  '/admin/auth/2fa',
  '/admin/auth/forgot-password',
  '/admin/auth/reset-password',
];

// ---- helper ----
const isAuthRoute = (url?: string) => {
  return AUTH_ROUTES.some((route) => url?.includes(route));
};
// ---- Request interceptor (immutable headers) ----
api.interceptors.request.use((config) => {
  // ✅ Don't attach token to refresh call — it only needs the cookie
  if (
    config.url?.includes('/admin/auth/refresh-token/exchange') ||
    isAuthRoute(config.url)
  ) {
    return config;
  }
  const token = tokenStore.get();

  if (token) {
    // @ts-expect-error i dont want to touch it
    config.headers = {
      ...(config.headers || {}),
      Authorization: `Bearer ${token}`,
    };
  }

  return config;
});

// ---- Refresh state ----
let isRefreshing = false;

let failedQueue: Array<{
  resolve: (token: string | null) => void;
  reject: (error: unknown) => void;
}> = [];

const processQueue = (error: unknown, token: string | null = null) => {
  failedQueue.forEach((p) => {
    if (error) p.reject(error);
    else p.resolve(token);
  });
  failedQueue = [];
};

// ---- Response interceptor ----


api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config as CustomAxiosRequestConfig;

    // 1. HANDLER: TOKEN REFRESH LOGIC
    // We only try to refresh if it's a 401 and not an auth/refresh route
    const shouldAttemptRefresh = 
      error.response?.status === 401 && 
      !originalRequest._retry && 
      !originalRequest.url?.includes('/admin/auth/refresh-token/exchange') &&
      !isAuthRoute(originalRequest.url) &&
      !isFrontendAuthRoute();

    if (shouldAttemptRefresh) {
      if (!tokenStore.get()) return Promise.reject(error);

      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then((token) => {
          originalRequest.headers!['Authorization'] = `Bearer ${token}`;
          return api(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const newToken = await getRefreshPromise(/* ... your refresh logic ... */);
        tokenStore.set(newToken);
        isRefreshing = false;
        processQueue(null, newToken);
        
        originalRequest.headers!['Authorization'] = `Bearer ${newToken}`;
        return api(originalRequest);
      } catch (refreshError: any) {
        isRefreshing = false;
        processQueue(refreshError, null);
        tokenStore.clear();
        if (refreshError.response?.status === 401) onAuthFailure?.();
        
        // Pass the error down to the "Shield" logic below
        error = refreshError; 
      }
    }

    // 2. THE SHIELD: NORMALIZE ERROR
    // This runs for EVERY error that wasn't fixed by the refresh logic
    let normalizedError: ApiError = {
      message: 'An unexpected error occurred.',
    };

    if (error.response) {
      normalizedError = {
        message: error.response.data?.message || 'Server Error',
        code: error.response.status,
        errors: error.response.data?.errors,
      };
    } else if (error.request) {
      normalizedError = {
        message: 'Network error. Please check your connection.',
        code: 'NETWORK_ERROR',
      };
    } else {
      normalizedError.message = error.message;
    }

    return Promise.reject(normalizedError);
  }
);

export default api;
```

## THUNCK BECOMES 

```batch
export const getComissionSettings = createAsyncThunk(
    'admin/getComissionSettings',
    async (_, { rejectWithValue }) => {
        try {
            const response = await client.adminGetAllComissionSettings();
            return response.data;
        } catch (error: any) {
            if (!error.response) {
                return rejectWithValue(error.message ?? 'Network error. Please try again.');
            }
            return rejectWithValue(error.response.data);
        }
    }
);
```

to 

```batch
export const getCommissionSettings = createAsyncThunk(
  'admin/getCommissionSettings',
  async (_, { rejectWithValue }) => {
    try {
      const { data } = await client.adminGetAllComissionSettings();
      return data; // Success logic stays simple
    } catch (error) {
      // 'error' is now guaranteed to be our ApiError object 
      // because the interceptor caught it first!
      return rejectWithValue(error);
    }
  }
);
```
