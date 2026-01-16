# UI Designer - Technical Reference

## Workflow 1: Creating a Design System from Scratch

**Goal:** Build production-ready design system with tokens, components, and documentation in 4-6 weeks.

### Step 1: Design Token Architecture (Week 1)

```json
// tokens/colors.json
{
  "color": {
    "brand": {
      "primary": { "value": "#3b82f6" },
      "secondary": { "value": "#8b5cf6" }
    },
    "semantic": {
      "success": { "value": "#10b981" },
      "error": { "value": "#ef4444" },
      "warning": { "value": "#f59e0b" }
    },
    "neutral": {
      "50": { "value": "#f9fafb" },
      "900": { "value": "#111827" }
    }
  },
  "font": {
    "family": {
      "sans": { "value": "Inter, system-ui, sans-serif" },
      "mono": { "value": "Fira Code, monospace" }
    },
    "size": {
      "xs": { "value": "12px" },
      "sm": { "value": "14px" },
      "base": { "value": "16px" },
      "lg": { "value": "18px" },
      "xl": { "value": "20px" }
    }
  },
  "spacing": {
    "xs": { "value": "4px" },
    "sm": { "value": "8px" },
    "md": { "value": "16px" },
    "lg": { "value": "24px" },
    "xl": { "value": "32px" }
  }
}
```

### Step 2: Component Inventory & Prioritization (Week 1)

**Priority 1 (Critical - Week 2):**
- Button (primary, secondary, ghost, danger)
- Input (text, email, password, textarea)
- Select / Dropdown
- Checkbox / Radio
- Modal / Dialog

**Priority 2 (Important - Week 3):**
- Card
- Table (with sorting, pagination)
- Tabs
- Accordion
- Toast / Alert

**Priority 3 (Nice-to-have - Week 4):**
- Tooltip
- Badge / Chip
- Avatar
- Skeleton loader
- Progress bar

### Step 3: Component Design with States (Weeks 2-4)

**Example: Button Component**

States:
- Default
- Hover
- Active (pressed)
- Focus (keyboard)
- Disabled
- Loading

Variants:
- Primary (filled, brand color)
- Secondary (outlined)
- Ghost (text only)
- Danger (filled, error color)

Sizes:
- Small (32px height, 12px padding)
- Medium (40px height, 16px padding)
- Large (48px height, 20px padding)

### Step 4: Accessibility Annotations

For each component:

```markdown
## Button Accessibility

- **Keyboard:** 
  - Tab: Focus button
  - Enter/Space: Activate button
  
- **Screen Reader:**
  - Announce button label
  - Announce disabled state if applicable
  - Loading state: aria-live="polite" announcement
  
- **Focus Indicator:**
  - 2px solid outline with 2px offset
  - Color: brand-primary-600
  - Contrast ratio: 3:1 against background
  
- **HTML Semantics:**
  - Use `<button>` element (not div with onClick)
  - type="button" for non-submit buttons
```

### Step 5: Documentation with Storybook (Weeks 5-6)

```jsx
// Button.stories.tsx
export default {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'ghost', 'danger']
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg']
    }
  }
};

export const Primary = {
  args: {
    children: 'Click me',
    variant: 'primary',
    size: 'md'
  }
};

export const AllStates = () => (
  <div>
    <Button>Default</Button>
    <Button disabled>Disabled</Button>
    <Button loading>Loading</Button>
  </div>
);
```

## Workflow 2: Accessibility Audit & Remediation

**Goal:** Fix all WCAG 2.1 AA violations in existing UI.

### Step 1: Run Automated Audit

```bash
# Using axe DevTools or Lighthouse
npm install -D @axe-core/cli
axe https://yourapp.com --tags wcag2a,wcag2aa
```

### Step 2: Categorize Violations

Common issues found:
- Color contrast failures (text/background)
- Missing form labels
- Buttons without accessible names
- Missing focus indicators
- Images without alt text
- Keyboard navigation broken

### Step 3: Fix Color Contrast

**Before:**
```css
.secondary-text {
  color: #9ca3af; /* gray-400 */
  background: #ffffff;
  /* Contrast: 2.8:1 ❌ */
}
```

**After:**
```css
.secondary-text {
  color: #6b7280; /* gray-500 */
  background: #ffffff;
  /* Contrast: 4.6:1 ✅ */
}
```

### Step 4: Add Focus Indicators

```css
button:focus-visible {
  outline: 2px solid #3b82f6;
  outline-offset: 2px;
  /* Contrast vs background: 3:1 minimum ✅ */
}
```

### Step 5: Fix Form Accessibility

**Before:**
```html
<input type="email" placeholder="Email" />
```

**After:**
```html
<label for="email">Email Address</label>
<input 
  type="email" 
  id="email" 
  aria-describedby="email-error"
  aria-invalid="true"
/>
<span id="email-error" role="alert">Please enter valid email</span>
```

## Color Palette Strategy

