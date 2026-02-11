# Migration Strategy: Vanilla TypeScript to React

## Overview

This document outlines a detailed 4-phase migration strategy for transitioning the workshop data client from vanilla TypeScript Web Components to React-based Web Components.

**Duration**: 10-15 weeks with 1-2 developers
**Approach**: Incremental component-by-component migration
**Risk Level**: Low-Medium (validated by POC, gradual transition)

## Migration Principles

### Core Principles

1. **Incremental Migration**: Replace components one at a time, not all at once
2. **Continuous Validation**: Test each component before moving to next
3. **No Breaking Changes**: Maintain Web Component interface throughout
4. **Measured Progress**: Track bundle size, test coverage, functionality
5. **Rollback Capability**: Each component can revert to vanilla if needed
6. **Team Learning**: On-the-job training during migration

### Success Criteria

- ✅ All existing functionality preserved
- ✅ Bundle size increase <20% (gzipped)
- ✅ Test coverage >80% (from ~5%)
- ✅ Web Component encapsulation maintained
- ✅ No accessibility regressions
- ✅ Improved developer experience

## Phase Overview

```
Phase 1: Setup & Infrastructure          [Weeks 1-2]   15% of effort
├─ React dependencies & wrapper
├─ Webpack configuration
├─ Testing framework setup
└─ Bundle size baseline

Phase 2: Component Migration             [Weeks 3-6]   35% of effort
├─ Simple components (button, toggle)
├─ Form inputs (checkboxes, dropdowns)
└─ Complex components (rich-text, file-upload)

Phase 3: Enhanced Features               [Weeks 7-12]  40% of effort
├─ Main modules (data, about, services)
├─ State management integration
└─ Test coverage >80%

Phase 4: Cleanup & Optimization          [Weeks 13-15] 10% of effort
├─ Remove vanilla components
├─ Bundle optimization
└─ Documentation updates
```

---

## Phase 1: Setup & Infrastructure (Weeks 1-2)

### Objectives

- Establish React development environment
- Create platform-owned Web Component wrapper
- Configure build tooling
- Measure bundle size baseline
- Setup testing infrastructure

### Tasks

#### 1.1 Install React Dependencies

**Dependencies to Add**:
```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/user-event": "^14.0.0",
    "eslint-plugin-react": "^7.35.0",
    "eslint-plugin-react-hooks": "^5.0.0"
  }
}
```

**Action**: Run `npm install --save react react-dom` and dev dependencies

**Validation**: Verify React 19+ installed (better Shadow DOM support)

#### 1.2 Create Web Component Wrapper

**Location**: `src/web-component-wrappers/react-to-webcomponent.ts`

**Purpose**: Platform-owned adapter to convert React components to Web Components

**Design**: See [05-WEB-COMPONENT-WRAPPER.md](./05-WEB-COMPONENT-WRAPPER.md) for detailed specification

**Key Features**:
- Shadow DOM creation and management
- Attribute/property synchronization
- Event dispatching
- React root lifecycle management
- CSS encapsulation for Bosch Frontend Kit

**Pseudocode Pattern**:
```typescript
export function reactToWebComponent(
  Component: React.ComponentType<any>,
  propTypes: Record<string, 'string' | 'boolean' | 'number' | 'object'>
) {
  return class extends HTMLElement {
    private root: ReactRoot;
    private props: any = {};

    connectedCallback() {
      const shadowRoot = this.attachShadow({ mode: 'open' });
      this.root = createRoot(shadowRoot);
      this.renderComponent();
    }

    disconnectedCallback() {
      this.root.unmount();
    }

    static get observedAttributes() {
      return Object.keys(propTypes);
    }

    attributeChangedCallback(name: string, oldValue: string, newValue: string) {
      this.props[name] = this.parseAttribute(name, newValue);
      this.renderComponent();
    }

    renderComponent() {
      this.root.render(<Component {...this.props} />);
    }

    // Additional methods for event handling, property parsing, etc.
  };
}
```

**Testing**: Write comprehensive tests for wrapper (edge cases, lifecycle, events)

