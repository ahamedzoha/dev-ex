# Backend Development Learning Path

> A comprehensive guide to Node.js backend development, focusing on enterprise-level architectures and best practices.

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Express.js Advanced Topics](#expressjs-advanced-topics)
3. [NestJS Enterprise Patterns](#nestjs-enterprise-patterns)
4. [Authentication & Authorization](#authentication--authorization)
5. [Database & ORM](#database--orm)
6. [API Design & Implementation](#api-design--implementation)
7. [Microservices Architecture](#microservices-architecture)
8. [Testing & Quality Assurance](#testing--quality-assurance)
9. [DevOps & Deployment](#devops--deployment)
10. [Performance & Optimization](#performance--optimization)

## Core Concepts

### 1. Node.js Fundamentals
- Event Loop & Asynchronous Programming
- Streams & Buffers
- Worker Threads & Multi-threading
- Clustering & Load Balancing
- Memory Management
- Error Handling Patterns
- Module System (CommonJS vs ESM)

### 2. TypeScript Integration
- Type System Fundamentals
- Interfaces & Types
- Decorators & Metadata
- Advanced Types
- Configuration & Best Practices

## Express.js Advanced Topics

### 1. Middleware Architecture
- Custom Middleware Development
- Error Handling Middleware
- Authentication Middleware
- Validation Middleware
- Compression & Security Middleware

### 2. Application Structure
- Route Organization
- Controller Patterns
- Service Layer
- Repository Pattern
- Error Handling
- Validation Strategies

## NestJS Enterprise Patterns

### 1. Core Concepts
- Modules & Dependencies
- Providers & Services
- Controllers & Routes
- Pipes & Filters
- Guards & Interceptors

### 2. Advanced Features
- Custom Decorators
- Dynamic Modules
- Circular Dependencies
- Exception Filters
- Custom Providers

## Authentication & Authorization

### 1. Authentication Strategies
- JWT Implementation
- OAuth 2.0 & OpenID Connect
- Session-based Authentication
- Passport.js Integration
- Multi-factor Authentication

### 2. Authorization
- Role-based Access Control (RBAC)
- Attribute-based Access Control (ABAC)
- Permission Management
- API Keys & Rate Limiting

## Database & ORM

### 1. TypeORM Example
```typescript
// Entity definition
@Entity()
export class Product {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  name: string;

  @Column('decimal', { precision: 10, scale: 2 })
  price: number;

  @ManyToOne(() => Category, category => category.products)
  category: Category;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}

// Repository pattern
@Injectable()
export class ProductRepository {
  constructor(
    @InjectRepository(Product)
    private readonly repository: Repository<Product>
  ) {}

  async findWithPagination(
    page: number,
    limit: number,
    filters: Partial<Product>
  ): Promise<[Product[], number]> {
    return this.repository.findAndCount({
      where: filters,
      skip: (page - 1) * limit,
      take: limit,
      relations: ['category']
    });
  }
}
```

### 2. Prisma Example
```typescript
// Schema definition
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  posts     Post[]
  profile   Profile?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

// Service implementation
@Injectable()
export class UserService {
  constructor(private prisma: PrismaService) {}

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
      include: {
        profile: true,
        posts: {
          take: 5,
          orderBy: { createdAt: 'desc' }
        }
      }
    });
  }
}
```

## API Design & Implementation

### 1. RESTful API Example
```typescript
@Controller('api/v1/products')
export class ProductController {
  constructor(private productService: ProductService) {}

  @Get()
  @UseGuards(AuthGuard)
  @ApiOperation({ summary: 'Get all products' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiQuery({ name: 'limit', required: false, type: Number })
  async getProducts(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number
  ): Promise<PaginatedResponse<Product>> {
    return this.productService.findAll({ page, limit });
  }

  @Post()
  @UseGuards(AuthGuard, RolesGuard)
  @Roles('admin')
  @ApiOperation({ summary: 'Create a product' })
  async createProduct(
    @Body() createProductDto: CreateProductDto
  ): Promise<Product> {
    return this.productService.create(createProductDto);
  }
}
```

### 2. GraphQL API Example
```typescript
// Schema definition
@ObjectType()
class User {
  @Field(() => ID)
  id: string;

  @Field()
  email: string;

  @Field(() => [Post])
  posts: Post[];
}

@Resolver(() => User)
export class UserResolver {
  constructor(private userService: UserService) {}

  @Query(() => [User])
  async users(
    @Args('page') page: number,
    @Args('limit') limit: number
  ): Promise<User[]> {
    return this.userService.findAll({ page, limit });
  }

  @ResolveField()
  async posts(
    @Parent() user: User,
    @Args('status', { nullable: true }) status?: PostStatus
  ): Promise<Post[]> {
    return this.userService.getUserPosts(user.id, status);
  }
}
```

## Microservices Architecture

### 1. NestJS Microservice Example
```typescript
// User microservice
@Controller()
export class UserController {
  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() id: string): Promise<User> {
    return this.userService.findById(id);
  }

  @EventPattern('user_created')
  async handleUserCreated(@Payload() data: CreateUserEvent) {
    // Handle user creation event
    await this.emailService.sendWelcomeEmail(data.email);
  }
}

// API Gateway
@Controller('api/users')
export class UserGatewayController {
  constructor(
    @Inject('USER_SERVICE') private userClient: ClientProxy
  ) {}

  @Get(':id')
  async getUser(@Param('id') id: string) {
    return this.userClient
      .send({ cmd: 'get_user' }, id)
      .pipe(
        timeout(5000),
        catchError(err => throwError(() => new ServiceUnavailableException()))
      );
  }
}
```

### 2. Message Queue Implementation
```typescript
// RabbitMQ producer
@Injectable()
export class OrderProducer {
  constructor(
    @InjectQueue('orders') private ordersQueue: Queue
  ) {}

  async createOrder(order: CreateOrderDto) {
    await this.ordersQueue.add('process_order', order, {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 1000
      }
    });
  }
}

// RabbitMQ consumer
@Processor('orders')
export class OrderConsumer {
  @Process('process_order')
  async processOrder(job: Job<CreateOrderDto>) {
    try {
      await this.orderService.process(job.data);
      await this.notificationService.notify('order_processed', job.data);
    } catch (error) {
      throw new Error('Order processing failed');
    }
  }
}
```

## Testing & Quality Assurance

### 1. Unit Testing Example
```typescript
describe('UserService', () => {
  let service: UserService;
  let repository: MockType<Repository<User>>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(User),
          useFactory: repositoryMockFactory
        }
      ]
    }).compile();

    service = module.get(UserService);
    repository = module.get(getRepositoryToken(User));
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      const user = { id: '1', email: 'test@example.com' };
      repository.findOne.mockReturnValue(user);

      expect(await service.findOne('1')).toEqual(user);
      expect(repository.findOne).toHaveBeenCalledWith('1');
    });
  });
});
```

### 2. E2E Testing Example
```typescript
describe('Auth (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule]
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/auth/login (POST)', () => {
    return request(app.getHttpServer())
      .post('/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123'
      })
      .expect(200)
      .expect(res => {
        expect(res.body).toHaveProperty('access_token');
        expect(res.body).toHaveProperty('refresh_token');
      });
  });
});
```

## DevOps & Deployment

### 1. Docker Configuration
```dockerfile
# Multi-stage build
FROM node:16-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:16-alpine

WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

EXPOSE 3000
CMD ["npm", "run", "start:prod"]
```

### 2. GitHub Actions CI/CD
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '16'
      - run: npm ci
      - run: npm run test
      - run: npm run test:e2e

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v2
      - uses: docker/setup-buildx-action@v1
      - uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v2
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

## Performance & Optimization

### 1. Application Performance
- Caching Strategies
- Memory Optimization
- CPU Profiling
- Database Optimization
- Load Balancing

### 2. Monitoring & Logging
- Prometheus & Grafana
- ELK Stack
- Application Metrics
- Error Tracking
- APM Tools

## Resources

### Documentation
- [Node.js Documentation](https://nodejs.org/docs)
- [Express.js Documentation](https://expressjs.com/)
- [NestJS Documentation](https://docs.nestjs.com)
- [TypeScript Documentation](https://www.typescriptlang.org/docs)

### Recommended Books
- "Node.js Design Patterns" by Mario Casciaro
- "Node.js in Practice" by Alex Young
- "Microservices Patterns" by Chris Richardson
- "Clean Architecture" by Robert C. Martin

### Tools & Platforms
- [TypeScript](https://www.typescriptlang.org/)
- [PM2](https://pm2.keymetrics.io/)
- [Docker](https://www.docker.com/)
- [Kubernetes](https://kubernetes.io/)
- [MongoDB](https://www.mongodb.com/)
- [PostgreSQL](https://www.postgresql.org/)
