---
description: Backend Testing Guidelines for Node.js/Express applications
globs: *.test.ts,*.spec.ts,**/test/**/*.ts,**/tests/**/*.ts
alwaysApply: false
---

# Backend Testing Guidelines

## Testing Philosophy and Core Principles

### ✅ DO: Focus on Business Logic and API Contracts

- Test API endpoints from **client perspective**
- Concentrate on **business rules and data transformations**
- Verify **side effects** of operations (database changes, external API calls)

### ❌ DON'T: Test Framework Implementation Details

- Avoid testing Express middleware registration itself
- Don't write tests that distrust the platform (Node.js, Express, etc.)
- Avoid tests that only verify function signatures without behavior

```typescript
// ❌ Testing implementation details
test('should have POST route registered', () => {
    expect(app._router.stack).toBeDefined();
});

// ✅ Behavior-driven testing
test('should create user and return 201 status', async () => {
    const userData = { name: 'John', email: 'john@example.com' };

    const response = await request(app).post('/api/users').send(userData).expect(201);

    expect(response.body.user.name).toBe('John');
    expect(mockUserRepository.create).toHaveBeenCalledWith(userData);
});
```

---

## Architecture and Design

### ✅ DO: Separate Business Logic from HTTP Layer

- Extract logic into **services**, **repositories**, **utilities**
- Controllers should only handle HTTP concerns (request/response)
- Isolate business logic into testable units

### ✅ DO: Design with Testing in Mind

- Hard-to-test code signals design problems
- Maximize pure function usage
- Design external dependencies to be injectable

```typescript
// ✅ Testable structure
export class AuthService {
    constructor(
        private userRepository: UserRepository,
        private tokenService: TokenService,
    ) {}

    async authenticateUser(email: string, password: string): Promise<AuthResult> {
        // Pure business logic
        const user = await this.userRepository.findByEmail(email);
        if (!user || !(await this.validatePassword(password, user.hashedPassword))) {
            throw new UnauthorizedError('Invalid credentials');
        }

        return {
            user,
            token: await this.tokenService.generate(user.id),
        };
    }
}

// ✅ Controller handles HTTP only
export class AuthController {
    constructor(private authService: AuthService) {}

    async login(req: Request, res: Response) {
        try {
            const { email, password } = req.body;
            const result = await this.authService.authenticateUser(email, password);
            res.status(200).json(result);
        } catch (error) {
            // Error handling
        }
    }
}
```

---

## Test Writing Methodology

### ✅ DO: Use Given-When-Then Pattern

Template for consistent test writing:

```typescript
test('should return user profile when valid token provided', async () => {
    // Given: Prepare test data and mocks
    const userId = 'user-123';
    const mockUser = { id: userId, name: 'John', email: 'john@example.com' };
    mockUserRepository.findById.mockResolvedValue(mockUser);
    mockTokenService.verify.mockResolvedValue({ userId });

    // When: Execute the action to test
    const response = await request(app).get('/api/users/profile').set('Authorization', 'Bearer valid-token');

    // Then: Verify results
    expect(response.status).toBe(200);
    expect(response.body.user).toEqual(mockUser);
    expect(mockTokenService.verify).toHaveBeenCalledWith('valid-token');
});
```

### ✅ DO: One Intention Per Test Case

- Each test should verify one specific behavior
- Enable quick identification of failure causes
- Write descriptive test names that explain the scenario

### ❌ DON'T: Write Tests Dependent on External Factors

- Mock external services (APIs, databases, file systems)
- Avoid dependencies on current time, random values
- Ensure tests are deterministic and isolated

---

## Test Type Guidelines

### Unit Testing

#### ✅ DO: Actively Test Service Layer Logic

