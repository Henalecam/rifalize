# Technical Implementation Guide

## Frontend Architecture (Next.js 14)
### Core Structure
```typescript
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="pt-BR">
      <body>
        <Providers>
          <Header />
          {children}
          <Footer />
        </Providers>
      </body>
    </html>
  )
}

// app/providers.tsx
export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider>
        <AuthProvider>
          {children}
        </AuthProvider>
      </ThemeProvider>
    </QueryClientProvider>
  )
}
```

### State Management
```typescript
// store/useAuthStore.ts
interface AuthState {
  user: User | null
  token: string | null
  isAuthenticated: boolean
}

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  token: null,
  isAuthenticated: false,
  setUser: (user: User) => set({ user, isAuthenticated: true }),
  logout: () => set({ user: null, token: null, isAuthenticated: false })
}))

// store/useRaffleStore.ts
interface RaffleState {
  currentRaffle: Raffle | null
  tickets: Ticket[]
  loading: boolean
}

export const useRaffleStore = create<RaffleState>((set) => ({
  currentRaffle: null,
  tickets: [],
  loading: false,
  setRaffle: (raffle: Raffle) => set({ currentRaffle: raffle }),
  addTickets: (tickets: Ticket[]) => set((state) => ({
    tickets: [...state.tickets, ...tickets]
  }))
}))
```

### API Integration
```typescript
// lib/api.ts
const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
})

api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// hooks/useRaffle.ts
export function useRaffle(id: string) {
  return useQuery({
    queryKey: ['raffle', id],
    queryFn: () => api.get(`/raffles/${id}`).then(res => res.data),
    enabled: !!id
  })
}
```

## Backend Architecture (Nest.js)
### Core Modules
```typescript
// src/modules/auth/auth.module.ts
@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '1d' }
    }),
    UsersModule
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy]
})
export class AuthModule {}

// src/modules/raffles/raffles.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([Raffle, Ticket]),
    UsersModule,
    PaymentsModule
  ],
  controllers: [RafflesController],
  providers: [RafflesService, TicketService]
})
export class RafflesModule {}
```

### Database Entities
```typescript
// src/entities/user.entity.ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column({ unique: true })
  email: string

  @Column()
  password: string

  @Column()
  name: string

  @Column()
  role: UserRole

  @OneToMany(() => Raffle, raffle => raffle.owner)
  raffles: Raffle[]

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date
}

// src/entities/raffle.entity.ts
@Entity('raffles')
export class Raffle {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  title: string

  @Column('text')
  description: string

  @Column()
  totalNumbers: number

  @Column('decimal', { precision: 10, scale: 2 })
  pricePerNumber: number

  @Column()
  status: RaffleStatus

  @ManyToOne(() => User, user => user.raffles)
  owner: User

  @OneToMany(() => Ticket, ticket => ticket.raffle)
  tickets: Ticket[]

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date
}
```

### Services Implementation
```typescript
// src/services/raffle.service.ts
@Injectable()
export class RaffleService {
  constructor(
    @InjectRepository(Raffle)
    private raffleRepository: Repository<Raffle>,
    private ticketService: TicketService,
    private paymentService: PaymentService
  ) {}

  async createRaffle(data: CreateRaffleDto, userId: string): Promise<Raffle> {
    const raffle = this.raffleRepository.create({
      ...data,
      owner: { id: userId }
    })
    await this.raffleRepository.save(raffle)
    await this.ticketService.generateTickets(raffle.id, data.totalNumbers)
    return raffle
  }

  async purchaseTickets(
    raffleId: string,
    userId: string,
    numbers: number[]
  ): Promise<Ticket[]> {
    const tickets = await this.ticketService.reserveTickets(raffleId, numbers)
    const payment = await this.paymentService.createPayment({
      userId,
      raffleId,
      tickets,
      amount: tickets.length * tickets[0].price
    })
    return tickets
  }
}
```

## Payment Integration
```typescript
// src/services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(
    private pixService: PixService,
    private creditCardService: CreditCardService
  ) {}

  async createPayment(data: CreatePaymentDto): Promise<Payment> {
    switch (data.method) {
      case 'pix':
        return this.pixService.createPayment(data)
      case 'credit_card':
        return this.creditCardService.createPayment(data)
      default:
        throw new Error('Invalid payment method')
    }
  }
}

// src/services/pix.service.ts
@Injectable()
export class PixService {
  async createPayment(data: CreatePaymentDto): Promise<Payment> {
    const pixData = await this.generatePixData(data)
    return this.paymentRepository.save({
      ...data,
      pixData,
      status: 'pending'
    })
  }

  private async generatePixData(data: CreatePaymentDto) {
    // Implement PIX generation logic
  }
}
```

## Security Implementation
```typescript
// src/guards/auth.guard.ts
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private usersService: UsersService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest()
    const token = this.extractToken(request)
    
    if (!token) {
      throw new UnauthorizedException()
    }

    try {
      const payload = await this.jwtService.verifyAsync(token)
      const user = await this.usersService.findOne(payload.sub)
      request.user = user
      return true
    } catch {
      throw new UnauthorizedException()
    }
  }
}

// src/interceptors/logging.interceptor.ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest()
    const { method, url, body } = request
    const now = Date.now()

    return next.handle().pipe(
      tap(() => {
        const response = context.switchToHttp().getResponse()
        const delay = Date.now() - now
        this.logger.log(
          `${method} ${url} ${response.statusCode} ${delay}ms`
        )
      })
    )
  }
}
```

## Testing Setup
```typescript
// src/tests/raffle.service.spec.ts
describe('RaffleService', () => {
  let service: RaffleService
  let repository: Repository<Raffle>

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        RaffleService,
        {
          provide: getRepositoryToken(Raffle),
          useClass: Repository
        }
      ]
    }).compile()

    service = module.get<RaffleService>(RaffleService)
    repository = module.get<Repository<Raffle>>(getRepositoryToken(Raffle))
  })

  it('should create a raffle', async () => {
    const raffleData = {
      title: 'Test Raffle',
      description: 'Test Description',
      totalNumbers: 100,
      pricePerNumber: 10
    }

    const result = await service.createRaffle(raffleData, 'user-id')
    expect(result).toBeDefined()
    expect(result.title).toBe(raffleData.title)
  })
})
``` 