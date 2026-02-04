# Upgrade Lab - REST API Design

## 1. Authentication & Authorization

### 1.1 Roles & Permissions Matrix

| Resource | Public | Student/Lead | Editor | Super Admin |
|----------|--------|--------------|--------|-------------|
| **Projects** |
| List/View | ✅ | ✅ | ✅ | ✅ |
| Create | ❌ | ❌ | ✅ | ✅ |
| Update | ❌ | ❌ | ✅ | ✅ |
| Delete | ❌ | ❌ | ❌ | ✅ |
| **Posts** |
| List/View (published) | ✅ | ✅ | ✅ | ✅ |
| List/View (drafts) | ❌ | ❌ | ✅ | ✅ |
| Create/Update | ❌ | ❌ | ✅ | ✅ |
| Delete | ❌ | ❌ | ❌ | ✅ |
| **Leads** |
| Create (self-register) | ✅ | ❌ | ❌ | ❌ |
| List/View | ❌ | Own only | ✅ | ✅ |
| Update | ❌ | Own only | ✅ | ✅ |
| Delete | ❌ | ❌ | ❌ | ✅ |
| **Sessions** |
| List/View | ✅ | ✅ | ✅ | ✅ |
| Register | ✅ | ✅ | ✅ | ✅ |
| Create/Update | ❌ | ❌ | ✅ | ✅ |
| Mark Attendance | ❌ | ❌ | ✅ | ✅ |
| **Code Reviews** |
| Request | ❌ | ✅ | ✅ | ✅ |
| List | ❌ | Own only | ✅ | ✅ |
| Complete Review | ❌ | ❌ | ✅ | ✅ |
| **Admin Users** |
| List/View | ❌ | ❌ | View only | ✅ |
| Create/Update/Delete | ❌ | ❌ | ❌ | ✅ |
| **Testimonials** |
| Submit | ❌ | ✅ | ✅ | ✅ |
| Approve/Reject | ❌ | ❌ | ✅ | ✅ |

### 1.2 JWT Strategy

#### Access Token
```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "editor",
  "type": "access",
  "iat": 1738684800,
  "exp": 1738686600
}
```
- **Expiry**: 30 minutes
- **Storage**: Memory (not localStorage for security)
- **Usage**: Every API request in `Authorization: Bearer <token>`

#### Refresh Token
```json
{
  "sub": "user_id",
  "type": "refresh",
  "iat": 1738684800,
  "exp": 1746460800
}
```
- **Expiry**: 90 days
- **Storage**: HttpOnly cookie (secure, sameSite)
- **Usage**: Only on `/auth/refresh` endpoint
- **Rotation**: Issue new refresh token on each use

#### Token Storage in Database
```sql
CREATE TABLE refresh_token (
    id SERIAL PRIMARY KEY,
    admin_user_id INTEGER NOT NULL REFERENCES admin_user(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    revoked BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 2. API Endpoints

### Base URL
- **Production**: `https://api.upgradelab.dev/v1`
- **Development**: `http://localhost:3000/api/v1`

---

### 2.1 Authentication Endpoints

#### POST /auth/login
**Description**: Admin login  
**Access**: Public  
**Request**:
```json
{
  "email": "admin@upgradelab.dev",
  "password": "SecurePassword123!"
}
```
**Response** (200):
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": 1,
    "email": "admin@upgradelab.dev",
    "fullName": "Mohamed Amgad",
    "role": "super_admin"
  }
}
```
**Errors**:
- `401`: Invalid credentials
- `403`: Account deactivated
- `429`: Too many login attempts

---

#### POST /auth/refresh
**Description**: Refresh access token  
**Access**: Public (requires refresh token in cookie)  
**Response** (200):
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs..."
}
```
**Errors**:
- `401`: Invalid/expired refresh token
- `403`: Token revoked

---

#### POST /auth/logout
**Description**: Revoke refresh token  
**Access**: Authenticated  
**Response** (204): No content

---

### 2.2 Project Endpoints

#### GET /projects
**Description**: List all published projects  
**Access**: Public  
**Query Params**:
- `page` (default: 1)
- `limit` (default: 12, max: 50)
- `featured` (boolean)
- `type` (mvp|dashboard|landing|ui_upgrade)
- `tech` (slug, can be repeated: `?tech=react&tech=nextjs`)
- `tag` (slug, can be repeated)
- `sort` (recent|popular|alpha, default: recent)
- `status` (draft|published|archived) - Admin only

