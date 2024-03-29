# Next.js Concepts

## Getting Started

### Creating Next App

```bash
npx create-next-app@latest nextjs-dashboard --use-npm
```

### Folder Structure

- /app: Contains all the routes, components, and logic for your application, this is where you'll be mostly working from.
- /app/lib: Contains functions used in your application, such as reusable utility functions and data fetching functions.
- /app/ui: Contains all the UI components for your application, such as cards, tables, and forms. To save time, we've pre-styled these components for you.
- /public: Contains all the static assets for your application, such as images.
- /scripts: Contains a seeding script that you'll use to populate your database in a later chapter.
- Config Files: You'll also notice config files such as next.config.js at the root of your application. Most of these files are created and pre-configured when you start a new project using create-next-app. You will not need to modify them in this course.

## Styling

You can store styles globally in `@/app/ui/global.css`

You can also store them inside modules in `@/app/ui/home.module.css`

## Font and Image Optimization

It is important to optimize fonts and images because google ranks pages based on Cumulative Layout Shift (CLS) this happens when the page is rendered before the font loads and once it loads it swaps it out and shifts the content slightly differently from the initial paint. When it comes to images we also face CLS but we also have to ensure our image is responsive, specify image sizes for different devices, serving images in modern formats and lazy load images that are outside the viewport. Luckily Next.js takes care of this for us.

### Optimizing Fonts

```ts
// In /app/ui/fonts.ts

import { Inter } from 'next/font/google';
 
export const inter = Inter({ subsets: ['latin'] });

// In /app/layout.tsx

import '@/app/ui/global.css';
import { inter } from '@/app/ui/fonts';
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={`${inter.className} antialiased`}>{children}</body>
    </html>
  );
}

```

### Optimizing Images

We can use the `<Image />` component to optimize our Image:

```ts
import AcmeLogo from '@/app/ui/acme-logo';
import { ArrowRightIcon } from '@heroicons/react/24/outline';
import Link from 'next/link';
import { lusitana } from '@/app/ui/fonts';
import Image from 'next/image';
 
export default function Page() {
  return (
    // ...
    <div className="flex items-center justify-center p-6 md:w-3/5 md:px-28 md:py-12">
      {/* Add Hero Images Here */}
      <Image
        src="/hero-desktop.png"
        width={1000}
        height={760}
        className="hidden md:block"
        alt="Screenshots of the dashboard project showing desktop version"
      />
      <Image
        src="/hero-mobile.png"
        width={560}
        height={620}
        className="block md:hidden"
        alt="Screenshot of the dashboard project showing mobile version"
      />
    </div>
    //...
  );
}
```

### Creating Layout and Pages

Next.js uses file system routing: this means that each folder representing a **route segment** maps to a **URL segment**. We use `page.tsx` to export a page component. We can also have a `layout.tsx` file that returns a `<Layout />` component and receives a `{children}` prop. This allows us to partially rerender some parts of the page.

Cool side note: unlike a traditional single page application (SPA). Since Next.js is server side rendering (SSR), the failure of other components won't affect our current component.

#### Link Component

The `<Link />` component allows you to do client side navigation with JS. Import it from `next/link`.

We can also make these Links active by using the `usePathname` hook from `next/navigation`.

This is where it gets tricky so don't lose me on this. Hooks are a client side concept; therefore, we need to make our component a client side component.

```ts

'use client'; //Make this a client component
 
import {
  UserGroupIcon,
  HomeIcon,
  DocumentDuplicateIcon,
} from '@heroicons/react/24/outline';
import Link from 'next/link';
import { usePathname } from 'next/navigation';
import clsx from 'clsx';
 
// ...
 
export default function NavLinks() {
  const pathname = usePathname();
 
  return (
    <>
      {links.map((link) => {
        const LinkIcon = link.icon;
        return (
          <Link
            key={link.name}
            href={link.href}
            className={clsx( //Add conditional styling for links dependent on pathname === link.href
              'flex h-[48px] grow items-center justify-center gap-2 rounded-md bg-gray-50 p-3 text-sm font-medium hover:bg-sky-100 hover:text-blue-600 md:flex-none md:justify-start md:p-2 md:px-3',
              {
                'bg-sky-100 text-blue-600': pathname === link.href,
              },
            )}
          >
            <LinkIcon className="w-6" />
            <p className="hidden md:block">{link.name}</p>
          </Link>
        );
      })}
    </>
  );
}
```

### Setting Up PostgreSQL

Next.js is a framework created by Vercel. If you aren't using vercel, you are missing out. It is a great choice for hosting client side apps. Since Next.js is a Vercel product. All you have to do is deploy your app with vercel. Navigate to the Storage tab. Create the database and copy the secrets for your `.env` file locally. Congratulations, easiest database setup ever.

