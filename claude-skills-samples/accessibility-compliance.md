# Accessibility Compliance Skill

## Description
Enforce WCAG 2.1 AA compliance for all customer-facing user interfaces at Home Depot. Ensures legal compliance and inclusive shopping experience for all customers.

**Use when:** Creating UI components, web pages, React components, forms, navigation, or any customer-facing interface.

**Activate for phrases like:** "accessibility", "a11y", "WCAG", "screen reader", "keyboard navigation", "UI component", "form", "button", "modal", "ARIA", "contrast", "focus"

## Instructions

### 1. Core WCAG 2.1 AA Requirements

Every UI must meet these criteria:

| Principle | Requirement | Test |
|-----------|-------------|------|
| **Perceivable** | Text contrast ≥ 4.5:1 | Use contrast checker |
| **Operable** | All interactive elements keyboard accessible | Tab through entire page |
| **Understandable** | Form errors clearly identified | Test with screen reader |
| **Robust** | Valid HTML, proper ARIA | Run axe-core |

### 2. React Component Standards

Always include accessibility props:

```tsx
// ❌ WRONG: Missing accessibility
const ProductCard = ({ product }) => (
  <div onClick={() => navigate(`/product/${product.sku}`)}>
    <img src={product.image} />
    <span>{product.name}</span>
    <span>${product.price}</span>
  </div>
);

// ✅ CORRECT: Fully accessible
const ProductCard = ({ product }: ProductCardProps) => {
  const handleClick = () => navigate(`/product/${product.sku}`);
  const handleKeyDown = (e: KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      handleClick();
    }
  };

  return (
    <article
      role="article"
      tabIndex={0}
      onClick={handleClick}
      onKeyDown={handleKeyDown}
      aria-labelledby={`product-name-${product.sku}`}
      className="product-card"
    >
      <img
        src={product.image}
        alt={`${product.name} - ${product.brand}`}  // Descriptive alt text
        loading="lazy"
      />
      <h3 id={`product-name-${product.sku}`}>
        {product.name}
      </h3>
      <p aria-label={`Price: ${product.price} dollars`}>
        ${product.price}
      </p>
      {product.onSale && (
        <span
          role="status"
          aria-live="polite"
          className="sale-badge"
        >
          On Sale
        </span>
      )}
    </article>
  );
};
```

### 3. Form Accessibility

All forms must include proper labeling and error handling:

```tsx
// Accessible form with validation
const CheckoutForm = () => {
  const [errors, setErrors] = useState<Record<string, string>>({});
  const formRef = useRef<HTMLFormElement>(null);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    const newErrors = validateForm();
    setErrors(newErrors);

    if (Object.keys(newErrors).length > 0) {
      // Focus first error for screen readers
      const firstErrorField = formRef.current?.querySelector('[aria-invalid="true"]');
      (firstErrorField as HTMLElement)?.focus();

      // Announce error count
      announceToScreenReader(`Form has ${Object.keys(newErrors).length} errors. Please correct them.`);
    }
  };

  return (
    <form
      ref={formRef}
      onSubmit={handleSubmit}
      aria-labelledby="checkout-heading"
      noValidate  // Use custom validation
    >
      <h1 id="checkout-heading">Checkout</h1>

      {/* Error summary for screen readers */}
      {Object.keys(errors).length > 0 && (
        <div
          role="alert"
          aria-live="assertive"
          className="error-summary"
        >
          <h2>Please correct the following errors:</h2>
          <ul>
            {Object.entries(errors).map(([field, message]) => (
              <li key={field}>
                <a href={`#${field}`}>{message}</a>
              </li>
            ))}
          </ul>
        </div>
      )}

      {/* Accessible input field */}
      <div className="form-group">
        <label htmlFor="email">
          Email Address
          <span aria-hidden="true" className="required">*</span>
          <span className="visually-hidden">(required)</span>
        </label>
        <input
          type="email"
          id="email"
          name="email"
          required
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? "email-error" : "email-hint"}
          autoComplete="email"
        />
        <span id="email-hint" className="hint">
          We'll send your receipt here
        </span>
        {errors.email && (
          <span id="email-error" role="alert" className="error">
            {errors.email}
          </span>
        )}
      </div>

      {/* Accessible select */}
      <div className="form-group">
        <label htmlFor="state">State</label>
        <select
          id="state"
          name="state"
          aria-required="true"
          aria-invalid={!!errors.state}
          aria-describedby={errors.state ? "state-error" : undefined}
        >
          <option value="">Select a state</option>
          {states.map(state => (
            <option key={state.code} value={state.code}>
              {state.name}
            </option>
          ))}
        </select>
        {errors.state && (
          <span id="state-error" role="alert" className="error">
            {errors.state}
          </span>
        )}
      </div>

      <button
        type="submit"
        aria-describedby="submit-hint"
      >
        Complete Purchase
      </button>
      <span id="submit-hint" className="visually-hidden">
        You will be charged after clicking this button
      </span>
    </form>
  );
};
```

### 4. Color and Contrast

```css
/* Home Depot accessible color palette */
:root {
  /* Primary colors with AA contrast ratios */
  --hd-orange: #F96302;           /* 4.5:1 on white */
  --hd-orange-dark: #C44E00;      /* Use for text on white - 7:1 ratio */
  --hd-text-primary: #1A1A1A;     /* 16:1 on white */
  --hd-text-secondary: #595959;   /* 7:1 on white */

  /* Error/success colors - must meet contrast */
  --hd-error: #C41E3A;            /* 5.9:1 on white */
  --hd-success: #1E7F34;          /* 5.4:1 on white */

  /* Focus indicator - highly visible */
  --hd-focus-ring: 3px solid #005FCC;  /* Blue focus ring */
}

