# The `:has()` Selector for Parent Selection

<div align="center">

![CSS](https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white)
![Feature](https://img.shields.io/badge/Modern_CSS-:has()_Selector-green?style=for-the-badge)

*CSS `:has()` is the "parent selector" we've waited 20 years for!*

</div>

## Basic Usage

```css
/* Select parent that contains a specific child */
.card:has(img) {
  /* Card with an image */
  padding: 0;
}

.card:has(.badge) {
  /* Card with a badge */
  border: 2px solid gold;
}
```

## Form Validation Styling

```css
/* Style label when input is invalid */
label:has(+ input:invalid) {
  color: red;
}

/* Style form group when input is focused */
.form-group:has(input:focus) {
  background: #f0f0f0;
}
```

## Conditional Layouts

```css
/* Grid with sidebar only if aside exists */
.page:has(aside) {
  display: grid;
  grid-template-columns: 1fr 300px;
}

.page:not(:has(aside)) {
  display: block;
}
```

## Navigation Active State

```css
/* Highlight nav item with active link */
nav li:has(a.active) {
  background: #007bff;
}
```

## Quantity Queries

```css
/* Style list differently based on item count */
ul:has(li:nth-child(5)) {
  /* Has 5+ items - use grid */
  display: grid;
  grid-template-columns: repeat(3, 1fr);
}

ul:not(:has(li:nth-child(5))) {
  /* Less than 5 items - use flexbox */
  display: flex;
}
```

## Combining with Other Selectors

```css
/* Article with both image and blockquote */
article:has(img):has(blockquote) {
  border-left: 4px solid blue;
}

/* Table row with checkbox checked */
tr:has(input[type="checkbox"]:checked) {
  background: #e3f2fd;
}
```

## Browser Support

- Chrome 105+
- Firefox 121+
- Safari 15.4+

---

*Learned: December 20, 2025*
*Tags: CSS, :has() Selector, Parent Selector*