```typescript
// Service logic testing
describe('AuthService', () => {
    let authService: AuthService;
    let mockUserRepository: jest.Mocked<UserRepository>;
    let mockTokenService: jest.Mocked<TokenService>;

    beforeEach(() => {
        mockUserRepository = createMockUserRepository();
        mockTokenService = createMockTokenService();
        authService = new AuthService(mockUserRepository, mockTokenService);
    });

    test('should throw UnauthorizedError when user not found', async () => {
        // Given
        mockUserRepository.findByEmail.mockResolvedValue(null);

        // When & Then
        await expect(authService.authenticateUser('nonexistent@example.com', 'password')).rejects.toThrow(
            UnauthorizedError,
        );
    });

    test('should return auth result when credentials are valid', async () => {
        // Given
        const mockUser = { id: 'user-123', email: 'john@example.com', hashedPassword: 'hashed' };
        const mockToken = 'jwt-token';
        mockUserRepository.findByEmail.mockResolvedValue(mockUser);
        mockTokenService.generate.mockResolvedValue(mockToken);
        jest.spyOn(authService as any, 'validatePassword').mockResolvedValue(true);

        // When
        const result = await authService.authenticateUser('john@example.com', 'password');

        // Then
        expect(result).toEqual({ user: mockUser, token: mockToken });
        expect(mockTokenService.generate).toHaveBeenCalledWith('user-123');
    });
});
```

#### ✅ DO: Test Repository Layer with Database

```typescript
// Repository testing with test database
describe('UserRepository', () => {
    let userRepository: UserRepository;
    let testDb: Database;

    beforeEach(async () => {
        testDb = await createTestDatabase();
        userRepository = new UserRepository(testDb);
    });

    afterEach(async () => {
        await cleanupTestDatabase(testDb);
    });

    test('should create user with generated ID', async () => {
        // Given
        const userData = { name: 'John', email: 'john@example.com' };

        // When
        const createdUser = await userRepository.create(userData);

        // Then
        expect(createdUser.id).toBeDefined();
        expect(createdUser.name).toBe('John');
        expect(createdUser.createdAt).toBeInstanceOf(Date);
    });
});
```

### Integration Testing

#### ✅ DO: Test API Endpoints End-to-End

```typescript
describe('POST /api/auth/login', () => {
    let app: Express;
    let testDb: Database;

    beforeAll(async () => {
        testDb = await setupTestDatabase();
        app = createTestApp(testDb);
    });

    afterAll(async () => {
        await teardownTestDatabase(testDb);
    });

    beforeEach(async () => {
        await seedTestData(testDb);
    });

    test('should login user with valid credentials', async () => {
        // Given
        const loginData = { email: 'test@example.com', password: 'password123' };

        // When
        const response = await request(app).post('/api/auth/login').send(loginData);

        // Then
        expect(response.status).toBe(200);
        expect(response.body.token).toBeDefined();
        expect(response.body.user.email).toBe('test@example.com');
        expect(response.headers['set-cookie']).toBeDefined(); // Session cookie
    });

    test('should return 401 for invalid credentials', async () => {
        // Given
        const invalidData = { email: 'test@example.com', password: 'wrongpassword' };

        // When
        const response = await request(app).post('/api/auth/login').send(invalidData);

        // Then
        expect(response.status).toBe(401);
        expect(response.body.error).toBe('Invalid credentials');
    });
});
```

#### ✅ DO: Test Middleware Functionality

```typescript
describe('rateLimiter middleware', () => {
    let app: Express;

    beforeEach(() => {
        app = express();
        app.use(rateLimiter);
        app.get('/test', (req, res) => res.json({ success: true }));
    });

    test('should allow requests within limit', async () => {
        // Given & When
        const response = await request(app).get('/test');

        // Then
        expect(response.status).toBe(200);
        expect(response.headers['x-ratelimit-remaining']).toBeDefined();
    });

    test('should block requests exceeding limit', async () => {
        // Given - Make requests up to the limit
        const maxRequests = parseInt(process.env.RATE_LIMIT_MAX_REQUESTS || '100');

        for (let i = 0; i < maxRequests; i++) {
            await request(app).get('/test');
        }

        // When - Exceed the limit
        const response = await request(app).get('/test');

        // Then
        expect(response.status).toBe(429);
        expect(response.body.error).toBe('Too Many Requests');
        expect(response.headers['retry-after']).toBeDefined();
    });
});
```

