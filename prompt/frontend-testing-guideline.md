---
description: 
globs: *.test.(ts|tsx),*.spec.(ts|tsx),**/test/**/*.(ts|tsx),**/tests/**/*.(ts|tsx),  
alwaysApply: false
---

# Frontend Testing Guidelines

## Testing Philosophy and Core Principles

### ✅ DO: Focus on Behavior-Driven Testing

- Test application behavior from **user perspective**
- Concentrate on **business logic and data flow**
- Verify **side effects** of user interactions

### ❌ DON'T: Test Implementation Details

- Avoid testing component rendering itself
- Don't write tests that distrust the platform (React, Vue, etc.)
- Avoid tests that only verify event handler registration

```javascript
// ❌ Testing implementation details
test('should render button element', () => {
    const { getByRole } = render(<Button />);
    expect(getByRole('button')).toBeInTheDocument();
});

// ✅ Behavior-driven testing
test('should update user status when button is clicked', () => {
    const { getByText } = render(<UserProfile />);
    fireEvent.click(getByText('Activate'));
    expect(mockUpdateUserStatus).toHaveBeenCalledWith({ active: true });
});
```

---

## Architecture and Design

### ✅ DO: Separate Business Logic from UI Rendering

- Extract logic into **custom hooks**, **middleware**, **selectors**
- Components should only handle UI rendering
- Isolate logic into testable units

### ✅ DO: Design with Testing in Mind

- Hard-to-test code signals design problems
- Maximize pure function usage
- Design external dependencies to be injectable

```javascript
// ✅ Testable structure
const useUserManagement = () => {
    const updateUser = useCallback(userData => {
        // Business logic
        return userService.update(userData);
    }, []);

    return { updateUser };
};

// ✅ Component handles UI only
const UserProfile = () => {
    const { updateUser } = useUserManagement();

    return <button onClick={() => updateUser({ active: true })}>Activate</button>;
};
```

---

## Test Writing Methodology

### ✅ DO: Use Given-When-Then Pattern

Template for consistent test writing:

```javascript
test('should [expected result]', () => {
    // Given: Prepare data and state for testing
    const mockData = { id: 1, name: 'John' };

    // When: Execute the action to test
    const result = processUser(mockData);

    // Then: Verify results
    expect(result).toEqual({ id: 1, name: 'John', processed: true });
});
```

### ✅ DO: One Intention Per Test Case

- Each test should contain only one assumption and one verification
- Enable quick identification of failure causes
- Write meaningful test names

### ❌ DON'T: Write Tests Dependent on External Factors

- Test results should be determined solely by assumptions
- Avoid dependencies on network, time, random values
- Ensure identical results for identical inputs

---

## Test Type Guidelines

### Unit Testing

#### ✅ DO: Actively Test Store Logic

```javascript
// Reducer testing
test('should update user when UPDATE_USER action is dispatched', () => {
    // Given
    const initialState = { user: null };
    const action = { type: 'UPDATE_USER', payload: { id: 1, name: 'John' } };

    // When
    const newState = userReducer(initialState, action);

    // Then
    expect(newState.user).toEqual({ id: 1, name: 'John' });
});

// Selector testing
test('should return active users only', () => {
    // Given
    const state = {
        users: [
            { id: 1, active: true },
            { id: 2, active: false },
        ],
    };

    // When
    const activeUsers = selectActiveUsers(state);

    // Then
    expect(activeUsers).toHaveLength(1);
    expect(activeUsers[0].id).toBe(1);
});
```

#### ✅ DO: Test Asynchronous Logic (Middleware)

```javascript
test('should handle user creation flow', async () => {
    // Given
    const userData = { name: 'John', email: 'john@example.com' };

    // When
    const { effects } = await expectSaga(createUserSaga)
        .provide([[matchers.call.fn(apiService.createUser), { id: 1, ...userData }]])
        .put(loadingActions.start())
        .call(apiService.createUser, userData)
        .put(userActions.setUser({ id: 1, ...userData }))
        .put(loadingActions.finish())
        .run();

    // Then
    expect(effects).toBeDefined();
});
```

### Integration Testing

#### ✅ DO: Test Real User Scenarios

```javascript
test('should complete user registration flow', () => {
    // Given
    render(<RegistrationFlow />);

    // When
    fireEvent.change(screen.getByLabelText('Name'), {
        target: { value: 'John Doe' },
    });
    fireEvent.change(screen.getByLabelText('Email'), {
        target: { value: 'john@example.com' },
    });
    fireEvent.click(screen.getByText('Register'));

    // Then
    expect(screen.getByText('Registration successful')).toBeInTheDocument();
});
```

### ❌ DON'T: Overuse Certain Test Types

#### Avoid Snapshot Testing

- Fragile to changes and difficult to debug
- Detects changes rather than finding actual bugs
- Use only when regression testing is absolutely necessary

#### Use E2E Testing Judiciously

- Long execution time and high maintenance cost
- Very fragile to changes
- Apply only to core user scenarios

---

## Practical Testing Strategies

### ✅ DO: Leverage Storybook for Visual Validation

- Write page-level stories to verify complex states
- Reuse mock data for various scenario validation
- Enable quick UI bug identification and debugging

```javascript
// Page-level story example
const meta = {
    title: 'Pages/UserDashboard',
    component: UserDashboard,
    decorators: [
        withProvider({
            user: mockUserData,
            settings: mockSettingsData,
        }),
    ],
};

export const Default = {};
export const WithNotifications = {
    decorators: [
        withProvider({
            user: mockUserData,
            notifications: mockNotifications,
        }),
    ],
};
```

### ✅ DO: Prioritize Quality Over Coverage

- Avoid writing tests just to meet coverage metrics
- Focus on meaningful test cases
- Prioritize business-critical components

### ✅ DO: Establish Testing Strategy Based on Project Nature

```javascript
// Long-term projects: Comprehensive testing
describe('User Management', () => {
    // Include unit, integration, and visual tests
});

// Short-term event pages: Minimal testing
describe('Event Landing', () => {
    // Test only core functionality
    test('should track event participation', () => {
        // Business-critical parts only
    });
});
```

---

## Test Optimization

### ✅ DO: Manage Mock Data Efficiently

- Reuse mock data created for store tests
- Write mock factory functions for various state combinations
- Ensure independence between tests

```javascript
// Mock factory pattern
const createMockUser = (overrides = {}) => ({
    id: 1,
    name: 'John Doe',
    email: 'john@example.com',
    active: true,
    ...overrides,
});

// Usage example
test('should handle inactive user', () => {
    const inactiveUser = createMockUser({ active: false });
    // Test logic
});
```

### ✅ DO: Consider Test Maintainability

- Extract repetitive test setup into helper functions
- Manage test data in separate files
- Design structure to minimize test breakage frequency

### ❌ DON'T: Test Platform Features

- Trust framework for React rendering, event handling, etc.
- Don't test browser API behavior
- Exclude basic functionality of third-party libraries

---

## Core Principles Summary

1. **Purpose Clarity**: Verify application works as users intend
2. **Design First**: Structure code to be easily testable
3. **Behavior Focus**: Concentrate on behavior, not implementation
4. **Efficiency**: Establish testing strategy considering ROI
5. **Practicality**: Flexible approach based on project characteristics

---

**Remember**: Testing is not about finding bugs, but creating **reliable software**. It provides a safety net for code changes, enabling **sustainable development**.
