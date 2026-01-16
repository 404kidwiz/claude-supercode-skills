# UI Designer - Code Examples & Patterns

## Pattern 1: Design Token Structure (CSS Variables)

```css
:root {
  /* Colors - Semantic */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-success: #10b981;
  --color-error: #ef4444;
  
  /* Colors - Neutral */
  --color-background: #ffffff;
  --color-surface: #f9fafb;
  --color-text-primary: #111827;
  --color-text-secondary: #6b7280;
  --color-border: #e5e7eb;
  
  /* Typography */
  --font-sans: 'Inter', system-ui, sans-serif;
  --font-mono: 'Fira Code', monospace;
  
  --font-size-xs: 0.75rem;    /* 12px */
  --font-size-sm: 0.875rem;   /* 14px */
  --font-size-base: 1rem;     /* 16px */
  --font-size-lg: 1.125rem;   /* 18px */
  --font-size-xl: 1.25rem;    /* 20px */
  
  /* Spacing */
  --spacing-xs: 0.25rem;  /* 4px */
  --spacing-sm: 0.5rem;   /* 8px */
  --spacing-md: 1rem;     /* 16px */
  --spacing-lg: 1.5rem;   /* 24px */
  --spacing-xl: 2rem;     /* 32px */
  
  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
  
  /* Border Radius */
  --radius-sm: 0.25rem;  /* 4px */
  --radius-md: 0.5rem;   /* 8px */
  --radius-lg: 1rem;     /* 16px */
  --radius-full: 9999px; /* Pill shape */
}

/* Dark Mode Overrides */
[data-theme="dark"] {
  --color-background: #0f172a;
  --color-surface: #1e293b;
  --color-text-primary: #f1f5f9;
  --color-text-secondary: #94a3b8;
  --color-border: #334155;
  
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.3);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.4);
}
```

## Pattern 2: Accessible Button Component (React + Tailwind)

```tsx
// Button.tsx - Production-ready accessible button
import { forwardRef, ButtonHTMLAttributes } from 'react';
import { Loader2 } from 'lucide-react';
import { cn } from '@/lib/utils';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', size = 'md', loading, disabled, children, className, ...props }, ref) => {
    return (
      <button
        ref={ref}
        disabled={disabled || loading}
        aria-busy={loading}
        className={cn(
          // Base styles
          'inline-flex items-center justify-center font-medium rounded-md',
          'transition-colors focus-visible:outline-none',
          'focus-visible:ring-2 focus-visible:ring-offset-2',
          'disabled:opacity-50 disabled:pointer-events-none',
          
          // Variants
          {
            'bg-blue-600 text-white hover:bg-blue-700 focus-visible:ring-blue-600': variant === 'primary',
            'border-2 border-blue-600 text-blue-600 hover:bg-blue-50 focus-visible:ring-blue-600': variant === 'secondary',
            'text-blue-600 hover:bg-blue-50 focus-visible:ring-blue-600': variant === 'ghost',
            'bg-red-600 text-white hover:bg-red-700 focus-visible:ring-red-600': variant === 'danger',
          },
          
          // Sizes
          {
            'h-8 px-3 text-sm': size === 'sm',
            'h-10 px-4 text-base': size === 'md',
            'h-12 px-6 text-lg': size === 'lg',
          },
          
          className
        )}
        {...props}
      >
        {loading && <Loader2 className="mr-2 h-4 w-4 animate-spin" aria-hidden="true" />}
        {children}
      </button>
    );
  }
);

Button.displayName = 'Button';
```

## Pattern 3: Responsive Card Layout

```tsx
// Card.tsx - Responsive card component
export const Card = ({ title, description, image, action }) => (
  <div className="group rounded-lg border border-gray-200 bg-white overflow-hidden hover:shadow-lg transition-shadow">
    {/* Image - 16:9 aspect ratio */}
    <div className="relative aspect-[16/9] overflow-hidden bg-gray-100">
      <img 
        src={image} 
        alt={title}
        className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
        loading="lazy"
      />
    </div>
    
    {/* Content - responsive padding */}
    <div className="p-4 md:p-6">
      <h3 className="text-lg md:text-xl font-semibold text-gray-900 mb-2">
        {title}
      </h3>
      <p className="text-sm md:text-base text-gray-600 mb-4">
        {description}
      </p>
      
      {/* Action button */}
      <Button size="sm" variant="primary">
        {action}
      </Button>
    </div>
  </div>
);
```

## Anti-Pattern 1: Ignoring Color Contrast Requirements

### What it looks like (BAD):
```css
/* ❌ BAD: Insufficient contrast */
.secondary-text {
  color: #a0aec0; /* gray-400 */
  background-color: #ffffff;
  /* Contrast ratio: 2.3:1 ❌ (fails WCAG AA 4.5:1) */
}

.button-ghost {
  color: #cbd5e0; /* gray-300 */
  background-color: #f7fafc; /* gray-50 */
  /* Contrast ratio: 1.4:1 ❌ (completely unreadable) */
}
```

### Why it fails:
- Users with low vision cannot read text
- Fails WCAG 2.1 Level AA compliance
- Poor usability in bright environments
- Accessibility lawsuits risk

### Correct approach:
```css
/* ✅ GOOD: Meets 4.5:1 contrast */
.secondary-text {
  color: #718096; /* gray-600 */
  background-color: #ffffff;
  /* Contrast ratio: 5.1:1 ✅ */
}

.button-ghost {
  color: #2d3748; /* gray-800 */
  background-color: #f7fafc; /* gray-50 */
  /* Contrast ratio: 12.6:1 ✅ */
}

/* Use contrast checker: */
/* https://webaim.org/resources/contrastchecker/ */
```

## Anti-Pattern 2: Over-Animating UI

