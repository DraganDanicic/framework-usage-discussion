# Testing Strategy: Achieving >80% Coverage with React

## Overview

This document outlines a comprehensive testing strategy for the workshop data client migration, targeting an improvement from ~5% test coverage (5 test files) to >80% coverage with React Testing Library, integration tests, and end-to-end testing.

**Current State**: ~5% coverage (5 test files / 94 files)
**Target State**: >80% coverage with comprehensive test suite
**Testing Framework**: React Testing Library + Jest + Playwright/Cypress

## Current Testing State Analysis

### Baseline Assessment

**Current Metrics**:
```
Test Files:       5 files
Source Files:     94 files
Coverage:         ~5% (estimated)
Test Types:       Unit tests only (minimal)
Testing Tools:    Jest (configured but underutilized)
```

**Pain Points**:
- Manual Web Component testing requires boilerplate
- Shadow DOM testing complexity
- No testing utilities for vanilla components
- Difficult to mock component interactions
- No component testing best practices
- Limited assertion helpers

**Consequences**:
- High regression risk during changes
- Manual QA burden
- Difficult to refactor with confidence
- Slow feature development (fear of breaking things)
- No automated validation of component behavior

### Why Low Coverage?

**Root Causes**:

1. **Vanilla Web Component Testing Difficulty**:
```typescript
// Vanilla testing requires manual setup
describe('InputField', () => {
  let element: InputField;

  beforeEach(() => {
    element = document.createElement('input-field');
    document.body.appendChild(element);
    // Wait for connectedCallback, Shadow DOM creation, etc.
  });

  test('updates value attribute', () => {
    element.setAttribute('value', 'test');
    // Need to query Shadow DOM manually
    const input = element.shadowRoot.querySelector('input');
    expect(input.value).toBe('test');
  });

  // Lots of boilerplate for each test
});
```

2. **No Testing Library for Vanilla Components**:
- React Testing Library: extensive utilities for React
- Testing Library (DOM): generic, but doesn't handle Shadow DOM well
- Vanilla Web Components: manual testing only

3. **Manual Mocking Complexity**:
- API calls require manual fetch mocking
- Event handling testing is verbose
- State changes difficult to verify

## React Testing Library Approach

### Why React Testing Library?

**Philosophy**: "The more your tests resemble the way your software is used, the more confidence they can give you."

**Advantages for Workshop Data Client**:

| Feature | Vanilla Testing | React Testing Library | Improvement |
|---------|----------------|----------------------|-------------|
| **Component Rendering** | Manual createElement, appendChild | `render(<Component />)` | ⬇️ 90% less boilerplate |
| **Querying Elements** | Manual querySelector | `screen.getByRole()`, `getByLabelText()` | ⬇️ 80% less code |
| **User Interactions** | Manual dispatchEvent | `userEvent.click()`, `userEvent.type()` | ⬇️ More realistic |
| **Async Testing** | Manual waitFor with timers | `waitFor()`, `findBy` queries | ⬇️ 70% less code |
| **Accessibility** | Manual ARIA checks | Built-in role queries | ⬆️ Better a11y |
| **Assertions** | Generic expect() | jest-dom matchers | ⬆️ More readable |

### Core Principles

1. **Test User Behavior, Not Implementation**:
```typescript
// ❌ Bad: Testing implementation details
expect(component.state.value).toBe('test');

// ✅ Good: Testing user-visible behavior
expect(screen.getByDisplayValue('test')).toBeInTheDocument();
```

2. **Query by Accessibility Attributes**:
```typescript
// Priority order:
screen.getByRole('button', { name: 'Submit' })     // 1st choice
screen.getByLabelText('Email address')             // 2nd choice
screen.getByPlaceholderText('Enter email')         // 3rd choice
screen.getByText('Click here')                     // 4th choice
screen.getByTestId('submit-button')                // Last resort
```

3. **Use userEvent for Interactions**:
```typescript
import userEvent from '@testing-library/user-event';

// ❌ Bad: Direct event firing
fireEvent.click(button);

// ✅ Good: Realistic user interaction
await userEvent.click(button);
```

4. **Async Testing with findBy and waitFor**:
```typescript
// Wait for element to appear
const message = await screen.findByText('Success');

// Wait for condition
await waitFor(() => {
  expect(screen.getByRole('alert')).toHaveTextContent('Saved');
});
```

## Unit Testing Strategy for Components

### Component Test Structure

**Standard Test Template**:

