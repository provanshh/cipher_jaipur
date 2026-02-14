# Design Document: CipherGuard Parental Control System

## Overview

CipherGuard is a full-stack parental control platform built with a three-tier architecture: a Node.js/Express backend API, a React/TypeScript frontend dashboard, and a Chrome browser extension. The system enables parents to monitor children's browsing activity, block inappropriate content, detect incognito mode usage, and generate activity reports.

### Architecture Philosophy

The system follows a client-server architecture with clear separation of concerns:
- **Backend**: Centralized data storage and business logic
- **Frontend**: Parent-facing dashboard for monitoring and management
- **Extension**: Client-side monitoring and enforcement on child devices

All components communicate via REST APIs with JWT-based authentication.

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     CipherGuard System                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐         ┌──────────────┐                  │
│  │   Frontend   │◄────────┤   Backend    │                  │
│  │  Dashboard   │  REST   │     API      │                  │
│  │ (React/TS)   │  API    │ (Node/Express│                  │
│  └──────────────┘         │  /MongoDB)   │                  │
│         │                 └──────┬───────┘                  │
│         │                        │                           │
│         │                        │                           │
│         │                 ┌──────▼───────┐                  │
│         │                 │   MongoDB    │                  │
│         │                 │   Database   │                  │
│         │                 └──────────────┘                  │
│         │                        ▲                           │
│         │                        │                           │
│  ┌──────▼──────┐                │                           │
│  │   Browser   │────────────────┘                           │
│  │  Extension  │     REST API                               │
│  │ (Vanilla JS)│                                            │
│  └─────────────┘                                            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

**Backend:**
- Runtime: Node.js
- Framework: Express.js
- Database: MongoDB with Mongoose ODM
- Authentication: JWT (jsonwebtoken)
- Password Hashing: bcryptjs (currently disabled in code)
- Notifications: Twilio (for Telegram integration)
- Environment: dotenv for configuration

**Frontend:**
- Framework: React 18
- Language: TypeScript
- Build Tool: Vite
- Styling: Tailwind CSS
- UI Components: shadcn-ui
- Routing: React Router DOM
- State Management: React Query (@tanstack/react-query)
- Charts: Recharts
- Icons: Lucide React
- PDF Generation: jsPDF with jspdf-autotable

**Browser Extension:**
- Manifest Version: 3
- Language: Vanilla JavaScript
- Content Filtering: DOM manipulation and MutationObserver
- Image Analysis: External NSFW classification API (optional)

### Deployment Architecture

- Backend: Runs on port 5000 (configurable via environment)
- Frontend: Vite dev server on port 5173 (development)
- Database: MongoDB connection via MONGO_URI environment variable
- CORS: Configured for local development, ngrok tunnels, and Chrome extensions

## Components and Interfaces

### Backend API Components

#### 1. Server Entry Point (`server.js`)

**Responsibilities:**
- Initialize Express application
- Configure middleware (CORS, body-parser)
- Connect to MongoDB
- Mount route handlers
- Start HTTP server

**Key Configuration:**
```javascript
CORS Origins:
- http://localhost:5173 (Vite dev server)
- https://*.ngrok-free.app (tunneling)
- chrome-extension://* (browser extension)
```

#### 2. Data Models

**Parent Model (`models/parent.js`):**
```javascript
{
  name: String (required),
  email: String (required, unique),
  password: String (required),
  children: [ObjectId] (references Child),
  telegramChatId: String (optional)
}
```

**Child Model (`models/child.js`):**
```javascript
{
  name: String,
  email: String (unique),
  extensionToken: String (UUID),
  blockedUrls: [String],
  monitoredUrls: [{
    url: String,
    domain: String,
    category: String,
    dailyTimeSpent: Map<String, Number>, // date -> seconds
    searchQueries: [String],
    lastUpdated: Date
  }],
  incognitoAlerts: [{
    url: String,
    timestamp: Date
  }],
  lastHeartbeat: Date,
  status: String (enum: 'online', 'offline')
}
```

