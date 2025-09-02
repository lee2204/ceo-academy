# CEO 아카데미 홈페이지 설계 명세서

## 🎯 프로젝트 개요
CEO 아카데미 기업 홈페이지 + 커뮤니티 플랫폼

### 핵심 기능
- 회사 소개 (About Company)
- 과정 소개 (Course Introduction)  
- 기수별 활동내역 (Activity History by Generation)
- 회원가입 (User Registration)
- 게시판 (Board/Forum)

## 🏗️ 시스템 아키텍처

### 기술 스택
- **Frontend**: Next.js 13+ (App Router) + TypeScript
- **Styling**: Tailwind CSS + Headless UI
- **Database**: PostgreSQL + Prisma ORM
- **Authentication**: NextAuth.js
- **File Storage**: Cloudinary (이미지/동영상)
- **Deployment**: Vercel

### 핵심 아키텍처 결정
```
Frontend (Next.js) ⟷ API Routes ⟷ Prisma ⟷ PostgreSQL
                   ↓
              NextAuth.js ⟷ Session Management
                   ↓  
              Cloudinary ⟷ Media Storage
```

## 📁 디렉토리 구조

```
ceo-academy/
├── src/
│   ├── app/                    # Next.js 13 App Router
│   │   ├── (auth)/            # 인증 라우트 그룹
│   │   │   ├── login/page.tsx
│   │   │   └── register/page.tsx
│   │   ├── about/             # 회사 소개
│   │   │   └── page.tsx
│   │   ├── courses/           # 과정 소개
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── activities/        # 기수별 활동내역
│   │   │   ├── page.tsx
│   │   │   └── [generation]/page.tsx
│   │   ├── board/            # 게시판
│   │   │   ├── page.tsx
│   │   │   ├── [id]/page.tsx
│   │   │   └── create/page.tsx
│   │   ├── api/              # API routes
│   │   │   ├── auth/[...nextauth]/route.ts
│   │   │   ├── posts/route.ts
│   │   │   ├── users/route.ts
│   │   │   └── upload/route.ts
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx          # 홈페이지
│   ├── components/           # 재사용 컴포넌트
│   │   ├── ui/              # 기본 UI 컴포넌트
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── card.tsx
│   │   │   └── modal.tsx
│   │   ├── layout/          # 레이아웃 컴포넌트
│   │   │   ├── header.tsx
│   │   │   ├── footer.tsx
│   │   │   └── navigation.tsx
│   │   └── features/        # 기능별 컴포넌트
│   │       ├── about/
│   │       ├── courses/
│   │       ├── activities/
│   │       ├── board/
│   │       └── auth/
│   ├── lib/                 # 유틸리티와 설정
│   │   ├── prisma.ts       # Prisma client
│   │   ├── auth.ts         # NextAuth 설정
│   │   ├── utils.ts        # 공통 유틸리티
│   │   └── validations.ts  # Zod 스키마
│   └── types/              # TypeScript 타입 정의
│       ├── auth.ts
│       ├── post.ts
│       └── user.ts
├── public/                 # 정적 파일
│   ├── images/
│   ├── icons/
│   └── favicon.ico
├── prisma/                # 데이터베이스 스키마
│   ├── schema.prisma
│   └── migrations/
├── package.json
├── tailwind.config.js
├── next.config.js
└── tsconfig.json
```

## 🗄️ 데이터베이스 스키마

```prisma
// prisma/schema.prisma

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  password      String
  name          String
  generation    Int?      // 기수 번호
  role          Role      @default(USER)
  profileImage  String?   // 프로필 이미지 URL
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  // Relations
  posts         Post[]
  comments      Comment[]
  activities    Activity[]
  
  @@map("users")
}

model Course {
  id          String   @id @default(cuid())
  title       String
  description String   @db.Text
  duration    String   // "8주", "3개월" 등
  price       Int?     // 가격 (null = 무료)
  image       String?  // 과정 이미지 URL
  isActive    Boolean  @default(true)
  order       Int      @default(0) // 표시 순서
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("courses")
}

model Activity {
  id          String   @id @default(cuid())
  generation  Int      // 기수
  title       String
  description String   @db.Text
  date        DateTime // 활동 날짜
  images      Json     // 이미지 URL 배열
  createdBy   String   // User.id
  createdAt   DateTime @default(now())
  
  // Relations
  creator     User     @relation(fields: [createdBy], references: [id])
  
  @@map("activities")
}

model Post {
  id          String      @id @default(cuid())
  title       String
  content     String      @db.Text
  authorId    String
  category    PostCategory @default(FREE)
  isPinned    Boolean     @default(false)
  viewCount   Int         @default(0)
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  
  // Relations
  author      User        @relation(fields: [authorId], references: [id])
  comments    Comment[]
  
  @@map("posts")
}

model Comment {
  id        String   @id @default(cuid())
  postId    String
  authorId  String
  content   String   @db.Text
  createdAt DateTime @default(now())
  
  // Relations
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  author    User     @relation(fields: [authorId], references: [id])
  
  @@map("comments")
}

enum Role {
  ADMIN
  USER
}

enum PostCategory {
  NOTICE    // 공지사항
  QNA       // 질문답변
  FREE      // 자유게시판
}
```

