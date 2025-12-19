# Object.groupBy for Array Grouping

ES2024 introduces `Object.groupBy()` for native array grouping without lodash.

## Basic Usage

```javascript
const people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 30 },
  { name: "Charlie", age: 25 },
  { name: "Diana", age: 30 }
];

const byAge = Object.groupBy(people, person => person.age);

// Result:
// {
//   25: [{ name: "Alice", age: 25 }, { name: "Charlie", age: 25 }],
//   30: [{ name: "Bob", age: 30 }, { name: "Diana", age: 30 }]
// }
```

## Grouping by Category

```javascript
const products = [
  { name: "iPhone", category: "electronics", price: 999 },
  { name: "Shirt", category: "clothing", price: 29 },
  { name: "MacBook", category: "electronics", price: 1999 },
  { name: "Jeans", category: "clothing", price: 59 }
];

const byCategory = Object.groupBy(products, p => p.category);

// Access groups
console.log(byCategory.electronics); // [{...iPhone}, {...MacBook}]
console.log(byCategory.clothing);    // [{...Shirt}, {...Jeans}]
```

## Custom Grouping Logic

```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

const grouped = Object.groupBy(numbers, n =>
  n % 2 === 0 ? "even" : "odd"
);
// { odd: [1, 3, 5, 7, 9], even: [2, 4, 6, 8, 10] }
```

## Map.groupBy for Non-String Keys

```javascript
const items = [
  { name: "A", date: new Date("2024-01-01") },
  { name: "B", date: new Date("2024-01-01") },
  { name: "C", date: new Date("2024-02-01") }
];

// Use Map.groupBy for object keys
const byDate = Map.groupBy(items, item => item.date);
```

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: JavaScript, ES2024, Arrays*