```typescript
// Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  // Setup
  const defaultProps = {
    label: 'Click me',
    onClick: jest.fn()
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  // Rendering tests
  describe('rendering', () => {
    test('renders with label', () => {
      render(<Button {...defaultProps} />);
      expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
    });

    test('applies variant class', () => {
      render(<Button {...defaultProps} variant="primary" />);
      expect(screen.getByRole('button')).toHaveClass('bosch-button--primary');
    });
  });

  // Interaction tests
  describe('interactions', () => {
    test('calls onClick when clicked', async () => {
      const user = userEvent.setup();
      render(<Button {...defaultProps} />);

      await user.click(screen.getByRole('button'));
      expect(defaultProps.onClick).toHaveBeenCalledTimes(1);
    });

    test('does not call onClick when disabled', async () => {
      const user = userEvent.setup();
      render(<Button {...defaultProps} disabled />);

      await user.click(screen.getByRole('button'));
      expect(defaultProps.onClick).not.toHaveBeenCalled();
    });
  });

  // State tests
  describe('states', () => {
    test('disables button when disabled prop is true', () => {
      render(<Button {...defaultProps} disabled />);
      expect(screen.getByRole('button')).toBeDisabled();
    });
  });

  // Accessibility tests
  describe('accessibility', () => {
    test('has correct ARIA attributes', () => {
      render(<Button {...defaultProps} aria-label="Custom label" />);
      expect(screen.getByLabelText('Custom label')).toBeInTheDocument();
    });
  });
});
```

### Test Coverage Targets Per Component Type

#### Simple Components (button, toggle, chip)

**Target**: 90-95% coverage

**Test Categories**:
1. Rendering with various props
2. Click/interaction handling
3. Disabled/enabled states
4. Variant/style classes
5. Accessibility attributes

**Example: Toggle Component**:
```typescript
describe('Toggle', () => {
  test('renders in off state by default', () => {
    render(<Toggle label="Enable feature" />);
    expect(screen.getByRole('switch')).not.toBeChecked();
  });

  test('renders in on state when checked prop is true', () => {
    render(<Toggle label="Enable feature" checked />);
    expect(screen.getByRole('switch')).toBeChecked();
  });

  test('toggles state when clicked', async () => {
    const user = userEvent.setup();
    const onChange = jest.fn();
    render(<Toggle label="Enable feature" onChange={onChange} />);

    await user.click(screen.getByRole('switch'));
    expect(onChange).toHaveBeenCalledWith(true);
  });

  test('does not toggle when disabled', async () => {
    const user = userEvent.setup();
    const onChange = jest.fn();
    render(<Toggle label="Enable feature" disabled onChange={onChange} />);

    await user.click(screen.getByRole('switch'));
    expect(onChange).not.toHaveBeenCalled();
  });

  test('has correct accessibility attributes', () => {
    render(<Toggle label="Enable feature" />);
    expect(screen.getByLabelText('Enable feature')).toBeInTheDocument();
    expect(screen.getByRole('switch')).toHaveAccessibleName('Enable feature');
  });
});
```

#### Form Input Components (text field, dropdown, checkbox)

**Target**: 85-90% coverage

**Test Categories**:
1. Rendering with default values
2. User input handling (typing, selecting)
3. Validation (required, pattern, custom)
4. Error message display
5. Label association
6. Controlled/uncontrolled behavior
7. Accessibility (labels, ARIA)

**Example: InputField Component**:
```typescript
describe('InputField', () => {
  test('renders with label', () => {
    render(<InputField label="Email" />);
    expect(screen.getByLabelText('Email')).toBeInTheDocument();
  });

  test('accepts user input', async () => {
    const user = userEvent.setup();
    const onChange = jest.fn();
    render(<InputField label="Email" onChange={onChange} />);

    await user.type(screen.getByLabelText('Email'), 'test@example.com');
    expect(onChange).toHaveBeenLastCalledWith('test@example.com');
  });

  test('shows error when validation fails', async () => {
    const user = userEvent.setup();
    render(<InputField label="Email" required />);

    const input = screen.getByLabelText('Email');
    await user.click(input);
    await user.tab(); // Blur

    expect(screen.getByText('Email is required')).toBeInTheDocument();
  });

  test('validates email format', async () => {
    const user = userEvent.setup();
    render(<InputField label="Email" type="email" />);

    await user.type(screen.getByLabelText('Email'), 'invalid-email');
    await user.tab();

    expect(screen.getByText('Invalid email format')).toBeInTheDocument();
  });

  test('clears error when valid input provided', async () => {
    const user = userEvent.setup();
    render(<InputField label="Email" type="email" />);

    await user.type(screen.getByLabelText('Email'), 'invalid');
    await user.tab();
    expect(screen.getByText('Invalid email format')).toBeInTheDocument();

    await user.clear(screen.getByLabelText('Email'));
    await user.type(screen.getByLabelText('Email'), 'valid@example.com');
    expect(screen.queryByText('Invalid email format')).not.toBeInTheDocument();
  });
});
```

