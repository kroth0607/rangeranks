# Range Ranks: Migration to Online App Plan

## Overview
This document outlines the plan to migrate Range Ranks from a local-only browser app to a full-stack online application with shared database, real-time updates, and multi-device support.

---

## 1. Architecture Overview

### Current State
- **Frontend**: Single HTML file with vanilla JavaScript
- **Storage**: Browser localStorage (client-side only)
- **Data**: Isolated per device/browser
- **No backend**: Everything runs client-side

### Target State
- **Frontend**: Modern web app (React/Vue/Vanilla JS)
- **Backend**: REST API + WebSocket server
- **Database**: Shared PostgreSQL/MongoDB database
- **Authentication**: User accounts with secure login
- **Real-time**: Live updates when friends log sessions

### Recommended Stack Options

#### Option A: Full-Stack JavaScript (Recommended for simplicity)
- **Frontend**: React or Vue.js
- **Backend**: Node.js + Express
- **Database**: PostgreSQL (with Prisma ORM) or MongoDB
- **Real-time**: Socket.io
- **Auth**: JWT tokens + bcrypt
- **Hosting**: Vercel (frontend) + Railway/Render (backend)

#### Option B: Python Backend
- **Frontend**: React/Vue
- **Backend**: Python + FastAPI or Django
- **Database**: PostgreSQL
- **Real-time**: WebSockets (FastAPI) or Django Channels
- **Auth**: JWT or Django Auth
- **Hosting**: Vercel (frontend) + Railway/Render (backend)

#### Option C: Serverless (Scalable but more complex)
- **Frontend**: React/Vue
- **Backend**: AWS Lambda / Vercel Functions
- **Database**: Supabase or Firebase
- **Real-time**: Supabase Realtime or Firebase Realtime
- **Auth**: Supabase Auth or Firebase Auth
- **Hosting**: Vercel (all-in-one)

---

## 2. Database Schema Design

### Core Tables

#### `users` table
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  display_name VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_login TIMESTAMP,
  is_active BOOLEAN DEFAULT true
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

#### `sessions` table (renamed from "shooters" to match user concept)
```sql
CREATE TABLE shooting_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  drill_id VARCHAR(50) NOT NULL,
  shots INTEGER NOT NULL CHECK (shots > 0),
  hits INTEGER NOT NULL CHECK (hits >= 0 AND hits <= shots),
  xp_earned INTEGER NOT NULL DEFAULT 0,
  timestamp TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW(),
  
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_sessions_user_id ON shooting_sessions(user_id);
CREATE INDEX idx_sessions_timestamp ON shooting_sessions(timestamp DESC);
CREATE INDEX idx_sessions_user_timestamp ON shooting_sessions(user_id, timestamp DESC);
```

#### `follows` table
```sql
CREATE TABLE follows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  follower_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  following_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(follower_id, following_id),
  CHECK (follower_id != following_id),
  
  FOREIGN KEY (follower_id) REFERENCES users(id),
  FOREIGN KEY (following_id) REFERENCES users(id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);
```

#### `drills` table (for future customization)
```sql
CREATE TABLE drills (
  id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  default_shots INTEGER NOT NULL,
  difficulty DECIMAL(3,2) NOT NULL,
  distance INTEGER NOT NULL,
  is_active BOOLEAN DEFAULT true
);
```

### Computed Fields (Views or Application Logic)

#### User XP Calculation
```sql
-- XP is calculated from sessions, stored as computed field or cached
CREATE VIEW user_stats AS
SELECT 
  u.id,
  u.username,
  u.display_name,
  COALESCE(SUM(ss.xp_earned), 0) as total_xp,
  COUNT(ss.id) as session_count,
  MAX(ss.timestamp) as last_session_at
FROM users u
LEFT JOIN shooting_sessions ss ON u.id = ss.user_id
GROUP BY u.id, u.username, u.display_name;
```

