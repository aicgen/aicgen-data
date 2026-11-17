# Test Doubles and Mocking

## Rules

- Mock only boundaries you do not own or that are slow or non-deterministic: network, database, clock, filesystem, email. Do not mock the code under test or simple value objects.
- Pick the lightest double: stub for canned answers, fake for working in-memory behavior, spy or mock only when the interaction itself is the behavior.
- Verify interactions sparingly; prefer asserting on the result.
- Reset or recreate doubles between tests so state cannot leak.
- If a test needs many mocks, the unit has too many dependencies; simplify the design instead.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Types of Test Doubles

```typescript
// STUB: Returns canned responses
const stubUserRepo = {
  findById: () => ({ id: '1', name: 'Test User' })
};

// MOCK: Pre-programmed with expectations
const mockPaymentGateway = {
  charge: jest.fn()
    .mockResolvedValueOnce({ success: true, transactionId: 'tx1' })
    .mockResolvedValueOnce({ success: false, error: 'Declined' })
};

// SPY: Records calls for verification
const spy = jest.spyOn(emailService, 'send');

// FAKE: Working implementation (not for production)
class FakeDatabase implements Database {
  private data = new Map<string, any>();

  async save(id: string, entity: any) { this.data.set(id, entity); }
  async find(id: string) { return this.data.get(id); }
}
```

## When to Mock

```typescript
// ✅ Mock external services (APIs, databases)
const mockHttpClient = {
  get: jest.fn().mockResolvedValue({ data: userData })
};

// ✅ Mock time-dependent operations
jest.useFakeTimers();
jest.setSystemTime(new Date('2024-01-15'));

// ✅ Mock random/non-deterministic functions
jest.spyOn(Math, 'random').mockReturnValue(0.5);

// ❌ Don't mock the code you're testing
// ❌ Don't mock simple data structures
```

## Mock Verification

```typescript
it('should send welcome email after registration', async () => {
  const mockEmail = { send: jest.fn().mockResolvedValue(true) };
  const service = new UserService({ emailService: mockEmail });

  await service.register({ email: 'new@example.com' });

  expect(mockEmail.send).toHaveBeenCalledWith({
    to: 'new@example.com',
    template: 'welcome',
    subject: 'Welcome!'
  });
  expect(mockEmail.send).toHaveBeenCalledTimes(1);
});
```

## Partial Mocks

```typescript
// Mock only specific methods
const service = new OrderService();

jest.spyOn(service, 'validateOrder').mockReturnValue(true);
jest.spyOn(service, 'calculateTotal').mockReturnValue(100);
// Other methods use real implementation

const result = await service.processOrder(orderData);
expect(result.total).toBe(100);
```

## Resetting Mocks

```typescript
describe('PaymentService', () => {
  const mockGateway = { charge: jest.fn() };
  const service = new PaymentService(mockGateway);

  beforeEach(() => {
    jest.clearAllMocks(); // Clear call history
    // or jest.resetAllMocks() to also reset return values
  });

  it('should process payment', async () => {
    mockGateway.charge.mockResolvedValue({ success: true });
    await service.charge(100);
    expect(mockGateway.charge).toHaveBeenCalledTimes(1);
  });
});
```

## Mock Modules

```typescript
// Mock entire module
jest.mock('./email-service', () => ({
  EmailService: jest.fn().mockImplementation(() => ({
    send: jest.fn().mockResolvedValue(true)
  }))
}));

// Mock with partial implementation
jest.mock('./config', () => ({
  ...jest.requireActual('./config'),
  API_KEY: 'test-key'
}));
```