#### Complex Components (rich-text, file-upload, image-upload)

**Target**: 80-85% coverage

**Test Categories**:
1. Basic rendering
2. Core functionality
3. Loading states
4. Error handling
5. Third-party library integration
6. File validation (for upload components)
7. User flows (upload → preview → remove)

**Example: RichText Component**:
```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { RichText } from './RichText';

describe('RichText', () => {
  test('renders editor with toolbar', () => {
    render(<RichText />);
    expect(screen.getByRole('textbox')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Bold' })).toBeInTheDocument();
  });

  test('formats text as bold when bold button clicked', async () => {
    const user = userEvent.setup();
    const onChange = jest.fn();
    render(<RichText onChange={onChange} />);

    const editor = screen.getByRole('textbox');
    await user.click(editor);
    await user.type(editor, 'Test text');

    // Select all text
    await user.keyboard('{Control>}a{/Control}');

    // Click bold button
    await user.click(screen.getByRole('button', { name: 'Bold' }));

    await waitFor(() => {
      expect(onChange).toHaveBeenCalledWith(
        expect.stringContaining('<strong>Test text</strong>')
      );
    });
  });

  test('sanitizes HTML content', () => {
    const maliciousContent = '<script>alert("XSS")</script><p>Safe content</p>';
    const onChange = jest.fn();

    render(<RichText initialContent={maliciousContent} onChange={onChange} />);

    // Should not contain script tag
    expect(screen.queryByText(/alert/)).not.toBeInTheDocument();
    // Should contain safe content
    expect(screen.getByText('Safe content')).toBeInTheDocument();
  });
});
```

**Example: FileUpload Component**:
```typescript
describe('FileUpload', () => {
  test('accepts file via input', async () => {
    const user = userEvent.setup();
    const onUpload = jest.fn();
    render(<FileUpload onUpload={onUpload} />);

    const file = new File(['content'], 'test.pdf', { type: 'application/pdf' });
    const input = screen.getByLabelText('Upload file');

    await user.upload(input, file);

    expect(onUpload).toHaveBeenCalledWith(file);
  });

  test('validates file size', async () => {
    const user = userEvent.setup();
    const onUpload = jest.fn();
    render(<FileUpload onUpload={onUpload} maxSize={1000000} />); // 1MB

    const largeFile = new File(['x'.repeat(2000000)], 'large.pdf', { type: 'application/pdf' });
    const input = screen.getByLabelText('Upload file');

    await user.upload(input, largeFile);

    expect(screen.getByText('File size must be less than 1 MB')).toBeInTheDocument();
    expect(onUpload).not.toHaveBeenCalled();
  });

  test('validates file type', async () => {
    const user = userEvent.setup();
    const onUpload = jest.fn();
    render(<FileUpload onUpload={onUpload} accept=".pdf,.doc" />);

    const invalidFile = new File(['content'], 'test.jpg', { type: 'image/jpeg' });
    const input = screen.getByLabelText('Upload file');

    await user.upload(input, invalidFile);

    expect(screen.getByText(/Invalid file type/)).toBeInTheDocument();
    expect(onUpload).not.toHaveBeenCalled();
  });

  test('shows upload progress', async () => {
    const user = userEvent.setup();
    render(<FileUpload onUpload={jest.fn()} />);

    const file = new File(['content'], 'test.pdf', { type: 'application/pdf' });
    const input = screen.getByLabelText('Upload file');

    await user.upload(input, file);

    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  test('handles upload error', async () => {
    const user = userEvent.setup();
    const onError = jest.fn();
    render(<FileUpload onUpload={() => Promise.reject('Network error')} onError={onError} />);

    const file = new File(['content'], 'test.pdf', { type: 'application/pdf' });
    await user.upload(screen.getByLabelText('Upload file'), file);

    await waitFor(() => {
      expect(screen.getByText(/Upload failed/)).toBeInTheDocument();
    });
  });
});
```

## Integration Testing for Web Component Wrappers

### Why Integration Tests?

**Purpose**: Validate React components work correctly when wrapped as Web Components

**What to Test**:
1. Attribute → React prop synchronization
2. React events → Custom Event dispatching
3. Shadow DOM encapsulation
4. CSS isolation
5. Bosch Frontend Kit styling
6. Lifecycle (mount, update, unmount)

### Web Component Integration Test Pattern