### Alternative: MongoDB Schema
```javascript
// users collection
{
  _id: ObjectId,
  username: String (unique),
  email: String (unique),
  passwordHash: String,
  displayName: String,
  createdAt: Date,
  lastLogin: Date,
  // Computed fields (updated on session creation)
  totalXP: Number,
  sessionCount: Number
}

// sessions collection
{
  _id: ObjectId,
  userId: ObjectId (ref: users),
  drillId: String,
  shots: Number,
  hits: Number,
  xpEarned: Number,
  timestamp: Date,
  createdAt: Date
}

// follows collection
{
  _id: ObjectId,
  followerId: ObjectId (ref: users),
  followingId: ObjectId (ref: users),
  createdAt: Date,
  // Compound index on (followerId, followingId) for uniqueness
}
```

---

## 3. API Design

### REST Endpoints

#### Authentication
```
POST   /api/auth/register     - Register new user
POST   /api/auth/login        - Login (returns JWT token)
POST   /api/auth/logout       - Logout (invalidate token)
GET    /api/auth/me           - Get current user info
PUT    /api/auth/me           - Update current user profile
```

#### Users
```
GET    /api/users              - List users (with pagination, search)
GET    /api/users/:id          - Get user profile
GET    /api/users/:id/stats    - Get user stats (XP, rank, sessions count)
GET    /api/users/:id/sessions - Get user's sessions (paginated)
```

#### Sessions
```
GET    /api/sessions           - Get current user's sessions
POST   /api/sessions           - Create new session
GET    /api/sessions/:id       - Get specific session
DELETE /api/sessions/:id       - Delete session (own only)
```

#### Follows
```
GET    /api/users/:id/followers   - Get user's followers
GET    /api/users/:id/following   - Get users that :id follows
POST   /api/users/:id/follow      - Follow a user
DELETE /api/users/:id/follow      - Unfollow a user
```

#### Feed
```
GET    /api/feed                - Get activity feed (followed users' sessions)
```

#### Leaderboard
```
GET    /api/leaderboard         - Get global leaderboard
GET    /api/leaderboard/friends - Get friends-only leaderboard
```

### WebSocket Events (Real-time)

```javascript
// Client → Server
socket.emit('join_user_feed', userId)
socket.emit('leave_user_feed', userId)

// Server → Client
socket.on('new_session', (sessionData) => {
  // New session from followed user
})

socket.on('user_updated', (userData) => {
  // User stats changed (XP, rank)
})

socket.on('new_follower', (followerData) => {
  // Someone followed you
})
```

---

## 4. Authentication & Security

### Authentication Flow
1. User registers with username, email, password
2. Password is hashed with bcrypt (10+ rounds)
3. JWT token issued on successful login
4. Token stored in httpOnly cookie or localStorage
5. Token included in Authorization header for API calls
6. Token expires after 7-30 days (configurable)

### Security Measures
- **Password**: Minimum 8 chars, hashed with bcrypt
- **JWT**: Signed with secret, includes expiration
- **CORS**: Configured for your domain only
- **Rate Limiting**: Prevent brute force (e.g., 5 login attempts per minute)
- **Input Validation**: Sanitize all user inputs
- **SQL Injection**: Use parameterized queries (ORM handles this)
- **XSS**: Sanitize user-generated content
- **HTTPS**: Enforce SSL/TLS in production

### Authorization Rules
- Users can only edit/delete their own sessions
- Users can view any public profile
- Follow/unfollow requires authentication
- Leaderboard is public (read-only)

---

## 5. Frontend Changes Required

### Current Structure to Refactor

