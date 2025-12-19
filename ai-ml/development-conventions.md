# Development Conventions & Team Guidelines

<div align="center">

![Standards](https://img.shields.io/badge/Team-Conventions-green?style=for-the-badge)
![Git](https://img.shields.io/badge/Git-Workflow-F05032?style=for-the-badge&logo=git&logoColor=white)

*Standardized conventions for consistent code quality and team alignment.*

</div>

## Git Workflow

### Branch Naming

```
feature/<ticket-id>-<description>
bugfix/<ticket-id>-<description>
hotfix/<ticket-id>-<description>
chore/<description>
```

Examples:
```
feature/PEAK-102-add-payment-method
bugfix/PEAK-88-fix-invalid-jwt
hotfix/PEAK-200-critical-auth-fix
chore/update-dependencies
```

### Commit Message Standard

Use Conventional Commit format: `type(scope): message`

| Type | Description |
|------|-------------|
| feat | New feature |
| fix | Bug fix |
| refactor | Code or structure improvements |
| chore | Maintenance/update dependencies |
| test | Add or update tests |
| docs | Documentation change |

Examples:
```
feat(auth): implement JWT refresh token
fix(user): handle duplicated phone number validation
refactor(order): simplify pricing calculation function
test(payment): add integration tests for Stripe webhook
docs(api): update OpenAPI specification
```

## Code Naming Rules

### General Conventions

| Case | Used For |
|------|----------|
| camelCase | Variables and functions |
| PascalCase | Interfaces, components, classes |
| SNAKE_UPPERCASE | Constants |

```typescript
// Constants
const API_TIMEOUT_MS = 3000;
const MAX_RETRY_ATTEMPTS = 5;

// Interfaces
interface ProductRequestDto {
  name: string;
  price: number;
}

// Functions
function handlePayment(orderId: string): Promise<PaymentResult> {
  // ...
}

// Variables
const isLoading = true;
const hasError = false;
```

## Frontend Standards

### Stack
- Next.js (App Router)
- React
- TypeScript (strict mode)

### Folder Structure

```
/app              # Next.js App Router pages
/components       # Reusable UI components
/hooks            # Custom React hooks
/services         # API client and services
/interfaces       # TypeScript interfaces
/utils            # Utility functions
/constants        # Application constants
/enums            # TypeScript enums
```

### File Naming

```
LoginForm.tsx           # Component
useOrders.ts            # Hook
order.service.ts        # Service
order.interface.ts      # Interface
order.enum.ts           # Enum
order.constants.ts      # Constants
```

### Rules
- Logic goes into hooks
- UI goes into components
- Use strict TypeScript
- Avoid large files (>200 lines per component)

## Backend Standards

### REST Naming Rules

Bad:
```
/getUserList
/updateUserById
/deleteUserRecord
```

Good:
```
GET    /users
GET    /users/:id
POST   /users
PATCH  /users/:id
DELETE /users/:id
```

### DTO Naming

```typescript
// Request DTOs
interface CreateUserRequestDto { }
interface UpdateUserRequestDto { }
interface LoginRequestDto { }

// Response DTOs
interface UserResponseDto { }
interface LoginResponseDto { }
interface PaginatedResponseDto<T> { }
```

### Business Rules
- No business logic inside controllers
- Service layer handles core logic
- Validation layer must run before execution

## Pull Request Workflow

### Title Format

```
PEAK-### - short description
```

Example: `PEAK-120 - Implement new deposit validation`

### Description Template

```markdown
## Summary
- Provide short summary of changes

## Implementation Details
- Explain changes and reasoning
- List key decisions made

## Test Instructions
1. Step one
2. Step two

## Risks & Notes
- Edge cases
- Warnings
- Breaking changes
```

### Reviewer Checklist
- [ ] No console logs
- [ ] Remove unused imports
- [ ] Readable and structured code
- [ ] Small and focused functions
- [ ] Strict typings
- [ ] Testable behavior

## Team Communication

### Status Updates

**When Starting:**
```
Starting PEAK-XXX <task>, ETA: <Time>
```

**When Blocked:**
```
Blocked on PEAK-XXX
Reason: <reason>
Next Step: waiting response from <person/team>
```

**When Completed:**
```
PEAK-XXX completed, PR submitted -> ready for review
```

## Deployment Checklist

- [ ] DB migration verified and reversible
- [ ] Rollback plan prepared
- [ ] QA passed
- [ ] Test cases covered
- [ ] Sensitive logs removed
- [ ] Backward API compatibility validated
- [ ] Environment variables configured
- [ ] Performance benchmarks checked

## Benefits

- Faster onboarding for new team members
- Better developer alignment across teams
- Less repeated defects
- Predictable delivery timeline
- Better traceability
- Higher customer trust

---

*Learned: December 20, 2025*
*Tags: Conventions, Git, Team, Best Practices, Standards*