#### 1.3 Update Webpack Configuration

**File**: `webpack.config.js`

**Changes Required**:

1. **JSX/TSX Support**:
```javascript
module.exports = {
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx', '.scss']
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/
      }
    ]
  }
};
```

2. **Bundle Analysis**:
```javascript
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

plugins: [
  new BundleAnalyzerPlugin({
    analyzerMode: 'static',
    openAnalyzer: false,
    reportFilename: 'bundle-report.html'
  })
];
```

3. **Code Splitting** (for large components):
```javascript
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      react: {
        test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
        name: 'react-vendor',
        priority: 10
      }
    }
  }
};
```

4. **Size Budgets**:
```javascript
performance: {
  maxAssetSize: 500000,  // 500 KB (adjust based on baseline)
  maxEntrypointSize: 500000,
  hints: 'error'
};
```

**Validation**: Build successfully with React components

#### 1.4 Update TypeScript Configuration

**File**: `tsconfig.json`

**Changes**:
```json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"]
}
```

**Validation**: TypeScript compiles React components without errors

#### 1.5 Setup React Testing Library

**File**: `jest.config.js`

**Configuration**:
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/test-setup.ts'],
  moduleNameMapper: {
    '\\.(scss|css)$': 'identity-obj-proxy'
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/*.d.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

**Test Setup File** (`src/test-setup.ts`):
```typescript
import '@testing-library/jest-dom';
```

**Validation**: Run `npm test` successfully with React Testing Library

#### 1.6 Establish Linting Rules

**File**: `.eslintrc.json`

**React-Specific Rules**:
```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended"
  ],
  "plugins": ["react", "react-hooks"],
  "rules": {
    "react/react-in-jsx-scope": "off",  // React 19+ doesn't require import
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn",
    "react/prop-types": "off"  // Using TypeScript for prop validation
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
```

**Validation**: Run `npm run lint` without errors on React components

#### 1.7 Bundle Size Baseline Measurement

**Critical Task**: Establish current bundle size before any React code

**Steps**:
1. Build current vanilla application in production mode
2. Measure gzipped bundle size (not just compiled size)
3. Document per-module sizes (data, about, services)
4. Create size tracking spreadsheet

**Measurement Commands**:
```bash
npm run build
gzip -c dist/workshop-data.js | wc -c > baseline-data.txt
gzip -c dist/workshop-about.js | wc -c > baseline-about.txt
gzip -c dist/workshop-services.js | wc -c > baseline-services.txt
```

**Documentation**: Record baseline in `BUNDLE_SIZE_BASELINE.md`

**Example Format**:
```
Baseline (2026-02-11):
- workshop-data.js:     1,234 KB (gzipped)
- workshop-about.js:    567 KB (gzipped)
- workshop-services.js: 890 KB (gzipped)
- Total:                2,691 KB (gzipped)

Budget (20% increase):  3,229 KB (gzipped)
Headroom:               538 KB (gzipped)
```

### Phase 1 Deliverables

- ✅ React dependencies installed and configured
- ✅ Web Component wrapper implemented and tested
- ✅ Webpack configured for React/TypeScript
- ✅ Testing framework ready (React Testing Library + Jest)
- ✅ Linting configured with React rules
- ✅ Bundle size baseline documented
- ✅ CI/CD updated to track bundle size

### Phase 1 Validation

**Checkpoint**: Before moving to Phase 2, verify:

1. ✅ Hello World React component builds successfully
2. ✅ Web Component wrapper converts React → Web Component correctly
3. ✅ Shadow DOM encapsulation works
4. ✅ Tests run with React Testing Library
5. ✅ Linting enforces React best practices
6. ✅ Bundle size measurement tooling in place

**Duration**: 1-2 weeks (includes learning curve for wrapper development)

---

## Phase 2: Component Migration (Weeks 3-6)

### Objectives

- Migrate all 14 common components from vanilla to React
- Maintain Web Component interface (no breaking changes)
- Achieve >80% test coverage for migrated components
- Validate bundle size stays within budget

### Migration Priority Order

#### Tier 1: Simple Components (Week 3)

**Characteristics**: Minimal state, simple rendering, no external dependencies

1. **general-button** (1 day)
   - Simple props (label, disabled, variant)
   - Click event handling
   - Bosch Frontend Kit styling

2. **toggle** (1 day)
   - Boolean state (on/off)
   - Change event
   - Accessibility (ARIA attributes)

3. **activity-indicator** (0.5 day)
   - No state, just rendering
   - Size variants

4. **general-chip** (0.5 day)
   - Display text with optional close button
   - Remove event

**Total**: ~3 days

**Learning Outcome**: Team learns basic React patterns, Web Component wrapper usage

#### Tier 2: Form Input Components (Week 4)

**Characteristics**: Form state, validation, event handling

5. **input-checkbox-field** (1 day)
   - Checkbox state
   - Label association
   - Validation messages

6. **input-text-field** (1 day)
   - Text input state
   - Validation (required, pattern)
   - Error display

7. **input-type-field** (1 day)
   - Type-specific validation (email, URL, phone)
   - Input masking
   - Custom validators

8. **input-textarea-field** (1 day)
   - Multi-line text
   - Character count
   - Resize handling

9. **input-dropdown-field** (1.5 days)
   - Options management
   - Selection state
   - Search/filter functionality

**Total**: ~5.5 days

**Learning Outcome**: Team masters React hooks (useState, useEffect), form patterns

#### Tier 3: Complex Components (Weeks 5-6)

**Characteristics**: External library integration, complex state, rich functionality

10. **rich-text** (3 days)
    - TipTap React integration
    - Toolbar state management
    - Content sanitization (DOMPurify)
    - Format buttons

11. **file-upload** (2 days)
    - File selection state
    - Drag-and-drop (React Dropzone)
    - Upload progress
    - File validation

12. **image-upload** (2 days)
    - Image preview generation
    - Cropping functionality
    - File size validation
    - Image optimization

13. **multi-selector** (2 days)
    - Multiple selections state
    - Checkbox list
    - Select all / deselect all
    - Search filtering

14. **field-dropdown** (2 days)
    - Advanced dropdown with custom rendering
    - Virtualization for large lists
    - Keyboard navigation

**Total**: ~11 days

**Learning Outcome**: Team handles complex React patterns, third-party integrations

### Per-Component Migration Process

**Standard Process** (apply to each component):

#### Step 1: Create React Component (30% of time)

**File Structure**:
```
src/react-components/common/Button/
├── Button.tsx           # React component
├── Button.test.tsx      # Tests
├── Button.styles.scss   # Styles
└── index.ts             # Re-export
```

**Component Template**:
```typescript
// Button.tsx
import React from 'react';
import './Button.styles.scss';

interface ButtonProps {
  label: string;
  disabled?: boolean;
  variant?: 'primary' | 'secondary';
  onClick?: () => void;
}

export function Button({ label, disabled = false, variant = 'primary', onClick }: ButtonProps) {
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
```

#### Step 2: Write Tests (30% of time)

**Test Template**:
```typescript
// Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  test('renders with label', () => {
    render(<Button label="Click me" />);
    expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
  });

  test('calls onClick when clicked', async () => {
    const handleClick = jest.fn();
    render(<Button label="Click me" onClick={handleClick} />);

    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  test('disables button when disabled prop is true', () => {
    render(<Button label="Click me" disabled />);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  // Additional tests for variants, accessibility, etc.
});
```

**Coverage Target**: >80% per component

#### Step 3: Wrap with Web Component (20% of time)

**File**: `src/web-component-wrappers/GeneralButton.ts`

```typescript
import { reactToWebComponent } from './react-to-webcomponent';
import { Button } from '../react-components/common/Button';

customElements.define(
  'general-button',
  reactToWebComponent(Button, {
    label: 'string',
    disabled: 'boolean',
    variant: 'string'
  })
);
```

**Event Handling** (if needed):
```typescript
// For components that dispatch custom events
const WrappedButton = reactToWebComponent(Button, propTypes, {
  events: {
    onClick: 'button-click'  // React onClick → Custom Event 'button-click'
  }
});
```

#### Step 4: Integration Testing (10% of time)

**Test Web Component Wrapper**:
```typescript
// GeneralButton.integration.test.ts
describe('general-button Web Component', () => {
  test('creates element with attributes', () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Test');
    button.setAttribute('disabled', 'true');
    document.body.appendChild(button);

    expect(button.shadowRoot.querySelector('button')).toBeDisabled();
  });

  test('dispatches custom event on click', async () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Test');
    document.body.appendChild(button);

    const clickHandler = jest.fn();
    button.addEventListener('button-click', clickHandler);

    button.shadowRoot.querySelector('button').click();
    expect(clickHandler).toHaveBeenCalled();
  });
});
```

#### Step 5: Replace Vanilla Component (5% of time)

**Update Imports**:
```typescript
// Before (vanilla)
import './common-components/general-button';

// After (React)
import './web-component-wrappers/GeneralButton';
```

**Validation**:
- All usages still work
- No console errors
- Visual appearance unchanged
- Events still fire correctly

#### Step 6: Measure Bundle Size (5% of time)

**After Each Component**:
```bash
npm run build
gzip -c dist/workshop-data.js | wc -c
```

**Compare to Baseline**:
- Record size increase per component
- Ensure cumulative increase stays <20%
- If approaching budget, investigate code splitting

### Phase 2 Deliverables

- ✅ All 14 common components migrated to React
- ✅ >80% test coverage for each component
- ✅ Web Component wrappers functional
- ✅ Bundle size within budget (<20% increase)
- ✅ No visual regressions
- ✅ All existing functionality preserved

### Phase 2 Validation

**Checkpoint**: Before moving to Phase 3, verify:

1. ✅ All common components work in isolation
2. ✅ Web Component interface unchanged (no breaking changes)
3. ✅ Shadow DOM encapsulation maintained
4. ✅ Bosch Frontend Kit styling preserved
5. ✅ Bundle size acceptable (measured after each component)
6. ✅ Test coverage >80% for migrated components
7. ✅ Team comfortable with React patterns

**Duration**: 3-4 weeks (14 components, varying complexity)

---

## Phase 3: Enhanced Features (Weeks 7-12)

### Objectives

- Migrate main application modules (data, about, services)
- Integrate state management for complex state
- Add form library (React Hook Form) for validation
- Improve test coverage across entire application
- Optimize performance

### Tasks

#### 3.1 Migrate Workshop Data Module (Weeks 7-9)

**Complexity**: High (most complex module)

**Sub-Components**:
- Data entry forms
- Field validation
- Image upload integration
- Rich text integration
- API integration

**Approach**:

1. **Create Module Structure** (1 day):
```
src/react-components/workshop-data/
├── WorkshopDataModule.tsx      # Main module component
├── components/
│   ├── DataEntryForm.tsx       # Form component
│   ├── ImageGallery.tsx        # Image management
│   └── FieldValidation.tsx     # Validation logic
├── hooks/
│   ├── useWorkshopData.ts      # API data fetching
│   └── useFormValidation.ts    # Form state
└── WorkshopDataModule.test.tsx
```

2. **Integrate React Hook Form** (2 days):
```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const workshopSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email'),
  description: z.string().optional(),
  // ... additional fields
});

function DataEntryForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(workshopSchema)
  });

  const onSubmit = (data) => {
    // API call
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}
      {/* Additional fields */}
    </form>
  );
}
```

3. **API Integration Hooks** (2 days):
```typescript
// useWorkshopData.ts
import { useState, useEffect } from 'react';