**Log Model (`models/log.js`):**
```javascript
{
  child: ObjectId (references Child),
  type: String, // 'BLOCKED_URL', 'INCOGNITO_ALERT'
  domain: String,
  timestamp: Date,
  message: String
}
```

#### 3. Controllers

**Authentication Controller (`controller/authContoller.js`):**
- `register(req, res)`: Create new parent account, generate JWT
- `login(req, res)`: Validate credentials, return JWT
- `getParent(req, res)`: Return authenticated parent's profile

**Child Controller (`controller/childController.js`):**
- `createChild(req, res)`: Create child profile, generate extension token
- `getChildren(req, res)`: List all children for parent
- `getWebUsageStats(req, res)`: Calculate total time for today
- `getWebUsageStatsFull(req, res)`: Detailed per-domain usage
- `getAlerts(req, res)`: Count of incognito alerts
- `getAlertsFull(req, res)`: Detailed alert list with timestamps
- `clearAlerts(req, res)`: Remove all alerts for child
- `getBlockedStats(req, res)`: Count of blocked URLs
- `getBlockedStatsFull(req, res)`: List of all blocked domains
- `getSearchActivities(req, res)`: Filtered activity timeline

**Monitor Controller (`controller/monitorController.js`):**
- `monitorUrl(req, res)`: Record URL visit and time spent
- `alertIncognito(req, res)`: Store incognito detection event
- `checkUrl(req, res)`: Verify if URL is blocked
- `activateExtension(req, res)`: Mark extension as online
- `disconnectExtension(req, res)`: Mark extension as offline

**Parent Controller (`controller/parentContoller.js`):**
- `getAllChildren(req, res)`: Get children with populated profiles
- `getChildDetails(req, res)`: Get specific child by ID
- `getChildUrls(req, res)`: Get monitored URLs for child
- `getChildAlerts(req, res)`: Get incognito alerts for child
- `blockUrl(req, res)`: Add URL to blocked list
- `unblockUrl(req, res)`: Remove URL from blocked list
- `resetTimeSpent(req, res)`: Clear time tracking data

#### 4. Middleware

**Auth Middleware (`middleware/authMiddleware.js`):**
- `verifyToken(req, res, next)`: Validate JWT from Authorization header
- Extracts email from token and attaches to `req.user`

**Extension Auth Middleware (`middleware/extensionAuth.js`):**
- Similar to authMiddleware but specifically for extension requests

#### 5. Routes

**Auth Routes (`/api/auth`):**
- POST `/signup` - Parent registration
- POST `/login` - Parent login
- GET `/user` - Get current parent (protected)

**Child Routes (`/api/child`):**
- POST `/add-child` - Create child profile (protected)
- GET `/all` - List all children (protected)
- GET `/web-usage/:email` - Summary stats
- GET `/web-usagefull/:email` - Detailed usage
- GET `/alerts/:email` - Alert count
- GET `/alertsfull/:email` - Alert details
- DELETE `/delete-alerts/:email` - Clear alerts
- GET `/blocked/:email` - Blocked count
- GET `/blockedfull/:email` - Blocked list
- POST `/web-usage-filtered` - Filtered activity timeline

**Monitor Routes (`/api/monitor`):**
- POST `/monitor-url` - Record URL visit (extension auth)
- POST `/incognito-alert` - Report incognito detection (extension auth)
- POST `/check-url` - Check if URL is blocked (extension auth)
- POST `/activate` - Extension activation (extension auth)
- POST `/disconnect` - Extension disconnection (extension auth)

**Parent Routes (`/api/parent`):**
- GET `/children` - Get all children (protected)
- GET `/children/:id` - Get child details (protected)
- GET `/children/:id/urls` - Get child URLs (protected)
- GET `/children/:id/alerts` - Get child alerts (protected)
- POST `/children/block` - Block URL (protected)
- POST `/children/unblock` - Unblock URL (protected)
- POST `/children/:id/reset` - Reset time tracking (protected)

#### 6. Utilities

**JWT Utility (`utillity/jwt.js`):**
- `generateToken(email)`: Create JWT with email payload