```typescript
// GeneralButton.integration.test.ts
describe('general-button Web Component', () => {
  beforeAll(() => {
    // Register Web Component (done by wrapper)
    // customElements.define('general-button', ...);
  });

  afterEach(() => {
    document.body.innerHTML = '';
  });

  test('creates element with default attributes', () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Test Button');
    document.body.appendChild(button);

    // Wait for React to render in Shadow DOM
    return new Promise(resolve => {
      setTimeout(() => {
        const shadowButton = button.shadowRoot.querySelector('button');
        expect(shadowButton).toBeTruthy();
        expect(shadowButton.textContent).toBe('Test Button');
        resolve();
      }, 100);
    });
  });

  test('synchronizes attribute changes', async () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Initial');
    document.body.appendChild(button);

    await waitFor(() => {
      expect(button.shadowRoot.querySelector('button').textContent).toBe('Initial');
    });

    button.setAttribute('label', 'Updated');

    await waitFor(() => {
      expect(button.shadowRoot.querySelector('button').textContent).toBe('Updated');
    });
  });

  test('dispatches custom event on click', async () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Click me');
    document.body.appendChild(button);

    const clickHandler = jest.fn();
    button.addEventListener('button-click', clickHandler);

    await waitFor(() => {
      expect(button.shadowRoot.querySelector('button')).toBeTruthy();
    });

    button.shadowRoot.querySelector('button').click();

    expect(clickHandler).toHaveBeenCalled();
  });

  test('maintains Shadow DOM encapsulation', () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Test');
    document.body.appendChild(button);

    expect(button.shadowRoot).toBeTruthy();
    expect(button.shadowRoot.mode).toBe('open');

    // Shadow DOM children not accessible from document
    expect(document.querySelector('button')).toBeNull();
    expect(button.shadowRoot.querySelector('button')).toBeTruthy();
  });

  test('applies Bosch Frontend Kit styles in Shadow DOM', async () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Styled');
    button.setAttribute('variant', 'primary');
    document.body.appendChild(button);

    await waitFor(() => {
      const shadowButton = button.shadowRoot.querySelector('button');
      expect(shadowButton.classList.contains('bosch-button')).toBe(true);
      expect(shadowButton.classList.contains('bosch-button--primary')).toBe(true);
    });
  });

  test('handles boolean attributes correctly', async () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Test');
    button.setAttribute('disabled', 'true');
    document.body.appendChild(button);

    await waitFor(() => {
      expect(button.shadowRoot.querySelector('button')).toBeDisabled();
    });

    button.setAttribute('disabled', 'false');

    await waitFor(() => {
      expect(button.shadowRoot.querySelector('button')).not.toBeDisabled();
    });
  });

  test('unmounts React component when removed from DOM', () => {
    const button = document.createElement('general-button');
    button.setAttribute('label', 'Test');
    document.body.appendChild(button);

    const unmountSpy = jest.spyOn(button, 'disconnectedCallback');

    button.remove();

    expect(unmountSpy).toHaveBeenCalled();
  });
});
```

### Integration Test Coverage

**Target**: 90-95% coverage for wrapper functionality

**Critical Paths**:
- ✅ Attribute synchronization (string, boolean, number, object)
- ✅ Event dispatching (React → Custom Event)
- ✅ Shadow DOM creation and encapsulation
- ✅ CSS isolation
- ✅ Lifecycle (connectedCallback, disconnectedCallback)
- ✅ Property updates (setAttribute vs direct property)
- ✅ Multiple instances (no state leakage)

## E2E Testing Approach (Playwright/Cypress)

### Why E2E Tests?

**Purpose**: Validate complete user flows work end-to-end

**When to Use**:
- Critical user journeys (form submission, file upload)
- Multi-step workflows
- API integration validation
- Cross-browser compatibility
- Visual regression testing

### Playwright vs Cypress Comparison

| Aspect | Playwright | Cypress | Recommendation |
|--------|-----------|---------|----------------|
| **Multi-browser** | ✅ Chrome, Firefox, Safari, Edge | ⚠️ Chrome, Firefox, Edge (no Safari) | Playwright (Safari support) |
| **Speed** | ✅ Fast (parallel tests) | ⚡ Very fast | Tie |
| **API** | Modern, async/await | Chainable commands | Personal preference |
| **Debugging** | ✅ Excellent (trace viewer) | ✅ Excellent (time-travel) | Tie |
| **Network Mocking** | ✅ Built-in | ✅ Built-in | Tie |
| **Screenshots/Video** | ✅ Yes | ✅ Yes | Tie |
| **Learning Curve** | ⚡ Standard async/await | ⚡ Unique syntax | Playwright (familiar) |

**Recommendation**: **Playwright** for better cross-browser support (Safari) and familiar async/await syntax.

### E2E Test Structure

**Workshop Data Module E2E Test**:

