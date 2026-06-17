# 🏗️ Архитектура Honestly

Полное описание архитектуры приложения Honestly - как компоненты взаимодействуют и почему такой дизайн.

## 📊 Общая архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
├──────────────────────┬──────────────────────────────────────┤
│  Web (React)         │  Mobile (React Native)               │
│  - Single Page App   │  - iOS & Android                     │
│  - TypeScript        │  - Native Performance                │
└──────────────────────┴──────────────────────────────────────┘
                            ↓
        ┌───────────────────────────────────────┐
        │      API Gateway / Load Balancer      │
        │  (nginx / AWS ELB)                    │
        └───────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
├──────────────────────────────────────────────────────────────┤
│  Backend Services (Node.js + Express)                       │
│  ├── Authentication Service                                 │
│  ├── User Service                                           │
│  ├── Matching Service                                       │
│  ├── Chat Service                                           │
│  └── Search Service                                         │
└──────────────────────────────────────────────────────────────┘
         ↓                           ↓
    ┌─────────────┐          ┌─────────────────┐
    │  Database   │          │  Cache Layer    │
    │ PostgreSQL  │          │  Redis          │
    └─────────────┘          └─────────────────┘
         ↓
    ┌─────────────┐
    │   File      │
    │  Storage    │
    │   (S3)      │
    └─────────────┘
```

## 🎯 Компоненты системы

### 1. Frontend (Web)
**Технология**: React 18 + TypeScript + Redux + TailwindCSS

**Структура**:
```
src/
├── components/      # Переиспользуемые компоненты
├── pages/          # Страницы приложения
├── store/          # Redux store и slices
├── services/       # API клиенты
├── hooks/          # Custom React hooks
├── types/          # TypeScript типы
└── utils/          # Утилиты и хелперы
```

**Ключевые особенности**:
- State management через Redux
- Real-time updates через WebSocket
- Lazy loading для оптимизации
- Responsive design
- Progressive Web App (PWA) поддержка

### 2. Frontend (Mobile)
**Технология**: React Native + Expo + TypeScript

**Структура похожа на Web, но с нативными компонентами**

**Преимущества**:
- Один код для iOS и Android
- Нативная производительность
- Доступ к device features (камера, GPS и т.д.)

### 3. Backend API
**Технология**: Node.js + Express + TypeScript

**Основные сервисы**:

#### Authentication Service
```typescript
- POST /auth/register        - Регистрация
- POST /auth/login           - Логин
- POST /auth/refresh         - Refresh токен
- POST /auth/logout          - Логаут
- POST /auth/verify-email    - Верификация email
- POST /auth/reset-password  - Сброс пароля
```

#### User Service
```typescript
- GET /users/:id             - Получить профиль
- PUT /users/:id             - Обновить профиль
- POST /users/:id/photos     - Загрузить фото
- DELETE /users/:id/photos/:photoId - Удалить фото
- PUT /users/:id/preferences - Обновить предпочтения
```

#### Matching Service
```typescript
- GET /matches               - Список матчей
- GET /matches/cards         - Карточки для свайпа
- POST /likes                - Лайк профиля
- POST /passes               - Пасс профиля
- GET /matches/:id           - Детали матча
```

#### Chat Service
```typescript
- GET /chats                 - Список чатов
- GET /chats/:id             - История сообщений
- POST /messages             - Отправить сообщение (REST)
- WebSocket для real-time    - Вебсокет для real-time
- GET /messages/:id          - Получить сообщение
```

#### Search Service
```typescript
- GET /search                - Поиск пользователей
- GET /search/filters        - Доступные фильтры
- POST /search/saved         - Сохраненный поиск
```

### 4. База данных (PostgreSQL)

**Основные таблицы**:

```sql
-- Пользователи
users
├── id (PK)
├── email (UNIQUE)
├── phone (UNIQUE)
├── password_hash
├── created_at
├── updated_at
└── is_verified

-- Профили
profiles
├── id (PK)
├── user_id (FK)
├── first_name
├── age
├── bio
├── location (GIS)
├── interests[]
└── preferences (JSONB)

-- Фото
photos
├── id (PK)
├── user_id (FK)
├── url
├── order
├── is_verified
└── created_at

-- Лайки
likes
├── id (PK)
├── from_user_id (FK)
├── to_user_id (FK)
├── created_at
└── UNIQUE(from_user_id, to_user_id)

-- Матчи (взаимные лайки)
matches
├── id (PK)
├── user1_id (FK)
├── user2_id (FK)
├── created_at
└── is_active

-- Сообщения
messages
├── id (PK)
├── match_id (FK)
├── sender_id (FK)
├── content
├── created_at
├── is_read
└── read_at

-- Блокировки
blocks
├── id (PK)
├── blocker_id (FK)
├── blocked_id (FK)
└── created_at
```

**Индексы**:
```sql
-- Performance индексы
CREATE INDEX idx_profiles_user_id ON profiles(user_id);
CREATE INDEX idx_likes_from_user ON likes(from_user_id);
CREATE INDEX idx_likes_to_user ON likes(to_user_id);
CREATE INDEX idx_messages_match_id ON messages(match_id);
CREATE INDEX idx_messages_created_at ON messages(created_at);
CREATE INDEX idx_photos_user_id ON photos(user_id);
CREATE INDEX idx_blocks_blocker ON blocks(blocker_id);
```

### 5. Redis Cache

**Использование**:

```typescript
// Session store
redis:session:{sessionId}