**Telegram Utility (`utillity/telegram.js`):**
- `sendTelegramNotification(parentEmail, message)`: Send notification via Telegram

**Cron Monitor (`utillity/cronMonitor.js`):**
- Scheduled tasks for periodic operations (implementation details not visible)

### Frontend Dashboard Components

#### 1. Application Structure

**Entry Point (`main.tsx`):**
- Renders `App` component
- Sets up React root

**App Component (`App.tsx`):**
- Configures React Query client
- Sets up routing with React Router
- Provides theme and toast context

**Routes:**
- `/` - Landing page (Index)
- `/dashboard` - Main dashboard (Dashboard)
- `/login` - Parent login (signin)
- `/signup` - Parent registration (signup)
- `/add-child` - Create child profile (AddChild)
- `/contact` - Contact page (Contact)
- `*` - 404 page (NotFound)

#### 2. Page Components

**Dashboard (`pages/Dashboard.tsx`):**
- Main parent dashboard with multiple views
- Manages selected child state
- Fetches parent and children data on mount
- Displays statistics, activity monitor, and controls
- Handles report generation
- Manages notifications and settings menu

**AddChild (`pages/AddChild.tsx`):**
- Form for creating child profiles
- Generates and displays extension token
- Provides token copy functionality

**Signin/Signup (`pages/signin.tsx`, `pages/signup.tsx`):**
- Authentication forms
- JWT token storage in localStorage
- Redirect to dashboard on success

#### 3. Dashboard Sub-Components

**DashboardStats (`components/DashboardStats.tsx`):**
- Displays 4 key metrics: Screen Time, Alerts, Blocked Content, Active Sessions
- Fetches data every 5 seconds
- Shows percentage changes

**ActivityMonitor (`components/ActivityMonitor.tsx`):**
- Timeline of browsing activities
- Time frame filtering (today, yesterday, week, month)
- Displays searches and website visits
- Auto-refreshes every 5 seconds

**DashboardPreview (`components/DashboardPreview.tsx`):**
- Visual preview of activity data
- Charts and graphs (implementation details not fully visible)

**DashboardControls (`components/DashboardControls.tsx`):**
- View switcher (overview, devices, protect, profiles)

**DeviceList (`components/DeviceList.tsx`):**
- List of monitored devices
- Device status indicators

**ProtectionStatus (`components/ProtectionStatus.tsx`):**
- Shows protection active/inactive status

**UserProfile (`components/UserProfile.tsx`):**
- Parent profile information
- Edit profile functionality

#### 4. UI Components

**Navbar (`components/Navbar.tsx`):**
- Top navigation bar
- Links to main sections

**Footer (`components/Footer.tsx`):**
- Footer with links and information

**Button (`components/Button.tsx`):**
- Reusable button component with variants

**Toast (`hooks/use-toast.ts`):**
- Toast notification system

#### 5. Utilities

**PDF Generator (`utils/pdfGenerator.ts`):**
- `generatePDFReport()`: Creates PDF with activity data
- Includes usage stats, alerts, blocked content, and timeline

### Browser Extension Components

#### 1. Manifest Configuration (`manifest.json`)

**Permissions:**
- `tabs`: Access to tab information
- `activeTab`: Access to currently active tab
- `scripting`: Execute scripts in pages
- `storage`: Store extension token
- `<all_urls>`: Access all websites

**Components:**
- Background service worker: `urlMonitoring.js`
- Content script: `content.js` (runs on all pages)
- Popup: `popup.html` with `popup.js`
- Incognito: `spanning` mode (works in incognito)

#### 2. Background Service Worker (`urlMonitoring.js`)

**Responsibilities:**
- Monitor active tab changes
- Track URL visits and send to backend
- Detect incognito window creation
- Check URLs against blocked list
- Send heartbeat signals
- Handle extension lifecycle events

**Key Functions:**
- `updateActiveTabToBackend()`: Send current URL to API every 60 seconds
- `checkUrlWithBackend(domain)`: Verify if URL is blocked
- `handleTab(tabId, url)`: Process tab navigation
- `alertIncognitoOpen(url)`: Report incognito detection
- `sendHeartbeat()`: Maintain online status
- `notifyLifecycleEvent(endpoint)`: Report activation/disconnection