```typescript
// workshop-data.e2e.test.ts
import { test, expect } from '@playwright/test';

test.describe('Workshop Data Module', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:8080/workshop-data.html');
  });

  test('user can submit workshop data form', async ({ page }) => {
    // Fill out form
    await page.getByLabel('Workshop Name').fill('React Testing Workshop');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Description').fill('Learn React Testing Library');

    // Select dropdown option
    await page.getByLabel('Category').selectOption('Technology');

    // Upload image
    const fileInput = page.getByLabel('Upload Image');
    await fileInput.setInputFiles('./test-fixtures/image.jpg');

    // Wait for image preview
    await expect(page.getByAltText('Preview')).toBeVisible();

    // Submit form
    await page.getByRole('button', { name: 'Submit' }).click();

    // Wait for success message
    await expect(page.getByText('Workshop saved successfully')).toBeVisible();
  });

  test('shows validation errors for invalid input', async ({ page }) => {
    // Submit without filling required fields
    await page.getByRole('button', { name: 'Submit' }).click();

    // Should show validation errors
    await expect(page.getByText('Workshop Name is required')).toBeVisible();
    await expect(page.getByText('Email is required')).toBeVisible();
  });

  test('validates email format', async ({ page }) => {
    await page.getByLabel('Email').fill('invalid-email');
    await page.getByLabel('Email').blur();

    await expect(page.getByText('Invalid email format')).toBeVisible();
  });

  test('can upload and remove image', async ({ page }) => {
    const fileInput = page.getByLabel('Upload Image');
    await fileInput.setInputFiles('./test-fixtures/image.jpg');

    await expect(page.getByAltText('Preview')).toBeVisible();

    // Remove image
    await page.getByRole('button', { name: 'Remove' }).click();

    await expect(page.getByAltText('Preview')).not.toBeVisible();
  });

  test('rich text editor formats content', async ({ page }) => {
    const editor = page.locator('[contenteditable="true"]');

    await editor.click();
    await editor.type('Test content');

    // Select all text
    await page.keyboard.press('Control+A');

    // Click bold button
    await page.getByRole('button', { name: 'Bold' }).click();

    // Verify bold formatting applied
    const boldText = page.locator('strong');
    await expect(boldText).toHaveText('Test content');
  });

  test('form persists data on navigation', async ({ page }) => {
    // Fill form
    await page.getByLabel('Workshop Name').fill('Persistence Test');

    // Navigate away (if applicable)
    // await page.goto('/other-page');
    // await page.goBack();

    // Verify data persisted (if using localStorage or session storage)
    // await expect(page.getByLabel('Workshop Name')).toHaveValue('Persistence Test');
  });
});
```

### API Mocking in E2E Tests

**Playwright API Mocking**:

```typescript
test('handles API errors gracefully', async ({ page }) => {
  // Mock API failure
  await page.route('**/api/workshop', route => {
    route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Internal Server Error' })
    });
  });

  await page.goto('http://localhost:8080/workshop-data.html');

  // Fill and submit form
  await page.getByLabel('Workshop Name').fill('Test');
  await page.getByRole('button', { name: 'Submit' }).click();

  // Should show error message
  await expect(page.getByText('Failed to save workshop. Please try again.')).toBeVisible();
});

test('loads existing workshop data', async ({ page }) => {
  // Mock API response
  await page.route('**/api/workshop/123', route => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        id: '123',
        name: 'Existing Workshop',
        email: 'existing@example.com',
        description: 'Pre-loaded data'
      })
    });
  });

  await page.goto('http://localhost:8080/workshop-data.html?id=123');

  // Verify form populated with API data
  await expect(page.getByLabel('Workshop Name')).toHaveValue('Existing Workshop');
  await expect(page.getByLabel('Email')).toHaveValue('existing@example.com');
});
```

### E2E Test Coverage Goals

**Target**: 5-10 critical user flows

**Priority Flows**:
1. ✅ Submit new workshop data (happy path)
2. ✅ Edit existing workshop data
3. ✅ Form validation (error states)
4. ✅ Image upload flow
5. ✅ Rich text editing
6. ✅ API error handling
7. ⚡ Cross-browser compatibility (Chrome, Firefox, Safari, Edge)

## Test Coverage Goals and Measurement

### Coverage Targets

| Component Category | Target Coverage | Rationale |
|-------------------|----------------|-----------|
| **Simple Components** | 90-95% | High confidence, easy to test |
| **Form Components** | 85-90% | Core business logic, critical |
| **Complex Components** | 80-85% | Third-party deps, harder to mock |
| **Utility Functions** | 95-100% | Pure functions, no side effects |
| **API Integration** | 80-85% | Mocked APIs, critical paths |
| **Overall Application** | >80% | Hard requirement met |

### Jest Coverage Configuration