**Response** (200):
```json
{
  "data": [
    {
      "id": 1,
      "slug": "saas-dashboard-rebuild",
      "title": "SaaS Analytics Dashboard",
      "subtitle": "Real-time metrics dashboard for 10K+ users",
      "projectType": "dashboard",
      "clientType": "business",
      "featured": true,
      "thumbnail": "https://cdn.upgradelab.dev/projects/1/thumb.jpg",
      "technologies": [
        {
          "name": "Next.js",
          "slug": "nextjs",
          "category": "frontend",
          "colorHex": "#000000"
        }
      ],
      "tags": ["React", "Dashboard", "SaaS"],
      "completedAt": "2026-01-20",
      "viewCount": 247
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 12,
    "total": 23,
    "totalPages": 2
  }
}
```

---

#### GET /projects/:slug
**Description**: Get single project details  
**Access**: Public (published only), Admin (all statuses)  
**Response** (200):
```json
{
  "id": 1,
  "slug": "saas-dashboard-rebuild",
  "title": "SaaS Analytics Dashboard",
  "subtitle": "Real-time metrics dashboard for 10K+ users",
  "problem": "Client had a slow, cluttered admin panel...",
  "solution": "Rebuilt the entire dashboard using Next.js 14...",
  "role": "Full-stack developer - UI/UX design...",
  "results": "Page load time reduced from 4.2s to 0.8s...",
  "lessonsLearned": "Always profile before optimizing...",
  "projectType": "dashboard",
  "clientType": "business",
  "liveUrl": "https://demo.example.com",
  "repoUrl": null,
  "featured": true,
  "media": [
    {
      "id": 1,
      "mediaType": "image",
      "fileUrl": "https://cdn.upgradelab.dev/projects/1/hero.jpg",
      "caption": "Dashboard homepage",
      "altText": "Analytics dashboard showing charts",
      "isThumbnail": true
    }
  ],
  "technologies": [...],
  "tags": [...],
  "startedAt": "2025-11-15",
  "completedAt": "2026-01-20",
  "viewCount": 247,
  "createdAt": "2026-01-21T10:30:00Z"
}
```
**Errors**:
- `404`: Project not found
- `403`: Draft project, admin access required

---

