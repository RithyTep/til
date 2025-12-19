# Using AbortController for Fetch Cancellation

AbortController lets you cancel fetch requests, useful for React cleanup and timeouts.

## Basic Cancellation

```javascript
const controller = new AbortController();

fetch('/api/data', { signal: controller.signal })
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(err => {
    if (err.name === 'AbortError') {
      console.log('Fetch was cancelled');
    }
  });

// Cancel the request
controller.abort();
```

## React useEffect Cleanup

```javascript
useEffect(() => {
  const controller = new AbortController();

  const fetchData = async () => {
    try {
      const response = await fetch('/api/users', {
        signal: controller.signal
      });
      const data = await response.json();
      setUsers(data);
    } catch (err) {
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    }
  };

  fetchData();

  // Cleanup: cancel on unmount
  return () => controller.abort();
}, []);
```

## Timeout with AbortSignal.timeout()

```javascript
// Cancel after 5 seconds
fetch('/api/slow-endpoint', {
  signal: AbortSignal.timeout(5000)
});
```

## Multiple Requests with Same Controller

```javascript
const controller = new AbortController();

Promise.all([
  fetch('/api/users', { signal: controller.signal }),
  fetch('/api/posts', { signal: controller.signal }),
  fetch('/api/comments', { signal: controller.signal })
]);

// Cancel all three requests at once
controller.abort();
```

## Custom Abort Reason

```javascript
controller.abort(new Error('User navigated away'));
```

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: JavaScript, Fetch, AbortController, React*