**jest.config.js**:
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/test-setup.ts'],

  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/test-utils/**',
    '!src/**/*.stories.tsx'  // Exclude Storybook stories
  ],

  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    // Stricter thresholds for critical modules
    'src/react-components/workshop-data/**/*.{ts,tsx}': {
      branches: 85,
      functions: 85,
      lines: 85,
      statements: 85
    }
  },

  coverageReporters: ['text', 'lcov', 'html'],
  coverageDirectory: 'coverage'
};
```

### Coverage Report Example

**Target Report**:
```
-------------------------|---------|----------|---------|---------|-------------------
File                     | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
-------------------------|---------|----------|---------|---------|-------------------
All files                |   85.23 |    82.45 |   87.12 |   84.98 |
 react-components/       |   89.45 |    84.34 |   90.12 |   88.76 |
  common/
   Button.tsx           |   95.67 |    91.23 |   94.56 |   95.12 |
   Toggle.tsx           |   94.23 |    89.45 |   93.78 |   93.98 |
   InputField.tsx       |   92.34 |    88.67 |   91.23 |   92.11 |
   RichText.tsx         |   82.45 |    76.34 |   85.67 |   81.98 | 45-52, 78-82
  workshop-data/
   WorkshopDataModule   |   88.56 |    83.45 |   89.78 |   87.92 |
   DataEntryForm.tsx    |   90.12 |    85.67 |   91.34 |   89.87 |
 web-component-wrappers/
  react-to-webcomp...   |   94.78 |    90.23 |   95.67 |   94.45 |
-------------------------|---------|----------|---------|---------|-------------------
```

### Continuous Coverage Monitoring

**CI/CD Integration**:

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm test -- --coverage
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true
      - name: Comment PR with coverage
        uses: romeovs/lcov-reporter-action@v0.3.1
        with:
          lcov-file: ./coverage/lcov.info
```

**Coverage Badge**:
```markdown
[![Coverage Status](https://codecov.io/gh/org/repo/branch/main/graph/badge.svg)](https://codecov.io/gh/org/repo)
```

## Testing Tools and Configuration

### Core Testing Stack

**Package Dependencies**:
```json
{
  "devDependencies": {
    "jest": "^29.7.0",
    "ts-jest": "^29.1.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/jest-dom": "^6.1.0",
    "@testing-library/user-event": "^14.5.0",
    "@playwright/test": "^1.40.0",
    "jest-axe": "^8.0.0",
    "jest-environment-jsdom": "^29.7.0"
  }
}
```

### Test Setup File

**src/test-setup.ts**:
```typescript
import '@testing-library/jest-dom';
import { configure } from '@testing-library/react';

// Configure Testing Library
configure({
  testIdAttribute: 'data-testid',
  asyncUtilTimeout: 3000
});

// Mock window.matchMedia (for responsive components)
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});

// Mock IntersectionObserver (for lazy loading)
global.IntersectionObserver = class IntersectionObserver {
  constructor() {}
  disconnect() {}
  observe() {}
  takeRecords() { return []; }
  unobserve() {}
};

// Suppress console errors in tests (optional)
const originalError = console.error;
beforeAll(() => {
  console.error = (...args) => {
    if (
      typeof args[0] === 'string' &&
      args[0].includes('Warning: ReactDOM.render')
    ) {
      return;
    }
    originalError.call(console, ...args);
  };
});

afterAll(() => {
  console.error = originalError;
});
```

### Custom Test Utilities

**src/test-utils/render.tsx**:
```typescript
import { render, RenderOptions } from '@testing-library/react';
import { ReactElement } from 'react';

// Custom render with providers (if using Context)
function customRender(
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) {
  return render(ui, {
    wrapper: ({ children }) => (
      // Add providers here if needed
      // <ThemeProvider><IntlProvider>{children}</IntlProvider></ThemeProvider>
      <>{children}</>
    ),
    ...options
  });
}

export * from '@testing-library/react';
export { customRender as render };
```

**src/test-utils/mock-api.ts**:
```typescript
export const mockWorkshopData = {
  id: '123',
  name: 'Test Workshop',
  email: 'test@example.com',
  description: 'Test description',
  services: [],
  images: []
};

export const mockFetch = (data: any, status = 200) => {
  global.fetch = jest.fn(() =>
    Promise.resolve({
      ok: status >= 200 && status < 300,
      status,
      json: () => Promise.resolve(data),
    })
  ) as jest.Mock;
};

export const mockFetchError = (error: string) => {
  global.fetch = jest.fn(() =>
    Promise.reject(new Error(error))
  ) as jest.Mock;
};
```

## Example Test Patterns for React Components

### Pattern 1: Form Validation Testing