### What it looks like (BAD):
```css
/* ❌ BAD: Excessive animation */
.card {
  transition: all 0.8s ease-in-out;
  transform: rotate(0deg) scale(1);
}

.card:hover {
  transform: rotate(360deg) scale(1.5);
  box-shadow: 0 50px 100px rgba(0,0,0,0.5);
  /* Distracting, nauseating, unprofessional */
}

.button {
  animation: rainbow 2s linear infinite;
  /* Constantly animating buttons = accessibility nightmare */
}
```

### Why it fails:
- Triggers motion sickness for users with vestibular disorders
- Distracts from actual content
- Violates WCAG 2.1 Success Criterion 2.3.3
- Drains battery on mobile devices
- Looks unprofessional

### Correct approach:
```css
/* ✅ GOOD: Subtle, purposeful animation */
.card {
  transition: box-shadow 0.2s ease, transform 0.2s ease;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  /* Subtle lift effect */
}

/* ✅ Respect user preference */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

/* ✅ Purposeful loading spinner (not decorative) */
.spinner {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
```

## Form Accessibility Examples

### Accessible Input Component

```tsx
// Input.tsx - Accessible input with error handling
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
  helpText?: string;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, helpText, id, ...props }, ref) => {
    const inputId = id || `input-${label.toLowerCase().replace(/\s/g, '-')}`;
    const errorId = `${inputId}-error`;
    const helpId = `${inputId}-help`;
    
    return (
      <div className="space-y-1">
        <label 
          htmlFor={inputId}
          className="block text-sm font-medium text-gray-700"
        >
          {label}
        </label>
        
        <input
          ref={ref}
          id={inputId}
          aria-invalid={!!error}
          aria-describedby={`${error ? errorId : ''} ${helpText ? helpId : ''}`.trim() || undefined}
          className={cn(
            'w-full px-3 py-2 border rounded-md',
            'focus:outline-none focus:ring-2 focus:ring-offset-0',
            error 
              ? 'border-red-500 focus:ring-red-500' 
              : 'border-gray-300 focus:ring-blue-500'
          )}
          {...props}
        />
        
        {error && (
          <p id={errorId} role="alert" className="text-sm text-red-600">
            {error}
          </p>
        )}
        
        {helpText && !error && (
          <p id={helpId} className="text-sm text-gray-500">
            {helpText}
          </p>
        )}
      </div>
    );
  }
);
```

### Focus Management in Modal

```tsx
// Modal.tsx - Accessible modal with focus trap
import { useEffect, useRef } from 'react';

export const Modal = ({ isOpen, onClose, title, children }) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousActiveElement = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Save current focus
      previousActiveElement.current = document.activeElement as HTMLElement;
      
      // Focus first focusable element in modal
      const firstFocusable = modalRef.current?.querySelector<HTMLElement>(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      firstFocusable?.focus();
    } else {
      // Restore focus when closing
      previousActiveElement.current?.focus();
    }
  }, [isOpen]);

  // Handle Escape key
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && isOpen) {
        onClose();
      }
    };
    
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div 
      className="fixed inset-0 z-50 flex items-center justify-center bg-black/50"
      onClick={onClose}
      aria-modal="true"
      role="dialog"
      aria-labelledby="modal-title"
    >
      <div 
        ref={modalRef}
        className="bg-white rounded-lg shadow-xl max-w-md w-full p-6"
        onClick={e => e.stopPropagation()}
      >
        <h2 id="modal-title" className="text-xl font-semibold mb-4">
          {title}
        </h2>
        
        {children}
        
        <button 
          onClick={onClose}
          className="mt-4 px-4 py-2 bg-gray-200 rounded hover:bg-gray-300"
        >
          Close
        </button>
      </div>
    </div>
  );
};
```

## Responsive Grid Pattern

```css
/* Responsive grid that adapts to content */
.grid-auto {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 280px), 1fr));
  gap: var(--spacing-lg);
}

/* Explicit breakpoint grid */
.grid-responsive {
  display: grid;
  gap: var(--spacing-md);
  
  /* Mobile: 1 column */
  grid-template-columns: 1fr;
}

@media (min-width: 640px) {
  .grid-responsive {
    /* Tablet: 2 columns */
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .grid-responsive {
    /* Desktop: 3 columns */
    grid-template-columns: repeat(3, 1fr);
  }
}

@media (min-width: 1280px) {
  .grid-responsive {
    /* Large desktop: 4 columns */
    grid-template-columns: repeat(4, 1fr);
  }
}
```

## Loading Skeleton Pattern

```tsx
// Skeleton.tsx - Content placeholder during loading
export const Skeleton = ({ 
  width = '100%', 
  height = '1rem', 
  rounded = 'md' 
}: {
  width?: string;
  height?: string;
  rounded?: 'sm' | 'md' | 'lg' | 'full';
}) => (
  <div 
    className={cn(
      'animate-pulse bg-gray-200',
      {
        'rounded-sm': rounded === 'sm',
        'rounded-md': rounded === 'md',
        'rounded-lg': rounded === 'lg',
        'rounded-full': rounded === 'full',
      }
    )}
    style={{ width, height }}
    aria-hidden="true"
  />
);

// Usage: Card skeleton
export const CardSkeleton = () => (
  <div className="border rounded-lg p-4 space-y-4">
    <Skeleton height="200px" rounded="md" /> {/* Image */}
    <Skeleton height="1.5rem" width="70%" /> {/* Title */}
    <Skeleton height="1rem" width="100%" /> {/* Description line 1 */}
    <Skeleton height="1rem" width="80%" /> {/* Description line 2 */}
    <Skeleton height="2.5rem" width="120px" rounded="md" /> {/* Button */}
  </div>
);
```
