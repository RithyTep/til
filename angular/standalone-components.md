# Standalone Components Without NgModules

Angular 14+ allows components without NgModules using `standalone: true`.

## Creating a Standalone Component

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-hello',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h1>Hello, {{ name }}!</h1>
    <p *ngIf="showMessage">Welcome to standalone components</p>
  `
})
export class HelloComponent {
  name = 'Rithy';
  showMessage = true;
}
```

## Bootstrapping Without AppModule

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { provideRouter } from '@angular/router';
import { routes } from './app/routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes)
  ]
});
```

## Importing Other Standalone Components

```typescript
@Component({
  standalone: true,
  imports: [
    CommonModule,
    HeaderComponent,    // Other standalone component
    FooterComponent,    // Other standalone component
    RouterModule
  ],
  template: `
    <app-header />
    <router-outlet />
    <app-footer />
  `
})
export class AppComponent {}
```

## Benefits

- ✅ Simpler mental model
- ✅ Better tree-shaking
- ✅ Faster compilation
- ✅ Easier lazy loading
- ✅ Less boilerplate

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: Angular, Standalone Components, NgModules*
