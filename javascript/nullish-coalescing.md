# Nullish Coalescing vs OR Operator

<div align="center">

![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![ES2020](https://img.shields.io/badge/ES2020-Nullish_Coalescing-orange?style=for-the-badge)

*The `??` operator only checks for `null` or `undefined`, unlike `||` which checks for any falsy value.*

![JavaScript](https://upload.wikimedia.org/wikipedia/commons/6/6a/JavaScript-logo.png)

</div>

## The Problem with `||`

```javascript
const port = userPort || 3000;

// These all return 3000 (unexpected!)
userPort = 0;         // 0 || 3000 = 3000
userPort = "";        // "" || 3000 = 3000
userPort = false;     // false || 3000 = 3000

// These correctly return 3000
userPort = null;      // null || 3000 = 3000
userPort = undefined; // undefined || 3000 = 3000
```

## Solution with `??`

```javascript
const port = userPort ?? 3000;

// 0 is a valid port number!
userPort = 0;         // 0 ?? 3000 = 0
userPort = "";        // "" ?? 3000 = ""
userPort = false;     // false ?? 3000 = false

// Only null/undefined trigger fallback
userPort = null;      // null ?? 3000 = 3000
userPort = undefined; // undefined ?? 3000 = 3000
```

## Real-World Examples

```javascript
// Config with valid falsy values
const config = {
  retries: userConfig.retries ?? 3,      // 0 retries is valid
  debug: userConfig.debug ?? false,       // false is valid
  timeout: userConfig.timeout ?? 5000,    // 0 timeout is valid
};

// API response handling
const count = response.count ?? 0;        // null becomes 0
const name = response.name ?? "Unknown";  // undefined becomes "Unknown"
```

## Combining with Optional Chaining

```javascript
const city = user?.address?.city ?? "Unknown";
```

---

*Learned: December 20, 2025*
*Tags: JavaScript, ES2020, Operators*