**Event Listeners:**
- `chrome.windows.onCreated`: Detect incognito windows
- `chrome.tabs.onUpdated`: Track URL changes
- `chrome.tabs.onActivated`: Track tab switches
- `chrome.runtime.onStartup`: Browser start
- `chrome.runtime.onInstalled`: Extension install/update

**Intervals:**
- URL tracking: Every 60 seconds
- Heartbeat: Every 60 seconds

#### 3. Content Script (`content.js`)

**Responsibilities:**
- Filter offensive text in web pages
- Blur offensive images
- Optionally classify images with NSFW API

**Key Features:**
- Maintains list of 500+ offensive words (English and Hindi)
- Uses MutationObserver for dynamic content
- Regex-based text filtering
- Alt-text based image filtering
- Optional external NSFW classification

**Key Functions:**
- `cleanText(node)`: Replace offensive words with asterisks
- `blurOffensiveImages()`: Apply blur filter to flagged images
- `classifyImage(img)`: Send image to NSFW API for analysis
- `traverseDOM(root)`: Recursively scan DOM tree

**CSS Injection:**
```css
.blurred-safe {
  filter: blur(15px) !important;
  transition: filter 0.3s ease;
}
```

#### 4. Popup Interface (`popup.html`, `popup.js`)

**Responsibilities:**
- Display extension status
- Allow token input
- Show quick stats
- Provide settings access

## Data Models

### Parent Data Model

```typescript
interface Parent {
  _id: ObjectId;
  name: string;
  email: string;  // unique
  password: string;  // plain text (security issue)
  children: ObjectId[];  // references to Child documents
  telegramChatId?: string;
}
```

**Relationships:**
- One-to-many with Child (one parent has many children)

**Indexes:**
- Unique index on `email`

### Child Data Model

```typescript
interface Child {
  _id: ObjectId;
  name: string;
  email: string;  // unique
  extensionToken: string;  // UUID v4
  blockedUrls: string[];
  monitoredUrls: MonitoredUrl[];
  incognitoAlerts: IncognitoAlert[];
  lastHeartbeat: Date;
  status: 'online' | 'offline';
}

interface MonitoredUrl {
  url: string;
  domain: string;
  category: string;
  dailyTimeSpent: Map<string, number>;  // ISO date string -> seconds
  searchQueries: string[];
  lastUpdated: Date;
}

interface IncognitoAlert {
  url: string;
  timestamp: Date;
}
```

**Relationships:**
- Many-to-one with Parent (many children belong to one parent)

**Indexes:**
- Unique index on `email`

**Data Patterns:**
- `dailyTimeSpent` uses Map for efficient date-based lookups
- `searchQueries` array prevents duplicates via code logic
- `monitoredUrls` is an embedded array (not normalized)

### Log Data Model

```typescript
interface Log {
  _id: ObjectId;
  child: ObjectId;  // reference to Child
  type: string;  // 'BLOCKED_URL', 'INCOGNITO_ALERT'
  domain: string;
  timestamp: Date;
  message: string;
}
```

**Usage:**
- Currently defined but not actively used in the codebase
- Intended for audit trail and historical logging

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Authentication Token Validity

*For any* authenticated request to the Backend_API, the JWT_Token must be valid, unexpired, and contain a registered parent email, otherwise the request should be rejected with 401 status.

**Validates: Requirements 1.6**

### Property 2: Child Profile Uniqueness

*For any* child profile creation request, if a child with the same email already exists, the Backend_API should reject the request and return an error, ensuring email uniqueness across all child profiles.

**Validates: Requirements 2.1**

### Property 3: Time Tracking Accumulation

*For any* sequence of URL monitoring requests for the same domain on the same date, the total Daily_Time_Spent should equal the sum of all timeSpent values sent in those requests.

**Validates: Requirements 4.3, 4.4, 4.5**

### Property 4: Blocked URL Enforcement