// User cache
redis:user:{userId}

// Match suggestions (pre-computed)
redis:suggestions:{userId}

// Chat cache
redis:chat:{matchId}

// Rate limiting
redis:ratelimit:{userId}:{endpoint}
```

**TTL Strategy**:
- Sessions: 24 часа
- User cache: 1 час
- Match suggestions: 30 минут
- Chat messages: 5 минут (для быстрого доступа)

### 6. File Storage (S3/MinIO)

**Структура**:
```
/users/{userId}/
├── profile/
│   └── avatar.jpg
└── photos/
    ├── photo_0.jpg
    ├── photo_1.jpg
    └── photo_2.jpg
```

**Image Processing**:
- Оригинал (макс. 10MB)
- Thumbnail (200x200)
- Medium (500x500)
- Large (1000x1000)

## 🔐 Безопасность

### Authentication Flow
```
User Login
    ↓
Validate Credentials
    ↓
Generate JWT Token
├── Access Token (15 минут TTL)
└── Refresh Token (7 дней TTL)
    ↓
Send to Client
    ↓
Client stores in httpOnly Cookie
```

### Password Security
- Bcrypt with 12 salt rounds
- Никогда не логируй пароли
- Min 8 characters, uppercase, lowercase, number, special char

### Data Encryption
```typescript
// Sensitive fields in DB
- email (encrypted)
- phone (encrypted)
- location (geohashed)
- preferences (encrypted JSONB)
```

## 🚀 Масштабируемость

### Горизонтальное масштабирование

```
Load Balancer
├── Server 1 (Express instance)
├── Server 2 (Express instance)
└── Server 3 (Express instance)
    ↓
Shared Redis (Session store)
    ↓
Primary DB + Replicas
```

### Кэширование стратегия

**Cache-Aside Pattern**:
```typescript
1. Check cache
   ↓ Hit → Return
   ↓ Miss
2. Query database
3. Update cache
4. Return to client
```

### Queue система (для тяжелых операций)

```typescript
// Using Bull queue
- Email sending
- Image processing
- Matching calculations
- Notifications
```

## 📊 Data Flow

### User Registration Flow
```
Client: POST /auth/register
    ↓
Backend: Validate input
    ↓
Backend: Hash password
    ↓
Backend: Create user in DB
    ↓
Backend: Send verification email (async queue)
    ↓
Backend: Generate JWT tokens
    ↓
Client: Store tokens, redirect to profile
```

### Matching Flow
```
User A opens app
    ↓
GET /matches/cards
    ↓
Backend: Get suggestions from Redis
    ↓ Cache miss
    ↓
Backend: Calculate matches (algorithm)
    ↓
Backend: Store in Redis
    ↓
Return cards to frontend
    ↓
User swipes: POST /likes or /passes
    ↓
Backend: Update DB
    ↓
Check for mutual like (match)
    ↓ if matched
    ↓
Send notification (WebSocket)
```

### Chat Flow
```
User opens chat
    ↓
WebSocket: ws://api/chat/:matchId
    ↓
Backend: Subscribe to room
    ↓
Load history: GET /messages?matchId={id}
    ↓
User sends message: emit('message', {...})
    ↓
Backend: Save to DB
    ↓
Broadcast to match partner
```

## 🧪 Testing Strategy

### Unit Tests
- Services (auth, user, matching, etc.)
- Utils и helpers
- Coverage: >90%

### Integration Tests
- API endpoints
- Database interactions
- Cache operations

### E2E Tests
- User registration flow
- Matching flow
- Chat flow

### Performance Tests
- Load testing (k6 / Artillery)
- Database query optimization
- Cache hit rates

## 📈 Monitoring & Logging

### Logs
```typescript
- Application logs → Elasticsearch / Datadog
- Access logs (nginx) → Log aggregator
- Error tracking → Sentry
- Performance → New Relic
```

### Metrics
```
- API response times
- Database query times
- Cache hit rate
- User registration rate
- Match success rate
```

## 🔄 CI/CD Pipeline

```
Developer Push
    ↓
GitHub Actions
    ├── Lint & Format
    ├── Unit Tests
    ├── Integration Tests
    ├── Build Docker image
    └── Upload to registry
        ↓
    Deploy to Staging
        ↓
    Smoke Tests
        ↓ if passed
        ↓
    Manual approval
        ↓
    Deploy to Production
        ↓
    Health checks
        ↓
    Monitor for errors
```

## 📱 API Versioning

```
/api/v1/users
/api/v1/matches
/api/v1/chat

New breaking changes:
/api/v2/users
```

## 🎯 Performance Targets

- API response: <200ms (p95)
- Database query: <50ms (p95)
- Chat message delivery: <500ms
- Image upload: <5s
- Search results: <1s
- App startup: <2s

---

**Created with 💪 for scalability and reliability**