The cool thing of having vercel host both your server and DB is that latency is reduced. The server and DB are in the same region.

Install the Vercel Posstgres SDK `npm i @vercel/postgres`

### Fetching Data

There are multiple ways to fetch data such as:

- API Layer - You can create Next API layers using Route Handlers
- Database Queries - You can directly fetch the data from your react server components

#### Using React Server Components to Fetch Data

- Server Components support promises, providing a simpler solution for asynchronous tasks like data fetching. You can use async/await syntax without reaching out for useEffect, useState or data fetching libraries.
- Server Components execute on the server, so you can keep expensive data fetches and logic on the server and only send the result to the client.
- As mentioned before, since Server Components execute on the server, you can query the database directly without an additional API layer.

```ts
// In /app/lib/data.ts

//Import sql function from @vercel/postgres

import { sql } from '@vercel/postgres';

// Fetch the last 5 invoices, sorted by date
const data = await sql<LatestInvoiceRaw>`
  SELECT invoices.amount, customers.name, customers.image_url, customers.email
  FROM invoices
  JOIN customers ON invoices.customer_id = customers.id
  ORDER BY invoices.date DESC
  LIMIT 5`;
```

We do face two issues when fetching our data like this:

The first one is that we are creating a request waterfall which is when data requests are blocking each other. This is great when the data is dependent on other data; however, if the data is standalone data. We should maybe fetch those asynchronously.

Well lets bust out some of our vanillaJS to solve this issue.

```ts
//In app/lib/data.ts

export async function fetchCardData() {
  try {
    const invoiceCountPromise = sql`SELECT COUNT(*) FROM invoices`;
    const customerCountPromise = sql`SELECT COUNT(*) FROM customers`;
    const invoiceStatusPromise = sql`SELECT
         SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END) AS "paid",
         SUM(CASE WHEN status = 'pending' THEN amount ELSE 0 END) AS "pending"
         FROM invoices`;
 
    const data = await Promise.all([
      invoiceCountPromise,
      customerCountPromise,
      invoiceStatusPromise,
    ]);
    // ...
  }
}

```

Ok but what if one of these takes a long time? This leads us to our second issue. Without running them asynchronously, we may face the issue that Next.js pre-renders routes to improve performance (Static Rendering) which may cause inconsistent data. If they are run asynchronously, and we use a Promise.all() then we have to wait till all requests are complete. This doesn't seem optimal. How about we implement Dynamic Rendering?

#### Dynamic Rendering

Lets take a second to compare dynamic vs static rendering. Both are special in their own use cases.

| Feature                    | Static Rendering                                                        | Dynamic Rendering                                                    |
|----------------------------|-------------------------------------------------------------------------|-----------------------------------------------------------------------|
| Rendering Time             | At build time (when you deploy) or during revalidation                  | At request time (when the user visits the page)                       |
| Data Fetching              | Happens on the server                                                   | Happens on the server                                                 |
| Distribution               | Cached and distributed in a Content Delivery Network (CDN)              | Rendered for each user request                                        |
| Server Load                | Reduced, as content is cached                                           | Higher, as content is dynamically generated for each request          |
| Speed                      | Faster, as content is prerendered and cached                            | Potentially slower, as content is rendered on demand                  |
| SEO                        | Improved, as content is prerendered                                     | Depends on implementation, can be challenging for search engine crawlers |
| Use Cases                  | Static blog posts, product pages, any UI with no data or shared data    | Dashboards, user profiles, real-time data, personalized content       |
| Benefits                   | - Faster websites<br>- Reduced server load<br>- Improved SEO            | - Real-time data<br>- User-specific content<br>- Access to request time information |

In order to implement dynamic rendering, we have to make sure we aren't using cached data. In order to do this we can use the Next.js API named `unstable_noStore` from `next/cache` to opt out from static rendering.

```ts

// In app/lib/datta.ts

// ...
import { unstable_noStore as noStore } from 'next/cache';
 
export async function fetchRevenue() {
  // Add noStore() here to prevent the response from being cached.
  // This is equivalent to in fetch(..., {cache: 'no-store'}).
  noStore();
 
  // ...
}
 
export async function fetchLatestInvoices() {
  noStore();
  // ...
}
 
export async function fetchCardData() {
  noStore();
  // ...
}
 
export async function fetchFilteredInvoices(
  query: string,
  currentPage: number,
) {
  noStore();
  // ...
}
 
export async function fetchInvoicesPages(query: string) {
  noStore();
  // ...
}
 
export async function fetchFilteredCustomers(query: string) {
  noStore();
  // ...
}
 
export async function fetchInvoiceById(query: string) {
  noStore();
  // ...
}
```

This is great and all but what about the data that takes a while to fetch. Well with dynamic rendering, your application is only as fast as your slowest data fetch. Introducing Streaming.