*For any* URL that exists in a child's blockedUrls array, when the Browser_Extension checks that URL, the Backend_API should return blocked status as true, and the extension should close the tab.

**Validates: Requirements 7.5, 7.6, 7.7, 7.8**

### Property 5: Incognito Detection Completeness

*For any* incognito window creation event detected by the Browser_Extension, an incognito alert should be stored in the child's incognitoAlerts array and the window should be immediately closed.

**Validates: Requirements 6.1, 6.2, 6.3, 6.4**

### Property 6: Search Query Deduplication

*For any* search query sent to the Backend_API for a specific domain, if that exact query already exists in the searchQueries array for that Monitored_URL, it should not be added again.

**Validates: Requirements 5.4**

### Property 7: Heartbeat Status Consistency

*For any* child profile, when a heartbeat is received, the status should be set to "online" and lastHeartbeat should be updated to the current timestamp; when disconnection is signaled, status should be set to "offline".

**Validates: Requirements 15.1, 15.2, 15.3**

### Property 8: Activity Timeline Ordering

*For any* filtered activity request, the returned activities should be sorted by timestamp in descending order (newest first), ensuring chronological consistency in the activity monitor.

**Validates: Requirements 9.3**

### Property 9: Report Data Completeness

*For any* PDF report generation request, the report should include all four data categories (web usage, alerts, blocked content, and activity timeline) for the selected child, or fail with an error if any data fetch fails.

**Validates: Requirements 11.2, 11.3, 11.4, 11.5, 11.6, 11.7**

### Property 10: Content Filtering Idempotence

*For any* web page content, applying the offensive word filter multiple times should produce the same result as applying it once, ensuring consistent filtering regardless of DOM mutations.

**Validates: Requirements 8.2, 8.3, 8.4**

### Property 11: Parent-Child Association Integrity

*For any* child profile, the child's _id should exist in exactly one parent's children array, maintaining referential integrity between parents and children.

**Validates: Requirements 2.2**

### Property 12: Daily Time Reset

*For any* Monitored_URL, the Daily_Time_Spent for a specific date should only accumulate time for visits on that date, and should not affect time counts for other dates.

**Validates: Requirements 4.3, 4.5**

## Error Handling

### Backend Error Handling

**Authentication Errors:**
- 401 Unauthorized: Invalid or missing JWT token
- 404 Not Found: Parent or child not found
- 400 Bad Request: Invalid credentials or duplicate email

**Data Validation Errors:**
- 400 Bad Request: Missing required fields
- 500 Internal Server Error: Database connection failures

**Error Response Format:**
```javascript
{
  message: string,
  error?: string  // optional error details
}
```

**Current Issues:**
- Passwords stored in plain text (bcrypt code commented out)
- Inconsistent error handling across controllers
- Some endpoints lack proper error responses

### Frontend Error Handling

**API Request Errors:**
- Toast notifications for failed requests
- Console logging for debugging
- Graceful degradation (show "0m" for missing data)

**Authentication Errors:**
- Redirect to login on 401
- Clear token on logout

**User Feedback:**
- Toast messages for success/error states
- Loading states during async operations
- Error boundaries (not explicitly implemented)

### Extension Error Handling

**Network Errors:**
- Console logging for failed API requests
- Silent failures (no user notification)
- Retry logic not implemented

**Permission Errors:**
- Incognito access check every 60 seconds
- Console logging when permissions missing

**Content Filtering Errors:**
- Try-catch blocks around image classification
- Silent failures for NSFW API errors

## Testing Strategy

### Current Testing Status

**Backend:**
- No automated tests implemented
- `npm test` exits with error
- Manual testing via API clients

**Frontend:**
- No automated tests implemented
- Manual testing in browser

**Extension:**
- No automated tests implemented
- Manual testing in Chrome

### Recommended Testing Approach

**Unit Testing:**
- Backend controllers: Test individual endpoint logic
- Frontend components: Test rendering and user interactions
- Utility functions: Test JWT generation, PDF creation, etc.