#### 1. Separate API Layer
```javascript
// api.js
const API_BASE = 'https://api.rangeranks.com';

export const api = {
  async get(endpoint, token) {
    const response = await fetch(`${API_BASE}${endpoint}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    return response.json();
  },
  
  async post(endpoint, data, token) {
    const response = await fetch(`${API_BASE}${endpoint}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify(data)
    });
    return response.json();
  }
};
```

#### 2. State Management
- Replace localStorage with API calls
- Add loading states
- Add error handling
- Implement optimistic updates

#### 3. Authentication UI
- Login/Register forms
- Token management
- Auto-logout on token expiry
- "Remember me" functionality

#### 4. Real-time Updates
```javascript
// websocket.js
import io from 'socket.io-client';

const socket = io('https://api.rangeranks.com', {
  auth: { token: userToken }
});

socket.on('new_session', (session) => {
  // Update feed in real-time
  updateFeed(session);
});
```

#### 5. Error Handling
- Network error messages
- Retry logic for failed requests
- Offline mode detection
- Graceful degradation

---

## 6. Migration Strategy

### Phase 1: Backend Setup (Week 1-2)
1. Set up backend project structure
2. Create database schema
3. Implement authentication endpoints
4. Implement basic CRUD for sessions
5. Add follow/unfollow endpoints
6. Set up WebSocket server

### Phase 2: Frontend Refactor (Week 2-3)
1. Create API service layer
2. Add authentication UI
3. Replace localStorage calls with API calls
4. Add loading/error states
5. Implement real-time feed updates

### Phase 3: Testing & Polish (Week 3-4)
1. Test all features end-to-end
2. Add input validation
3. Implement error handling
4. Performance optimization
5. Security audit

### Phase 4: Deployment (Week 4)
1. Set up production database
2. Deploy backend to hosting service
3. Deploy frontend to CDN
4. Configure domain & SSL
5. Monitor & fix issues

### Data Migration (if needed)
If users have existing localStorage data:
1. Create export feature in old app (JSON download)
2. Create import feature in new app (upload JSON)
3. Map old data structure to new schema
4. Validate imported data

---

## 7. Technology Recommendations

### Recommended: Supabase (Easiest Path)
**Why**: All-in-one solution, PostgreSQL + Auth + Realtime + Storage

**Setup**:
1. Create Supabase project
2. Run SQL migrations
3. Enable Auth (email/password)
4. Use Supabase JS client
5. Enable Realtime for feed updates

**Pros**:
- Fastest to implement
- Built-in authentication
- Real-time subscriptions
- Generous free tier
- Auto-scaling

**Cons**:
- Less control over backend
- Vendor lock-in

### Alternative: Node.js + Express + PostgreSQL
**Why**: Full control, popular stack, lots of resources

**Setup**:
1. Express.js server
2. PostgreSQL database (Railway/Render)
3. Prisma ORM
4. Socket.io for real-time
5. JWT for auth

**Pros**:
- Full control
- Flexible
- Large ecosystem
- Can self-host

**Cons**:
- More setup required
- Need to manage auth yourself

### Alternative: Firebase
**Why**: Google-backed, real-time by default

**Setup**:
1. Firebase project
2. Firestore database
3. Firebase Auth
4. Firestore real-time listeners

**Pros**:
- Real-time built-in
- Good free tier
- Easy to scale

**Cons**:
- NoSQL (different query patterns)
- Vendor lock-in
- Can get expensive at scale

---

## 8. Deployment Plan

### Backend Deployment Options

#### Option A: Railway (Recommended)
- **Cost**: $5/month starter, $20/month for better performance
- **Setup**: Connect GitHub, auto-deploys
- **Database**: Managed PostgreSQL included
- **Pros**: Easy, includes database, good free tier
- **URL**: `https://your-app.railway.app`

#### Option B: Render
- **Cost**: Free tier available, $7/month for database
- **Setup**: Similar to Railway
- **Database**: Managed PostgreSQL
- **Pros**: Free tier, easy setup

#### Option C: DigitalOcean App Platform
- **Cost**: $5/month minimum
- **Setup**: More configuration needed
- **Database**: Managed database available
- **Pros**: More control, good docs

### Frontend Deployment

#### Option A: Vercel (Recommended)
- **Cost**: Free for personal projects
- **Setup**: Connect GitHub, auto-deploys
- **Pros**: Fast CDN, great DX, free SSL
- **URL**: `https://your-app.vercel.app`

#### Option B: Netlify
- **Cost**: Free tier available
- **Setup**: Similar to Vercel
- **Pros**: Good free tier, easy setup

### Database (if separate)

#### Option A: Supabase
- **Cost**: Free tier (500MB, 2GB bandwidth)
- **Pros**: Includes auth, real-time, easy

#### Option B: Railway/Render Managed PostgreSQL
- **Cost**: Included with hosting
- **Pros**: Integrated, easy backups

#### Option C: Neon (Serverless Postgres)
- **Cost**: Free tier available
- **Pros**: Serverless, auto-scaling

---

## 9. Cost Estimates

### Free Tier (MVP)
- **Frontend**: Vercel (Free)
- **Backend**: Render Free Tier
- **Database**: Supabase Free Tier
- **Total**: $0/month

### Small Scale (100-1000 users)
- **Frontend**: Vercel Pro ($20/month)
- **Backend**: Railway ($5/month)
- **Database**: Included with Railway
- **Total**: ~$25/month

### Medium Scale (1000-10000 users)
- **Frontend**: Vercel Pro ($20/month)
- **Backend**: Railway ($20/month)
- **Database**: Managed PostgreSQL ($15/month)
- **Total**: ~$55/month

---

## 10. Implementation Checklist

### Backend
- [ ] Set up project structure
- [ ] Configure database connection
- [ ] Create database migrations
- [ ] Implement user model/controller
- [ ] Implement session model/controller
- [ ] Implement follow model/controller
- [ ] Set up authentication (JWT)
- [ ] Create REST API endpoints
- [ ] Set up WebSocket server
- [ ] Add input validation
- [ ] Add error handling
- [ ] Write API tests
- [ ] Set up CI/CD

### Frontend
- [ ] Set up build system (Vite/Webpack)
- [ ] Create API service layer
- [ ] Add authentication UI (login/register)
- [ ] Replace localStorage with API calls
- [ ] Add loading states
- [ ] Add error handling
- [ ] Implement real-time updates
- [ ] Add offline detection
- [ ] Optimize bundle size
- [ ] Add responsive design
- [ ] Test on multiple devices

### DevOps
- [ ] Set up production database
- [ ] Configure environment variables
- [ ] Set up domain name
- [ ] Configure SSL certificates
- [ ] Set up monitoring (Sentry, etc.)
- [ ] Set up logging
- [ ] Create backup strategy
- [ ] Document deployment process

---

## 11. Next Steps

1. **Choose your stack** (Recommend: Supabase for fastest path)
2. **Set up development environment**
3. **Create database schema**
4. **Build authentication first**
5. **Migrate core features one by one**
6. **Add real-time features**
7. **Deploy to staging**
8. **Test thoroughly**
9. **Deploy to production**

---

## 12. Additional Features to Consider

### Short-term
- Email verification
- Password reset
- Profile pictures
- User search
- Notifications (new follower, etc.)

### Medium-term
- Custom drills
- Session photos
- Comments on sessions
- Achievements/badges
- Private groups/teams

### Long-term
- Mobile app (React Native)
- Analytics dashboard
- Export data (CSV/PDF)
- Social sharing
- API for third-party integrations

---

## Questions to Answer Before Starting

1. **Budget**: Free tier or paid hosting?
2. **Timeline**: How quickly do you need this?
3. **Technical Skill**: Comfortable with full-stack or prefer simpler solution?
4. **Scale**: Expected number of users?
5. **Features**: What's essential vs. nice-to-have?

---

## Recommended Starting Point

**For fastest implementation**: Use **Supabase**
- Set up takes ~30 minutes
- All features included (auth, database, real-time)
- Free tier is generous
- Great documentation

**For learning/control**: Use **Node.js + Express + PostgreSQL**
- More setup but full control
- Learn full-stack development
- Can customize everything

Would you like me to start implementing any specific part of this plan?

