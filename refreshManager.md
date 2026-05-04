```batch

// # global refresh manager
// Route ALL refresh logic through ONE shared promise

let refreshPromise: Promise<string> | null = null;

export const getRefreshPromise = (
  refreshFn: () => Promise<string>,
  onSuccess: (token: string) => void,
  onError?: () => void
) => {
  if (!refreshPromise) {
    refreshPromise = refreshFn()
      .then((token) => {
        onSuccess(token); // ✅ single execution
        refreshPromise = null;
        return token;
      })
      .catch((err) => {
        refreshPromise = null;
        onError?.(); // ✅ single execution
        throw err;
      });
  }

  return refreshPromise;
};

```
