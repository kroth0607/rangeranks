# Range Ranks API Specification

Quick reference for API endpoints and data structures.

## Base URL
```
Production: https://api.rangeranks.com
Development: http://localhost:3000
```

## Authentication
All protected endpoints require a JWT token in the Authorization header:
```
Authorization: Bearer <token>
```

---

## Endpoints

### Authentication

#### Register
```http
POST /api/auth/register
Content-Type: application/json

{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "securepassword123",
  "displayName": "John Doe"
}

Response: 201 Created
{
  "user": {
    "id": "uuid",
    "username": "johndoe",
    "displayName": "John Doe",
    "email": "john@example.com"
  },
  "token": "jwt_token_here"
}
```

#### Login
```http
POST /api/auth/login
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "securepassword123"
}

Response: 200 OK
{
  "user": { ... },
  "token": "jwt_token_here"
}
```

#### Get Current User
```http
GET /api/auth/me
Authorization: Bearer <token>

Response: 200 OK
{
  "id": "uuid",
  "username": "johndoe",
  "displayName": "John Doe",
  "email": "john@example.com",
  "totalXP": 1250,
  "sessionCount": 15,
  "rank": "Sharpshooter"
}
```

---

### Users

#### List Users
```http
GET /api/users?page=1&limit=20&search=john
Authorization: Bearer <token>

Response: 200 OK
{
  "users": [
    {
      "id": "uuid",
      "username": "johndoe",
      "displayName": "John Doe",
      "totalXP": 1250,
      "sessionCount": 15,
      "rank": "Sharpshooter",
      "isFollowing": false
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

#### Get User Profile
```http
GET /api/users/:id
Authorization: Bearer <token>

Response: 200 OK
{
  "id": "uuid",
  "username": "johndoe",
  "displayName": "John Doe",
  "totalXP": 1250,
  "sessionCount": 15,
  "rank": "Sharpshooter",
  "followersCount": 12,
  "followingCount": 8,
  "isFollowing": true,
  "isCurrentUser": false
}
```

#### Get User Stats
```http
GET /api/users/:id/stats
Authorization: Bearer <token>

Response: 200 OK
{
  "totalXP": 1250,
  "sessionCount": 15,
  "rank": "Sharpshooter",
  "currentRankXP": 250,
  "nextRankXP": 750,
  "progressPercent": 33.3,
  "averageAccuracy": 0.85,
  "lastSessionAt": "2024-01-15T10:30:00Z"
}
```

---

### Sessions

#### Create Session
```http
POST /api/sessions
Authorization: Bearer <token>
Content-Type: application/json

{
  "drillId": "basic7",
  "shots": 10,
  "hits": 9
}