export function useWorkshopData(id: string) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(`/api/workshop/${id}`)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [id]);

  return { data, loading, error };
}
```

4. **Component Integration** (3 days):
- Use migrated common components (input fields, rich-text, image-upload)
- Wire up form validation
- Connect to API hooks
- Implement error handling

5. **Testing** (2 days):
- Unit tests for components
- Integration tests for form submission
- API mock tests
- Validation tests

**Duration**: 2-3 weeks

#### 3.2 Migrate Workshop About Module (Week 10)

**Complexity**: Medium (simpler than data module)

**Approach**:
- Similar structure to workshop-data
- Fewer components
- Simpler validation
- Reuse patterns from workshop-data migration

**Duration**: 1 week

#### 3.3 Migrate Workshop Services Module (Week 10)

**Complexity**: Medium

**Approach**:
- List-based interface
- Service management forms
- Multi-selector integration
- API integration

**Duration**: 1 week

#### 3.4 State Management Integration (Week 11)

**Evaluation**: Determine if centralized state needed

**Options**:

**Option 1: React Context** (if state sharing is minimal)
```typescript
// WorkshopContext.tsx
const WorkshopContext = createContext(null);

export function WorkshopProvider({ children }) {
  const [workshopData, setWorkshopData] = useState(null);

  return (
    <WorkshopContext.Provider value={{ workshopData, setWorkshopData }}>
      {children}
    </WorkshopContext.Provider>
  );
}
```

**Option 2: Zustand** (if more complex state needed)
```typescript
import create from 'zustand';

