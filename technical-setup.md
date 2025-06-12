# Initial Technical Setup

## Development Environment
### Required Tools
- [ ] Node.js (v18+)
- [ ] Git
- [ ] VS Code (recommended)
- [ ] Docker
- [ ] PostgreSQL
- [ ] Redis (for caching)

### VS Code Extensions
- [ ] ESLint
- [ ] Prettier
- [ ] GitLens
- [ ] Docker
- [ ] PostgreSQL
- [ ] Tailwind CSS IntelliSense

## Project Structure
### Frontend (Next.js)
```
frontend/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   ├── (dashboard)/
│   │   └── (marketing)/
│   ├── components/
│   │   ├── common/
│   │   ├── forms/
│   │   └── layouts/
│   ├── hooks/
│   ├── lib/
│   ├── styles/
│   └── types/
├── public/
└── package.json
```

### Backend (Nest.js)
```
backend/
├── src/
│   ├── modules/
│   │   ├── auth/
│   │   ├── users/
│   │   ├── raffles/
│   │   └── payments/
│   ├── common/
│   │   ├── decorators/
│   │   ├── filters/
│   │   └── guards/
│   └── config/
├── test/
└── package.json
```

## Initial Dependencies
### Frontend
```json
{
  "dependencies": {
    "next": "latest",
    "react": "latest",
    "react-dom": "latest",
    "tailwindcss": "latest",
    "@headlessui/react": "latest",
    "@heroicons/react": "latest",
    "axios": "latest",
    "react-query": "latest",
    "zustand": "latest"
  }
}
```

### Backend
```json
{
  "dependencies": {
    "@nestjs/common": "latest",
    "@nestjs/core": "latest",
    "@nestjs/platform-express": "latest",
    "@nestjs/typeorm": "latest",
    "typeorm": "latest",
    "pg": "latest",
    "class-validator": "latest",
    "class-transformer": "latest",
    "passport": "latest",
    "passport-jwt": "latest"
  }
}
```

## Database Schema (Initial)
```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Subscriptions
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  plan_type VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL,
  start_date TIMESTAMP NOT NULL,
  end_date TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Raffles
CREATE TABLE raffles (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  title VARCHAR(255) NOT NULL,
  description TEXT,
  total_numbers INTEGER NOT NULL,
  price_per_number DECIMAL(10,2) NOT NULL,
  status VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

## Initial Configuration
### Environment Variables
```env
# Frontend
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Backend
PORT=3001
DATABASE_URL=postgresql://user:password@localhost:5432/rifalize
JWT_SECRET=your-secret-key
```

## Development Workflow
1. Clone repository
2. Install dependencies
3. Set up environment variables
4. Start development servers
5. Run database migrations
6. Start development

## First Steps
1. Set up basic authentication
2. Create user management
3. Implement basic raffle creation
4. Set up payment integration
5. Create basic admin panel 