# Web Component Wrapper: React to Web Component Integration

## Overview

This document provides the technical design for a platform-owned wrapper that converts React components into Web Components. This wrapper is critical for maintaining the Web Component boundary while leveraging React's development benefits.

## Design Goals

1. **Minimal Overhead**: Wrapper should add <5KB to bundle size
2. **Shadow DOM Support**: Proper encapsulation for CSS isolation
3. **Efficient Sync**: Optimize attribute/property synchronization
4. **Event Handling**: Support custom events and React callbacks
5. **Lifecycle Management**: Proper React root creation/cleanup
6. **Type Safety**: Full TypeScript support
7. **Reusability**: Single wrapper for all React components
8. **AI-Maintainable**: Simple enough for AI tools to understand and extend

## Architecture

### High-Level Design

```
┌─────────────────────────────────────┐
│      Web Component Interface        │
│  (Custom Element, Shadow DOM)       │
├─────────────────────────────────────┤
│                                     │
│        React-to-Web Component       │
│             Wrapper                 │
│  ┌───────────────────────────────┐ │
│  │ - Attribute observing         │ │
│  │ - Property mapping            │ │
│  │ - Event dispatching           │ │
│  │ - Shadow DOM creation         │ │
│  │ - React root lifecycle        │ │
│  └───────────────────────────────┘ │
│                                     │
├─────────────────────────────────────┤
│          React Component            │
│       (Props, State, Hooks)         │
└─────────────────────────────────────┘
```

### Data Flow

```
1. Attribute Change
   ↓
2. attributeChangedCallback
   ↓
3. Parse attribute → prop conversion
   ↓
4. Update React props
   ↓
5. React re-renders
   ↓
6. Shadow DOM updated

Event Flow:
React Component Event
   ↓
Wrapper event handler
   ↓
CustomEvent dispatch
   ↓
External listeners
```

## Implementation Design

### Core Wrapper Function

**File**: `src/web-component-wrappers/react-to-webcomponent.ts`

**Signature**:
```typescript
export function reactToWebComponent<P extends Record<string, any>>(
  Component: React.ComponentType<P>,
  propDefinitions: PropDefinitions,
  options?: WrapperOptions
): CustomElementConstructor
```

**Type Definitions**:
```typescript
type PropType = 'string' | 'boolean' | 'number' | 'object' | 'array';

interface PropDefinitions {
  [propName: string]: PropType;
}

interface WrapperOptions {
  // CSS to inject into Shadow DOM
  styles?: string | string[];

  // Event mappings (React prop → Custom Event name)
  events?: {
    [reactProp: string]: string;
  };

  // Shadow DOM mode
  shadowMode?: 'open' | 'closed';

  // Disable Shadow DOM (use Light DOM)
  useLightDOM?: boolean;
}
```

### Wrapper Implementation Pattern

**Pseudocode** (not actual implementation):

