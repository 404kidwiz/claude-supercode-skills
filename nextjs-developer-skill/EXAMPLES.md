# Next.js Developer - Code Examples & Patterns

## Advanced App Router with Server Actions

```typescript
// app/products/[slug]/page.tsx - Server Component
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { Metadata } from 'next';
import { ProductDetail } from './components/product-detail';
import { ProductReviews } from './components/product-reviews';
import { RelatedProducts } from './components/related-products';
import { getProduct, getProductReviews, getRelatedProducts } from '@/lib/products';
import { addToCart } from './actions/add-to-cart';

interface ProductPageProps {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
}

export async function generateMetadata(
  { params }: ProductPageProps
): Promise<Metadata> {
  const product = await getProduct(params.slug);
  
  if (!product) {
    return {
      title: 'Product Not Found',
    };
  }

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      description: product.description,
      images: [product.image],
      type: 'website',
    },
    twitter: {
      card: 'summary_large_image',
      title: product.name,
      description: product.description,
      images: [product.image],
    },
  };
}

export default async function ProductPage({ params }: ProductPageProps) {
  const product = await getProduct(params.slug);
  
  if (!product) {
    notFound();
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <ProductDetail product={product} addToCartAction={addToCart} />
      
      <div className="mt-12 grid grid-cols-1 lg:grid-cols-2 gap-8">
        <Suspense fallback={<div>Loading reviews...</div>}>
          <ProductReviews productId={product.id} />
        </Suspense>
        
        <Suspense fallback={<div>Loading related products...</div>}>
          <RelatedProducts productId={product.id} category={product.category} />
        </Suspense>
      </div>
    </div>
  );
}
```

## Server Action Example

```typescript
// app/products/[slug]/actions/add-to-cart.ts - Server Action
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { cartService } from '@/lib/cart-service';
import { CartItemInput } from '@/types/cart';

interface AddToCartFormState {
  error?: string;
  success?: string;
}

export async function addToCart(
  productId: string,
  quantity: number,
  prevState: AddToCartFormState,
  formData: FormData
): Promise<AddToCartFormState> {
  const session = await getServerSession(authOptions);
  
  if (!session?.user) {
    return { error: 'You must be logged in to add items to cart' };
  }

  try {
    const cartItem: CartItemInput = {
      productId,
      quantity,
      userId: session.user.id,
    };

    await cartService.addItem(cartItem);
    
    revalidatePath('/cart');
    revalidatePath('/products/' + productId);
    
    return { success: 'Item added to cart!' };
    
  } catch (error) {
    console.error('Failed to add item to cart:', error);
    return { error: 'Failed to add item to cart. Please try again.' };
  }
}
```

## Client Component with Server Action

```typescript
// app/products/[slug]/components/product-detail.tsx - Client Component
'use client';

import { useState } from 'react';
import { useTransition } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { toast } from 'sonner';
import { Product } from '@/types/product';
import { CartItemInput } from '@/types/cart';

interface ProductDetailProps {
  product: Product;
  addToCartAction: (
    productId: string,
    quantity: number,
    prevState: { error?: string; success?: string },
    formData: FormData
  ) => Promise<{ error?: string; success?: string }>;
}

export function ProductDetail({ product, addToCartAction }: ProductDetailProps) {
  const [isPending, startTransition] = useTransition();
  const [quantity, setQuantity] = useState(1);

  const handleAddToCart = (formData: FormData) => {
    startTransition(async () => {
      const result = await addToCartAction(
        product.id,
        quantity,
        { error: '', success: '' },
        formData
      );

      if (result.success) {
        toast.success(result.success);
      } else {
        toast.error(result.error);
      }
    });
  };

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
      <div>
        <div className="relative aspect-square overflow-hidden rounded-lg">
          <img
            src={product.image}
            alt={product.name}
            className="object-cover w-full h-full"
          />
        </div>
      </div>
      
      <div className="space-y-4">
        <h1 className="text-3xl font-bold">{product.name}</h1>
        <p className="text-gray-600">{product.description}</p>
        
        <div className="flex items-center space-x-2">
          <span className="text-3xl font-bold">${product.price}</span>
          {product.comparePrice && (
            <span className="text-lg text-gray-500 line-through">
              ${product.comparePrice}
            </span>
          )}
        </div>

        <div className="flex items-center space-x-4">
          <div className="flex items-center space-x-2">
            <label htmlFor="quantity">Quantity:</label>
            <Input
              id="quantity"
              type="number"
              min="1"
              max={product.stock}
              value={quantity}
              onChange={(e) => setQuantity(Math.max(1, parseInt(e.target.value) || 1))}
              className="w-20"
            />
          </div>
          
          <form action={handleAddToCart}>
            <Button
              type="submit"
              disabled={isPending || product.stock === 0}
              className="flex-1"
            >
              {isPending ? 'Adding...' : 'Add to Cart'}
            </Button>
          </form>
        </div>

        {product.stock === 0 && (
          <p className="text-red-500">This product is currently out of stock</p>
        )}
      </div>
    </div>
  );
}
```

## Route Handler with Advanced Caching

