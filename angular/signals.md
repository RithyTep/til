# Signals for Reactive State Management

Angular Signals (v16+) provide a simpler way to handle reactive state without RxJS complexity.

## Basic Usage

```typescript
import { signal, computed, effect } from '@angular/core';

@Component({
  template: `
    <p>Count: {{ count() }}</p>
    <p>Double: {{ double() }}</p>
    <button (click)="increment()">+1</button>
  `
})
export class CounterComponent {
  // Writable signal
  count = signal(0);

  // Computed signal (derived state)
  double = computed(() => this.count() * 2);

  constructor() {
    // Effect runs when dependencies change
    effect(() => {
      console.log('Count changed:', this.count());
    });
  }

  increment() {
    this.count.update(c => c + 1);
    // or: this.count.set(this.count() + 1);
  }
}
```

## Signal Methods

```typescript
const name = signal('Rithy');

// Read value
console.log(name()); // 'Rithy'

// Set new value
name.set('Tep');

// Update based on previous value
name.update(n => n.toUpperCase());

// Mutate (for objects/arrays)
const user = signal({ name: 'Rithy', age: 25 });
user.mutate(u => u.age++);
```

## When to Use Signals vs RxJS

| Use Signals | Use RxJS |
|-------------|----------|
| Local component state | HTTP requests |
| Simple derived state | Complex async operations |
| Form values | Event streams |
| UI state | Combining multiple sources |

## Benefits

- ✅ No subscriptions to manage
- ✅ Automatic change detection
- ✅ Better performance (fine-grained updates)
- ✅ Simpler mental model

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: Angular, Signals, State Management*