const useWorkshopStore = create((set) => ({
  data: null,
  setData: (data) => set({ data }),
  loading: false,
  setLoading: (loading) => set({ loading })
}));
```

**Decision Criteria**:
- If state mostly local to components → Use component state (useState)
- If state shared between 2-3 components → Use React Context
- If complex state with many updates → Use Zustand

**Duration**: 1 week (only if needed)

#### 3.5 Improve Test Coverage (Week 12)

**Goal**: Achieve >80% overall test coverage

**Approach**:
1. Run coverage report: `npm run test -- --coverage`
2. Identify untested code paths
3. Write missing tests (prioritize critical paths)
4. Add integration tests for user flows
5. Add accessibility tests (jest-axe)

**Example Coverage Report**:
```
-------------------------|---------|----------|---------|---------|
File                     | % Stmts | % Branch | % Funcs | % Lines |
-------------------------|---------|----------|---------|---------|
All files                |   85.23 |    78.45 |   87.12 |   84.98 |
 react-components/       |   89.45 |    82.34 |   90.12 |   88.76 |
  common/               |   92.34 |    88.23 |   94.56 |   91.87 |
  workshop-data/        |   84.56 |    76.45 |   86.78 |   83.92 |
  workshop-about/       |   88.23 |    79.34 |   89.45 |   87.65 |
  workshop-services/    |   86.78 |    78.23 |   87.89 |   86.12 |