### ❌ DON'T: Overuse Certain Test Types

#### Avoid Testing Third-Party Libraries

- Don't test Express.js router functionality
- Don't test database driver behavior
- Don't test external API responses (mock them instead)

#### Use E2E Testing Judiciously

- Long execution time and high maintenance cost
- Very fragile to changes
- Apply only to critical user workflows

---

## Practical Testing Strategies

### ✅ DO: Use Test Databases and Transactions

```typescript
// Test database setup
export const createTestDatabase = async (): Promise<Database> => {
    const testDb = new Database({
        ...config.database,
        database: `${config.database.database}_test_${Date.now()}`,
    });

    await testDb.migrate();
    return testDb;
};

// Transaction-based test isolation
describe('UserService', () => {
    let transaction: Transaction;

    beforeEach(async () => {
        transaction = await db.beginTransaction();
    });

    afterEach(async () => {
        await transaction.rollback();
    });

    test('should create user within transaction', async () => {
        const userService = new UserService(transaction);
        const user = await userService.createUser({ name: 'John' });
        expect(user.id).toBeDefined();
        // Transaction will be rolled back, no cleanup needed
    });
});
```

### ✅ DO: Mock External Dependencies

```typescript
// Mock factory for external services
export const createMockKakaoAPI = () => ({
    getToken: jest.fn(),
    getUserInfo: jest.fn(),
    refreshToken: jest.fn(),
});

describe('KakaoAuthService', () => {
    let kakaoAuthService: KakaoAuthService;
    let mockKakaoAPI: ReturnType<typeof createMockKakaoAPI>;

    beforeEach(() => {
        mockKakaoAPI = createMockKakaoAPI();
        kakaoAuthService = new KakaoAuthService(mockKakaoAPI);
    });

    test('should handle Kakao API errors gracefully', async () => {
        // Given
        mockKakaoAPI.getToken.mockRejectedValue(new Error('Kakao API Error'));

        // When & Then
        await expect(kakaoAuthService.authenticate('invalid-code')).rejects.toThrow('Authentication failed');
    });
});
```

### ✅ DO: Prioritize Quality Over Coverage

- Avoid writing tests just to meet coverage metrics
- Focus on business-critical paths
- Test edge cases and error scenarios

### ✅ DO: Establish Testing Strategy Based on Risk

```typescript
// High-risk authentication flow: Comprehensive testing
describe('Authentication Flow', () => {
    // Unit tests for service logic
    // Integration tests for API endpoints
    // Error scenario tests
    // Security tests
});

// Low-risk utility functions: Basic testing
describe('Date Utilities', () => {
    test('should format date correctly', () => {
        // Simple behavior verification
    });
});
```

---

## Test Environment and Setup

### ✅ DO: Isolate Test Environment

```typescript
// Test configuration
export const testConfig = {
    database: {
        host: process.env.TEST_DB_HOST || 'localhost',
        port: parseInt(process.env.TEST_DB_PORT || '5433'),
        database: `${process.env.DB_NAME}_test`,
    },
    redis: {
        host: process.env.TEST_REDIS_HOST || 'localhost',
        port: parseInt(process.env.TEST_REDIS_PORT || '6380'),
    },
};

// Test setup helper
export const setupTestApp = async (): Promise<Express> => {
    const app = express();

    // Use test-specific middleware and configuration
    app.use(express.json());
    app.use('/api', createTestRoutes());

    return app;
};
```

### ✅ DO: Manage Test Data Effectively