/* Never rely on color alone */
.error-message {
  color: var(--hd-error);
  /* Also include icon for non-color indication */
}
.error-message::before {
  content: "⚠ ";  /* Visual indicator beyond color */
}

/* High contrast focus states */
*:focus-visible {
  outline: var(--hd-focus-ring);
  outline-offset: 2px;
}

/* Ensure links are distinguishable */
a {
  color: #0066CC;
  text-decoration: underline;  /* Don't rely on color alone */
}
a:hover {
  text-decoration-thickness: 2px;
}

/* Skip link for keyboard users */
.skip-link {
  position: absolute;
  left: -9999px;
  z-index: 9999;
}
.skip-link:focus {
  left: 10px;
  top: 10px;
  padding: 1rem;
  background: white;
}
```

### 5. Keyboard Navigation

Ensure full keyboard operability:

```tsx
// Custom dropdown with keyboard support
const AccessibleDropdown = ({ options, value, onChange, label }: DropdownProps) => {
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const listRef = useRef<HTMLUListElement>(null);

  const handleKeyDown = (e: KeyboardEvent) => {
    switch (e.key) {
      case 'Enter':
      case ' ':
        e.preventDefault();
        if (isOpen && activeIndex >= 0) {
          onChange(options[activeIndex]);
          setIsOpen(false);
        } else {
          setIsOpen(true);
        }
        break;

      case 'ArrowDown':
        e.preventDefault();
        if (!isOpen) {
          setIsOpen(true);
        } else {
          setActiveIndex(prev => Math.min(prev + 1, options.length - 1));
        }
        break;

      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(prev => Math.max(prev - 1, 0));
        break;

      case 'Escape':
        setIsOpen(false);
        break;

      case 'Home':
        e.preventDefault();
        setActiveIndex(0);
        break;

      case 'End':
        e.preventDefault();
        setActiveIndex(options.length - 1);
        break;

      default:
        // Type-ahead search
        const char = e.key.toLowerCase();
        const matchIndex = options.findIndex(opt =>
          opt.label.toLowerCase().startsWith(char)
        );
        if (matchIndex >= 0) {
          setActiveIndex(matchIndex);
        }
    }
  };

  return (
    <div className="dropdown">
      <label id={`${label}-label`}>{label}</label>
      <button
        type="button"
        role="combobox"
        aria-expanded={isOpen}
        aria-haspopup="listbox"
        aria-labelledby={`${label}-label`}
        aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
        onKeyDown={handleKeyDown}
        onClick={() => setIsOpen(!isOpen)}
      >
        {value?.label || 'Select...'}
      </button>

      {isOpen && (
        <ul
          ref={listRef}
          role="listbox"
          aria-labelledby={`${label}-label`}
          tabIndex={-1}
        >
          {options.map((option, index) => (
            <li
              key={option.value}
              id={`option-${index}`}
              role="option"
              aria-selected={value?.value === option.value}
              className={index === activeIndex ? 'active' : ''}
              onClick={() => {
                onChange(option);
                setIsOpen(false);
              }}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

### 6. Modal/Dialog Accessibility

```tsx
// Accessible modal with focus trap
const AccessibleModal = ({ isOpen, onClose, title, children }: ModalProps) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousActiveElement = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Store previously focused element
      previousActiveElement.current = document.activeElement as HTMLElement;

      // Focus the modal
      modalRef.current?.focus();

      // Trap focus inside modal
      const handleTab = (e: KeyboardEvent) => {
        if (e.key === 'Tab') {
          const focusableElements = modalRef.current?.querySelectorAll(
            'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
          );
          const firstElement = focusableElements?.[0] as HTMLElement;
          const lastElement = focusableElements?.[focusableElements.length - 1] as HTMLElement;

          if (e.shiftKey && document.activeElement === firstElement) {
            e.preventDefault();
            lastElement?.focus();
          } else if (!e.shiftKey && document.activeElement === lastElement) {
            e.preventDefault();
            firstElement?.focus();
          }
        }

        if (e.key === 'Escape') {
          onClose();
        }
      };

      document.addEventListener('keydown', handleTab);
      return () => document.removeEventListener('keydown', handleTab);
    } else {
      // Return focus when modal closes
      previousActiveElement.current?.focus();
    }
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div
      className="modal-overlay"
      onClick={(e) => e.target === e.currentTarget && onClose()}
      aria-hidden="true"
    >
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        aria-describedby="modal-description"
        tabIndex={-1}
        className="modal"
      >
        <h2 id="modal-title">{title}</h2>
        <div id="modal-description">
          {children}
        </div>
        <button
          onClick={onClose}
          aria-label="Close dialog"
          className="modal-close"
        >
          ×
        </button>
      </div>
    </div>
  );
};
```

### 7. Image and Media Accessibility

```tsx
// Product image gallery with accessibility
const ProductImageGallery = ({ images, productName }: GalleryProps) => {
  const [activeIndex, setActiveIndex] = useState(0);

  return (
    <div
      role="region"
      aria-label={`${productName} image gallery`}
      className="gallery"
    >
      {/* Main image */}
      <figure>
        <img
          src={images[activeIndex].src}
          alt={images[activeIndex].alt || `${productName} - view ${activeIndex + 1}`}
          // Don't use alt="" for decorative only - products need descriptions
        />
        <figcaption className="visually-hidden">
          {images[activeIndex].description}
        </figcaption>
      </figure>

      {/* Thumbnails */}
      <div
        role="tablist"
        aria-label="Product image thumbnails"
      >
        {images.map((image, index) => (
          <button
            key={image.id}
            role="tab"
            aria-selected={index === activeIndex}
            aria-controls={`image-panel-${index}`}
            onClick={() => setActiveIndex(index)}
            onKeyDown={(e) => {
              if (e.key === 'ArrowRight') {
                setActiveIndex((prev) => (prev + 1) % images.length);
              } else if (e.key === 'ArrowLeft') {
                setActiveIndex((prev) => (prev - 1 + images.length) % images.length);
              }
            }}
          >
            <img
              src={image.thumbnail}
              alt=""  // Decorative thumbnail, main image has alt
              aria-hidden="true"
            />
            <span className="visually-hidden">
              View image {index + 1} of {images.length}
            </span>
          </button>
        ))}
      </div>

      {/* Live region for announcements */}
      <div aria-live="polite" className="visually-hidden">
        Now showing image {activeIndex + 1} of {images.length}
      </div>
    </div>
  );
};
```

### 8. Required Testing

All UI components must pass:

```typescript
// jest + @testing-library/react + jest-axe
import { axe, toHaveNoViolations } from 'jest-axe';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

expect.extend(toHaveNoViolations);

describe('ProductCard Accessibility', () => {
  it('should have no accessibility violations', async () => {
    const { container } = render(<ProductCard product={mockProduct} />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('should be keyboard navigable', async () => {
    const user = userEvent.setup();
    render(<ProductCard product={mockProduct} />);

    const card = screen.getByRole('article');
    await user.tab();
    expect(card).toHaveFocus();

    await user.keyboard('{Enter}');
    // Assert navigation occurred
  });

  it('should announce dynamic content to screen readers', async () => {
    render(<ProductCard product={saleProduct} />);

    const saleBadge = screen.getByRole('status');
    expect(saleBadge).toHaveAttribute('aria-live', 'polite');
  });

  it('should have sufficient color contrast', async () => {
    const { container } = render(<ProductCard product={mockProduct} />);

    // Run axe contrast checks
    const results = await axe(container, {
      rules: { 'color-contrast': { enabled: true } }
    });
    expect(results).toHaveNoViolations();
  });

  it('should have descriptive alt text for images', () => {
    render(<ProductCard product={mockProduct} />);

    const image = screen.getByRole('img');
    expect(image).toHaveAttribute('alt');
    expect(image.getAttribute('alt')).not.toBe('');
  });
});
```

### 9. Checklist Before Deployment

```markdown
## Accessibility Checklist

### Automated Testing
- [ ] All jest-axe tests passing
- [ ] Lighthouse accessibility score ≥ 90
- [ ] No console warnings about accessibility

### Manual Testing
- [ ] Tab through entire page - logical order
- [ ] Escape closes all modals/dropdowns
- [ ] Focus visible on all interactive elements
- [ ] Screen reader announces all content (test with VoiceOver/NVDA)
- [ ] Page works at 200% zoom
- [ ] No content lost at 320px width

### Content Review
- [ ] All images have descriptive alt text
- [ ] Link text is descriptive (not "click here")
- [ ] Headings follow hierarchy (h1 → h2 → h3)
- [ ] Form fields have visible labels
- [ ] Error messages are clear and specific

### Color/Visual
- [ ] Color is not only indicator of state
- [ ] Contrast ratio ≥ 4.5:1 for text
- [ ] Focus indicators visible
- [ ] Animations can be disabled (prefers-reduced-motion)
```

## Output Format

When creating UI components, always include:
1. Proper ARIA attributes and roles
2. Keyboard event handlers
3. Focus management
4. Screen reader announcements
5. Color contrast compliance
6. Jest-axe accessibility tests

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