```typescript
export function reactToWebComponent<P extends Record<string, any>>(
  Component: React.ComponentType<P>,
  propDefinitions: PropDefinitions,
  options: WrapperOptions = {}
): CustomElementConstructor {

  return class ReactWebComponent extends HTMLElement {
    // React root for rendering
    private root: ReactRoot | null = null;

    // Current props to pass to React component
    private props: Partial<P> = {};

    // Shadow root or light DOM container
    private container: ShadowRoot | HTMLElement;

    // Track if component is connected
    private isConnected = false;

    constructor() {
      super();

      // Create Shadow DOM or use light DOM
      if (options.useLightDOM) {
        this.container = this;
      } else {
        this.container = this.attachShadow({
          mode: options.shadowMode || 'open'
        });
      }

      // Inject styles if provided
      if (options.styles && !options.useLightDOM) {
        this.injectStyles(options.styles);
      }
    }

    // Define observed attributes based on prop definitions
    static get observedAttributes(): string[] {
      return Object.keys(propDefinitions);
    }

    // Called when element added to DOM
    connectedCallback() {
      this.isConnected = true;

      // Initialize props from attributes
      this.syncAttributesToProps();

      // Create React root and render
      this.createReactRoot();
    }

    // Called when element removed from DOM
    disconnectedCallback() {
      this.isConnected = false;

      // Cleanup React root
      if (this.root) {
        this.root.unmount();
        this.root = null;
      }
    }

    // Called when observed attribute changes
    attributeChangedCallback(
      name: string,
      oldValue: string | null,
      newValue: string | null
    ) {
      if (oldValue === newValue) return;

      // Parse attribute value based on prop type
      const propType = propDefinitions[name];
      this.props[name] = this.parseAttribute(name, newValue, propType);

      // Re-render React component if connected
      if (this.isConnected && this.root) {
        this.renderComponent();
      }
    }

    // Parse attribute string to typed prop value
    private parseAttribute(
      name: string,
      value: string | null,
      type: PropType
    ): any {
      if (value === null) {
        return undefined;
      }

      switch (type) {
        case 'boolean':
          // Boolean attributes: presence = true, absence = false
          return value !== 'false' && value !== null;

        case 'number':
          const num = Number(value);
          return isNaN(num) ? undefined : num;

        case 'object':
        case 'array':
          try {
            return JSON.parse(value);
          } catch (e) {
            console.warn(`Failed to parse ${type} attribute "${name}":`, e);
            return undefined;
          }

        case 'string':
        default:
          return value;
      }
    }

    // Initialize props from current attributes
    private syncAttributesToProps() {
      for (const [propName, propType] of Object.entries(propDefinitions)) {
        const attrName = propName.toLowerCase();
        if (this.hasAttribute(attrName)) {
          const attrValue = this.getAttribute(attrName);
          this.props[propName] = this.parseAttribute(propName, attrValue, propType);
        }
      }
    }

    // Inject CSS into Shadow DOM
    private injectStyles(styles: string | string[]) {
      const styleArray = Array.isArray(styles) ? styles : [styles];

      styleArray.forEach(css => {
        const styleElement = document.createElement('style');
        styleElement.textContent = css;
        this.container.appendChild(styleElement);
      });
    }

    // Create React root and render component
    private createReactRoot() {
      if (this.root) return;

      // Create wrapper div for React (optional, for better encapsulation)
      const reactContainer = document.createElement('div');
      this.container.appendChild(reactContainer);

      // Create React 19+ root
      this.root = ReactDOM.createRoot(reactContainer);

      // Initial render
      this.renderComponent();
    }

    // Render React component with current props
    private renderComponent() {
      if (!this.root) return;

      // Create event handlers from options
      const eventHandlers = this.createEventHandlers();

      // Merge props with event handlers
      const componentProps = {
        ...this.props,
        ...eventHandlers
      } as P;

      // Render React component
      this.root.render(React.createElement(Component, componentProps));
    }

    // Create event handlers that dispatch custom events
    private createEventHandlers(): Record<string, Function> {
      if (!options.events) return {};

      const handlers: Record<string, Function> = {};

      for (const [reactProp, eventName] of Object.entries(options.events)) {
        handlers[reactProp] = (...args: any[]) => {
          // Dispatch custom event
          const customEvent = new CustomEvent(eventName, {
            detail: args.length === 1 ? args[0] : args,
            bubbles: true,
            composed: true  // Cross Shadow DOM boundary
          });

          this.dispatchEvent(customEvent);
        };
      }

      return handlers;
    }

    // Allow setting properties directly (in addition to attributes)
    // This enables: element.someProperty = value
    set property(value: any) {
      // Define property setters dynamically if needed
      // For now, prefer attributes for consistency
    }
  };
}
```

### Key Implementation Details

#### 1. Shadow DOM Creation

**Purpose**: Encapsulate styles and DOM to prevent conflicts