**Property-Based Testing:**
- Test correctness properties with randomized inputs
- Use fast-check (JavaScript) or similar PBT library
- Minimum 100 iterations per property test
- Tag each test with property number and feature name

**Integration Testing:**
- Test API endpoints with real database
- Test frontend-backend communication
- Test extension-backend communication

**End-to-End Testing:**
- Test complete user flows (signup → add child → monitor activity)
- Test extension installation and activation
- Test report generation

**Security Testing:**
- Test JWT validation and expiration
- Test SQL injection prevention (MongoDB)
- Test XSS prevention in content filtering
- Test CORS configuration

### Testing Priorities

1. **Critical Path Testing:**
   - Parent authentication
   - Child profile creation
   - URL monitoring and time tracking
   - URL blocking enforcement

2. **Property-Based Testing:**
   - Implement tests for all 12 correctness properties
   - Focus on data integrity properties (time accumulation, deduplication)

3. **Security Testing:**
   - Fix password hashing (enable bcrypt)
   - Test authentication bypass attempts
   - Test injection attacks

4. **Performance Testing:**
   - Test with large numbers of monitored URLs
   - Test with high-frequency URL updates
   - Test dashboard rendering with large datasets

### Test Configuration

**Property-Based Tests:**
- Library: fast-check (for JavaScript/TypeScript)
- Iterations: 100 minimum per test
- Tag format: `// Feature: cipherguard-documentation, Property N: [property description]`

**Unit Tests:**
- Framework: Jest (recommended)
- Coverage target: 80% for critical paths
- Mock external dependencies (MongoDB, Telegram API)

**Integration Tests:**
- Framework: Supertest (for API testing)
- Test database: Separate MongoDB instance
- Cleanup: Reset database between tests

## Security Considerations

### Current Security Issues

1. **Password Storage:**
   - Passwords stored in plain text
   - bcrypt hashing code is commented out
   - **Risk:** Database breach exposes all passwords
   - **Fix:** Enable bcrypt hashing immediately

2. **JWT Secret:**
   - Stored in environment variable (good)
   - No token expiration configured
   - **Risk:** Stolen tokens valid indefinitely
   - **Fix:** Add expiration time to JWT generation

3. **CORS Configuration:**
   - Allows ngrok tunnels (development convenience)
   - **Risk:** Potential for unauthorized access
   - **Fix:** Restrict CORS in production

4. **Extension Token:**
   - UUID v4 (good randomness)
   - No expiration or rotation
   - **Risk:** Stolen token grants permanent access
   - **Fix:** Implement token rotation

5. **Input Validation:**
   - Limited validation on API endpoints
   - **Risk:** Injection attacks, data corruption
   - **Fix:** Add comprehensive input validation

### Recommended Security Enhancements

1. **Enable Password Hashing:**
   ```javascript
   const hashedPassword = await bcrypt.hash(password, 10);
   ```

2. **Add JWT Expiration:**
   ```javascript
   jwt.sign({ email }, JWT_SECRET, { expiresIn: '24h' })
   ```

3. **Implement Rate Limiting:**
   - Limit login attempts
   - Limit API requests per minute

4. **Add Input Sanitization:**
   - Validate email formats
   - Sanitize URL inputs
   - Validate time values

5. **Implement HTTPS:**
   - Require HTTPS in production
   - Use secure cookies for tokens

6. **Add CSRF Protection:**
   - Implement CSRF tokens for state-changing operations

7. **Audit Logging:**
   - Use Log model for security events
   - Track failed login attempts
   - Track URL blocking changes

## Performance Considerations

### Current Performance Characteristics

**Backend:**
- MongoDB queries not optimized
- No caching layer
- No pagination on list endpoints
- Synchronous processing of requests

**Frontend:**
- Polling every 5 seconds for updates
- No data caching (React Query configured but not fully utilized)
- Full page re-renders on state changes

**Extension:**
- URL updates every 60 seconds
- DOM scanning on every mutation
- No throttling or debouncing

### Performance Optimization Recommendations