Response: 201 Created
{
  "id": "uuid",
  "userId": "uuid",
  "drillId": "basic7",
  "shots": 10,
  "hits": 9,
  "xpEarned": 45,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

#### Get User Sessions
```http
GET /api/sessions?userId=uuid&page=1&limit=20
Authorization: Bearer <token>

Response: 200 OK
{
  "sessions": [
    {
      "id": "uuid",
      "userId": "uuid",
      "drillId": "basic7",
      "drillName": "Basic 7 – 10 shots @ 7 yards",
      "shots": 10,
      "hits": 9,
      "xpEarned": 45,
      "timestamp": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": { ... }
}
```

#### Delete Session
```http
DELETE /api/sessions/:id
Authorization: Bearer <token>

Response: 204 No Content
```

---

### Follows

#### Follow User
```http
POST /api/users/:id/follow
Authorization: Bearer <token>

Response: 201 Created
{
  "followerId": "current_user_uuid",
  "followingId": "target_user_uuid",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

#### Unfollow User
```http
DELETE /api/users/:id/follow
Authorization: Bearer <token>

Response: 204 No Content
```

#### Get Followers
```http
GET /api/users/:id/followers
Authorization: Bearer <token>

Response: 200 OK
{
  "followers": [
    {
      "id": "uuid",
      "username": "johndoe",
      "displayName": "John Doe",
      "totalXP": 1250
    }
  ],
  "count": 12
}
```

#### Get Following
```http
GET /api/users/:id/following
Authorization: Bearer <token>

Response: 200 OK
{
  "following": [
    {
      "id": "uuid",
      "username": "johndoe",
      "displayName": "John Doe",
      "totalXP": 1250
    }
  ],
  "count": 8
}
```

---

### Feed

#### Get Activity Feed
```http
GET /api/feed?page=1&limit=20
Authorization: Bearer <token>

Response: 200 OK
{
  "feed": [
    {
      "id": "uuid",
      "user": {
        "id": "uuid",
        "username": "johndoe",
        "displayName": "John Doe"
      },
      "session": {
        "id": "uuid",
        "drillId": "basic7",
        "drillName": "Basic 7 – 10 shots @ 7 yards",
        "shots": 10,
        "hits": 9,
        "xpEarned": 45,
        "timestamp": "2024-01-15T10:30:00Z"
      }
    }
  ],
  "pagination": { ... }
}
```

---

### Leaderboard

#### Get Global Leaderboard
```http
GET /api/leaderboard?page=1&limit=50
Authorization: Bearer <token> (optional)

Response: 200 OK
{
  "leaderboard": [
    {
      "rank": 1,
      "user": {
        "id": "uuid",
        "username": "johndoe",
        "displayName": "John Doe"
      },
      "totalXP": 5000,
      "sessionCount": 50,
      "rankName": "Legend",
      "isFollowing": false
    }
  ],
  "pagination": { ... }
}
```

#### Get Friends Leaderboard
```http
GET /api/leaderboard/friends
Authorization: Bearer <token>

Response: 200 OK
{
  "leaderboard": [
    {
      "rank": 1,
      "user": { ... },
      "totalXP": 5000,
      "sessionCount": 50,
      "rankName": "Legend"
    }
  ]
}
```

---

## WebSocket Events

### Connection
```javascript
const socket = io('https://api.rangeranks.com', {
  auth: { token: 'jwt_token_here' }
});
```

### Client → Server Events

#### Join User Feed
```javascript
socket.emit('join_user_feed', userId);
```

#### Leave User Feed
```javascript
socket.emit('leave_user_feed', userId);
```

### Server → Client Events

#### New Session
```javascript
socket.on('new_session', (data) => {
  // data: {
  //   userId: "uuid",
  //   username: "johndoe",
  //   session: { ... }
  // }
});
```

#### User Updated
```javascript
socket.on('user_updated', (data) => {
  // data: {
  //   userId: "uuid",
  //   totalXP: 1250,
  //   rank: "Sharpshooter"
  // }
});
```

#### New Follower
```javascript
socket.on('new_follower', (data) => {
  // data: {
  //   followerId: "uuid",
  //   followerUsername: "johndoe"
  // }
});
```

---

## Data Models

### User
```typescript
interface User {
  id: string;
  username: string;
  email: string;
  displayName: string;
  totalXP: number;
  sessionCount: number;
  rank: string;
  followersCount?: number;
  followingCount?: number;
  isFollowing?: boolean;
  isCurrentUser?: boolean;
  createdAt: string;
  lastLogin?: string;
}
```

### Session
```typescript
interface Session {
  id: string;
  userId: string;
  drillId: string;
  drillName?: string;
  shots: number;
  hits: number;
  xpEarned: number;
  timestamp: string;
  createdAt: string;
}
```

### Follow
```typescript
interface Follow {
  id: string;
  followerId: string;
  followingId: string;
  createdAt: string;
}
```

### Drill
```typescript
interface Drill {
  id: string;
  name: string;
  defaultShots: number;
  difficulty: number;
  distance: number;
  isActive: boolean;
}
```

---

## Error Responses

### Standard Error Format
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {}
  }
}
```

### Common Error Codes

- `UNAUTHORIZED` (401) - Missing or invalid token
- `FORBIDDEN` (403) - Not allowed to perform action
- `NOT_FOUND` (404) - Resource not found
- `VALIDATION_ERROR` (400) - Invalid input data
- `CONFLICT` (409) - Resource already exists (e.g., duplicate username)
- `INTERNAL_ERROR` (500) - Server error

### Example Error Response
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "shots": "Must be greater than 0",
      "hits": "Must be between 0 and shots"
    }
  }
}
```

---

## Rate Limiting

- **Authentication endpoints**: 5 requests per minute
- **Other endpoints**: 100 requests per minute per user
- Headers included in response:
  ```
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 95
  X-RateLimit-Reset: 1642248000
  ```

---

## Pagination

All list endpoints support pagination:
```
GET /api/endpoint?page=1&limit=20
```

Response includes pagination metadata:
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

