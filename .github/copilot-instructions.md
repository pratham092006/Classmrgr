# AI Copilot Instructions - Room Manager System

## Project Overview
Multi-tier room management system: Flutter mobile app → Express.js backend → React admin panel. Manages classroom schedules, bookings, sessions, and NFC-based access control with role-based access (ADMIN, HOD, STAFF, FACULTY).

## Architecture Decisions

### Three Independent Services
- **Flutter (lib/)**: Client-side app using Riverpod state management, no backend dependency yet
- **Backend (backend/)**: Express.js + MySQL, RESTful API with JWT auth
- **Admin Panel (admin-panel/)**: React 18 + Vite, admin-only web interface

Each has independent build/run commands and .env configuration. They do NOT share code—duplicate patterns when necessary.

### Auth Flow Pattern
- **Backend**: JWT issued on login, 24h expiration, user role/dept in token payload
- **Frontend**: Store token in localStorage (admin) or flutter_secure_storage (mobile)
- **Interceptors**: Auto-logout on 401, redirect to /access-denied on 403, clear token from storage

### State Management
- **Flutter**: Use Riverpod (already in pubspec.yaml)
  - `StateNotifierProvider` for auth (see `AuthController` in `lib/core/auth_controller.dart`)
  - `FutureProvider` for async data with `.autoDispose` for cleanup
  - `FutureProvider.family` for parameterized queries (e.g., room ID)
- **Admin Panel**: React Context API + localStorage (see `useAuth.jsx`), no Redux/Zustand

### Data Fetching
- **Mobile**: Mock API fallback in `lib/core/mock_api.dart` when backend unavailable
- **Admin Panel**: Axios client with base URL from `VITE_API_URL` env, Bearer token in headers
- **Models**: Keep separate JSON serialization in each platform—Flutter uses `.fromJson()` factory, React uses plain objects

## Developer Workflows

### Start All Services (PowerShell)
```powershell
# Terminal 1: Backend
cd backend
npm run dev  # Runs on :4000, requires MySQL running

# Terminal 2: Admin Panel
cd admin-panel
npm run dev  # Runs on :5173

# Terminal 3: Flutter
flutter run -d chrome  # or -d windows for desktop
```

### Database Setup
```powershell
mysql -u root -p < backend/create_test_user.sql
# Creates app_db, users table, test user (admin@test.com / password)
```

### Environment Files
- **Backend**: `backend/.env` (DB_HOST, DB_USER, DB_PASSWORD, JWT_SECRET)
- **Admin Panel**: `admin-panel/.env` (VITE_API_URL=http://localhost:4000/api)
- **Flutter**: No .env needed; uses hardcoded API URLs in `lib/core/api_service.dart`

## Project-Specific Patterns

### API Endpoint Structure (Backend)
Follow modular routing: `modules/{feature}/` contains routes, controller, service
- Route: Handles HTTP, calls controller
- Controller: Request validation, response formatting
- Service: Database queries, business logic

Example: `src/modules/auth/` for `/api/auth/*` endpoints

### Field Name Mismatches
Flutter expects camelCase keys but backend sends snake_case:
- Backend sends `department`, Flutter expects `dept` (see `User.fromJson()`)
- Backend sends `start_time`/`end_time`, Flutter maps to `start`/`end` (see `TimetableSlot`)
- Always use `.fromJson()` constructors to handle mappings

### Role-Based Access
Three-tier system:
- **ADMIN**: Full access, React admin panel only
- **HOD/STAFF/FACULTY**: Mobile app access, view/book rooms
- **401/403 handling**: Auto-logout or redirect; never show error stacks to users

### DateTime Handling
- Backend: Store as ISO 8601 strings or DATETIME columns
- Flutter: Parse with `DateTime.parse()`, timezone-aware
- Admin: Format with `date-fns` library (already in package.json)

## Common Tasks & Examples

### Add New Backend Endpoint
1. Create `src/modules/feature/feature.routes.js` with route handlers
2. Add controller in `feature.controller.js`—queries go to service
3. Service (`feature.service.js`) handles DB with `pool.execute()`
4. Return JSON: `{ data: {...}, error: null }` or throw error caught by middleware

### Add Page to Admin Panel
1. Create `src/pages/NewPage.jsx`
2. Wrap with `<ProtectedRoute>` in `App.jsx` router
3. Fetch with `apiClient` from `src/services/api.js`
4. Use Tailwind (installed) + Lucide icons (already in deps) for UI
5. Handle loading/error states before rendering

### Add Riverpod Provider for Mobile
1. Define in `lib/core/providers.dart` with `.autoDispose` modifier
2. In widget: `final data = ref.watch(myProvider);`
3. For mutations: use `ref.read(authControllerProvider.notifier).myMethod()`
4. Always handle AsyncValue states: `when(data: ..., loading: ..., error: ...)`

## Key Files to Review
- **Architecture**: [lib/core/providers.dart](../lib/core/providers.dart), [backend/src/app.js](../backend/src/app.js), [admin-panel/src/App.jsx](../admin-panel/src/App.jsx)
- **Auth**: [lib/core/auth_controller.dart](../lib/core/auth_controller.dart), [admin-panel/src/hooks/useAuth.jsx](../admin-panel/src/hooks/useAuth.jsx)
- **Routing**: [lib/app.dart](../lib/app.dart), [admin-panel/src/App.jsx](../admin-panel/src/App.jsx)
- **Database**: [backend/src/config/db.js](../backend/src/config/db.js), [backend/create_test_user.sql](../backend/create_test_user.sql)

## Critical Gotchas
- **No code sharing**: Frontend and backend have duplicate types; don't try to unify
- **Mock API fallback**: Flutter's `mockApi` is a development feature; don't build features that depend on it for production
- **JWT token format**: Payload includes `{ sub: userId, email, role, dept }`—use these fields consistently
- **CORS**: Backend enables cors(); ensure admin panel URLs are whitelisted if deploying
- **MySQL pool**: Connections are pooled; always use `pool.execute()`, never direct queries
