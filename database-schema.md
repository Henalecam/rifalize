# Esquema do Banco de Dados

## Tabelas Principais

### Users
- [ ] id (PK)
- [ ] name
- [ ] email
- [ ] password
- [ ] role (admin/client)
- [ ] created_at
- [ ] updated_at

### Subscriptions
- [ ] id (PK)
- [ ] user_id (FK)
- [ ] plan_id (FK)
- [ ] status
- [ ] start_date
- [ ] end_date
- [ ] created_at
- [ ] updated_at

### Plans
- [ ] id (PK)
- [ ] name
- [ ] price
- [ ] features
- [ ] duration
- [ ] created_at
- [ ] updated_at

### Templates
- [ ] id (PK)
- [ ] name
- [ ] description
- [ ] preview_image
- [ ] html_content
- [ ] css_content
- [ ] created_at
- [ ] updated_at

### Raffles
- [ ] id (PK)
- [ ] user_id (FK)
- [ ] template_id (FK)
- [ ] title
- [ ] description
- [ ] total_numbers
- [ ] price_per_number
- [ ] status
- [ ] subdomain
- [ ] created_at
- [ ] updated_at

### Sales
- [ ] id (PK)
- [ ] raffle_id (FK)
- [ ] user_id (FK)
- [ ] numbers
- [ ] total_price
- [ ] status
- [ ] created_at
- [ ] updated_at

### SalesFunnel
- [ ] id (PK)
- [ ] raffle_id (FK)
- [ ] name
- [ ] description
- [ ] trigger_type
- [ ] trigger_value
- [ ] offer_type
- [ ] offer_value
- [ ] created_at
- [ ] updated_at 