# 데이터베이스 설계

## 🗄️ ERD (Entity Relationship Diagram)

```
User ||--o{ Post : creates
User ||--o{ Comment : writes  
User ||--o{ Activity : manages
Post ||--o{ Comment : has
```

## 📊 테이블 상세 설계

### Users Table
```sql
CREATE TABLE users (
  id VARCHAR(30) PRIMARY KEY,      -- cuid
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,  -- bcrypt hash
  name VARCHAR(100) NOT NULL,
  generation INTEGER,              -- 기수 (nullable)
  role user_role DEFAULT 'USER',   -- ADMIN, USER
  profile_image TEXT,              -- Cloudinary URL
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TYPE user_role AS ENUM ('ADMIN', 'USER');
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_generation ON users(generation);
```

### Courses Table
```sql
CREATE TABLE courses (
  id VARCHAR(30) PRIMARY KEY,
  title VARCHAR(200) NOT NULL,
  description TEXT NOT NULL,
  duration VARCHAR(50) NOT NULL,   -- "8주", "3개월"
  price INTEGER,                   -- NULL = 무료
  image TEXT,                      -- Cloudinary URL
  is_active BOOLEAN DEFAULT true,
  display_order INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_courses_active ON courses(is_active, display_order);
```

### Activities Table
```sql
CREATE TABLE activities (
  id VARCHAR(30) PRIMARY KEY,
  generation INTEGER NOT NULL,
  title VARCHAR(200) NOT NULL,
  description TEXT NOT NULL,
  activity_date DATE NOT NULL,    -- 활동 날짜
  images JSONB,                   -- 이미지 URL 배열
  created_by VARCHAR(30) NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_activities_generation ON activities(generation, activity_date DESC);
CREATE INDEX idx_activities_date ON activities(activity_date DESC);
```

### Posts Table
```sql
CREATE TABLE posts (
  id VARCHAR(30) PRIMARY KEY,
  title VARCHAR(300) NOT NULL,
  content TEXT NOT NULL,
  author_id VARCHAR(30) NOT NULL REFERENCES users(id),
  category post_category DEFAULT 'FREE',
  is_pinned BOOLEAN DEFAULT false,
  view_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TYPE post_category AS ENUM ('NOTICE', 'QNA', 'FREE');
CREATE INDEX idx_posts_category ON posts(category, is_pinned DESC, created_at DESC);
CREATE INDEX idx_posts_author ON posts(author_id);
```

### Comments Table
```sql
CREATE TABLE comments (
  id VARCHAR(30) PRIMARY KEY,
  post_id VARCHAR(30) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  author_id VARCHAR(30) NOT NULL REFERENCES users(id),
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_comments_post ON comments(post_id, created_at ASC);
CREATE INDEX idx_comments_author ON comments(author_id);
```

## 🔍 쿼리 패턴 최적화

### 자주 사용되는 쿼리
```sql
-- 게시판 목록 (페이지네이션)
SELECT p.*, u.name as author_name 
FROM posts p 
JOIN users u ON p.author_id = u.id 
WHERE p.category = $1 
ORDER BY p.is_pinned DESC, p.created_at DESC 
LIMIT $2 OFFSET $3;

-- 기수별 활동내역
SELECT * FROM activities 
WHERE generation = $1 
ORDER BY activity_date DESC;

-- 사용자 프로필 with 게시글 수
SELECT u.*, COUNT(p.id) as post_count
FROM users u 
LEFT JOIN posts p ON u.id = p.author_id 
WHERE u.id = $1 
GROUP BY u.id;
```

### 인덱스 전략
- **복합 인덱스**: 자주 함께 사용되는 컬럼들
- **정렬 최적화**: created_at DESC for 최신순 정렬
- **외래키 인덱스**: JOIN 성능 최적화

## 📈 확장성 고려사항

### 데이터 증가 대비
- **게시판**: 카테고리별 테이블 분리 (필요시)
- **활동내역**: 연도별 파티셔닝 (필요시)
- **이미지**: CDN 활용으로 DB 부담 최소화

### 성능 최적화
- **Connection Pooling**: Prisma connection pool
- **캐싱**: Redis (필요시)
- **읽기 복제본**: 조회 성능 향상 (필요시)

## 🔐 보안 고려사항

### 데이터 보호
- 비밀번호: bcrypt 해싱
- 개인정보: 최소 수집 원칙
- 세션: NextAuth.js JWT

### 입력 검증
- Prisma ORM: SQL Injection 방지
- Zod: 타입 및 형식 검증
- Rate Limiting: API 남용 방지

## 💾 백업 및 복구
- **정기 백업**: 일간 자동 백업
- **Point-in-time 복구**: PostgreSQL WAL
- **데이터 마이그레이션**: Prisma migrate