```typescript
describe('DataEntryForm', () => {
  test('validates required fields on submit', async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();
    render(<DataEntryForm onSubmit={onSubmit} />);

    await user.click(screen.getByRole('button', { name: 'Submit' }));

    expect(screen.getByText('Workshop Name is required')).toBeInTheDocument();
    expect(screen.getByText('Email is required')).toBeInTheDocument();
    expect(onSubmit).not.toHaveBeenCalled();
  });

  test('submits form with valid data', async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();
    render(<DataEntryForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Workshop Name'), 'React Workshop');
    await user.type(screen.getByLabelText('Email'), 'test@example.com');
    await user.click(screen.getByRole('button', { name: 'Submit' }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        name: 'React Workshop',
        email: 'test@example.com'
      });
    });
  });
});
```

### Pattern 2: Async Data Loading

```typescript
describe('WorkshopList', () => {
  test('shows loading state while fetching data', () => {
    render(<WorkshopList />);
    expect(screen.getByRole('status')).toHaveTextContent('Loading...');
  });

  test('displays workshops after loading', async () => {
    mockFetch([
      { id: '1', name: 'Workshop 1' },
      { id: '2', name: 'Workshop 2' }
    ]);

    render(<WorkshopList />);

    expect(await screen.findByText('Workshop 1')).toBeInTheDocument();
    expect(screen.getByText('Workshop 2')).toBeInTheDocument();
  });

  test('shows error message on fetch failure', async () => {
    mockFetchError('Network error');

    render(<WorkshopList />);

    expect(await screen.findByText(/Failed to load workshops/)).toBeInTheDocument();
  });
});
```

### Pattern 3: User Interaction Flows

```typescript
describe('ImageUpload user flow', () => {
  test('complete upload flow: select → preview → remove', async () => {
    const user = userEvent.setup();
    const onUpload = jest.fn();
    render(<ImageUpload onUpload={onUpload} />);

    // Step 1: Select image
    const file = new File(['image'], 'test.jpg', { type: 'image/jpeg' });
    const input = screen.getByLabelText('Upload Image');
    await user.upload(input, file);

    // Step 2: Preview shown
    expect(await screen.findByAltText('Preview')).toBeInTheDocument();

    // Step 3: Upload confirmed
    expect(onUpload).toHaveBeenCalledWith(file);

    // Step 4: Remove image
    await user.click(screen.getByRole('button', { name: 'Remove' }));

    // Step 5: Preview removed
    expect(screen.queryByAltText('Preview')).not.toBeInTheDocument();
  });
});
```

### Pattern 4: Conditional Rendering

```typescript
describe('ErrorMessage', () => {
  test('does not render when no error', () => {
    render(<ErrorMessage error={null} />);
    expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  });

  test('renders error when provided', () => {
    render(<ErrorMessage error="Something went wrong" />);
    expect(screen.getByRole('alert')).toHaveTextContent('Something went wrong');
  });

  test('dismisses error when close button clicked', async () => {
    const user = userEvent.setup();
    const onDismiss = jest.fn();
    render(<ErrorMessage error="Error" onDismiss={onDismiss} />);

    await user.click(screen.getByRole('button', { name: 'Dismiss' }));
    expect(onDismiss).toHaveBeenCalled();
  });
});
```

## Accessibility Testing (jest-axe)

### Why Accessibility Testing?

**Benefits**:
- Catch a11y issues early in development
- Automated WCAG compliance checks
- Prevent accessibility regressions
- Complement manual screen reader testing

### jest-axe Integration

**Installation**:
```bash
npm install --save-dev jest-axe
```

**Setup** (in test-setup.ts):
```typescript
import { toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);
```