```typescript
// Shadow DOM mode
this.container = this.attachShadow({ mode: 'open' });

// Inject Bosch Frontend Kit styles
const boschStyles = `
  @import url('/styles/bosch-frontend-kit.css');
`;
this.injectStyles(boschStyles);
```

**Benefits**:
- CSS isolation (Bosch Frontend Kit won't leak out)
- DOM encapsulation (internal structure hidden)
- Style scoping (component styles don't affect global)

**Consideration**: CSS must be injected per component (Shadow DOM limitation)

#### 2. Attribute/Property Synchronization

**Challenge**: Web Components use attributes (strings), React uses props (typed values)

**Solution**: Type-aware parsing

```typescript
// Boolean: attribute presence = true
<general-button disabled></general-button>  // disabled = true
<general-button disabled="false"></general-button>  // disabled = false

// Number: parse to number
<input-field max="100"></input-field>  // max = 100 (number)

// Object/Array: JSON parse
<multi-selector options='["Option 1", "Option 2"]'></multi-selector>

// String: use as-is
<general-button label="Click me"></general-button>
```

**Performance**: Only re-render if attribute actually changed (oldValue !== newValue check)

#### 3. Event Handling

**Challenge**: React uses callback props, Web Components use custom events

**Solution**: Event mapping

```typescript
// React component expects onClick callback
function Button({ onClick }) {
  return <button onClick={onClick}>Click</button>;
}

// Wrapper maps onClick → 'button-click' custom event
reactToWebComponent(Button, { label: 'string' }, {
  events: {
    onClick: 'button-click'
  }
});

// Usage in HTML
<general-button label="Click" onclick="handleClick()"></general-button>

// Or with addEventListener
button.addEventListener('button-click', (e) => {
  console.log('Button clicked!', e.detail);
});
```

**Event Options**:
- `bubbles: true` - Event propagates up DOM tree
- `composed: true` - Event crosses Shadow DOM boundary

#### 4. React Root Lifecycle

**React 19+ createRoot API**:

```typescript
// Create root once
this.root = ReactDOM.createRoot(container);

// Render component
this.root.render(<Component {...props} />);

// Cleanup on disconnect
this.root.unmount();
```

**Benefits**:
- Concurrent rendering support
- Automatic batching
- Better performance than legacy ReactDOM.render

**Lifecycle**:
```
connectedCallback    → Create root, initial render
attributeChanged     → Re-render with new props
disconnectedCallback → Unmount, cleanup
```

#### 5. TypeScript Type Safety

**Type-safe wrapper usage**:

```typescript
interface ButtonProps {
  label: string;
  disabled?: boolean;
  onClick?: () => void;
}

const ButtonElement = reactToWebComponent<ButtonProps>(
  Button,
  {
    label: 'string',
    disabled: 'boolean'
  },
  {
    events: {
      onClick: 'button-click'
    }
  }
);

customElements.define('general-button', ButtonElement);
```

**Benefits**:
- Compile-time prop validation
- Autocomplete for prop definitions
- Type errors if prop types don't match component

## Usage Examples

### Simple Component (Button)

```typescript
// 1. Define React component
interface ButtonProps {
  label: string;
  disabled?: boolean;
  variant?: 'primary' | 'secondary';
  onClick?: () => void;
}

function Button({ label, disabled, variant = 'primary', onClick }: ButtonProps) {
  return (
    <button
      className={`bosch-button bosch-button--${variant}`}
      disabled={disabled}
      onClick={onClick}
    >
      {label}
    </button>
  );
}

// 2. Wrap as Web Component
const ButtonElement = reactToWebComponent<ButtonProps>(
  Button,
  {
    label: 'string',
    disabled: 'boolean',
    variant: 'string'
  },
  {
    events: {
      onClick: 'button-click'
    },
    styles: boschButtonStyles  // Inject Bosch styles
  }
);

// 3. Register custom element
customElements.define('general-button', ButtonElement);

// 4. Use in HTML
<general-button
  label="Click me"
  variant="primary"
  disabled="false"
></general-button>
```

### Complex Component (Rich Text Editor)

```typescript
// 1. React component with TipTap
interface RichTextProps {
  content: string;
  onChange?: (html: string) => void;
  placeholder?: string;
}

function RichText({ content, onChange, placeholder }: RichTextProps) {
  const editor = useEditor({
    extensions: [StarterKit],
    content,
    onUpdate: ({ editor }) => {
      onChange?.(editor.getHTML());
    }
  });

  return (
    <div className="rich-text-editor">
      <EditorContent editor={editor} />
    </div>
  );
}

// 2. Wrap with styles and events
const RichTextElement = reactToWebComponent<RichTextProps>(
  RichText,
  {
    content: 'string',
    placeholder: 'string'
  },
  {
    events: {
      onChange: 'content-change'
    },
    styles: [boschStyles, tiptapStyles]  // Multiple stylesheets
  }
);

// 3. Register
customElements.define('rich-text', RichTextElement);

// 4. Use with event listener
const editor = document.querySelector('rich-text');
editor.addEventListener('content-change', (e) => {
  console.log('Content updated:', e.detail);
});
```

### Component with Object Props (Multi-Selector)

```typescript
// 1. React component
interface MultiSelectorProps {
  options: Array<{ value: string; label: string }>;
  selected?: string[];
  onChange?: (selected: string[]) => void;
}

function MultiSelector({ options, selected = [], onChange }: MultiSelectorProps) {
  // Implementation...
}

// 2. Wrap with object prop support
const MultiSelectorElement = reactToWebComponent<MultiSelectorProps>(
  MultiSelector,
  {
    options: 'array',    // JSON parsing
    selected: 'array'    // JSON parsing
  },
  {
    events: {
      onChange: 'selection-change'
    }
  }
);

// 3. Use with JSON attributes
<multi-selector
  options='[{"value":"1","label":"Option 1"},{"value":"2","label":"Option 2"}]'
  selected='["1"]'
></multi-selector>

// Or set via JavaScript
const selector = document.querySelector('multi-selector');
selector.setAttribute('options', JSON.stringify(optionsArray));
```

## Bosch Frontend Kit Integration

### CSS Injection Strategy

**Challenge**: Shadow DOM isolates styles, so global Bosch Frontend Kit CSS doesn't apply inside

**Solution**: Inject Bosch styles into each Shadow Root

#### Approach 1: Import Bosch CSS in Wrapper

```typescript
// Import Bosch CSS at build time
import boschStyles from '@bosch/frontend-kit/styles.css';

// Pass to wrapper
reactToWebComponent(Component, propDefs, {
  styles: boschStyles
});
```

**Pros**: Single source of truth
**Cons**: Increases bundle size per component

#### Approach 2: CSS Custom Properties (Preferred)

```typescript
// Bosch Frontend Kit uses CSS custom properties
:root {
  --bosch-color-primary: #005691;
  --bosch-color-secondary: #00884b;
  /* ... */
}

// These inherit into Shadow DOM
// No need to re-inject, just use CSS vars
```

**Pros**: Minimal bundle overhead
**Cons**: Requires Bosch Frontend Kit to use CSS custom properties

#### Approach 3: Constructable Stylesheets (Modern Browsers)

```typescript
// Create stylesheet once
const boschStyleSheet = new CSSStyleSheet();
boschStyleSheet.replaceSync(boschCSSString);

// Reuse across all Shadow Roots
shadowRoot.adoptedStyleSheets = [boschStyleSheet];
```

**Pros**: Shared stylesheet, minimal memory
**Cons**: Browser support (Chrome 73+, Firefox 101+, Safari 16.4+)

**Recommendation**: Use Approach 3 (Constructable Stylesheets) with Approach 1 as fallback

### Example Implementation

```typescript
// Shared Bosch stylesheet (created once)
let boschStyleSheet: CSSStyleSheet | null = null;

function getBoschStyleSheet(): CSSStyleSheet {
  if (!boschStyleSheet) {
    boschStyleSheet = new CSSStyleSheet();
    boschStyleSheet.replaceSync(boschCSSString);
  }
  return boschStyleSheet;
}

// In wrapper
if (this.container instanceof ShadowRoot) {
  if ('adoptedStyleSheets' in this.container) {
    // Modern browsers: use constructable stylesheets
    this.container.adoptedStyleSheets = [getBoschStyleSheet()];
  } else {
    // Fallback: inject <style> tag
    this.injectStyles(boschCSSString);
  }
}
```

## Performance Considerations

### Bundle Size Impact

**Wrapper Overhead**:
- Wrapper function: ~3 KB (minified + gzipped)
- React imports: Shared across all components (single React runtime)
- Styles: Use constructable stylesheets to minimize duplication

**Total Overhead**: ~5 KB per application (not per component)

### Rendering Performance

**Optimization 1: Batched Updates**

React 19+ automatically batches updates:
```typescript
// Multiple prop changes
element.setAttribute('label', 'New Label');
element.setAttribute('disabled', 'true');

// React batches → single re-render
```

**Optimization 2: Memoization**

Use React.memo for expensive components:
```typescript
const ExpensiveComponent = React.memo(function ExpensiveComponent(props) {
  // Only re-renders if props change
});
```

**Optimization 3: Lazy Loading**

Lazy load large components:
```typescript
const RichText = React.lazy(() => import('./RichText'));

// Wrapper handles Suspense
reactToWebComponent(() => (
  <Suspense fallback={<div>Loading...</div>}>
    <RichText />
  </Suspense>
), propDefs);
```

### Memory Management

**Proper Cleanup**:
```typescript
disconnectedCallback() {
  // Unmount React (cleans up hooks, listeners, etc.)
  if (this.root) {
    this.root.unmount();
    this.root = null;
  }

  // Clear props
  this.props = {};
}
```

**Benefit**: No memory leaks from unmounted components

## Testing Strategy

### Unit Tests for Wrapper

**Test Coverage**:
```typescript
describe('reactToWebComponent', () => {
  test('creates custom element constructor', () => {
    const Element = reactToWebComponent(TestComponent, {});
    expect(Element).toBeDefined();
    expect(Element.prototype).toBeInstanceOf(HTMLElement);
  });

  test('observes attributes based on prop definitions', () => {
    const Element = reactToWebComponent(TestComponent, {
      label: 'string',
      count: 'number'
    });
    expect(Element.observedAttributes).toEqual(['label', 'count']);
  });

  test('parses boolean attributes correctly', () => {
    const Element = reactToWebComponent(TestComponent, { disabled: 'boolean' });
    const el = new Element();

    el.setAttribute('disabled', 'true');
    expect(el.props.disabled).toBe(true);

    el.setAttribute('disabled', 'false');
    expect(el.props.disabled).toBe(false);
  });

  test('parses number attributes correctly', () => {
    const Element = reactToWebComponent(TestComponent, { count: 'number' });
    const el = new Element();

    el.setAttribute('count', '42');
    expect(el.props.count).toBe(42);
  });

  test('parses object attributes correctly', () => {
    const Element = reactToWebComponent(TestComponent, { data: 'object' });
    const el = new Element();

    const testData = { foo: 'bar' };
    el.setAttribute('data', JSON.stringify(testData));
    expect(el.props.data).toEqual(testData);
  });

  test('creates Shadow DOM by default', () => {
    const Element = reactToWebComponent(TestComponent, {});
    const el = new Element();
    document.body.appendChild(el);

    expect(el.shadowRoot).toBeTruthy();
  });

  test('dispatches custom events from React callbacks', async () => {
    const Element = reactToWebComponent(
      ({ onClick }) => <button onClick={onClick}>Click</button>,
      {},
      { events: { onClick: 'test-click' } }
    );

    const el = new Element();
    document.body.appendChild(el);

    const clickHandler = jest.fn();
    el.addEventListener('test-click', clickHandler);

    const button = el.shadowRoot.querySelector('button');
    button.click();

    expect(clickHandler).toHaveBeenCalled();
  });

  test('unmounts React on disconnect', () => {
    const Element = reactToWebComponent(TestComponent, {});
    const el = new Element();
    document.body.appendChild(el);

    const unmountSpy = jest.spyOn(el.root, 'unmount');

    document.body.removeChild(el);

    expect(unmountSpy).toHaveBeenCalled();
  });
});
```

### Integration Tests

**Test with Actual React Components**:
```typescript
describe('Button Web Component', () => {
  let button: HTMLElement;

  beforeEach(() => {
    button = document.createElement('general-button');
    document.body.appendChild(button);
  });

  afterEach(() => {
    document.body.removeChild(button);
  });

  test('renders with label attribute', () => {
    button.setAttribute('label', 'Test Button');

    const renderedButton = button.shadowRoot.querySelector('button');
    expect(renderedButton.textContent).toBe('Test Button');
  });

  test('dispatches click event', async () => {
    const clickHandler = jest.fn();
    button.addEventListener('button-click', clickHandler);

    button.setAttribute('label', 'Click me');

    const renderedButton = button.shadowRoot.querySelector('button');
    renderedButton.click();

    expect(clickHandler).toHaveBeenCalled();
  });
});
```

## Maintenance & Evolution

### AI-Assisted Maintenance

The wrapper is designed to be **AI-maintainable**:

**Simple patterns**:
- Clear function signature
- Well-documented code
- Standard Web Component lifecycle
- TypeScript for type safety

**AI Tools Can**:
- Debug wrapper issues (clear stack traces)
- Extend wrapper functionality (add new features)
- Optimize performance (suggest improvements)
- Generate documentation (from code)

### Future Enhancements

**Potential Improvements**:

1. **Performance**: Virtual DOM diffing optimization
2. **DevTools**: Custom DevTools panel for debugging
3. **SSR Support**: Server-side rendering for Web Components
4. **Declarative Shadow DOM**: Use template-based Shadow DOM
5. **Attribute Reflection**: Two-way sync (prop change → attribute update)

**Extensibility**: Wrapper can evolve without breaking existing components

## Best Practices

### DO

✅ **Use wrapper for all React components** (consistency)
✅ **Define prop types explicitly** (type safety)
✅ **Map events to custom events** (Web Component convention)
✅ **Inject Bosch styles into Shadow DOM** (encapsulation)
✅ **Test wrapper thoroughly** (reliability)
✅ **Document wrapper usage** (team onboarding)

### DON'T

❌ **Don't bypass wrapper** (creates inconsistency)
❌ **Don't use deprecated React APIs** (legacy ReactDOM.render)
❌ **Don't forget disconnectedCallback cleanup** (memory leaks)
❌ **Don't assume Light DOM** (always check Shadow DOM)
❌ **Don't over-engineer** (keep wrapper simple)

## Conclusion

The React-to-Web Component wrapper is a **critical piece of infrastructure** that:

- ✅ **Enables React development** while maintaining Web Component boundary
- ✅ **Minimizes overhead** (<5KB, shared React runtime)
- ✅ **Preserves encapsulation** (Shadow DOM, CSS isolation)
- ✅ **Ensures type safety** (TypeScript throughout)
- ✅ **Supports AI tooling** (simple, well-structured code)

**Next**: See [06-BUNDLE-SIZE-ANALYSIS.md](./06-BUNDLE-SIZE-ANALYSIS.md) for bundle size optimization strategies.

---

**Design Version**: 1.0
**Last Updated**: 2026-02-11
**Status**: Ready for implementation (Phase 1 of migration)
