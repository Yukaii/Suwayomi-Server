# Multi-User Support Planning Document

## Executive Summary

This document outlines a simplified, focused plan for implementing multi-user support in Suwayomi-Server, enabling per-user bookmarks, subscriptions, and library management. The design prioritizes minimal modifications to the existing system while providing essential multi-user functionality.

**Last Updated:** 2024-10-08  
**Status:** Planning Phase  
**Priority:** High  
**Related Issues:** #623

**Scope:**
- Multi-user support with authenticated users only
- Per-user libraries, bookmarks, and reading progress
- Simplified migration from single-user to multi-user mode
- No anonymous/public library access (users must authenticate)

---

## Table of Contents

1. [Current Architecture Overview](#1-current-architecture-overview)
2. [Problem Statement](#2-problem-statement)
3. [Design Goals](#3-design-goals)
4. [Proposed Architecture](#4-proposed-architecture)
5. [Database Schema Changes](#5-database-schema-changes)
6. [Authentication & Authorization](#6-authentication--authorization)
7. [API Changes](#7-api-changes)
8. [Migration Strategy](#8-migration-strategy)
9. [Implementation Phases](#9-implementation-phases)
10. [Testing Strategy](#10-testing-strategy)
11. [Security Considerations](#11-security-considerations)
12. [Open Questions](#12-open-questions)

---

## 1. Current Architecture Overview

### 1.1 Authentication System

Suwayomi-Server currently supports four authentication modes defined in `AuthMode` enum:

```kotlin
enum class AuthMode {
    NONE,           // No authentication required
    BASIC_AUTH,     // HTTP Basic authentication
    SIMPLE_LOGIN,   // Simple session-based login
    UI_LOGIN,       // JWT-based authentication for UI
}
```

**Current User Model:**
- `UserType.Admin(id: Int)` - Authenticated admin user (currently always ID=1)
- `UserType.Visitor` - Unauthenticated visitor

**Key Limitation:** All authenticated users are treated as a single "admin" user (ID=1), meaning there's no true multi-user support.

### 1.2 Database Schema

#### Core Tables:

**MangaTable:**
- Primary key: `id`
- Key fields: `url`, `title`, `sourceReference`, `inLibrary`, `inLibraryAt`
- **Current Issue:** `inLibrary` is a boolean flag - doesn't support multiple users

**ChapterTable:**
- Primary key: `id`
- Foreign key: `manga` → `MangaTable.id`
- User-specific fields: `isRead`, `isBookmarked`, `lastPageRead`, `lastReadAt`
- **Current Issue:** These fields are global, not per-user

**CategoryTable:**
- Primary key: `id`
- Key fields: `name`, `order`, `isDefault`
- **Current Issue:** Categories are global, not user-specific

**CategoryMangaTable:**
- Junction table linking categories to manga
- **Current Issue:** No user association

**TrackRecordTable:**
- Stores tracking information for external services (AniList, MAL, etc.)
- **Current Issue:** Tracking is global, not per-user

### 1.3 Library Management

Current implementation in `Library.kt`:
- `addMangaToLibrary(mangaId)` - Sets `inLibrary = true`
- `removeMangaFromLibrary(mangaId)` - Sets `inLibrary = false`
- No user context considered

### 1.4 Controllers and API

Current endpoints (from `MangaController.kt`):
- `/api/v1/manga/{id}/library` - Add/remove from library
- `/api/v1/manga/{id}/category` - Manage categories
- All operations use `ctx.getAttribute(Attribute.TachideskUser).requireUser()` which returns a user ID, but currently always 1

---

## 2. Problem Statement

### 2.1 Core Issues

1. **Single User Limitation:** All users share the same library, bookmarks, and reading progress
2. **No User Isolation:** User A's reading progress affects User B's view
3. **Category Sharing:** Categories are global and cannot be personalized
4. **Tracking Confusion:** External tracker integration (AniList, MAL) conflicts when multiple users exist
5. **Privacy Concerns:** Users cannot have private libraries or reading lists

### 2.2 Use Cases Requiring Multi-User Support

1. **Family Sharing:** Multiple family members using the same server instance
2. **Shared Hosting:** Friends sharing a server for cost efficiency
3. **Public Instances:** Community-run servers with multiple users
4. **Per-Device Separation:** Same person wanting different libraries on different devices

---

## 3. Design Goals

### 3.1 Primary Goals

1. **User Isolation:** Each authenticated user has their own library, bookmarks, categories, and reading progress
2. **Minimal Changes:** Reuse existing authentication infrastructure with minimal modifications
3. **Simple Migration:** Clear upgrade path from single-user to multi-user mode
4. **Essential Features:** Focus on core multi-user functionality without complex edge cases

### 3.2 Non-Goals

1. Anonymous/public library access (users must authenticate)
2. Complex backward compatibility modes
3. Advanced permission granularity (keep simple initially)
4. Social features (sharing, recommendations)
5. OAuth2/SAML integration (use existing auth modes)

---

## 4. Proposed Architecture

### 4.1 User Management System

**New User Model:**

```kotlin
// New database table
object UserTable : IntIdTable() {
    val username = varchar("username", 128).uniqueIndex()
    val passwordHash = varchar("password_hash", 256).nullable()
    val email = varchar("email", 256).nullable()
    val role = varchar("role", 32).default("USER") // ADMIN, USER
    val isActive = bool("is_active").default(true)
    val createdAt = long("created_at")
    val lastLoginAt = long("last_login_at").default(0)
}
```

**User Roles:**
- `ADMIN` - Full system access, can manage users
- `USER` - Regular user with own library

**First User Setup:**
- First user created during initial setup is automatically set as ADMIN
- Subsequent users created by admin

### 4.2 Library Ownership Model

**User-Scoped Library:**

Each user has their own library entries. Manga can be in multiple users' libraries independently.

```kotlin
object UserMangaLibraryTable : IntIdTable() {
    val user = reference("user_id", UserTable)
    val manga = reference("manga_id", MangaTable)
    val addedAt = long("added_at")
    val isFavorite = bool("is_favorite").default(false)
    
    init {
        uniqueIndex(user, manga)
    }
}
```

**Benefits:**
- Clear ownership per user
- Simple queries
- Natural data isolation
- Easy to implement

### 4.3 Reading Progress Isolation

**Current State:** Reading progress is stored directly in `ChapterTable`

**Proposed Solution:** Move user-specific data to separate table

```kotlin
object UserChapterTable : IntIdTable() {
    val user = reference("user_id", UserTable)
    val chapter = reference("chapter_id", ChapterTable)
    val isRead = bool("is_read").default(false)
    val isBookmarked = bool("is_bookmarked").default(false)
    val lastPageRead = integer("last_page_read").default(0)
    val lastReadAt = long("last_read_at").default(0)
    
    init {
        uniqueIndex(user, chapter)
    }
}
```

### 4.4 Category System

**User-Scoped Categories:**

```kotlin
object CategoryTable : IntIdTable() {
    val user = reference("user_id", UserTable)
    val name = varchar("name", 64)
    val order = integer("sort_order").default(0)
    val isDefault = bool("is_default").default(false)
    val includeInUpdate = integer("include_in_update").default(0)
    val includeInDownload = integer("include_in_download").default(0)
    
    init {
        uniqueIndex(user, name) // Unique category names per user
    }
}
```

**Category-Manga Junction:**

```kotlin
object UserCategoryMangaTable : IntIdTable() {
    val user = reference("user_id", UserTable)
    val category = reference("category_id", CategoryTable)
    val manga = reference("manga_id", MangaTable)
    
    init {
        uniqueIndex(user, category, manga)
    }
}
```

### 4.5 Tracking System

**User-Scoped Tracking:**

```kotlin
object TrackRecordTable : IntIdTable() {
    val user = reference("user_id", UserTable)
    val manga = reference("manga_id", MangaTable)
    val trackerId = integer("tracker_id")
    val remoteId = long("remote_id")
    val libraryId = long("library_id").nullable()
    // ... other tracking fields
    
    init {
        uniqueIndex(user, manga, trackerId)
    }
}
```

---

## 5. Database Schema Changes

### 5.1 New Tables

1. **UserTable** - Core user management
2. **UserMangaLibraryTable** - Per-user library entries
3. **UserChapterTable** - Per-user reading progress
4. **UserCategoryMangaTable** - Per-user category assignments

### 5.2 Modified Tables

1. **CategoryTable** - Add `user_id` foreign key (required, not nullable)
2. **TrackRecordTable** - Add `user_id` foreign key
3. **ChapterTable** - Keep as-is; user-specific data moves to UserChapterTable
4. **MangaTable** - Keep as-is; library status moves to UserMangaLibraryTable

### 5.3 Migration Tables

Temporary tables for data migration:

```kotlin
// Store old values during migration
object MigrationBackupTable : Table() {
    val tableName = varchar("table_name", 128)
    val recordId = integer("record_id")
    val fieldName = varchar("field_name", 128)
    val oldValue = text("old_value")
    val migratedAt = long("migrated_at")
}
```

---

## 6. Authentication & Authorization

### 6.1 Enhanced Authentication

**JWT Token Structure (for UI_LOGIN mode):**

```json
{
  "sub": "user_id",
  "username": "john_doe",
  "role": "USER",
  "iat": 1234567890,
  "exp": 1234567890
}
```

**Session Management (for SIMPLE_LOGIN mode):**
- Store actual user ID in session instead of hardcoded 1
- Session attribute: `"user_id"` instead of `"logged-in"`

### 6.2 Authorization Model

**Access Control Rules:**

1. **Library Operations:**
   - Users can only access their own library
   - Admins can access all libraries for management purposes

2. **Manga Operations:**
   - Read access: All authenticated users (manga metadata is public)
   - Library add/remove: Owner only

3. **Chapter Operations:**
   - Read/bookmark: Owner only

4. **Category Operations:**
   - CRUD: Owner only
   - Categories are private to each user

5. **Administrative Operations:**
   - User management: Admins only
   - Extension management: All authenticated users
   - Source configuration: All authenticated users

### 6.3 Permission Checks

**Simplified UserType:**

```kotlin
sealed class UserType {
    data class Admin(val id: Int, val username: String) : UserType()
    data class User(val id: Int, val username: String) : UserType()
    data object Visitor : UserType() // Not authenticated
}

fun UserType.requireAdmin(): Int {
    return when (this) {
        is UserType.Admin -> id
        else -> throw ForbiddenException("Admin access required")
    }
}

fun UserType.requireUser(): Int {
    return when (this) {
        is UserType.Admin -> id
        is UserType.User -> id
        else -> throw UnauthorizedException()
    }
}

fun UserType.canAccessLibrary(userId: Int): Boolean {
    return when (this) {
        is UserType.Admin -> true // Admins can access all
        is UserType.User -> this.id == userId
        else -> false
    }
}
```

---

## 7. API Changes

### 7.1 API Behavior

**Endpoint Authentication:**
- All library/reading progress endpoints require authentication
- Manga metadata endpoints (browse, search) available to authenticated users
- No anonymous/public access

**Examples:**

```
GET /api/v1/manga/{mangaId}
- Requires authentication
- Returns manga metadata

POST /api/v1/manga/{mangaId}/library
- Requires authentication
- Adds to authenticated user's library

GET /api/v1/library
- Requires authentication
- Returns authenticated user's library
```

### 7.2 New Multi-User Endpoints

**User Management:**

```
POST /api/v1/admin/users
GET /api/v1/admin/users
GET /api/v1/admin/users/{userId}
PUT /api/v1/admin/users/{userId}
DELETE /api/v1/admin/users/{userId}

GET /api/v1/user/me
PUT /api/v1/user/me
```

**Library Management:**

```
GET /api/v1/users/{userId}/library
GET /api/v1/users/{userId}/library/{mangaId}
POST /api/v1/users/{userId}/library/{mangaId}
DELETE /api/v1/users/{userId}/library/{mangaId}

# For admins to view any user's library
GET /api/v1/admin/users/{userId}/library
```

**Category Management:**

```
GET /api/v1/users/{userId}/categories
POST /api/v1/users/{userId}/categories
PUT /api/v1/users/{userId}/categories/{categoryId}
DELETE /api/v1/users/{userId}/categories/{categoryId}
```

**Reading Progress:**

```
GET /api/v1/users/{userId}/chapters/{chapterId}
PUT /api/v1/users/{userId}/chapters/{chapterId}/progress
PUT /api/v1/users/{userId}/chapters/{chapterId}/bookmark
```

### 7.3 GraphQL Schema Changes

**New Types:**

```graphql
type User {
  id: Int!
  username: String!
  email: String
  role: UserRole!
  isActive: Boolean!
  createdAt: Long!
  lastLoginAt: Long!
  libraryCount: Int!
  settings: UserSettings!
}

enum UserRole {
  ADMIN
  USER
  GUEST
}

type UserSettings {
  theme: String
  language: String
  # ... other preferences
}

type UserLibraryEntry {
  user: User!
  manga: Manga!
  addedAt: Long!
  customTitle: String
  isFavorite: Boolean!
  categories: [Category!]!
}

type UserChapterProgress {
  user: User!
  chapter: Chapter!
  isRead: Boolean!
  isBookmarked: Boolean!
  lastPageRead: Int!
  lastReadAt: Long!
}
```

**New Queries:**

```graphql
type Query {
  # Current user
  me: User
  
  # User management (admin only)
  users: [User!]!
  user(id: Int!): User
  
  # Library queries
  myLibrary: [UserLibraryEntry!]!
  userLibrary(userId: Int!): [UserLibraryEntry!]!
  
  # Chapter progress
  myChapterProgress(chapterId: Int!): UserChapterProgress
  userChapterProgress(userId: Int!, chapterId: Int!): UserChapterProgress
}
```

**New Mutations:**

```graphql
type Mutation {
  # User management
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(input: DeleteUserInput!): DeleteUserPayload!
  
  # Library operations
  addToLibrary(input: AddToLibraryInput!): AddToLibraryPayload!
  removeFromLibrary(input: RemoveFromLibraryInput!): RemoveFromLibraryPayload!
  
  # Reading progress
  updateChapterProgress(input: UpdateChapterProgressInput!): UpdateChapterProgressPayload!
  toggleBookmark(input: ToggleBookmarkInput!): ToggleBookmarkPayload!
}
```

---

## 8. Migration Strategy

### 8.1 Migration from Single-User to Multi-User

**Approach:** Simple migration where existing data is assigned to the first admin user created.

**Step 1: First-Time Setup**
1. On first launch after upgrade, prompt for admin user creation
2. Create admin user account
3. Migrate all existing data to this admin user

**Step 2: Database Schema Update**
```sql
-- M00XX_AddUserTable.kt
CREATE TABLE User (
    id INTEGER PRIMARY KEY,
    username VARCHAR(128) UNIQUE NOT NULL,
    password_hash VARCHAR(256) NOT NULL,
    role VARCHAR(32) DEFAULT 'USER',
    is_active BOOLEAN DEFAULT true,
    created_at BIGINT NOT NULL,
    last_login_at BIGINT DEFAULT 0
);
```

**Step 3: Create User-Scoped Tables**
```sql
-- M00XX_AddUserMangaLibrary.kt
CREATE TABLE UserMangaLibrary (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES User(id),
    manga_id INTEGER NOT NULL REFERENCES Manga(id),
    added_at BIGINT NOT NULL,
    is_favorite BOOLEAN DEFAULT false,
    UNIQUE(user_id, manga_id)
);

-- M00XX_AddUserChapter.kt
CREATE TABLE UserChapter (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES User(id),
    chapter_id INTEGER NOT NULL REFERENCES Chapter(id),
    is_read BOOLEAN DEFAULT false,
    is_bookmarked BOOLEAN DEFAULT false,
    last_page_read INTEGER DEFAULT 0,
    last_read_at BIGINT DEFAULT 0,
    UNIQUE(user_id, chapter_id)
);
```

**Step 4: Migrate Existing Data**
```sql
-- After admin user is created (e.g., ID=1)
-- M00XX_MigrateLibraryToUserLibrary.kt
INSERT INTO UserMangaLibrary (user_id, manga_id, added_at)
SELECT 1, id, in_library_at
FROM Manga
WHERE in_library = true;

-- M00XX_MigrateChapterProgress.kt
INSERT INTO UserChapter (user_id, chapter_id, is_read, is_bookmarked, last_page_read, last_read_at)
SELECT 1, id, read, bookmark, last_page_read, last_read_at
FROM Chapter
WHERE read = true OR bookmark = true OR last_page_read > 0;
```

**Step 5: Update Categories**
```sql
-- M00XX_AddUserToCategory.kt
ALTER TABLE Category ADD COLUMN user_id INTEGER REFERENCES User(id);
UPDATE Category SET user_id = 1; -- Assign to first admin user
ALTER TABLE Category ALTER COLUMN user_id SET NOT NULL;
```

### 8.2 Migration UI Flow

1. **Server Upgrade Detection:**
   - Server detects it needs multi-user migration
   - Locks down access except for migration endpoint

2. **Admin Setup Screen:**
   - Prompt for username, password, email (optional)
   - Create first admin user
   - Trigger data migration

3. **Migration Execution:**
   - Run database migrations
   - Migrate existing data to admin user
   - Complete setup

4. **Post-Migration:**
   - Server resumes normal operation
   - Admin can create additional users
   - All existing data preserved under admin user's account

---

## 9. Implementation Phases

### Phase 1: Foundation (3-4 weeks)

**Goals:**
- Database schema changes
- User management system
- Basic authentication enhancements

**Tasks:**
1. Create migration: Add User table
2. Create migration: Add UserMangaLibrary table
3. Create migration: Add UserChapter table
4. Implement User management business logic
5. Update authentication to support real user IDs
6. Create admin API for user CRUD operations
7. Create first-time setup flow
8. Write unit tests for user management

**Deliverables:**
- User table and management
- API endpoints for user CRUD
- Migration scripts
- First-time setup UI/flow
- Unit tests

### Phase 2: Library Isolation (3-4 weeks)

**Goals:**
- Per-user library support
- Migrate existing data
- Update library APIs

**Tasks:**
1. Update Library.kt to be user-aware
2. Update MangaController endpoints
3. Implement data migration script for libraries
4. Update GraphQL schema for libraries
5. Update existing tests
6. Add integration tests for multi-user libraries

**Deliverables:**
- User-scoped library functionality
- Data migration tools
- Updated API endpoints
- Integration tests

### Phase 3: Reading Progress & Categories (3-4 weeks)

**Goals:**
- Per-user reading progress
- Per-user categories
- Chapter bookmark isolation

**Tasks:**
1. Update Chapter.kt to be user-aware
2. Update ChapterController endpoints
3. Update Category.kt to be user-aware
4. Update category management endpoints
5. Implement data migration for reading progress and categories
6. Add tests for isolation

**Deliverables:**
- User-scoped reading progress and categories
- Updated APIs
- Migration scripts
- Tests

### Phase 4: Tracking & Polish (2-3 weeks)

**Goals:**
- Per-user tracking
- Documentation
- Production readiness

**Tasks:**
1. Update TrackRecord to be user-aware
2. Update tracking controllers
3. Migrate existing tracking data
4. Security audit
5. Update user documentation
6. Create migration guide
7. Beta testing

**Deliverables:**
- User-scoped tracking
- Complete documentation
- Migration guide
- Production-ready release

**Total Estimated Time: 11-15 weeks (2.5-3.5 months)**

---

## 10. Testing Strategy

### 11.1 Unit Tests

**User Management:**
- User CRUD operations
- Password hashing and verification
- Role-based permissions
- User activation/deactivation

**Library Operations:**
- Add manga to user library
- Remove manga from user library
- List user library
- Library isolation between users

**Reading Progress:**
- Mark chapter as read for user
- Bookmark chapter for user
- Update reading position for user
- Progress isolation between users

**Categories:**
- Create user category
- Assign manga to user category
- Category isolation between users

### 11.2 Integration Tests

**Multi-User Scenarios:**
1. Two users with different libraries
2. Same manga in multiple user libraries
3. Different reading progress for same chapter
4. Category name conflicts between users
5. Concurrent operations by multiple users

**Authentication Flow:**
- Login as different users
- Session management
- JWT token validation
- Permission checks

### 11.3 Migration Tests

**Data Migration:**
- Migrate sample database from single-user to multi-user
- Verify data integrity after migration
- Test rollback procedure
- Performance testing on large datasets

**Backward Compatibility:**
- Test with auth mode NONE
- Test with existing client applications
- Verify anonymous library access
- Test API compatibility

### 11.4 Performance Tests

**Load Testing:**
- 1000 users with libraries
- 10,000 manga entries
- 100,000 chapters
- Concurrent API requests

**Query Performance:**
- Library listing performance
- Chapter progress queries
- Category filtering
- Search across users (admin)

**Benchmarks:**
- Compare single-user vs multi-user mode
- Database query optimization
- Index effectiveness
- Cache hit rates

### 11.5 Security Tests

**Authorization:**
- User cannot access other user's library
- Guest cannot modify data
- Admin can access all resources
- Unauthorized API access attempts

**Authentication:**
- Password strength requirements
- Token expiration
- Session hijacking prevention
- CSRF protection

### 11.6 End-to-End Tests

**User Workflows:**
1. New user registration → login → add manga → read chapter
2. Admin creates user → user logs in → manages library
3. Anonymous user accesses shared library
4. User switches devices (different sessions)
5. Concurrent reading by multiple users

---

## 11. Security Considerations

### 13.1 Authentication Security

**Password Storage:**
- Use bcrypt or Argon2 for password hashing
- Minimum 10 rounds for bcrypt
- Salt per password
- Never store plaintext passwords

```kotlin
object PasswordHasher {
    private const val BCRYPT_ROUNDS = 12
    
    fun hashPassword(password: String): String {
        return BCrypt.hashpw(password, BCrypt.gensalt(BCRYPT_ROUNDS))
    }
    
    fun verifyPassword(password: String, hash: String): Boolean {
        return BCrypt.checkpw(password, hash)
    }
}
```

**Token Security:**
- JWT tokens with short expiration (15 minutes for access, 7 days for refresh)
- Secure token storage (HTTP-only cookies or secure storage)
- Token rotation on refresh
- Revocation mechanism for compromised tokens

### 13.2 Authorization Security

**Access Control:**
- Validate user ID in all operations
- Prevent horizontal privilege escalation (user accessing other user's data)
- Admin-only operations properly protected
- Rate limiting on sensitive operations

**SQL Injection Prevention:**
- Use parameterized queries (Exposed framework handles this)
- Validate all user inputs
- Escape special characters

**Example Authorization Check:**

```kotlin
fun requireLibraryAccess(requestingUser: UserType, targetUserId: Int) {
    when (requestingUser) {
        is UserType.Admin -> return // Admins can access all
        is UserType.User -> {
            if (requestingUser.id != targetUserId) {
                throw ForbiddenException("Cannot access another user's library")
            }
        }
        else -> throw UnauthorizedException()
    }
}
```

### 13.3 Data Privacy

**User Data Isolation:**
- Database-level foreign key constraints
- Application-level access checks
- Audit logging for admin access to user data

**Privacy Controls:**
- Users can delete their own data
- Admins can view but should have audit trail
- Export user data (GDPR compliance)

**Sensitive Data:**
- Email addresses (optional, used for recovery)
- Reading history (private to user)
- Bookmarks and favorites (private to user)

### 13.4 Audit Logging

**Log Security Events:**

```kotlin
object SecurityAudit {
    fun logUserLogin(userId: Int, ip: String, success: Boolean) {
        logger.info { "User login: userId=$userId, ip=$ip, success=$success" }
    }
    
    fun logUnauthorizedAccess(userId: Int, resource: String, ip: String) {
        logger.warn { "Unauthorized access attempt: userId=$userId, resource=$resource, ip=$ip" }
    }
    
    fun logAdminAction(adminId: Int, action: String, targetUserId: Int?) {
        logger.info { "Admin action: adminId=$adminId, action=$action, targetUserId=$targetUserId" }
    }
}
```

**Logged Events:**
- User login/logout
- Failed authentication attempts
- Unauthorized access attempts
- Admin operations on user data
- Password changes
- User creation/deletion

---

## 12. Open Questions

### 12.1 Design Decisions Needed

1. **First-Run Setup UX:**
   - Q: Should the setup wizard be required or optional?
   - Recommendation: Required for security - force admin user creation on first launch

2. **Migration Timing:**
   - Q: When should migration happen - automatic on upgrade or manual trigger?
   - Options: A) Automatic on first launch, B) Manual trigger from admin panel
   - Recommendation: Automatic on first launch with clear progress indication

3. **Shared Resources:**
   - Q: Should manga metadata (descriptions, thumbnails) be shared or per-user?
   - Decision: Shared (to save storage and bandwidth)

4. **Category Limits:**
   - Q: Should there be a limit on categories per user?
   - Options: A) Unlimited, B) Soft limit (warn at 50), C) Hard limit
   - Recommendation: Soft limit with warning

### 12.2 Technical Concerns

1. **Database Size:**
   - Q: How much additional storage for multi-user data?
   - Estimate: 100-200 bytes per library entry, 50 bytes per chapter progress
   - For 10 users with 1000 manga each: ~1-2MB additional

2. **Concurrent Access:**
   - Q: How many concurrent users should be supported?
   - Target: 10-20 concurrent users on recommended hardware
   - Need: Load testing to verify

### 12.3 User Experience

1. **Admin Interface:**
   - Q: Should user management be in WebUI or API only?
   - Recommendation: Both - API first, then WebUI in Phase 2

2. **User Registration:**
   - Q: Should users self-register or admin-created only?
   - Decision: Admin-only for simplicity (self-registration can be added later)

### 12.4 Compatibility

1. **Client Apps:**
   - Q: Will existing client apps work with multi-user server?
   - Answer: Apps will need updates to support authentication flows
   - Action: Document required changes for client app developers

2. **Backup/Restore:**
   - Q: How to handle backups with multiple users?
   - Recommendation: Full backup (all users) - per-user backup can be added later

3. **External Trackers:**
   - Q: How to handle tracker credentials with multiple users?
   - Answer: Per-user tracker authentication (already supported in design)

---

## Conclusion

This planning document provides a focused roadmap for implementing multi-user support in Suwayomi-Server. The implementation is designed to:

1. **Minimize Modifications:** Reuse existing authentication and add minimal new tables
2. **Provide Clear Migration Path:** Simple first-run setup with automatic data migration
3. **Ensure Data Isolation:** Each user's library, progress, and preferences are private
4. **Keep It Simple:** Focus on core functionality, avoid complex edge cases

**Next Steps:**

1. Review and approve this design document
2. Create detailed technical specifications for Phase 1
3. Begin implementation of Phase 1 (Foundation)
4. Regular progress reviews and adjustments

**Contributors:**
- Community feedback welcome via GitHub issues
- Technical design review needed before implementation

**References:**
- Issue #623: Multi-user support request
- Current authentication implementation: `UserType.kt`
- Database schema: `server/src/main/kotlin/suwayomi/tachidesk/manga/model/table/`
- Migration framework: `server/src/main/kotlin/suwayomi/tachidesk/server/database/migration/`

---

*This document is a living document and will be updated as the implementation progresses and new requirements emerge.*
