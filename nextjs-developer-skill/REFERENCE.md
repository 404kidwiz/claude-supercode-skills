# Next.js Developer - Technical Reference

## Behavioral Traits

### Performance First
- Optimizes Core Web Vitals (LCP, FID, CLS) for best user experience
- Implements strategic code splitting and lazy loading
- Uses React 18 concurrent features for smooth interactions
- Optimizes images, fonts, and third-party scripts effectively
- Monitors and measures performance continuously

### SEO Excellence
- Implements comprehensive SEO metadata with next-seo or custom metadata
- Creates semantic HTML with proper heading hierarchy
- Optimizes for search engines with structured data and schema markup
- Implements proper URL structures and routing patterns
- Ensures accessibility (a11y) compliance throughout applications

### Developer Experience
- Sets up comprehensive development tooling and workflows
- Implements proper error boundaries and error handling
- Creates reusable component libraries and design systems
- Establishes consistent code patterns and conventions
- Leverages TypeScript for type safety and better DX

## Ideal Scenarios

- **E-commerce**: SEO-optimized online stores with server-side rendering
- **Content Management**: Blogs, news sites, and content-heavy applications
- **SaaS Platforms**: Full-stack applications with authentication and data management
- **Marketing Websites**: High-performance landing pages and marketing funnels
- **Dashboards**: Admin panels with complex data visualization
- **Enterprise Applications**: Scalable web applications with complex business logic

### Problem Areas Addressed
- Performance and SEO optimization challenges
- Complex state management in full-stack applications
- Database integration and data synchronization
- Authentication and authorization patterns
- Third-party integrations and API management

## Development Workflow

### Project Setup
- Initializes Next.js 14+ project with TypeScript and Tailwind CSS
- Configures ESLint, Prettier, and Husky for code quality
- Sets up Prisma or Drizzle ORM for database management
- Implements NextAuth.js or Clerk for authentication
- Configures Tailwind CSS with custom design system

### Component Development
- Uses component-driven development with Storybook
- Implements atomic design with reusable UI components
- Creates comprehensive prop interfaces and documentation
- Uses Radix UI or Shadcn/ui for accessible components
- Implements proper accessibility (a11y) from the start

### Performance Monitoring
- Sets up Next.js Analytics and Vercel Speed Insights
- Implements Core Web Vitals monitoring
- Uses Lighthouse CI for automated performance testing
- Monitors bundle size with @next/bundle-analyzer
- Implements performance budgets and alerts

## Caching Strategy Selection

```
Data Fetching Caching Strategy
├─ Static data (rarely changes)
│   └─ ✅ Static Generation with ISR
│       • fetch(url, { next: { revalidate: 3600 } })
│       • Revalidate every hour
│       • Example: Blog posts, product catalog
│       • Cost: $0 (served from CDN)
│       • Performance: Instant (cached)
│
├─ Frequently changing data (minutes)
│   ├─ Public data (same for all users)
│   │   └─ ✅ Time-based revalidation
│   │       • fetch(url, { next: { revalidate: 60 } })
│   │       • Revalidate every 60 seconds
│   │       • Example: Stock prices, news feed
│   │
│   └─ User-specific data
│       └─ ✅ No caching
│           • fetch(url, { cache: 'no-store' })
│           • Always fresh data
│           • Example: User cart, personalized dashboard
│
├─ Real-time data (seconds)
│   └─ ⚠️ Client-side fetching
│       • Use SWR or React Query in Client Component
│       • WebSocket for live updates
│       • Example: Live chat, trading dashboard
│
└─ On-demand revalidation
    └─ ✅ Tag-based revalidation
        • fetch(url, { next: { tags: ['products'] } })
        • revalidateTag('products') in Server Action
        • Example: Admin updates trigger cache refresh
        • Performance: Instant after first load
```

**Caching Strategy Comparison Table:**

| Strategy | Freshness | Cost | Performance | Use Case |
|----------|-----------|------|-------------|----------|
| `force-cache` (default) | Stale until revalidation | $0 | Instant (CDN) | Static content |
| `{ revalidate: 3600 }` | Max 1hr old | $0 | Instant (CDN) | Semi-static (blog, products) |
| `{ revalidate: 60 }` | Max 1min old | Low | Fast | Frequently updated (news) |
| `no-store` | Always fresh | High | Slower | User-specific (cart, profile) |
| `revalidateTag()` | On-demand fresh | Low | Fast after update | Admin-triggered updates |
| Client-side (SWR) | Configurable | Medium | Fast with stale-while-revalidate | Interactive dashboards |

## Routing Pattern Selection

```
Complex Routing Needs
├─ Modal overlays (don't change URL)
│   └─ ✅ Intercepting Routes
│       • app/@modal/(.)photos/[id]/page.tsx
│       • Intercepts /photos/[id] when navigated from same route
│       • Direct URL access shows full page
│       • Example: Image lightbox, quick view
│
├─ Multiple content areas (dashboard)
│   └─ ✅ Parallel Routes
│       • app/@analytics/page.tsx
│       • app/@users/page.tsx
│       • Render multiple slots simultaneously
│       • Independent loading/error states
│
├─ Conditional rendering based on auth
│   └─ ✅ Route Groups + Layouts
│       • app/(auth)/login/page.tsx
│       • app/(auth)/layout.tsx (no auth UI)
│       • app/(dashboard)/layout.tsx (with nav)
│       • Groups don't affect URL
│
└─ Loading states during navigation
    └─ ✅ Streaming with loading.tsx
        • app/products/loading.tsx
        • Automatic Suspense boundary
        • Progressive page rendering
```

## Type Safety Best Practices

- **End-to-End Types**: Share types between frontend and backend
- **API Types**: Generate types from API responses
- **Database Types**: Use ORM-generated types for database entities
- **Component Props**: Strongly type all component interfaces
- **Server Actions**: Type server actions with proper input/output types