#### POST /projects
**Description**: Create new project  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "title": "E-commerce Store Redesign",
  "subtitle": "Modern checkout experience",
  "slug": "ecommerce-store-redesign",
  "problem": "High cart abandonment rate...",
  "solution": "Redesigned checkout flow...",
  "role": "UI/UX Designer & Frontend Developer",
  "results": "Conversion increased by 23%",
  "lessonsLearned": "User testing revealed...",
  "projectType": "ui_upgrade",
  "clientType": "business",
  "liveUrl": "https://example-store.com",
  "repoUrl": null,
  "featured": false,
  "status": "draft",
  "technologies": [1, 5, 8],
  "tags": [2, 7],
  "startedAt": "2026-01-10",
  "completedAt": "2026-02-03"
}
```
**Validation**:
- `title`: Required, 3-255 chars
- `slug`: Required, unique, lowercase, alphanumeric + hyphens
- `problem`, `solution`, `role`: Required, min 20 chars
- `projectType`: Required, enum
- `technologies`: Array of tech_stack IDs
- `tags`: Array of tag IDs

**Response** (201):
```json
{
  "id": 24,
  "slug": "ecommerce-store-redesign",
  "message": "Project created successfully"
}
```

---

#### PATCH /projects/:id
**Description**: Update project  
**Access**: Editor, Super Admin  
**Request**: Partial update of any fields from POST  
**Response** (200):
```json
{
  "id": 24,
  "message": "Project updated successfully"
}
```

---

#### DELETE /projects/:id
**Description**: Delete project (hard delete)  
**Access**: Super Admin only  
**Response** (204): No content  
**Note**: Cascades to project_media, project_tech, project_tag

---

### 2.3 Post Endpoints

#### GET /posts
**Description**: List blog posts  
**Access**: Public (published only), Admin (all)  
**Query Params**:
- `page`, `limit`
- `type` (article|session_summary|tutorial|announcement)
- `tag` (slug)
- `featured` (boolean)
- `status` (draft|published|archived) - Admin only
- `sort` (recent|popular|alpha, default: recent)

**Response** (200):
```json
{
  "data": [
    {
      "id": 10,
      "slug": "start-programming-right",
      "title": "ابدأ برمجة صح: الخطة الأولى",
      "excerpt": "معظم المبتدئين بيبدأوا غلط. هنا الخطة اللي محتاجها...",
      "featuredImage": "https://cdn.upgradelab.dev/posts/10.jpg",
      "postType": "article",
      "readTimeMinutes": 8,
      "tags": ["Beginner", "Roadmap"],
      "viewCount": 523,
      "publishedAt": "2026-02-01T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

---

#### GET /posts/:slug
**Description**: Get single post  
**Access**: Public (published), Admin (all)  
**Response** (200):
```json
{
  "id": 10,
  "slug": "start-programming-right",
  "title": "ابدأ برمجة صح: الخطة الأولى",
  "excerpt": "...",
  "content": "# المقدمة\n\nمعظم المبتدئين...",
  "featuredImage": "...",
  "postType": "article",
  "featured": false,
  "readTimeMinutes": 8,
  "tags": [...],
  "viewCount": 524,
  "publishedAt": "2026-02-01T10:00:00Z",
  "createdAt": "2026-01-28T14:30:00Z",
  "updatedAt": "2026-02-01T10:00:00Z"
}
```

---

#### POST /posts
**Description**: Create post  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "title": "React vs Next.js: إمتى تستخدم إيه؟",
  "slug": "react-vs-nextjs-when-to-use",
  "excerpt": "الفرق الحقيقي ومتى تختار كل واحد",
  "content": "# المقدمة\n\n...",
  "featuredImage": "https://cdn.upgradelab.dev/posts/new.jpg",
  "postType": "article",
  "status": "draft",
  "tags": [1, 3, 7]
}
```
**Validation**:
- `title`: Required, 3-255 chars
- `slug`: Required, unique
- `content`: Required, min 100 chars
- `postType`: Required, enum
- Auto-calculate `readTimeMinutes` from content length

**Response** (201):
```json
{
  "id": 45,
  "slug": "react-vs-nextjs-when-to-use",
  "message": "Post created successfully"
}
```

---

### 2.4 Lead Endpoints

#### POST /leads
**Description**: Public lead submission form  
**Access**: Public  
**Rate Limit**: 3 requests per IP per hour  
**Request**:
```json
{
  "fullName": "Ahmed Hassan",
  "email": "ahmed@gmail.com",
  "phone": "+201012345678",
  "whatsapp": "+201012345678",
  "leadType": "student",
  "source": "website_form",
  "message": "عايز أعرف تفاصيل الـ Mentorship",
  "interestedIn": "mentorship",
  "budgetRange": "500-1000 EGP/month",
  "urgency": "this_month"
}
```
**Validation**:
- `fullName`: Required, 2-255 chars
- `email` OR `phone` OR `whatsapp`: At least one required
- `email`: Valid email format
- `leadType`: Required, enum (student|client|partner|other)
- `source`: Required
- `message`: Optional, max 2000 chars

**Response** (201):
```json
{
  "id": 150,
  "message": "شكرًا! هنتواصل معاك قريب",
  "estimatedResponseTime": "24 hours"
}
```
**Errors**:
- `429`: Rate limit exceeded - "يُرجى الانتظار قبل إرسال طلب آخر"

---

#### GET /leads
**Description**: List all leads (CRM)  
**Access**: Editor, Super Admin  
**Query Params**:
- `page`, `limit`
- `type` (student|client|partner)
- `status` (new|contacted|qualified|proposal_sent|closed_won|closed_lost|nurturing)
- `source` (website_form|whatsapp|instagram|facebook|referral|session)
- `assignedTo` (admin user ID)
- `search` (search in name, email, phone)
- `sort` (recent|oldest|last_contact, default: recent)

**Response** (200):
```json
{
  "data": [
    {
      "id": 150,
      "fullName": "Ahmed Hassan",
      "email": "ahmed@gmail.com",
      "phone": "+201012345678",
      "whatsapp": "+201012345678",
      "leadType": "student",
      "source": "website_form",
      "interestedIn": "mentorship",
      "status": "new",
      "budgetRange": "500-1000 EGP/month",
      "urgency": "this_month",
      "assignedTo": null,
      "lastContactAt": null,
      "createdAt": "2026-02-04T13:22:00Z"
    }
  ],
  "pagination": {...},
  "statistics": {
    "new": 23,
    "contacted": 45,
    "qualified": 12,
    "closed_won": 8
  }
}
```

---

#### GET /leads/:id
**Description**: Get lead details with notes  
**Access**: Editor, Super Admin  
**Response** (200):
```json
{
  "id": 150,
  "fullName": "Ahmed Hassan",
  "email": "ahmed@gmail.com",
  "phone": "+201012345678",
  "whatsapp": "+201012345678",
  "leadType": "student",
  "source": "website_form",
  "message": "عايز أعرف تفاصيل الـ Mentorship",
  "interestedIn": "mentorship",
  "status": "contacted",
  "budgetRange": "500-1000 EGP/month",
  "urgency": "this_month",
  "assignedTo": {
    "id": 1,
    "fullName": "Mohamed Amgad"
  },
  "lastContactAt": "2026-02-04T14:30:00Z",
  "createdAt": "2026-02-04T13:22:00Z",
  "notes": [
    {
      "id": 89,
      "note": "تم التواصل عبر WhatsApp. مهتم بخطة شهرية.",
      "noteType": "whatsapp",
      "createdBy": {
        "id": 1,
        "fullName": "Mohamed Amgad"
      },
      "createdAt": "2026-02-04T14:30:00Z"
    }
  ]
}
```

---

#### PATCH /leads/:id
**Description**: Update lead status/assignment  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "status": "qualified",
  "assignedTo": 1,
  "lastContactAt": "2026-02-04T14:30:00Z"
}
```
**Response** (200):
```json
{
  "id": 150,
  "message": "Lead updated successfully"
}
```

---

#### POST /leads/:id/notes
**Description**: Add note to lead  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "note": "Follow-up call scheduled for tomorrow 3 PM",
  "noteType": "call"
}
```
**Response** (201):
```json
{
  "id": 90,
  "message": "Note added successfully"
}
```

---

### 2.5 Session Endpoints

#### GET /sessions
**Description**: List sessions/events  
**Access**: Public  
**Query Params**:
- `page`, `limit`
- `type` (weekly_live|workshop|code_review|masterclass)
- `status` (scheduled|live|completed|cancelled)
- `upcoming` (boolean) - Filter scheduled sessions after now
- `sort` (upcoming|recent, default: upcoming)

**Response** (200):
```json
{
  "data": [
    {
      "id": 201,
      "slug": "upgrade-lab-live-week-12",
      "title": "Upgrade Lab Live: Start & Ship - Week 12",
      "description": "This week: Building authentication with Next.js 14...",
      "sessionType": "weekly_live",
      "scheduledAt": "2026-02-07T19:00:00Z",
      "durationMinutes": 60,
      "isPaid": false,
      "priceEgp": null,
      "maxAttendees": 100,
      "registeredCount": 47,
      "spotsAvailable": 53,
      "status": "scheduled"
    }
  ],
  "pagination": {...}
}
```

---

#### GET /sessions/:slug
**Description**: Get session details  
**Access**: Public  
**Response** (200):
```json
{
  "id": 201,
  "slug": "upgrade-lab-live-week-12",
  "title": "Upgrade Lab Live: Start & Ship - Week 12",
  "description": "...",
  "sessionType": "weekly_live",
  "scheduledAt": "2026-02-07T19:00:00Z",
  "durationMinutes": 60,
  "meetingUrl": null,
  "recordingUrl": null,
  "isPaid": false,
  "priceEgp": null,
  "maxAttendees": 100,
  "registeredCount": 47,
  "spotsAvailable": 53,
  "status": "scheduled",
  "createdAt": "2026-01-31T10:00:00Z"
}
```
**Note**: `meetingUrl` only revealed 1 hour before session

---

#### POST /sessions/:id/register
**Description**: Register interest in session  
**Access**: Public  
**Rate Limit**: 5 per IP per day  
**Request**:
```json
{
  "fullName": "Omar Khaled",
  "email": "omar@gmail.com",
  "whatsapp": "+201055555555"
}
```
**Validation**:
- Check capacity: `registeredCount < maxAttendees`
- Prevent duplicate registration (same email)

**Response** (201):
```json
{
  "id": 312,
  "message": "تم التسجيل بنجاح!",
  "sessionTitle": "Upgrade Lab Live: Start & Ship - Week 12",
  "scheduledAt": "2026-02-07T19:00:00Z",
  "confirmationSent": true
}
```
**Errors**:
- `400`: Already registered
- `409`: Session full
- `400`: Session already completed

---

#### POST /sessions
**Description**: Create new session  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "title": "Upgrade Lab Live: Week 13",
  "slug": "upgrade-lab-live-week-13",
  "description": "Topic: Deploying to production with CI/CD",
  "sessionType": "weekly_live",
  "scheduledAt": "2026-02-14T19:00:00Z",
  "durationMinutes": 60,
  "meetingUrl": "https://meet.google.com/abc-xyz",
  "maxAttendees": 100,
  "isPaid": false
}
```
**Response** (201):
```json
{
  "id": 202,
  "slug": "upgrade-lab-live-week-13",
  "message": "Session created successfully"
}
```

---

### 2.6 Code Review Endpoints

#### POST /code-reviews
**Description**: Request code review  
**Access**: Public (creates lead + review request)  
**Rate Limit**: 3 per email per month  
**Request**:
```json
{
  "requesterName": "Omar Khaled",
  "requesterEmail": "omar.dev@outlook.com",
  "reviewType": "github_profile",
  "githubUrl": "https://github.com/omarkhaled",
  "linkedinUrl": "https://linkedin.com/in/omarkhaled",
  "description": "Looking for feedback before applying to jobs"
}
```
**Validation**:
- `reviewType`: Required, enum (github_profile|linkedin_profile|project_code|ui_review)
- Corresponding URL required based on type

**Response** (201):
```json
{
  "id": 78,
  "estimatedDelivery": "3-5 business days",
  "priceEgp": 300,
  "paymentInstructions": "تواصل عبر WhatsApp: +20xxxxxxxxx",
  "message": "تم استلام الطلب. سيتم التواصل قريبًا"
}
```

---

#### GET /code-reviews
**Description**: List code review requests  
**Access**: Editor, Super Admin  
**Query Params**: `status`, `reviewType`, `paid`, `page`, `limit`  
**Response** (200):
```json
{
  "data": [
    {
      "id": 78,
      "requesterName": "Omar Khaled",
      "requesterEmail": "omar.dev@outlook.com",
      "reviewType": "github_profile",
      "githubUrl": "https://github.com/omarkhaled",
      "status": "pending",
      "paid": false,
      "priceEgp": 300,
      "createdAt": "2026-02-04T11:20:00Z"
    }
  ],
  "pagination": {...}
}
```

---

#### PATCH /code-reviews/:id
**Description**: Update review (mark paid, complete review)  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "status": "completed",
  "paid": true,
  "reviewNotes": "Detailed feedback markdown content...",
  "videoReviewUrl": "https://youtube.com/watch?v=xxxxx"
}
```
**Response** (200):
```json
{
  "id": 78,
  "message": "Review updated successfully"
}
```

---

### 2.7 Testimonial Endpoints

#### POST /testimonials
**Description**: Submit testimonial  
**Access**: Public  
**Rate Limit**: 1 per email per 30 days  
**Request**:
```json
{
  "authorName": "Sara Mohamed",
  "authorRole": "Founder at StartupXYZ",
  "authorAvatar": "https://cdn.example.com/avatar.jpg",
  "content": "Working with Upgrade Lab was amazing. Fast delivery and clean code!",
  "rating": 5,
  "testimonialType": "client",
  "projectId": 1
}
```
**Validation**:
- `rating`: Required, 1-5
- `content`: Required, 20-1000 chars
- `testimonialType`: Required, enum

**Response** (201):
```json
{
  "id": 45,
  "message": "شكرًا! سيتم المراجعة والنشر قريبًا",
  "status": "pending"
}
```

---

#### GET /testimonials
**Description**: List approved testimonials  
**Access**: Public (approved only), Admin (all)  
**Query Params**: `type`, `featured`, `status` (admin), `page`, `limit`  
**Response** (200):
```json
{
  "data": [
    {
      "id": 45,
      "authorName": "Sara Mohamed",
      "authorRole": "Founder at StartupXYZ",
      "authorAvatar": "...",
      "content": "...",
      "rating": 5,
      "testimonialType": "client",
      "projectId": 1,
      "isFeatured": true,
      "createdAt": "2026-02-01T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

---

#### PATCH /testimonials/:id/approve
**Description**: Approve testimonial  
**Access**: Editor, Super Admin  
**Request**:
```json
{
  "approved": true,
  "featured": false
}
```
**Response** (200):
```json
{
  "id": 45,
  "message": "Testimonial approved"
}
```

---

### 2.8 Admin & Utility Endpoints

#### GET /tech-stack
**Description**: List all technologies  
**Access**: Public  
**Query Params**: `category`, `active`  
**Response** (200):
```json
{
  "data": [
    {
      "id": 1,
      "name": "React",
      "slug": "react",
      "category": "frontend",
      "logoUrl": "https://cdn.upgradelab.dev/tech/react.svg",
      "colorHex": "#61DAFB"
    }
  ]
}
```

---

#### GET /tags
**Description**: List all tags  
**Access**: Public  
**Query Params**: `type`, `sort` (popular|alpha)  
**Response** (200):
```json
{
  "data": [
    {
      "id": 1,
      "name": "Frontend",
      "slug": "frontend",
      "tagType": "topic",
      "useCount": 47
    }
  ]
}
```

---

#### GET /services
**Description**: List service packages  
**Access**: Public  
**Query Params**: `targetAudience` (student|client|both), `active`  
**Response** (200):
```json
{
  "data": [
    {
      "id": 1,
      "name": "MVP Sprint",
      "slug": "mvp-sprint",
      "description": "تحويل فكرة لمنتج أولي شغال",
      "features": [
        "Scope واضح",
        "تسليمات أسبوعية",
        "Code review"
      ],
      "targetAudience": "client",
      "priceEgp": 15000,
      "priceUsd": 500,
      "durationWeeks": 6,
      "deliverables": [
        "Deployed MVP",
        "Source code",
        "Documentation"
      ]
    }
  ]
}
```

---

#### GET /admin/dashboard/stats
**Description**: Admin dashboard statistics  
**Access**: Editor, Super Admin  
**Response** (200):
```json
{
  "leads": {
    "thisWeek": 18,
    "lastWeek": 12,
    "byType": {
      "student": 10,
      "client": 8
    },
    "byStatus": {
      "new": 5,
      "contacted": 8,
      "qualified": 3,
      "closed_won": 2
    }
  },
  "sessions": {
    "upcoming": 3,
    "completedThisMonth": 4,
    "totalRegistrations": 342,
    "averageAttendance": 67
  },
  "projects": {
    "published": 23,
    "drafts": 4,
    "totalViews": 8456
  },
  "posts": {
    "published": 34,
    "drafts": 7,
    "totalViews": 15234
  },
  "revenue": {
    "thisMonth": 45000,
    "lastMonth": 38000,
    "ytd": 180000
  }
}
```

---

## 3. Validation Rules

### Global Rules
- **Email**: RFC 5322 format validation
- **Phone/WhatsApp**: International format validation (E.164)
- **URLs**: Must be valid HTTP/HTTPS URLs
- **Slugs**: Lowercase, alphanumeric + hyphens, 3-255 chars, unique per resource
- **Dates**: ISO 8601 format (`YYYY-MM-DD` or full timestamp)

### Specific Endpoint Validations

#### POST /projects
```javascript
{
  title: {
    type: "string",
    required: true,
    minLength: 3,
    maxLength: 255
  },
  slug: {
    type: "string",
    required: true,
    pattern: /^[a-z0-9-]+$/,
    unique: true
  },
  problem: {
    type: "string",
    required: true,
    minLength: 20,
    maxLength: 5000
  },
  projectType: {
    type: "enum",
    required: true,
    values: ["mvp", "dashboard", "landing", "ui_upgrade", "other"]
  },
  technologies: {
    type: "array",
    items: "integer",
    validateExists: "tech_stack.id"
  }
}
```

#### POST /leads
```javascript
{
  fullName: {
    type: "string",
    required: true,
    minLength: 2,
    maxLength: 255
  },
  email: {
    type: "email",
    requiredIf: "!phone && !whatsapp"
  },
  leadType: {
    type: "enum",
    required: true,
    values: ["student", "client", "partner", "other"]
  },
  message: {
    type: "string",
    maxLength: 2000
  }
}
```

#### POST /sessions/:id/register
```javascript
{
  fullName: {
    type: "string",
    required: true,
    minLength: 2
  },
  email: {
    type: "email",
    required: true,
    unique: "per session" // No duplicate registrations
  },
  businessLogic: {
    checkCapacity: "registeredCount < maxAttendees",
    checkStatus: "session status === 'scheduled'"
  }
}
```

---

## 4. Pagination, Filtering, Sorting

### Pagination Strategy
**Offset-based** (suitable for CMS-like data with stable sorting):

```
GET /projects?page=2&limit=20
```

**Response format**:
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 156,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": true
  }
}
```

**Defaults**:
- `page`: 1
- `limit`: 20 (max: 100)

**Alternative for high-traffic endpoints**: Cursor-based pagination for `/sessions` (upcoming feature)

---

### Filtering

**Query String Format**:
```
GET /projects?type=dashboard&tech=react&tech=nextjs&featured=true&status=published
```

**Supported Filters by Resource**:

| Resource | Filters |
|----------|---------|
| Projects | `type`, `tech[]`, `tag[]`, `featured`, `status`, `clientType` |
| Posts | `type`, `tag[]`, `featured`, `status` |
| Leads | `type`, `status`, `source`, `assignedTo`, `search` |
| Sessions | `type`, `status`, `upcoming`, `isPaid` |
| Code Reviews | `reviewType`, `status`, `paid` |

**Search Implementation** (for `/leads?search=ahmed`):
- Full-text search on: `fullName`, `email`, `phone`, `message`
- PostgreSQL: Use `ts_vector` or `ILIKE` for Arabic support

---

### Sorting

**Query String Format**:
```
GET /projects?sort=popular
GET /posts?sort=-createdAt  // Descending
```

**Supported Sort Options**:

| Resource | Options | Default |
|----------|---------|---------|
| Projects | `recent`, `popular`, `alpha` | `recent` (completedAt DESC) |
| Posts | `recent`, `popular`, `alpha` | `recent` (publishedAt DESC) |
| Leads | `recent`, `oldest`, `last_contact` | `recent` (createdAt DESC) |
| Sessions | `upcoming`, `recent` | `upcoming` (scheduledAt ASC) |

**Implementation**:
- Use indexes on sorted columns for performance
- Default sort always stable (add `id` as secondary sort)

---

## 5. Rate Limiting

### Strategy
**Leaky bucket algorithm** with Redis backing

### Limits by Endpoint

| Endpoint | Limit | Window | Identifier |
|----------|-------|--------|------------|
| `POST /auth/login` | 5 requests | 15 minutes | IP + Email |
| `POST /leads` | 3 requests | 1 hour | IP |
| `POST /sessions/:id/register` | 5 requests | 24 hours | IP |
| `POST /code-reviews` | 3 requests | 30 days | Email |
| `POST /testimonials` | 1 request | 30 days | Email |
| `GET /projects` | 60 requests | 1 minute | IP (authenticated: 120/min) |
| `GET /posts` | 60 requests | 1 minute | IP |
| Admin endpoints | 300 requests | 1 minute | User ID |

### Headers
**Response headers on every request**:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1738687200
```

**On rate limit exceeded (429)**:
```json
{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please try again later.",
  "retryAfter": 3600
}
```

---

## 6. Security Considerations

### 6.1 Authentication Security
- ✅ **Password Hashing**: bcrypt with salt rounds = 12
- ✅ **JWT Secrets**: Rotate every 90 days
- ✅ **Refresh Token Rotation**: New token on each refresh
- ✅ **Token Revocation**: Database blacklist for logout
- ✅ **HTTPS Only**: Enforce TLS 1.3+

### 6.2 Input Validation
- ✅ **Sanitize HTML**: Strip tags from user input (DOMPurify for rich text)
- ✅ **SQL Injection**: Use parameterized queries (ORM: Prisma/TypeORM)
- ✅ **XSS Protection**: CSP headers + output encoding
- ✅ **File Uploads**: Validate MIME types, scan with ClamAV

### 6.3 CORS Policy
```javascript
{
  origin: [
    "https://upgradelab.dev",
    "https://www.upgradelab.dev",
    "http://localhost:3000" // Dev only
  ],
  credentials: true,
  allowedHeaders: ["Content-Type", "Authorization"],
  methods: ["GET", "POST", "PATCH", "DELETE"]
}
```

### 6.4 Additional Headers
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

### 6.5 API Key Authentication (Optional)
For integrations (e.g., WhatsApp bot, automation):
```
Authorization: Bearer api_key_xxxxxxxxxxxxxxxx
```
- Stored hashed in database
- Scoped permissions (read-only, write-leads, etc.)

### 6.6 Audit Logging
All destructive actions logged to `audit_log`:
- DELETE requests
- Status changes on leads/projects
- Admin user modifications
- Failed login attempts (security monitoring)

---

## 7. Error Response Format

### Standard Error Response
```json
{
  "error": "validation_error",
  "message": "Invalid request data",
  "details": [
    {
      "field": "email",
      "message": "Invalid email format"
    },
    {
      "field": "slug",
      "message": "Slug already exists"
    }
  ],
  "timestamp": "2026-02-04T15:53:26Z",
  "path": "/api/v1/projects",
  "requestId": "req_abc123xyz"
}
```

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| `200` | OK | Successful GET/PATCH |
| `201` | Created | Successful POST |
| `204` | No Content | Successful DELETE |
| `400` | Bad Request | Validation errors, business logic errors |
| `401` | Unauthorized | Missing/invalid token |
| `403` | Forbidden | Valid token but insufficient permissions |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Duplicate resource (unique constraint violation) |
| `422` | Unprocessable Entity | Semantic validation error |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server error |
| `503` | Service Unavailable | Database down, maintenance mode |

### Error Codes
```javascript
{
  // Authentication
  "invalid_credentials": 401,
  "token_expired": 401,
  "token_invalid": 401,
  "insufficient_permissions": 403,
  
  // Validation
  "validation_error": 400,
  "missing_required_field": 400,
  "invalid_format": 400,
  
  // Business Logic
  "resource_not_found": 404,
  "duplicate_resource": 409,
  "session_full": 409,
  "already_registered": 400,
  
  // Rate Limiting
  "rate_limit_exceeded": 429,
  
  // Server
  "internal_error": 500,
  "database_error": 500
}
```

---

## 8. OpenAPI-like Outline

```yaml
openapi: 3.0.0
info:
  title: Upgrade Lab API
  version: 1.0.0
  description: Backend API for Upgrade Lab platform
  contact:
    name: Upgrade Lab Support
    email: support@upgradelab.dev

servers:
  - url: https://api.upgradelab.dev/v1
    description: Production
  - url: http://localhost:3000/api/v1
    description: Development

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    Error:
      type: object
      properties:
        error:
          type: string
        message:
          type: string
        details:
          type: array
        timestamp:
          type: string
          format: date-time

    Pagination:
      type: object
      properties:
        page:
          type: integer
        limit:
          type: integer
        total:
          type: integer
        totalPages:
          type: integer

    Project:
      type: object
      properties:
        id:
          type: integer
        slug:
          type: string
        title:
          type: string
        subtitle:
          type: string
        problem:
          type: string
        solution:
          type: string
        # ... (full schema)

    Lead:
      type: object
      properties:
        id:
          type: integer
        fullName:
          type: string
        email:
          type: string
        # ... (full schema)

paths:
  /auth/login:
    post:
      tags: [Authentication]
      summary: Admin login
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                email:
                  type: string
                password:
                  type: string
      responses:
        200:
          description: Login successful
        401:
          description: Invalid credentials

  /projects:
    get:
      tags: [Projects]
      summary: List projects
      parameters:
        - name: page
          in: query
          schema:
            type: integer
        - name: limit
          in: query
          schema:
            type: integer
        - name: type
          in: query
          schema:
            type: string
      responses:
        200:
          description: Projects list
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Project'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

  /leads:
    post:
      tags: [Leads]
      summary: Submit lead form
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Lead'
      responses:
        201:
          description: Lead created
        429:
          description: Rate limit exceeded

# ... (continue for all endpoints)
```

---

## 9. Implementation Checklist

### Phase 1: Core API (Week 1-2)
- [ ] Set up Express/Fastify + PostgreSQL + Prisma
- [ ] Implement JWT authentication + refresh tokens
- [ ] Build Projects CRUD (public read, admin write)
- [ ] Build Posts CRUD
- [ ] Implement pagination/filtering/sorting utilities

### Phase 2: Lead Management (Week 2-3)
- [ ] Public lead submission with rate limiting
- [ ] Admin lead CRM endpoints
- [ ] Lead notes system
- [ ] Email notifications on new leads

### Phase 3: Sessions & Engagement (Week 3-4)
- [ ] Session CRUD
- [ ] Session registration with capacity checks
- [ ] Attendance tracking (admin)
- [ ] Email confirmations + calendar invites

### Phase 4: Monetization (Week 4-5)
- [ ] Code review request system
- [ ] Testimonial submission + approval
- [ ] Service packages display
- [ ] Payment integration (Stripe/Fawry)

### Phase 5: Polish & Production (Week 5-6)
- [ ] Admin dashboard statistics
- [ ] Audit logging for security
- [ ] Rate limiting with Redis
- [ ] Full test coverage (unit + integration)
- [ ] API documentation (Swagger UI)
- [ ] Deploy to production (Railway/Render/DigitalOcean)

---

## 10. Recommended Tech Stack

**Backend Framework**: Fastify (faster than Express) or NestJS (enterprise-grade)  
**ORM**: Prisma (best TypeScript DX)  
**Validation**: Zod or Joi  
**Authentication**: jose (JWT) + bcrypt  
**Rate Limiting**: express-rate-limit + Redis  
**File Uploads**: Multer + AWS S3/Cloudinary  
**Email**: Resend or SendGrid  
**Payments**: Stripe (international) + Fawry (Egypt)  
**Documentation**: Swagger/Scalar  
**Testing**: Vitest + Supertest  
**Deployment**: Railway (easy) or DigitalOcean App Platform  

---

## Next Steps

1. **Set up skeleton project** with authentication
2. **Implement Projects API first** (most visible feature)
3. **Build admin dashboard** to manage content
4. **Add lead capture** to start collecting data
5. **Integrate sessions** for community building
6. **Deploy MVP** and iterate based on real usage

This API design is production-ready and scalable. Let me know when you're ready to start implementation! 🚀
