# Serverless Best Practices

## Function Design

### Single Responsibility

```typescript
// ❌ Bad: Multiple responsibilities
export const handler = async (event: APIGatewayEvent) => {
  if (event.path === '/users') {
    // Handle users
  } else if (event.path === '/orders') {
    // Handle orders
  } else if (event.path === '/products') {
    // Handle products
  }
};

// ✅ Good: One function per responsibility
// createUser.ts
export const handler = async (event: APIGatewayEvent) => {
  const userData = JSON.parse(event.body);
  const user = await userService.create(userData);
  return { statusCode: 201, body: JSON.stringify(user) };
};
```

### Keep Functions Small

```typescript
// ✅ Good: Small, focused function
export const handler = async (event: SNSEvent) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);
    await processMessage(message);
  }
};

// Extract business logic to separate module
async function processMessage(message: OrderMessage): Promise<void> {
  const order = await orderService.process(message);
  await notificationService.sendConfirmation(order);
}
```

## Cold Start Optimization

### Minimize Dependencies

```typescript
// ❌ Bad: Heavy imports at top level
import * as AWS from 'aws-sdk';
import moment from 'moment';
import _ from 'lodash';

// ✅ Good: Import only what you need
import { DynamoDB } from '@aws-sdk/client-dynamodb';

// ✅ Good: Lazy load optional dependencies
let heavyLib: typeof import('heavy-lib') | undefined;

async function useHeavyFeature() {
  if (!heavyLib) {
    heavyLib = await import('heavy-lib');
  }
  return heavyLib.process();
}
```

### Initialize Outside Handler

```typescript
// ✅ Good: Reuse connections across invocations
import { DynamoDB } from '@aws-sdk/client-dynamodb';

// Created once, reused
const dynamodb = new DynamoDB({});
let cachedConnection: Connection | undefined;

export const handler = async (event: Event) => {
  // Reuse existing connection
  if (!cachedConnection) {
    cachedConnection = await createConnection();
  }

  return process(event, cachedConnection);
};
```

### Provisioned Concurrency

```yaml
# serverless.yml
functions:
  api:
    handler: handler.api
    provisionedConcurrency: 5  # Keep 5 instances warm
```

## Error Handling

### Structured Error Responses

```typescript
class LambdaError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message);
  }
}

export const handler = async (event: APIGatewayEvent) => {
  try {
    const result = await processRequest(event);
    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    if (error instanceof LambdaError) {
      return {
        statusCode: error.statusCode,
        body: JSON.stringify({
          error: { code: error.code, message: error.message }
        })
      };
    }

    console.error('Unexpected error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        error: { code: 'INTERNAL_ERROR', message: 'Internal server error' }
      })
    };
  }
};
```

### Retry and Dead Letter Queues

```yaml
# CloudFormation
Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      DeadLetterConfig:
        TargetArn: !GetAtt DeadLetterQueue.Arn

  DeadLetterQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: my-function-dlq
```

## State Management

### Use External State Stores

```typescript
// ❌ Bad: In-memory state (lost between invocations)
let requestCount = 0;

export const handler = async () => {
  requestCount++; // Unreliable!
};

// ✅ Good: External state store
import { DynamoDB } from '@aws-sdk/client-dynamodb';

const dynamodb = new DynamoDB({});

export const handler = async (event: Event) => {
  // Atomic counter in DynamoDB
  await dynamodb.updateItem({
    TableName: 'Counters',
    Key: { id: { S: 'requests' } },
    UpdateExpression: 'ADD #count :inc',
    ExpressionAttributeNames: { '#count': 'count' },
    ExpressionAttributeValues: { ':inc': { N: '1' } }
  });
};
```

### Step Functions for Workflows

```yaml
# Step Functions state machine
StartAt: ValidateOrder
States:
  ValidateOrder:
    Type: Task
    Resource: arn:aws:lambda:...:validateOrder
    Next: ProcessPayment

  ProcessPayment:
    Type: Task
    Resource: arn:aws:lambda:...:processPayment
    Catch:
      - ErrorEquals: [PaymentFailed]
        Next: NotifyFailure
    Next: FulfillOrder

  FulfillOrder:
    Type: Task
    Resource: arn:aws:lambda:...:fulfillOrder
    End: true

  NotifyFailure:
    Type: Task
    Resource: arn:aws:lambda:...:notifyFailure
    End: true
```

## Security

### Least Privilege IAM

```yaml
# serverless.yml
provider:
  iam:
    role:
      statements:
        # Only the permissions needed
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
          Resource: arn:aws:dynamodb:*:*:table/Users

        - Effect: Allow
          Action:
            - s3:GetObject
          Resource: arn:aws:s3:::my-bucket/*
```

### Secrets Management

```typescript
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const secretsManager = new SecretsManager({});
let cachedSecret: string | undefined;

async function getSecret(): Promise<string> {
  if (!cachedSecret) {
    const response = await secretsManager.getSecretValue({
      SecretId: 'my-api-key'
    });
    cachedSecret = response.SecretString;
  }
  return cachedSecret!;
}
```

## Monitoring and Observability

### Structured Logging

```typescript
import { Logger } from '@aws-lambda-powertools/logger';

const logger = new Logger({
  serviceName: 'order-service',
  logLevel: 'INFO'
});

export const handler = async (event: Event, context: Context) => {
  logger.addContext(context);

  logger.info('Processing order', {
    orderId: event.orderId,
    customerId: event.customerId
  });

  try {
    const result = await processOrder(event);
    logger.info('Order processed', { orderId: event.orderId });
    return result;
  } catch (error) {
    logger.error('Order processing failed', { error, event });
    throw error;
  }
};
```

### Tracing

```typescript
import { Tracer } from '@aws-lambda-powertools/tracer';

const tracer = new Tracer({ serviceName: 'order-service' });

export const handler = async (event: Event) => {
  const segment = tracer.getSegment();
  const subsegment = segment.addNewSubsegment('ProcessOrder');

  try {
    const result = await processOrder(event);
    subsegment.close();
    return result;
  } catch (error) {
    subsegment.addError(error);
    subsegment.close();
    throw error;
  }
};
```

## Cost Optimization

- Set appropriate memory (more memory = faster CPU)
- Use ARM architecture when possible (cheaper)
- Batch operations to reduce invocations
- Use reserved concurrency to limit costs
- Monitor and alert on spending
- Clean up unused functions and versions