1. **Backend Optimizations:**
   - Add database indexes on frequently queried fields (email, extensionToken)
   - Implement pagination for children and activity lists
   - Add Redis caching for frequently accessed data
   - Use aggregation pipelines for statistics

2. **Frontend Optimizations:**
   - Implement proper React Query caching
   - Use WebSocket for real-time updates instead of polling
   - Implement virtual scrolling for long activity lists
   - Lazy load dashboard components

3. **Extension Optimizations:**
   - Debounce DOM mutation observer
   - Throttle URL update requests
   - Cache blocked URL list locally
   - Batch multiple URL updates

4. **Database Optimizations:**
   - Create compound indexes for common queries
   - Use MongoDB aggregation for statistics
   - Implement data archiving for old activity logs
   - Consider time-series collection for activity data

5. **Network Optimizations:**
   - Implement request batching
   - Use compression for API responses
   - Implement CDN for frontend assets
   - Use HTTP/2 for multiplexing

## Deployment and Operations

### Environment Configuration

**Backend Environment Variables:**
```
MONGO_URI=<mongodb-connection-string>
JWT_SECRET=<secret-key>
PORT=5000
TELEGRAM_BOT_TOKEN=<telegram-token>
TELEGRAM_CHAT_ID=<chat-id>
```

**Frontend Environment Variables:**
```
VITE_BACKENDURL=<backend-api-url>
```

### Deployment Steps

1. **Backend Deployment:**
   - Install Node.js dependencies
   - Configure environment variables
   - Start MongoDB instance
   - Run `npm start` or `npm run dev`

2. **Frontend Deployment:**
   - Install Node.js dependencies
   - Configure backend URL
   - Build production bundle: `npm run build`
   - Serve static files from `dist/` directory

3. **Extension Deployment:**
   - Update `BACKEND_URL` in `urlMonitoring.js`
   - Load unpacked extension in Chrome
   - Distribute to child devices

### Monitoring and Logging

**Current State:**
- Console logging throughout codebase
- No structured logging
- No monitoring dashboards
- No error tracking service

**Recommendations:**
- Implement structured logging (Winston, Pino)
- Add application monitoring (New Relic, Datadog)
- Implement error tracking (Sentry)
- Add uptime monitoring
- Create operational dashboards

### Backup and Recovery

**Current State:**
- No automated backups
- No disaster recovery plan

**Recommendations:**
- Implement MongoDB automated backups
- Test restore procedures
- Document recovery steps
- Implement data export functionality

## Future Enhancements

### Planned Features (Based on UI Placeholders)

1. **Time Restrictions:**
   - School hours blocking
   - Bedtime restrictions
   - Weekend limits

2. **AI Protection Levels:**
   - Configurable image analysis sensitivity
   - Text analysis levels
   - Behavioral analysis

3. **Device Management:**
   - Multiple device support per child
   - Device-specific settings
   - Cross-device synchronization

4. **Advanced Analytics:**
   - Trend analysis
   - Category-based insights
   - Comparison reports

5. **Mobile App:**
   - iOS/Android parent app
   - Push notifications
   - Mobile-optimized dashboard

### Technical Debt

1. **Password Security:** Enable bcrypt hashing
2. **Error Handling:** Standardize error responses
3. **Testing:** Implement comprehensive test suite
4. **Documentation:** Add API documentation (Swagger/OpenAPI)
5. **Code Quality:** Add linting and formatting rules
6. **Type Safety:** Add TypeScript to backend
7. **Database Migrations:** Implement schema versioning
8. **API Versioning:** Add version prefix to routes

### Scalability Considerations

1. **Horizontal Scaling:**
   - Stateless backend design (good)
   - Need session management for multi-instance deployment
   - Consider message queue for notifications

2. **Database Scaling:**
   - MongoDB sharding for large datasets
   - Read replicas for query performance
   - Consider time-series database for activity logs

3. **Caching Strategy:**
   - Redis for session storage
   - Cache frequently accessed child profiles
   - Cache blocked URL lists

4. **CDN Integration:**
   - Serve frontend assets via CDN
   - Cache static resources
   - Reduce server load
