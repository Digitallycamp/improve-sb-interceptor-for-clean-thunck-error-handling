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
  (error) => {
    // This is the "Shield" logic
    let normalizedError: ApiError = {
      message: 'An unexpected error occurred.',
    };

    if (error.response) {
      // Server responded with a status code outside 2xx
      normalizedError = {
        message: error.response.data?.message || 'Server Error',
        code: error.response.status,
        errors: error.response.data?.errors, 
      };
    } else if (error.request) {
      // Request was made but no response received (Network issues)
      normalizedError = {
        message: 'Network error. Please check your connection.',
        code: 'NETWORK_ERROR',
      };
    } else {
      // Something else happened in setting up the request
      normalizedError.message = error.message;
    }

    // CRITICAL: We return a rejected promise with the NEW object
    return Promise.reject(normalizedError);
  }
);
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (!error.config || !error.response) {
      return Promise.reject(error);
    }
    const originalRequest = error.config as CustomAxiosRequestConfig;

    const isRefreshCall = originalRequest.url?.includes(
      '/admin/auth/refresh-token/exchange'
    );

    const isAuthCall = isAuthRoute(originalRequest.url);

    if (
      error.response?.status !== 401 ||
      originalRequest._retry ||
      isRefreshCall ||
      isAuthCall ||
      isFrontendAuthRoute()
    ) {
      return Promise.reject(error);
    }

    //  no token , don't refresh
    if (!tokenStore.get()) {
      return Promise.reject(error);
    }

    // don't refresh on auth pages
    if (isFrontendAuthRoute()) {
      return Promise.reject(error);
    }
    // ---- Queue while refreshing ----
    if (isRefreshing) {
      return new Promise<string | null>((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      }).then((token) => {
        if (!token) return Promise.reject(new Error('No token'));

        originalRequest.headers = originalRequest.headers || {};
        originalRequest.headers['Authorization'] = `Bearer ${token}`;

        return api(originalRequest);
      });
    }

    originalRequest._retry = true;
    isRefreshing = true;

    try {
      const newToken = await getRefreshPromise(
        async () => {
          const { data } = await refreshClient.post(
            '/admin/auth/refresh-token/exchange',
            {},
            { withCredentials: true }
          );

          if (!data.data.access_token) {
            throw new Error('Invalid refresh response');
          }

          return data.data.access_token;
        },
        (token) => {
          tokenStore.set(token);
        },
        () => {
          tokenStore.clear();
        }
      );
      tokenStore.set(newToken);

      // ---- Important: release lock BEFORE resolving queue ----
      isRefreshing = false;
      processQueue(null, newToken);

      originalRequest.headers = {
        ...(originalRequest.headers || {}),
        Authorization: `Bearer ${newToken}`,
      };

      return api(originalRequest);
    } catch (err: any) {
      tokenStore.clear();

      // ---- Network vs Auth failure ----
      if (!err?.response) {
        // Network issue → reject queue but DON'T force logout
        isRefreshing = false;
        processQueue(err, null);
        return Promise.reject(err);
      }

      // Auth failure → logout
      isRefreshing = false;
      processQueue(err, null);

      // logout if it's truly auth failure
      if (err?.response?.status === 401) {
        onAuthFailure?.();
      }

      return Promise.reject(err);
    }
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