```typescript
// Test data factories
export const createTestUser = (overrides: Partial<User> = {}): User => ({
    id: `user-${Date.now()}`,
    name: 'Test User',
    email: `test-${Date.now()}@example.com`,
    createdAt: new Date(),
    ...overrides,
});

// Test data seeding
export const seedTestData = async (db: Database) => {
    const users = [
        createTestUser({ email: 'admin@example.com', role: 'admin' }),
        createTestUser({ email: 'user@example.com', role: 'user' }),
    ];

    await db.users.insertMany(users);
    return { users };
};
```

### ✅ DO: Handle Async Operations Properly

```typescript
// Proper async testing
test('should handle concurrent user creation', async () => {
    // Given
    const userData1 = { name: 'User1', email: 'user1@example.com' };
    const userData2 = { name: 'User2', email: 'user2@example.com' };

    // When - Execute concurrent operations
    const [user1, user2] = await Promise.all([userService.createUser(userData1), userService.createUser(userData2)]);

    // Then
    expect(user1.id).not.toBe(user2.id);
    expect(user1.email).toBe('user1@example.com');
    expect(user2.email).toBe('user2@example.com');
});
```

---

## Error Handling and Edge Cases

### ✅ DO: Test Error Scenarios Thoroughly

```typescript
describe('Error Handling', () => {
    test('should handle database connection errors', async () => {
        // Given
        mockDatabase.findById.mockRejectedValue(new Error('Connection lost'));

        // When
        const response = await request(app).get('/api/users/123').expect(500);

        // Then
        expect(response.body.error).toBe('Internal Server Error');
        expect(mockLogger.error).toHaveBeenCalledWith(expect.stringContaining('Connection lost'));
    });

    test('should validate request input', async () => {
        // Given
        const invalidData = { email: 'invalid-email' }; // Missing required fields

        // When
        const response = await request(app).post('/api/users').send(invalidData).expect(400);

        // Then
        expect(response.body.errors).toContain('Name is required');
        expect(response.body.errors).toContain('Invalid email format');
    });
});
```

### ✅ DO: Test Authentication and Authorization

```typescript
describe('Authentication & Authorization', () => {
    test('should require authentication for protected routes', async () => {
        // When
        const response = await request(app).get('/api/users/profile').expect(401);

        // Then
        expect(response.body.error).toBe('Authentication required');
    });

    test('should deny access for insufficient permissions', async () => {
        // Given
        const userToken = await createTestToken({ role: 'user' });

        // When
        const response = await request(app)
            .delete('/api/users/123')
            .set('Authorization', `Bearer ${userToken}`)
            .expect(403);

        // Then
        expect(response.body.error).toBe('Insufficient permissions');
    });
});
```

---

## Performance and Load Testing

### ✅ DO: Test Performance-Critical Paths

```typescript
describe('Performance Tests', () => {
    test('should handle bulk operations efficiently', async () => {
        // Given
        const users = Array.from({ length: 1000 }, (_, i) => createTestUser({ email: `user${i}@example.com` }));

        // When
        const startTime = Date.now();
        await userService.bulkCreate(users);
        const duration = Date.now() - startTime;

        // Then
        expect(duration).toBeLessThan(5000); // Should complete within 5 seconds
    });

    test('should handle concurrent requests', async () => {
        // Given
        const requests = Array.from({ length: 50 }, () => request(app).get('/api/health'));

        // When
        const responses = await Promise.all(requests);

        // Then
        responses.forEach(response => {
            expect(response.status).toBe(200);
        });
    });
});
```

---

## Core Principles Summary

1. **API Contract Focus**: Test from client perspective, verify contracts
2. **Business Logic Isolation**: Separate and thoroughly test business rules
3. **Dependency Management**: Mock external dependencies, use test doubles
4. **Error Resilience**: Test error scenarios and edge cases comprehensively
5. **Performance Awareness**: Include performance considerations in critical paths
6. **Security First**: Test authentication, authorization, and input validation
7. **Environment Isolation**: Use dedicated test environments and data

---

**Remember**: Backend testing is about ensuring **reliable, secure, and performant APIs**. It provides confidence in business logic and system integrations, enabling **safe deployments and refactoring**.