## 🎨 컴포넌트 설계

### Layout Components

**Header.tsx**
```typescript
interface HeaderProps {
  user?: User | null;
}

// 기능: 로고, 네비게이션, 사용자 메뉴
// 반응형: 모바일에서 햄버거 메뉴
```

**Navigation.tsx**
```typescript
interface NavItem {
  label: string;
  href: string;
  children?: NavItem[];
}

const navItems: NavItem[] = [
  { label: "홈", href: "/" },
  { label: "회사소개", href: "/about" },
  { label: "과정소개", href: "/courses" },
  { label: "활동내역", href: "/activities" },
  { label: "게시판", href: "/board" }
];
```

### Feature Components

**CourseCard.tsx**
```typescript
interface CourseCardProps {
  course: Course;
  onClick?: () => void;
}

// 기능: 과정 정보 카드 표시
// 이미지, 제목, 설명, 기간, 가격
```

**ActivityGallery.tsx** 
```typescript
interface ActivityGalleryProps {
  activities: Activity[];
  generation?: number;
}

// 기능: 기수별 활동 갤러리
// 필터링, 페이지네이션, 이미지 모달
```

**PostList.tsx**
```typescript
interface PostListProps {
  category?: PostCategory;
  searchQuery?: string;
  page?: number;
}

// 기능: 게시글 목록 표시
// 카테고리 필터, 검색, 페이지네이션
```

## 🚦 라우팅 설계

```typescript
// app/layout.tsx - 글로벌 레이아웃
// app/page.tsx - 홈페이지 (회사 소개 요약)
// app/about/page.tsx - 회사 소개 상세
// app/courses/page.tsx - 과정 목록
// app/courses/[id]/page.tsx - 과정 상세
// app/activities/page.tsx - 전체 활동내역
// app/activities/[generation]/page.tsx - 특정 기수 활동
// app/board/page.tsx - 게시판 목록
// app/board/[id]/page.tsx - 게시글 상세
// app/board/create/page.tsx - 게시글 작성
// app/(auth)/login/page.tsx - 로그인
// app/(auth)/register/page.tsx - 회원가입
```

## 🔒 인증 및 권한

### 역할 기반 접근 제어
- **ADMIN**: 모든 기능 + 콘텐츠 관리
- **USER**: 조회 + 게시판 참여 + 프로필 수정

### 보호된 라우트
- `/board/create` - 로그인 필요
- `/admin/*` - ADMIN 권한 필요
- `/profile` - 본인 계정만 접근

## 📱 반응형 디자인

### 브레이크포인트
- Mobile: 0-768px
- Tablet: 768-1024px  
- Desktop: 1024px+

### 주요 반응형 요소
- 네비게이션: 모바일에서 햄버거 메뉴
- 활동갤러리: 그리드 레이아웃 (1→2→3 columns)
- 게시판: 테이블 → 카드 레이아웃

## 🎯 구현 우선순위

**Phase 1 (MVP)**: 정적 페이지 + 기본 레이아웃
**Phase 2 (Core)**: 인증 + 사용자 관리  
**Phase 3 (Features)**: 동적 콘텐츠 + 게시판
**Phase 4 (Polish)**: 최적화 + 고급 기능

## 📊 성능 고려사항

- **SSG**: 회사소개, 과정소개 (정적 생성)
- **ISR**: 활동내역 (1시간마다 재생성)
- **CSR**: 게시판 (실시간 업데이트)
- **Image Optimization**: next/image + Cloudinary
- **Code Splitting**: 페이지별 자동 분할