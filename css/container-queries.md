# Container Queries for Component-Based Styling

Container queries let components style themselves based on their container size, not viewport.

## The Problem with Media Queries

```css
/* Media queries check viewport width */
@media (min-width: 768px) {
  .card { flex-direction: row; }
}

/* But what if .card is in a narrow sidebar?
   It still gets "desktop" styles even though
   the container is small! */
```

## Solution: Container Queries

```css
/* Define a container */
.card-container {
  container-type: inline-size;
  container-name: card;
}

/* Style based on container width */
@container card (min-width: 400px) {
  .card {
    flex-direction: row;
  }
}

@container card (max-width: 399px) {
  .card {
    flex-direction: column;
  }
}
```

## Container Types

```css
/* Size-based queries (width/height) */
.container {
  container-type: inline-size;  /* width only */
  container-type: size;         /* width and height */
}

/* Style-based queries */
.container {
  container-type: style;
}
```

## Shorthand

```css
.container {
  container: card / inline-size;
  /* Same as:
     container-name: card;
     container-type: inline-size; */
}
```

## Real-World Example

```css
.sidebar {
  container: sidebar / inline-size;
}

@container sidebar (min-width: 300px) {
  .widget {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
}

@container sidebar (max-width: 299px) {
  .widget {
    display: block;
  }
}
```

## Browser Support

- ✅ Chrome 105+
- ✅ Firefox 110+
- ✅ Safari 16+

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: CSS, Container Queries, Responsive Design*