### Accessibility Test Pattern

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('Button accessibility', () => {
  test('has no accessibility violations', async () => {
    const { container } = render(<Button label="Click me" />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  test('has accessible name', () => {
    render(<Button label="Submit form" />);
    expect(screen.getByRole('button')).toHaveAccessibleName('Submit form');
  });

  test('disabled button has correct ARIA state', () => {
    render(<Button label="Submit" disabled />);
    expect(screen.getByRole('button')).toHaveAttribute('aria-disabled', 'true');
  });
});

describe('Form accessibility', () => {
  test('form has no accessibility violations', async () => {
    const { container } = render(<DataEntryForm />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  test('inputs have associated labels', () => {
    render(<InputField label="Email" />);
    const input = screen.getByLabelText('Email');
    expect(input).toBeInTheDocument();
  });

  test('error messages are associated with inputs', async () => {
    const user = userEvent.setup();
    render(<InputField label="Email" required />);

    const input = screen.getByLabelText('Email');
    await user.click(input);
    await user.tab();

    const errorMessage = screen.getByText('Email is required');
    expect(input).toHaveAccessibleDescription('Email is required');
    expect(input).toHaveAttribute('aria-invalid', 'true');
  });
});
```

### Common Accessibility Checks

**Checklist**:
- ✅ All interactive elements have accessible names
- ✅ Form inputs have associated labels
- ✅ Error messages linked to inputs (aria-describedby)
- ✅ Invalid states marked (aria-invalid)
- ✅ Buttons have descriptive text (not just icons)
- ✅ Focus indicators visible
- ✅ Keyboard navigation works (Tab, Enter, Space, Arrow keys)
- ✅ Color contrast meets WCAG AA standards
- ✅ No automatic focus traps
- ✅ Screen reader announces dynamic content (aria-live)

## Visual Regression Testing

### Why Visual Regression Testing?

**Purpose**: Catch unintended visual changes (CSS, layout, styling)

**Tools**: Playwright Visual Comparisons or Percy

### Playwright Visual Comparison

**Configuration**:
```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  expect: {
    toHaveScreenshot: {
      maxDiffPixels: 100  // Allow small differences
    }
  }
});
```

**Visual Regression Test**:
```typescript
test('button renders correctly', async ({ page }) => {
  await page.goto('http://localhost:8080');

  const button = page.getByRole('button', { name: 'Submit' });

  // Take screenshot and compare to baseline
  await expect(button).toHaveScreenshot('button-default.png');
});

test('button hover state', async ({ page }) => {
  await page.goto('http://localhost:8080');

  const button = page.getByRole('button', { name: 'Submit' });
  await button.hover();

  await expect(button).toHaveScreenshot('button-hover.png');
});

test('full page visual regression', async ({ page }) => {
  await page.goto('http://localhost:8080/workshop-data.html');

  // Wait for page to fully load
  await page.waitForLoadState('networkidle');

  await expect(page).toHaveScreenshot('workshop-data-page.png', {
    fullPage: true
  });
});
```

**Baseline Creation**:
```bash
# Generate baseline screenshots
npx playwright test --update-snapshots

# Run visual regression tests
npx playwright test
```

**Handling Changes**:
- Intentional change: Update snapshots with `--update-snapshots`
- Unintended change: Fix CSS/layout bug
- Small differences: Adjust `maxDiffPixels` threshold

## Testing Timeline Integration

### Phase 1: Setup (Weeks 1-2)
- ✅ Install React Testing Library, jest-axe, Playwright
- ✅ Configure Jest with coverage thresholds
- ✅ Create test-setup.ts and test utilities
- ✅ Write example tests for wrapper

### Phase 2: Component Testing (Weeks 3-6)
- ✅ Write unit tests for each migrated component (aim for >80%)
- ✅ Accessibility tests for all components
- ✅ Integration tests for Web Component wrappers
- ✅ Continuous coverage monitoring

### Phase 3: Module Testing (Weeks 7-12)
- ✅ Integration tests for main modules
- ✅ API mocking and data fetching tests
- ✅ Form validation comprehensive testing
- ✅ E2E tests for critical user flows (5-10 flows)

### Phase 4: Final Validation (Weeks 13-15)
- ✅ Achieve >80% overall coverage
- ✅ Visual regression test suite
- ✅ Cross-browser E2E testing
- ✅ Accessibility audit with jest-axe

## Success Metrics

### Quantitative Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **Test Coverage** | ~5% | >80% | Jest coverage report |
| **Test Count** | 5 test files | ~50-100 test files | File count |
| **Accessibility Issues** | Unknown | 0 critical | jest-axe violations |
| **E2E Test Count** | 0 | 5-10 critical flows | Playwright test count |
| **Test Execution Time** | N/A | <5 minutes | Jest + Playwright runtime |

### Qualitative Metrics

- ✅ **Developer Confidence**: Team comfortable making changes
- ✅ **Refactoring Safety**: Can refactor without fear
- ✅ **Regression Prevention**: Tests catch bugs before production
- ✅ **Documentation**: Tests serve as component usage documentation
- ✅ **Accessibility**: WCAG AA compliance validated

## Conclusion

**Testing Strategy Summary**:

1. **React Testing Library**: >80% unit test coverage for all components
2. **Integration Tests**: Web Component wrapper validation
3. **E2E Tests**: 5-10 critical user flows with Playwright
4. **Accessibility**: jest-axe for automated a11y testing
5. **Visual Regression**: Playwright screenshots for CSS stability

**Confidence**: High (9/10) that >80% test coverage achievable with React Testing Library

**Advantage over Vanilla**: React Testing Library reduces testing boilerplate by ~70%, making high coverage realistic.

**Next**: See [08-RISK-MITIGATION.md](./08-RISK-MITIGATION.md) for comprehensive risk analysis.

---

**Strategy Version**: 1.0
**Last Updated**: 2026-02-11
**Status**: Ready for implementation