-------------------------|---------|----------|---------|---------|
```

**Duration**: 1 week

### Phase 3 Deliverables

- ✅ All 3 main modules migrated to React
- ✅ React Hook Form integrated for validation
- ✅ API integration with custom hooks
- ✅ State management (if needed) implemented
- ✅ Overall test coverage >80%
- ✅ Bundle size within budget
- ✅ All features functional

### Phase 3 Validation

**Checkpoint**: Before moving to Phase 4, verify:

1. ✅ Full end-to-end user flows work
2. ✅ Form validation behaves correctly
3. ✅ API integration functional
4. ✅ Image upload and rich text working
5. ✅ Cross-browser testing passed (Chrome, Firefox, Safari, Edge)
6. ✅ Accessibility testing passed
7. ✅ Performance acceptable (Lighthouse scores maintained or improved)

**Duration**: 4-6 weeks (3 modules + state management + testing)

---

## Phase 4: Cleanup & Optimization (Weeks 13-15)

### Objectives

- Remove all vanilla TypeScript components
- Clean up unused dependencies
- Optimize bundle size
- Update documentation
- Finalize migration

### Tasks

#### 4.1 Remove Vanilla Components (Week 13)

**Process**:
1. Verify all vanilla components replaced with React versions
2. Remove vanilla component files from `src/common-components/`
3. Remove vanilla module files
4. Update imports throughout codebase
5. Run full test suite to verify nothing broken

**Validation**:
```bash
# Ensure no vanilla component imports remain
grep -r "common-components" src/
# Should return no results
```

#### 4.2 Dependency Cleanup (Week 13)

**Remove Unused Dependencies**:
- Any vanilla-specific libraries
- Deprecated polyfills
- Unused utility libraries

**Update package.json**:
```bash
npm prune
npm audit fix
```

**Validation**:
- `npm run build` succeeds
- No console warnings about missing dependencies

#### 4.3 Bundle Optimization (Week 14)

**Optimization Strategies**:

1. **Code Splitting** (if needed):
```typescript
// Lazy load rich-text editor (largest component)
const RichText = React.lazy(() => import('./RichText'));