```typescript
// app/api/products/route.ts - Route Handler
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { cache } from 'react';
import { productService } from '@/lib/product-service';
import { createProductSchema } from '@/lib/validations/product';

// Cached function for data fetching
const getProducts = cache(async (params: {
  page?: number;
  limit?: number;
  category?: string;
  search?: string;
}) => {
  return productService.findAll(params);
});

export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    
    const parsedParams = {
      page: parseInt(searchParams.get('page') || '1'),
      limit: parseInt(searchParams.get('limit') || '10'),
      category: searchParams.get('category') || undefined,
      search: searchParams.get('search') || undefined,
    };

    const { products, total, hasNext, hasPrev } = await getProducts(parsedParams);

    return NextResponse.json({
      data: products,
      meta: {
        total,
        page: parsedParams.page,
        limit: parsedParams.limit,
        hasNext,
        hasPrev,
        totalPages: Math.ceil(total / parsedParams.limit),
      },
    });

  } catch (error) {
    console.error('Error fetching products:', error);
    return NextResponse.json(
      { error: 'Failed to fetch products' },
      { status: 500 }
    );
  }
}

export async function POST(request: NextRequest) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session?.user || session.user.role !== 'admin') {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }

    const body = await request.json();
    const validatedData = createProductSchema.parse(body);

    const product = await productService.create(validatedData);
    
    // Revalidate the products page
    revalidatePath('/products');
    revalidatePath('/api/products');

    return NextResponse.json(product, { status: 201 });

  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      );
    }

    console.error('Error creating product:', error);
    return NextResponse.json(
      { error: 'Failed to create product' },
      { status: 500 }
    );
  }
}
```

## Middleware and Authentication Setup

```typescript
// middleware.ts - Request Middleware
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getToken } from 'next-auth/jwt';
import { authConfig } from '@/lib/auth.config';

export async function middleware(request: NextRequest) {
  const token = await getToken({ req: request, secret: authConfig.secret });
  const { pathname } = request.nextUrl;

  // Redirect authenticated users away from auth pages
  if (token && ['/login', '/register'].includes(pathname)) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  // Protect admin routes
  if (pathname.startsWith('/admin') || pathname.startsWith('/api/admin')) {
    if (!token || token.role !== 'admin') {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // Protect dashboard routes
  if (pathname.startsWith('/dashboard')) {
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // Add security headers
  const response = NextResponse.next();

  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self';"
  );

  return response;
}

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico|images|public).*)',
  ],
};
```

## Authentication Configuration

```typescript
// lib/auth.config.ts - Authentication Configuration
import type { NextAuthOptions } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import GitHubProvider from 'next-auth/providers/github';
import GoogleProvider from 'next-auth/providers/google';
import { PrismaAdapter } from '@next-auth/prisma-adapter';
import { prisma } from '@/lib/prisma';
import { userService } from '@/lib/user-service';
import { verifyPassword } from '@/lib/auth';

export const authConfig: NextAuthOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        const user = await userService.findByEmail(credentials.email);
        
        if (!user || !await verifyPassword(credentials.password, user.password)) {
          return null;
        }

        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role,
        };
      },
    }),
    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  session: {
    strategy: 'jwt',
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      if (token) {
        session.user.id = token.sub!;
        session.user.role = token.role as string;
      }
      return session;
    },
  },
  pages: {
    signIn: '/login',
    signUp: '/register',
    error: '/auth/error',
  },
  secret: process.env.NEXTAUTH_SECRET,
};
```

## Anti-Patterns & Fixes

### Anti-Pattern 1: Using Client Components Unnecessarily

**BAD:**
```typescript
// ❌ BAD: Making entire page a Client Component for one interactive element
'use client';

import { useState } from 'react';

export default function ProductPage() {
  const [isOpen, setIsOpen] = useState(false);
  
  // 500 lines of static content that could be Server Component
  const products = await fetchProducts(); // ❌ Can't use async in Client Component!
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {/* Tons of static content here */}
    </div>
  );
}
```

**GOOD:**
```typescript
// ✅ GOOD: Server Component with nested Client Component
import { ToggleButton } from './toggle-button';

export default async function ProductPage() {
  // ✅ Server-side data fetching (fast, secure)
  const products = await fetchProducts();
  
  return (
    <div>
      {/* ✅ Only interactive part is Client Component */}
      <ToggleButton />
      
      {/* ✅ Static content server-rendered */}
      <ProductList products={products} />
    </div>
  );
}

// toggle-button.tsx
'use client';
import { useState } from 'react';

export function ToggleButton() {
  const [isOpen, setIsOpen] = useState(false);
  return <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>;
}
```

**Impact:** 80% smaller JavaScript bundle, faster page loads, better SEO.

### Anti-Pattern 2: Improper Data Mutation with Server Actions

**BAD:**
```typescript
// ❌ BAD: Not revalidating cache after mutation
'use server';

export async function updateProduct(formData: FormData) {
  const id = formData.get('id');
  await db.product.update({ where: { id }, data: { ... } });
  
  // ❌ Missing revalidation - UI shows stale data!
  return { success: true };
}
```

**GOOD:**
```typescript
// ✅ GOOD: Revalidate cache after mutation
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function updateProduct(formData: FormData) {
  const id = formData.get('id') as string;
  
  await db.product.update({
    where: { id },
    data: {
      name: formData.get('name') as string,
      price: parseFloat(formData.get('price') as string),
    }
  });
  
  // ✅ Revalidate specific paths
  revalidatePath(`/products/${id}`);
  revalidatePath('/products');
  
  // ✅ Or revalidate by tag (if using tagged cache)
  revalidateTag('products');
  
  return { success: true };
}

// Even better: Use redirect after mutation (PRG pattern)
'use server';

import { redirect } from 'next/navigation';

export async function updateProduct(formData: FormData) {
  const id = formData.get('id') as string;
  
  await db.product.update({ where: { id }, data: { ... } });
  
  revalidatePath(`/products/${id}`);
  
  // ✅ Redirect to updated page (Post-Redirect-Get pattern)
  redirect(`/products/${id}`);
}
```

**Impact:** Fresh data after mutations, better UX, prevents stale cache issues.
