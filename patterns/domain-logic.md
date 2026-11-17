# Domain Logic Patterns

## Rules

- Choose by complexity: Transaction Script for simple procedural logic, Domain Model for rich rules, Service Layer to coordinate use cases and transactions.
- Put business rules in the domain objects (Domain Model), not in controllers or the service layer.
- Keep the service layer thin: orchestration, transactions and authorization, not rules.
- Start simple and move to a richer model when conditionals multiply.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Transaction Script

Procedural approach - one procedure per operation.

```typescript
async function transferMoney(fromId: string, toId: string, amount: number) {
  const db = await Database.connect();
  await db.beginTransaction();

  try {
    const from = await db.query('SELECT * FROM accounts WHERE id = $1', [fromId]);
    if (from.balance < amount) throw new Error('Insufficient funds');

    await db.execute('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, fromId]);
    await db.execute('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, toId]);
    await db.commit();
  } catch (e) {
    await db.rollback();
    throw e;
  }
}
```

**Use for:** Simple apps, CRUD, reports.

## Domain Model

Rich objects with behavior.

```typescript
class Account {
  constructor(private balance: Money, private overdraftLimit: Money) {}

  withdraw(amount: Money): void {
    if (!this.canWithdraw(amount)) throw new InsufficientFundsError();
    this.balance = this.balance.subtract(amount);
  }

  transfer(amount: Money, recipient: Account): void {
    this.withdraw(amount);
    recipient.deposit(amount);
  }
}
```

**Use for:** Complex business rules, rich domains.

## Service Layer

Application boundary coordinating domain objects.

```typescript
class AccountService {
  constructor(
    private accountRepo: AccountRepository,
    private unitOfWork: UnitOfWork
  ) {}

  async transfer(fromId: string, toId: string, amount: Money): Promise<void> {
    const from = await this.accountRepo.findById(fromId);
    const to = await this.accountRepo.findById(toId);

    from.transfer(amount, to); // Domain logic

    await this.accountRepo.save(from);
    await this.accountRepo.save(to);
    await this.unitOfWork.commit();
  }
}
```

**Use for:** API boundaries, multiple clients, transaction coordination.

## Best Practices

- Choose pattern based on complexity
- Service layer orchestrates, domain model contains logic
- Keep services thin, domain objects rich
- Combine Domain Model + Service Layer for complex apps