function Editor() {
  return (
    <Suspense fallback={<div>Loading editor...</div>}>
      <RichText />
    </Suspense>
  );
}
```

2. **Tree-Shaking Verification**:
- Ensure unused Bosch Frontend Kit CSS removed
- Verify React production build optimizations
- Check for duplicate dependencies

3. **Dynamic Imports** (if needed):
```typescript
// Load TipTap only when rich text component used
import('tiptap').then(module => {
  // Initialize editor
});
```

4. **Bundle Analysis**:
```bash
npm run build
# Review bundle-report.html from webpack-bundle-analyzer
```

**Optimization Checklist**:
- ✅ React production build enabled
- ✅ Source maps disabled in production
- ✅ Minification enabled
- ✅ Unused CSS purged
- ✅ Code splitting for large components
- ✅ Gzip compression enabled

**Target**: Achieve or exceed <20% bundle size increase goal

#### 4.4 Documentation Updates (Week 15)

**Documents to Create/Update**:

1. **Developer Guide**:
   - How to create new React components
   - Web Component wrapper usage
   - Testing guidelines
   - Linting rules

2. **Component Library Documentation**:
   - Catalog of all React components
   - Props documentation (auto-generated from TypeScript)
   - Usage examples
   - Accessibility guidelines

3. **Migration Retrospective**:
   - Lessons learned
   - What went well
   - What could be improved
   - Recommendations for future migrations

4. **README Updates**:
   - Update main README with React stack
   - Update build instructions
   - Update testing instructions

#### 4.5 Final Validation (Week 15)

**Comprehensive Testing**:

1. **Functional Testing**:
   - All user flows work end-to-end
   - Forms submit correctly
   - Validation triggers appropriately
   - API integration functional
   - Image upload works
   - Rich text editor functional

2. **Cross-Browser Testing**:
   - Chrome (latest)
   - Firefox (latest)
   - Safari (latest)
   - Edge (latest)

3. **Accessibility Testing**:
   - Screen reader compatibility (NVDA, JAWS)
   - Keyboard navigation
   - ARIA attributes present
   - Color contrast
   - Focus indicators

4. **Performance Testing**:
   - Lighthouse scores (target: >90)
   - Load time metrics
   - Runtime performance (React profiler)
   - Bundle size final measurement

5. **Regression Testing**:
   - Compare functionality to vanilla version
   - Ensure no features lost
   - Verify same visual appearance

**Final Bundle Size Measurement**:
```bash
npm run build
gzip -c dist/workshop-data.js | wc -c
# Compare to baseline, verify <20% increase
```

### Phase 4 Deliverables

- ✅ All vanilla components removed
- ✅ Dependencies cleaned up
- ✅ Bundle size optimized and within budget
- ✅ Documentation updated
- ✅ Final validation passed
- ✅ Migration complete

### Phase 4 Validation

**Final Checkpoint**:

1. ✅ 100% functionality preserved from vanilla version
2. ✅ Bundle size <20% increase (hard requirement met)
3. ✅ Test coverage >80%
4. ✅ Cross-browser compatibility verified
5. ✅ Accessibility standards met
6. ✅ Performance maintained or improved
7. ✅ Team trained on React development
8. ✅ Documentation complete

**Duration**: 2-3 weeks

---

## Risk Mitigation Throughout Migration

### Continuous Monitoring

**Weekly Checks**:
- Bundle size measurement (every component)
- Test coverage tracking
- Performance metrics
- Team velocity (burndown chart)

**Mitigation Triggers**:

| Risk | Trigger | Mitigation |
|------|---------|----------|
| **Bundle size bloat** | >15% increase after Phase 2 | Implement code splitting immediately |
| **Low test coverage** | <70% after Phase 2 | Dedicate extra time to testing before Phase 3 |
| **Performance degradation** | Lighthouse score <85 | Profile and optimize before continuing |
| **Timeline slippage** | >2 weeks behind schedule | Re-prioritize, consider reducing scope |

### Rollback Plan

**If Migration Fails**:

1. **Per-Component Rollback**:
   - Revert to vanilla component
   - Keep React infrastructure for future attempts
   - Document reason for rollback

2. **Full Rollback** (worst case):
   - Revert to vanilla codebase
   - Keep React learnings for future
   - Reassess Angular Elements or hybrid approach

**Rollback Triggers**:
- Bundle size exceeds 20% increase with no optimization path
- Critical functionality broken and unfixable
- Performance unacceptable and unoptimizable
- Team unable to maintain React codebase

---

## Success Metrics

### Quantitative Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **Test Coverage** | ~5% | >80% | Jest coverage report |
| **Bundle Size** | X KB (gzipped) | <1.2X KB | Gzipped bundle measurement |
| **Component Count** | 94 files | ~50 files | File count (React more concise) |
| **Boilerplate Lines** | ~3,000 lines | ~500 lines | Manual DOM sync code removed |
| **Lighthouse Score** | Y | ≥Y (maintain) | Lighthouse audit |
| **Build Time** | Z seconds | <1.2Z seconds | Webpack build time |

### Qualitative Metrics

- ✅ **Developer Experience**: Team reports faster development
- ✅ **Maintainability**: New features easier to add
- ✅ **AI Tooling**: Measurable productivity gain with Cursor/Copilot
- ✅ **Code Consistency**: Linting enforces patterns
- ✅ **Onboarding**: New developers productive faster

---

## Timeline Summary

```
Week 1-2:   Phase 1 - Setup & Infrastructure
            ├─ React deps, wrapper, Webpack, testing
            └─ Bundle size baseline

