# Image Optimization with next/image

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Images](https://img.shields.io/badge/Optimization-Images-green?style=for-the-badge)

*Automatic image optimization for better Core Web Vitals.*

</div>

## Basic Usage

```tsx
import Image from 'next/image';

export default function Page() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero image"
      width={1200}
      height={600}
      priority // Load immediately (above fold)
    />
  );
}
```

## Responsive Images

```tsx
<Image
  src="/photo.jpg"
  alt="Photo"
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  fill
  className="object-cover"
/>
```

## Fill Container

```tsx
<div className="relative h-64 w-full">
  <Image
    src="/banner.jpg"
    alt="Banner"
    fill
    className="object-cover"
    sizes="100vw"
  />
</div>
```

## Remote Images

```tsx
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'images.unsplash.com',
      },
      {
        protocol: 'https',
        hostname: '*.cloudinary.com',
      },
    ],
  },
};

// Usage
<Image
  src="https://images.unsplash.com/photo-xxx"
  alt="Unsplash photo"
  width={800}
  height={600}
/>
```

## Blur Placeholder

```tsx
// Static import (automatic blur)
import heroImage from '@/public/hero.jpg';

<Image
  src={heroImage}
  alt="Hero"
  placeholder="blur"
/>

// Remote image (provide blurDataURL)
<Image
  src="https://example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQ..."
/>
```

## Loading Priority

```tsx
// Above the fold - load immediately
<Image src="/hero.jpg" priority />

// Below the fold - lazy load (default)
<Image src="/footer.jpg" loading="lazy" />
```

## Quality & Format

```tsx
// next.config.js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200],
    imageSizes: [16, 32, 48, 64, 96, 128, 256],
  },
};

// Per-image quality
<Image src="/photo.jpg" quality={85} />
```

---

*Learned: December 20, 2025*
*Tags: Next.js, Images, Performance, Core Web Vitals*