### Brand Colors (primary, secondary)
Generate 10-step scale per color:
- Example: blue-50, blue-100, ..., blue-900
- Ensure 4.5:1 contrast for text
- Tool: coolors.co, huemint.com

### Semantic Colors (success, warning, error, info)
Create accessible variants:
- Green (success): #10b981 (background), #065f46 (text on light)
- Red (error): #ef4444 (background), #991b1b (text on light)
- Yellow (warning): #f59e0b, ensure text contrast
- Blue (info): #3b82f6

### Neutral Colors (grays for UI)
10-step grayscale:
- gray-50 (backgrounds) → gray-900 (text)
- Use for borders, dividers, disabled states

### Dark Mode

**Invert strategy (simple):**
- Flip lightness values
- May need manual tweaks

**Custom dark palette (recommended):**
- Avoid pure black (#000) → use #0f172a
- Adjust colors for OLED displays
- Reduce shadow intensity

## Integration Patterns

### frontend-ui-ux-engineer
- **Handoff:** ui-designer creates component specs → frontend-ui-ux-engineer implements with React/Vue
- **Collaboration:** ui-designer defines visual design → frontend-ui-ux-engineer adds interactions, animations
- **Tools:** ui-designer uses Figma, design tokens; frontend-ui-ux-engineer uses Tailwind, Framer Motion

### accessibility-tester
- **Handoff:** ui-designer completes design → accessibility-tester audits WCAG compliance
- **Collaboration:** Both ensure 4.5:1 contrast, keyboard navigation, screen reader compatibility
- **Tools:** ui-designer uses Stark (Figma plugin); accessibility-tester uses axe, WAVE, screen readers

### ux-researcher
- **Handoff:** ux-researcher provides user insights → ui-designer creates designs based on research
- **Collaboration:** ux-researcher validates design prototypes with users → ui-designer iterates
- **Tools:** ux-researcher uses UserTesting, Hotjar; ui-designer creates interactive prototypes in Figma

### react-specialist
- **Handoff:** ui-designer provides component specs → react-specialist implements complex component logic
- **Collaboration:** ui-designer defines component states → react-specialist implements state management
- **Tools:** ui-designer specifies props/states in Figma; react-specialist uses React, TypeScript

### product-manager
- **Handoff:** product-manager defines features → ui-designer creates UI flows and mockups
- **Collaboration:** product-manager prioritizes features → ui-designer designs iteratively based on priority
- **Tools:** product-manager uses Jira, Notion; ui-designer creates user flows in Figma, Miro

## Example Scenarios

### Scenario 1: Designing a Component Library

**User Request**: "We need a component library for our dashboard"

**Skill Response**:
1. Queries context-manager for existing brand guidelines and design system
2. Analyzes current UI patterns in the dashboard
3. Designs component system structure and organization
4. Creates design tokens for colors, typography, spacing
5. Designs core components (buttons, inputs, cards, tables)
6. Defines component states (hover, focus, disabled, error)
7. Documents accessibility requirements for each component
8. Provides Figma components and developer specifications

**Deliverable**: "Dashboard component library completed. Designed 47 components with full state variations. Design tokens established for colors (47 variables), typography (24 fonts), spacing (scale), and shadows. All components validated for WCAG 2.1 AA accessibility. Figma library with comprehensive documentation provided. Developer handoff specifications included."

### Scenario 2: Redesigning with Accessibility Focus

**User Request**: "Our app fails accessibility audit, can you fix it?"

**Skill Response**:
1. Reviews accessibility audit findings and violations
2. Analyzes current color contrast ratios
3. Identifies keyboard navigation issues
4. Reviews form labels and error handling
5. Redesigns components with accessibility standards
6. Adjusts colors for WCAG AA compliance
7. Adds focus indicators and skip links
8. Documents accessibility guidelines for future design

**Deliverable**: "Accessibility redesign completed. Fixed 87 WCAG violations. Updated color palette to meet 4.5:1 contrast ratio for all text and UI elements. Added focus indicators for all interactive elements. Improved form labels and error messages. All keyboard navigation flows verified. Accessibility documentation and guidelines provided for team."

### Scenario 3: Dark Mode Implementation

**User Request**: "We need to add dark mode to our application"

**Skill Response**:
1. Analyzes existing light mode design and color system
2. Creates dark mode color palette with proper contrast
3. Adapts shadows for dark backgrounds
4. Adjusts images and illustrations for dark mode
5. Designs theme toggle and preferences
6. Tests transitions and handling between modes
7. Validates accessibility in both modes
8. Provides design token mapping for both themes

**Deliverable**: "Dark mode implementation designed. Created complete dark mode color palette with 67 color tokens, all meeting WCAG AA contrast. Adapted shadows and depth for dark backgrounds. Designed theme toggle with smooth transitions. System integration with OS theme preference. Both light and dark modes validated for accessibility. Design token system supports easy theme switching."