Week 3:     Phase 2 - Simple Components
            ├─ button, toggle, indicator, chip
            └─ Team learns React basics

Week 4:     Phase 2 - Form Input Components
            ├─ checkbox, text, type, textarea, dropdown
            └─ Team masters hooks

Week 5-6:   Phase 2 - Complex Components
            ├─ rich-text, file-upload, image-upload
            ├─ multi-selector, field-dropdown
            └─ Phase 2 validation checkpoint

Week 7-9:   Phase 3 - Workshop Data Module
            ├─ Form integration with React Hook Form
            ├─ API hooks
            └─ Complex validation

Week 10:    Phase 3 - About & Services Modules
            ├─ Simpler modules
            └─ Reuse patterns from data module

Week 11:    Phase 3 - State Management (if needed)
            └─ Context or Zustand integration

Week 12:    Phase 3 - Test Coverage Improvement
            ├─ Achieve >80% coverage
            └─ Phase 3 validation checkpoint

Week 13:    Phase 4 - Cleanup
            ├─ Remove vanilla components
            └─ Dependency cleanup

Week 14:    Phase 4 - Bundle Optimization
            ├─ Code splitting
            └─ Tree-shaking verification

Week 15:    Phase 4 - Documentation & Final Validation
            ├─ Documentation updates
            ├─ Final testing
            └─ Migration complete
```

**Total**: 10-15 weeks with 1-2 developers

---

## Next Steps

1. **Get stakeholder approval** on migration strategy
2. **Execute proof-of-concept** (see [09-PROOF-OF-CONCEPT-PLAN.md](./09-PROOF-OF-CONCEPT-PLAN.md))
3. **Validate assumptions** before committing to full migration
4. **Begin Phase 1** if POC successful

---

**Strategy Version**: 1.0
**Last Updated**: 2026-02-11
**Status**: Ready for execution (pending POC validation)
