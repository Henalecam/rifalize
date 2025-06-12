# Brazil-Specific Technical Implementation

## PIX Integration
```typescript
// src/services/pix/pix.service.ts
@Injectable()
export class PixService {
  constructor(
    @InjectRepository(Payment)
    private paymentRepository: Repository<Payment>,
    private configService: ConfigService
  ) {}

  async generatePixQRCode(payment: Payment): Promise<PixQRCode> {
    const pixData = {
      merchantName: this.configService.get('PIX_MERCHANT_NAME'),
      merchantCity: this.configService.get('PIX_MERCHANT_CITY'),
      postalCode: this.configService.get('PIX_POSTAL_CODE'),
      amount: payment.amount,
      transactionId: payment.id,
      key: this.configService.get('PIX_KEY')
    }

    return this.generateQRCode(pixData)
  }

  async validatePixPayment(paymentId: string): Promise<boolean> {
    // Implement PIX validation logic
    return true
  }
}

// src/entities/pix.entity.ts
@Entity('pix_payments')
export class PixPayment {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  qrCode: string

  @Column()
  qrCodeText: string

  @Column()
  status: PaymentStatus

  @Column()
  amount: number

  @Column()
  transactionId: string

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date
}
```

## Boleto Integration
```typescript
// src/services/boleto/boleto.service.ts
@Injectable()
export class BoletoService {
  constructor(
    @InjectRepository(BoletoPayment)
    private boletoRepository: Repository<BoletoPayment>,
    private configService: ConfigService
  ) {}

  async generateBoleto(payment: Payment): Promise<BoletoPayment> {
    const boletoData = {
      amount: payment.amount,
      dueDate: this.calculateDueDate(),
      customer: {
        name: payment.user.name,
        document: payment.user.document,
        email: payment.user.email
      },
      instructions: ['Não receber após o vencimento'],
      description: `Pagamento da rifa ${payment.raffle.title}`
    }

    const boleto = await this.boletoRepository.save({
      ...boletoData,
      paymentId: payment.id,
      status: 'pending'
    })

    return boleto
  }
}

// src/entities/boleto.entity.ts
@Entity('boleto_payments')
export class BoletoPayment {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  barcode: string

  @Column()
  digitableLine: string

  @Column()
  dueDate: Date

  @Column()
  status: PaymentStatus

  @Column()
  amount: number

  @Column()
  paymentId: string

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date
}
```

## Tax Compliance
```typescript
// src/services/tax/tax.service.ts
@Injectable()
export class TaxService {
  constructor(
    @InjectRepository(Invoice)
    private invoiceRepository: Repository<Invoice>
  ) {}

  async generateInvoice(payment: Payment): Promise<Invoice> {
    const invoiceData = {
      number: await this.generateInvoiceNumber(),
      date: new Date(),
      customer: {
        name: payment.user.name,
        document: payment.user.document,
        email: payment.user.email
      },
      items: [{
        description: `Rifa: ${payment.raffle.title}`,
        quantity: payment.tickets.length,
        unitPrice: payment.raffle.pricePerNumber,
        total: payment.amount
      }],
      taxes: this.calculateTaxes(payment.amount)
    }

    return this.invoiceRepository.save(invoiceData)
  }

  private calculateTaxes(amount: number): TaxCalculation {
    // Implement Brazilian tax calculation logic
    return {
      pis: amount * 0.0065,
      cofins: amount * 0.03,
      csll: amount * 0.01,
      irrf: amount * 0.015
    }
  }
}

// src/entities/invoice.entity.ts
@Entity('invoices')
export class Invoice {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  number: string

  @Column()
  date: Date

  @Column('jsonb')
  customer: CustomerData

  @Column('jsonb')
  items: InvoiceItem[]

  @Column('jsonb')
  taxes: TaxCalculation

  @Column()
  total: number

  @Column()
  status: InvoiceStatus

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date
}
```

## LGPD Compliance
```typescript
// src/services/privacy/privacy.service.ts
@Injectable()
export class PrivacyService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
    @InjectRepository(PrivacyConsent)
    private consentRepository: Repository<PrivacyConsent>
  ) {}

  async recordConsent(userId: string, consent: ConsentData): Promise<PrivacyConsent> {
    return this.consentRepository.save({
      userId,
      ...consent,
      timestamp: new Date()
    })
  }

  async exportUserData(userId: string): Promise<UserDataExport> {
    const user = await this.userRepository.findOne({
      where: { id: userId },
      relations: ['raffles', 'payments', 'tickets']
    })

    return {
      personalData: this.extractPersonalData(user),
      activityData: this.extractActivityData(user),
      consentHistory: await this.getConsentHistory(userId)
    }
  }

  async deleteUserData(userId: string): Promise<void> {
    await this.userRepository.softDelete(userId)
    await this.anonymizeUserData(userId)
  }
}

// src/entities/privacy.entity.ts
@Entity('privacy_consents')
export class PrivacyConsent {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  userId: string

  @Column('jsonb')
  consentData: ConsentData

  @Column()
  timestamp: Date

  @Column()
  ipAddress: string

  @Column()
  userAgent: string
}
```

## Brazilian Localization
```typescript
// src/config/localization.config.ts
export const localizationConfig = {
  defaultLocale: 'pt-BR',
  supportedLocales: ['pt-BR', 'en'],
  dateFormat: 'DD/MM/YYYY',
  timeFormat: 'HH:mm',
  currency: {
    code: 'BRL',
    symbol: 'R$',
    decimalSeparator: ',',
    thousandSeparator: '.'
  },
  phone: {
    format: '(##) #####-####',
    countryCode: '+55'
  },
  address: {
    format: {
      street: 'Rua/Av.',
      number: 'Nº',
      complement: 'Complemento',
      neighborhood: 'Bairro',
      city: 'Cidade',
      state: 'Estado',
      zipCode: 'CEP'
    }
  }
}

// src/services/localization/localization.service.ts
@Injectable()
export class LocalizationService {
  constructor(
    private configService: ConfigService
  ) {}

  formatCurrency(amount: number): string {
    const { symbol, decimalSeparator, thousandSeparator } = this.configService.get('currency')
    return `${symbol} ${amount.toFixed(2)
      .replace('.', decimalSeparator)
      .replace(/\B(?=(\d{3})+(?!\d))/g, thousandSeparator)}`
  }

  formatPhone(phone: string): string {
    const { format } = this.configService.get('phone')
    return phone.replace(/(\d{2})(\d{5})(\d{4})/, format)
  }

  formatAddress(address: Address): string {
    const { format } = this.configService.get('address')
    return `${format.street} ${address.street}, ${format.number} ${address.number}, ` +
           `${format.complement} ${address.complement}, ${format.neighborhood} ${address.neighborhood}, ` +
           `${address.city}/${address.state}, ${format.zipCode} ${this.formatZipCode(address.zipCode)}`
  }
}
``` 