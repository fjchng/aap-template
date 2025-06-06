# 5_Testing

## Testing Strategy

### Testing Pyramid

```
🔺 E2E Tests (10%) - Critical user journeys
🔺 Integration Tests (20%) - API + Database + UI
🔺 Unit Tests (70%) - Functions, Components, Services
```

### Coverage Requirements

- **Overall Coverage**: Minimum 80%
- **Unit Test Coverage**: Minimum 85%
- **Critical Path Coverage**: 100% (auth, payments, data loss)

### Quality Standards

- **Test Success Rate**: 100% (all tests must pass)
- **Test Performance**: Unit tests < 5ms, Integration < 500ms
- **Test Naming**: Clear behavior descriptions

---

## Unit Testing

### Test Structure (AAA Pattern)

```typescript
describe("UserService", () => {
  describe("createUser", () => {
    it("should create user with valid data and return user object", async () => {
      // Arrange - Setup test data and mocks
      const userData = {
        email: "test@example.com",
        password: "securepassword123",
        firstName: "John",
        lastName: "Doe",
      };

      const mockUser = { id: "user-123", ...userData };
      jest.spyOn(userRepository, "create").mockResolvedValue(mockUser);

      // Act - Execute the function under test
      const result = await userService.createUser(userData);

      // Assert - Verify the expected outcome
      expect(result).toBeDefined();
      expect(result.id).toBe("user-123");
      expect(userRepository.create).toHaveBeenCalledWith(userData);
    });

    it("should throw ValidationError when email is missing", async () => {
      const invalidData = { password: "test123" } as CreateUserRequest;

      await expect(userService.createUser(invalidData)).rejects.toThrow(
        "Email is required"
      );
    });
  });
});
```

### Frontend Component Testing

```tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { UserCard } from "./UserCard";

describe("UserCard Component", () => {
  const mockUser = {
    id: "user-123",
    email: "john@example.com",
    firstName: "John",
    lastName: "Doe",
  };

  it("should render user information correctly", () => {
    render(<UserCard user={mockUser} />);

    expect(screen.getByText("John Doe")).toBeInTheDocument();
    expect(screen.getByText("john@example.com")).toBeInTheDocument();
  });

  it("should call onEdit when edit button is clicked", async () => {
    const mockOnEdit = jest.fn();
    render(<UserCard user={mockUser} onEdit={mockOnEdit} isEditable />);

    fireEvent.click(screen.getByRole("button", { name: /edit/i }));
    expect(mockOnEdit).toHaveBeenCalledWith(mockUser);
  });
});
```

---

## Integration Testing

### API Testing

```typescript
import request from "supertest";
import { app } from "../app";

describe("User API", () => {
  describe("POST /api/users", () => {
    it("should create new user with valid data", async () => {
      const userData = {
        email: "test@example.com",
        password: "securepassword123",
        firstName: "John",
        lastName: "Doe",
      };

      const response = await request(app)
        .post("/api/users")
        .send(userData)
        .expect(201);

      expect(response.body.success).toBe(true);
      expect(response.body.data.user.email).toBe(userData.email);
      expect(response.body.data.user.password).toBeUndefined();
    });

    it("should return 400 for invalid email", async () => {
      const invalidData = {
        email: "invalid-email",
        password: "securepassword123",
      };

      const response = await request(app)
        .post("/api/users")
        .send(invalidData)
        .expect(400);

      expect(response.body.error.code).toBe("VALIDATION_ERROR");
    });
  });
});
```

### Database Testing

```typescript
describe("UserRepository", () => {
  beforeEach(async () => {
    await dataSource.getRepository(User).clear();
  });

  it("should create and save user to database", async () => {
    const userData = {
      email: "repo@example.com",
      password: "hashedpassword",
      firstName: "Repo",
      lastName: "Test",
    };

    const createdUser = await userRepository.create(userData);

    expect(createdUser.id).toBeDefined();
    expect(createdUser.email).toBe(userData.email);
  });
});
```

---

## E2E Testing

### User Journey Testing

```typescript
import { test, expect } from "@playwright/test";

test.describe("User Authentication Flow", () => {
  test("should complete registration and login", async ({ page }) => {
    const testEmail = `test${Date.now()}@example.com`;

    // Registration
    await page.goto("/register");
    await page.fill('[data-testid="email-input"]', testEmail);
    await page.fill('[data-testid="password-input"]', "SecurePass123!");
    await page.click('[data-testid="register-button"]');

    await expect(page).toHaveURL(/\/verify-email/);

    // Login
    await page.goto("/login");
    await page.fill('[data-testid="login-email"]', testEmail);
    await page.fill('[data-testid="login-password"]', "SecurePass123!");
    await page.click('[data-testid="login-button"]');

    await expect(page).toHaveURL(/\/dashboard/);
  });
});
```

---

## Test Data Management

### Data Factories

```typescript
export class UserFactory {
  static create(overrides: Partial<User> = {}): User {
    return {
      id: `user-${Date.now()}-${Math.random()}`,
      email: `test-${Date.now()}@example.com`,
      firstName: "Test",
      lastName: "User",
      createdAt: new Date(),
      updatedAt: new Date(),
      ...overrides,
    };
  }

  static createMany(count: number): User[] {
    return Array.from({ length: count }, (_, index) =>
      this.create({ email: `test-${Date.now()}-${index}@example.com` })
    );
  }
}
```

### Mock Services

```typescript
export class MockEmailService implements EmailService {
  private sentEmails: Array<{
    to: string;
    subject: string;
    body: string;
  }> = [];

  async sendEmail(to: string, subject: string, body: string): Promise<boolean> {
    this.sentEmails.push({ to, subject, body });
    return true;
  }

  getSentEmails() {
    return this.sentEmails;
  }
}
```

---

## Configuration

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  preset: "ts-jest",
  testEnvironment: "node",
  maxWorkers: "50%",

  testMatch: ["**/__tests__/**/*.test.ts", "**/?(*.)+(spec|test).ts"],

  collectCoverageFrom: ["src/**/*.ts", "!src/**/*.d.ts", "!src/**/*.test.ts"],

  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

### Playwright Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  testDir: "./e2e",
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
  ],
});
```

---

## Best Practices

### Test Quality Checklist

- [ ] Test names clearly describe behavior
- [ ] Each test focuses on single scenario
- [ ] Tests are independent and isolated
- [ ] Realistic test data used
- [ ] Specific assertions, not just "truthy"
- [ ] Error cases tested alongside success cases
- [ ] AAA pattern followed consistently

### AI Agent Guidelines

```typescript
// ✅ Good - Clear inputs/outputs for AI agents
it("should calculate 10% tax on $100 as $10", () => {
  const result = calculateTax(100, 0.1);
  expect(result).toBe(10);
});

// ❌ Bad - Vague test
it("should work correctly", () => {
  const result = calculateTax(100, 0.1);
  expect(result).toBeTruthy();
});
```

### Common Anti-Patterns to Avoid

- Testing multiple behaviors in one test
- Vague test names and assertions
- Excessive mocking
- Tests that depend on each other
- Hard-coded test data without factories
