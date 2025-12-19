# Metadata API for SEO

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![SEO](https://img.shields.io/badge/SEO-Metadata_API-orange?style=for-the-badge&logo=google&logoColor=white)

*Generate dynamic meta tags for better SEO with the Metadata API.*

</div>

## Static Metadata

```tsx
// app/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'My App',
  description: 'Welcome to my application',
  keywords: ['Next.js', 'React', 'TypeScript'],
  authors: [{ name: 'Rithy Tep' }],
  openGraph: {
    title: 'My App',
    description: 'Welcome to my application',
    images: ['/og-image.jpg'],
  },
  twitter: {
    card: 'summary_large_image',
    title: 'My App',
    description: 'Welcome to my application',
    images: ['/twitter-image.jpg'],
  },
};
```

## Dynamic Metadata

```tsx
// app/posts/[id]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.id);

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage],
      type: 'article',
      publishedTime: post.createdAt,
      authors: [post.author.name],
    },
  };
}
```

## Template Titles

```tsx
// app/layout.tsx
export const metadata: Metadata = {
  title: {
    template: '%s | My App',
    default: 'My App',
  },
};

// app/about/page.tsx
export const metadata: Metadata = {
  title: 'About', // Renders: "About | My App"
};
```

## Robots & Sitemap

```tsx
// app/robots.ts
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: {
      userAgent: '*',
      allow: '/',
      disallow: '/admin/',
    },
    sitemap: 'https://example.com/sitemap.xml',
  };
}

// app/sitemap.ts
export default async function sitemap(): MetadataRoute.Sitemap {
  const posts = await getPosts();

  return [
    { url: 'https://example.com', lastModified: new Date() },
    ...posts.map(post => ({
      url: `https://example.com/posts/${post.slug}`,
      lastModified: post.updatedAt,
    })),
  ];
}
```

## JSON-LD Structured Data

```tsx
export default function Page() {
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: 'Product Name',
    description: 'Product description',
    price: '99.99',
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <main>...</main>
    </>
  );
}
```

---

*Learned: December 20, 2025*
*Tags: Next.js, SEO, Metadata, OpenGraph*
