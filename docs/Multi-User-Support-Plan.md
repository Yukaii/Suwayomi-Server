# Multi-User Support Planning Document

## Executive Summary

This document outlines the comprehensive plan for implementing multi-user support in Suwayomi-Server, including per-user bookmarks, subscriptions, and library management while maintaining backward compatibility with the existing shared/public library for anonymous users.

**Last Updated:** 2024-10-08  
**Status:** Planning Phase  
**Priority:** High  
**Related Issues:** #623

---

## Table of Contents

1. [Current Architecture Overview](#1-current-architecture-overview)
2. [Problem Statement](#2-problem-statement)
3. [Design Goals](#3-design-goals)
4. [Proposed Architecture](#4-proposed-architecture)
5. [Database Schema Changes](#5-database-schema-changes)
6. [Authentication & Authorization](#6-authentication--authorization)
7. [API Changes](#7-api-changes)
8. [Backward Compatibility Strategy](#8-backward-compatibility-strategy)
9. [Implementation Phases](#9-implementation-phases)
10. [Migration Strategy](#10-migration-strategy)
11. [Testing Strategy](#11-testing-strategy)
12. [Performance Considerations](#12-performance-considerations)
13. [Security Considerations](#13-security-considerations)
14. [Future Enhancements](#14-future-enhancements)
15. [Open Questions](#15-open-questions)

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

1. **User Isolation:** Each user has their own library, bookmarks, categories, and reading progress
2. **Backward Compatibility:** Existing single-user installations continue to work seamlessly
3. **Anonymous Access:** Support for public/shared libraries for unauthenticated users
4. **Migration Path:** Clear upgrade path from single-user to multi-user mode
5. **Performance:** Minimal performance impact for single-user deployments

### 3.2 Secondary Goals

1. **User Management:** Admin interface for creating/managing users
2. **Permission System:** Role-based access control (Admin, User, Guest)
3. **Shared Resources:** Ability to share manga/categories between users
4. **Data Privacy:** User data isolation and privacy controls

### 3.3 Non-Goals (Future Enhancements)

1. Full OAuth2/SAML integration (keep for future)
2. Advanced permission granularity (keep simple initially)
3. User groups/teams
4. Social features (sharing, recommendations)

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
    val role = varchar("role", 32).default("USER") // ADMIN, USER, GUEST
    val isActive = bool("is_active").default(true)
    val createdAt = long("created_at")
    val lastLoginAt = long("last_login_at").default(0)
    val settings = text("settings").default("{}") // JSON for user preferences
}
```

**User Types:**
- `ADMIN` - Full system access, can manage users
- `USER` - Regular user with own library
- `GUEST` - Read-only access to shared library

**Anonymous/Shared Library User:**
- Special user with ID = 0 or username = "anonymous"
- Used for backward compatibility and public access
- When auth mode is NONE, all requests use anonymous user

### 4.2 Library Ownership Model

**Option A: User-Scoped Library (Recommended)**

Each user has their own library entries. Manga can be in multiple users' libraries independently.

```kotlin
object UserMangaLibraryTable : IntIdTable() {
    val user = reference("user_id", UserTable)
    val manga = reference("manga_id", MangaTable)
    val addedAt = long("added_at")
    val customTitle = varchar("custom_title", 512).nullable()
    val isFavorite = bool("is_favorite").default(false)
    
    init {
        uniqueIndex(user, manga)
    }
}
```

**Pros:**
- Clear ownership
- Easy to implement
- Simple queries
- Natural data isolation

**Cons:**
- Potential data duplication (minimal - just relationships)

**Option B: Shared Library with Access Control**

Single library with user access permissions.

**Decision:** Choose Option A for clarity and simplicity.

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
    val user = reference("user_id", UserTable).nullable() // null = global/anonymous
    val name = varchar("name", 64)
    val order = integer("sort_order").default(0)
    val isDefault = bool("is_default").default(false)
    val includeInUpdate = integer("include_in_update").default(0)
    val includeInDownload = integer("include_in_download").default(0)
    
    init {
        // Ensure unique category names per user
        uniqueIndex(user, name)
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

1. **CategoryTable** - Add `user_id` foreign key (nullable for backward compatibility)
2. **TrackRecordTable** - Add `user_id` foreign key
3. **ChapterTable** - Deprecate user-specific fields (keep for migration)
4. **MangaTable** - Deprecate `inLibrary` field (keep for migration)

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
   - Admins can access all libraries (for management)
   - Guests can access anonymous/shared library (read-only)

2. **Manga Operations:**
   - Read access: All users (manga metadata is public)
   - Library add/remove: Owner only
   - Chapter read/bookmark: Owner only

3. **Category Operations:**
   - CRUD: Owner only
   - View: Owner only (categories are private)

4. **Administrative Operations:**
   - User management: Admins only
   - Extension management: Admins only
   - Source configuration: Admins only

### 6.3 Permission Checks

**Enhanced UserType:**

```kotlin
sealed class UserType {
    data class Admin(val id: Int, val username: String) : UserType()
    data class User(val id: Int, val username: String) : UserType()
    data class Guest(val id: Int) : UserType() // Read-only access
    data object Visitor : UserType() // Not authenticated
    data object Anonymous : UserType() // Public/shared library access
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
        is UserType.Guest -> userId == 0 // Only anonymous library
        else -> false
    }
}
```

---

## 7. API Changes

### 7.1 Backward Compatible Endpoints

**Existing endpoints remain unchanged in behavior:**
- When auth mode is NONE, all operations use anonymous user (ID=0)
- When auth mode requires auth, operations use authenticated user's ID
- Default user ID in multi-user mode: authenticated user's actual ID

**Examples:**

```
GET /api/v1/manga/{mangaId}
- No changes - manga metadata is public

POST /api/v1/manga/{mangaId}/library
- Before: Adds to global library
- After: Adds to authenticated user's library (or anonymous if no auth)

GET /api/v1/library
- Before: Returns global library
- After: Returns authenticated user's library
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

## 8. Backward Compatibility Strategy

### 8.1 Anonymous/Shared Library Support

**Default User (ID=0):**
- Username: `"anonymous"` or `"shared"`
- All operations in NONE auth mode use this user
- Existing data migrated to this user during upgrade
- Available for public access even in multi-user mode

### 8.2 Migration from Single-User to Multi-User

**Phase 1: Database Schema Update**
1. Create new tables (User, UserMangaLibrary, UserChapter, etc.)
2. Add foreign key columns to existing tables (nullable initially)
3. Create default "anonymous" user (ID=0)

**Phase 2: Data Migration**
1. Migrate all existing library entries to anonymous user
2. Migrate all reading progress to anonymous user
3. Migrate all categories to anonymous user
4. Update foreign key references

**Phase 3: Application Update**
1. Update business logic to use user context
2. Maintain fallback to user ID=0 when not authenticated
3. Update API endpoints to be user-aware

**Phase 4: Cleanup (Optional)**
1. Remove deprecated fields from old tables
2. Make foreign keys non-nullable (after migration complete)

### 8.3 Configuration Options

**New Server Config:**

```kotlin
// In ServerConfig
val multiUserEnabled = BooleanProperty(
    "server.multiUser.enabled",
    false
)

val defaultUserEnabled = BooleanProperty(
    "server.multiUser.allowAnonymous",
    true
)

val requireAuthForReading = BooleanProperty(
    "server.multiUser.requireAuthForReading",
    false
)
```

**Behavior:**
- `multiUserEnabled = false`: Legacy mode, everything uses user ID=1
- `multiUserEnabled = true, allowAnonymous = true`: Multi-user mode with shared library
- `multiUserEnabled = true, allowAnonymous = false`: Strict multi-user mode

---

## 9. Implementation Phases

### Phase 1: Foundation (4-6 weeks)

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
7. Write unit tests for user management

**Deliverables:**
- User table and management
- API endpoints for user CRUD
- Migration scripts
- Unit tests

### Phase 2: Library Isolation (4-6 weeks)

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

### Phase 3: Reading Progress Isolation (3-4 weeks)

**Goals:**
- Per-user reading progress
- Chapter bookmark isolation
- Progress tracking

**Tasks:**
1. Update Chapter.kt to be user-aware
2. Update ChapterController endpoints
3. Implement data migration for reading progress
4. Update UI clients (documentation)
5. Add tests for reading progress isolation

**Deliverables:**
- User-scoped reading progress
- Updated chapter APIs
- Migration scripts
- Tests

### Phase 4: Categories & Organization (3-4 weeks)

**Goals:**
- Per-user categories
- Category management
- Manga categorization

**Tasks:**
1. Update Category.kt to be user-aware
2. Update category management endpoints
3. Implement category data migration
4. Update GraphQL mutations
5. Add category isolation tests

**Deliverables:**
- User-scoped categories
- Updated category APIs
- Migration scripts
- Tests

### Phase 5: Tracking & External Services (2-3 weeks)

**Goals:**
- Per-user tracking
- External service integration isolation

**Tasks:**
1. Update TrackRecord to be user-aware
2. Update tracking controllers
3. Migrate existing tracking data
4. Test with AniList, MAL, etc.
5. Update documentation

**Deliverables:**
- User-scoped tracking
- Updated tracking APIs
- Tests with external services

### Phase 6: Polish & Production (2-3 weeks)

**Goals:**
- Performance optimization
- Documentation
- Production readiness

**Tasks:**
1. Performance testing and optimization
2. Security audit
3. Update user documentation
4. Update API documentation
5. Create migration guide
6. Beta testing period

**Deliverables:**
- Performance benchmarks
- Complete documentation
- Migration guide
- Production-ready release

**Total Estimated Time: 18-26 weeks (4.5-6.5 months)**

---

## 10. Migration Strategy

### 10.1 Database Migration Approach

**Strategy: Progressive Migration with Rollback Support**

1. **Non-Destructive Migrations:**
   - Add new tables/columns without removing old ones
   - Keep old data intact during migration
   - Use feature flags to toggle new behavior

2. **Data Migration Steps:**

**Step 1: Schema Addition**
```sql
-- M00XX_AddUserTable.kt
CREATE TABLE User (
    id INTEGER PRIMARY KEY,
    username VARCHAR(128) UNIQUE NOT NULL,
    password_hash VARCHAR(256),
    role VARCHAR(32) DEFAULT 'USER',
    is_active BOOLEAN DEFAULT true,
    created_at BIGINT NOT NULL,
    last_login_at BIGINT DEFAULT 0
);

-- Create anonymous user
INSERT INTO User (id, username, role, created_at) 
VALUES (0, 'anonymous', 'GUEST', CURRENT_TIMESTAMP);
```

**Step 2: Create User-Scoped Tables**
```sql
-- M00XX_AddUserMangaLibrary.kt
CREATE TABLE UserMangaLibrary (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES User(id),
    manga_id INTEGER NOT NULL REFERENCES Manga(id),
    added_at BIGINT NOT NULL,
    custom_title VARCHAR(512),
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

**Step 3: Migrate Existing Data**
```sql
-- M00XX_MigrateLibraryToUserLibrary.kt
-- Migrate all manga marked as in_library to anonymous user
INSERT INTO UserMangaLibrary (user_id, manga_id, added_at)
SELECT 0, id, in_library_at
FROM Manga
WHERE in_library = true;

-- M00XX_MigrateChapterProgress.kt
-- Migrate all chapter progress to anonymous user
INSERT INTO UserChapter (user_id, chapter_id, is_read, is_bookmarked, last_page_read, last_read_at)
SELECT 0, id, read, bookmark, last_page_read, last_read_at
FROM Chapter
WHERE read = true OR bookmark = true OR last_page_read > 0;
```

**Step 4: Update Categories**
```sql
-- M00XX_AddUserToCategory.kt
ALTER TABLE Category ADD COLUMN user_id INTEGER REFERENCES User(id);

-- Migrate existing categories to anonymous user
UPDATE Category SET user_id = 0;
```

### 10.2 Application Migration

**Config-Based Feature Toggle:**

```kotlin
object MultiUserMigration {
    private val config = serverConfig
    
    fun isMultiUserEnabled(): Boolean = 
        config.multiUserEnabled.value
    
    fun getUserIdForContext(ctx: Context): Int {
        if (!isMultiUserEnabled()) {
            return 1 // Legacy behavior
        }
        
        val userType = getUserFromContext(ctx)
        return when (userType) {
            is UserType.Admin -> userType.id
            is UserType.User -> userType.id
            UserType.Anonymous -> 0
            else -> 0 // Default to anonymous
        }
    }
}
```

### 10.3 Rollback Plan

**If Migration Fails:**

1. Keep old columns/tables intact
2. Feature flag to disable multi-user mode
3. Fallback queries that use old schema
4. Database backup before migration
5. Migration verification scripts

**Rollback Steps:**
```kotlin
object MigrationRollback {
    fun rollbackToSingleUser() {
        transaction {
            // Merge all user libraries back to global
            exec("UPDATE Manga SET in_library = true WHERE id IN (SELECT manga_id FROM UserMangaLibrary)")
            
            // Merge all reading progress back
            exec("""
                UPDATE Chapter 
                SET read = true 
                WHERE id IN (SELECT chapter_id FROM UserChapter WHERE is_read = true)
            """)
            
            // Clear multi-user tables (optional)
            // exec("DELETE FROM UserMangaLibrary")
            // exec("DELETE FROM UserChapter")
        }
    }
}
```

---

## 11. Testing Strategy

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

## 12. Performance Considerations

### 12.1 Database Indexing

**Required Indexes:**

```sql
-- User table
CREATE INDEX idx_user_username ON User(username);
CREATE INDEX idx_user_role ON User(role);

-- UserMangaLibrary
CREATE INDEX idx_user_manga_library_user ON UserMangaLibrary(user_id);
CREATE INDEX idx_user_manga_library_manga ON UserMangaLibrary(manga_id);
CREATE INDEX idx_user_manga_library_added ON UserMangaLibrary(user_id, added_at);

-- UserChapter
CREATE INDEX idx_user_chapter_user ON UserChapter(user_id);
CREATE INDEX idx_user_chapter_chapter ON UserChapter(chapter_id);
CREATE INDEX idx_user_chapter_read ON UserChapter(user_id, is_read);
CREATE INDEX idx_user_chapter_bookmarked ON UserChapter(user_id, is_bookmarked);

-- Category
CREATE INDEX idx_category_user ON Category(user_id);
CREATE INDEX idx_category_user_order ON Category(user_id, sort_order);

-- UserCategoryManga
CREATE INDEX idx_user_category_manga_user ON UserCategoryManga(user_id);
CREATE INDEX idx_user_category_manga_category ON UserCategoryManga(category_id);
```

### 12.2 Query Optimization

**Common Query Patterns:**

```kotlin
// Efficient library query
fun getUserLibrary(userId: Int): List<Manga> {
    return transaction {
        (MangaTable innerJoin UserMangaLibraryTable)
            .select(MangaTable.columns)
            .where { UserMangaLibraryTable.user eq userId }
            .orderBy(UserMangaLibraryTable.addedAt to SortOrder.DESC)
            .map { MangaTable.toDataClass(it) }
    }
}

// Efficient chapter progress query
fun getChapterProgress(userId: Int, mangaId: Int): List<UserChapterProgress> {
    return transaction {
        (ChapterTable innerJoin UserChapterTable)
            .select(ChapterTable.columns + UserChapterTable.columns)
            .where { 
                (ChapterTable.manga eq mangaId) and 
                (UserChapterTable.user eq userId) 
            }
            .map { row ->
                UserChapterProgress(
                    chapter = ChapterTable.toDataClass(row),
                    isRead = row[UserChapterTable.isRead],
                    isBookmarked = row[UserChapterTable.isBookmarked],
                    lastPageRead = row[UserChapterTable.lastPageRead]
                )
            }
    }
}
```

### 12.3 Caching Strategy

**Cache Layers:**

1. **User Session Cache:**
   - Cache user object for session duration
   - Invalidate on logout or profile update
   - TTL: Session lifetime

2. **Library Cache:**
   - Cache user library manga IDs
   - Invalidate on library add/remove
   - TTL: 5 minutes

3. **Reading Progress Cache:**
   - Cache recently accessed chapter progress
   - Write-through cache for updates
   - TTL: 1 minute

**Implementation:**

```kotlin
object UserCache {
    private val userLibraryCache = CacheBuilder.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .maximumSize(1000)
        .build<Int, Set<Int>>() // userId -> Set of mangaIds
    
    fun getUserLibraryMangaIds(userId: Int): Set<Int> {
        return userLibraryCache.get(userId) {
            transaction {
                UserMangaLibraryTable
                    .select(UserMangaLibraryTable.manga)
                    .where { UserMangaLibraryTable.user eq userId }
                    .map { it[UserMangaLibraryTable.manga].value }
                    .toSet()
            }
        }
    }
    
    fun invalidateUserLibrary(userId: Int) {
        userLibraryCache.invalidate(userId)
    }
}
```

### 12.4 Scalability Considerations

**Vertical Scaling:**
- Database connection pool sizing
- Memory allocation for caches
- Thread pool configuration

**Horizontal Scaling Challenges:**
- Session state management (use Redis or database sessions)
- Cache synchronization across instances
- Distributed locking for concurrent operations

**Database Sharding (Future):**
- Shard by user ID
- Manga metadata in shared database
- User data in user-specific shards

---

## 13. Security Considerations

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

## 14. Future Enhancements

### 14.1 Phase 2+ Features

**User Profile Enhancements:**
- Profile pictures/avatars
- Custom themes per user
- Reading statistics and analytics
- Reading goals and achievements

**Social Features:**
- Follow other users
- Share reading lists
- Recommendations based on similar users
- Comments and reviews

**Advanced Permissions:**
- Fine-grained permissions (read-only access to specific categories)
- Shared libraries (family sharing)
- Guest passes with expiration
- API tokens for third-party clients

**Integration Enhancements:**
- OAuth2/SAML for enterprise SSO
- LDAP/Active Directory integration
- Social login (Google, GitHub, etc.)
- Two-factor authentication (2FA)

### 14.2 API Enhancements

**GraphQL Subscriptions:**
- Real-time library updates
- Live reading progress sync
- Notification system

**Batch Operations:**
- Bulk add to library
- Bulk mark as read
- Batch category assignment

**Advanced Filtering:**
- Search across all users (admin)
- Complex library queries
- Custom sorting and grouping

### 14.3 Mobile App Support

**Offline Sync:**
- Download user library for offline access
- Sync reading progress when online
- Conflict resolution

**Push Notifications:**
- New chapter notifications (per user)
- Library update notifications
- Tracking updates from external services

### 14.4 Performance Enhancements

**Advanced Caching:**
- CDN for manga images
- Pre-computed library views
- Background job for cache warming

**Database Optimization:**
- Read replicas for scaling
- Database partitioning
- Archive old data

---

## 15. Open Questions

### 15.1 Design Decisions Needed

1. **Default User Behavior:**
   - Q: Should new installations create a default admin user or require setup?
   - Options: A) Auto-create admin, B) Require first-run setup, C) Continue with anonymous
   - Recommendation: Require first-run setup for security

2. **Anonymous Library Access:**
   - Q: Should anonymous library be read-only or read-write?
   - Options: A) Read-only, B) Read-write with limitations, C) Configurable
   - Recommendation: Configurable via server config

3. **Migration Timing:**
   - Q: When should migration happen - automatic on upgrade or manual trigger?
   - Options: A) Automatic, B) Manual with prompt, C) Delayed until multi-user enabled
   - Recommendation: Automatic with verification, rollback option

4. **Shared Resources:**
   - Q: Should manga metadata (descriptions, thumbnails) be shared or per-user?
   - Decision: Shared (to save storage and bandwidth)

5. **Category Limits:**
   - Q: Should there be a limit on categories per user?
   - Options: A) Unlimited, B) Soft limit (warn at 50), C) Hard limit
   - Recommendation: Soft limit with warning

### 15.2 Technical Concerns

1. **Performance Impact:**
   - Q: What is acceptable performance degradation for multi-user mode?
   - Target: < 10% overhead for single-user deployments
   - Need: Benchmarking before and after

2. **Database Size:**
   - Q: How much additional storage for multi-user data?
   - Estimate: 100-200 bytes per library entry, 50 bytes per chapter progress
   - For 100 users with 1000 manga each and 50,000 chapters: ~50-100MB

3. **Concurrent Access:**
   - Q: How many concurrent users should be supported?
   - Target: 100 concurrent users on recommended hardware
   - Need: Load testing to verify

### 15.3 User Experience

1. **Migration UX:**
   - Q: How to inform users about migration?
   - Options: A) Automatic with notification, B) Require confirmation, C) Opt-in
   - Recommendation: Automatic with notification and ability to rollback

2. **Admin Interface:**
   - Q: Should user management be in WebUI or API only?
   - Recommendation: Both - API first, then WebUI in later phase

3. **User Registration:**
   - Q: Should users self-register or admin-created only?
   - Options: A) Admin-only, B) Self-registration with approval, C) Self-registration open
   - Recommendation: Configurable, default to admin-only

### 15.4 Compatibility

1. **Client Apps:**
   - Q: Will existing client apps work with multi-user server?
   - Answer: Yes, with backward compatibility mode
   - Action: Document required changes for client app developers

2. **Backup/Restore:**
   - Q: How to handle backups with multiple users?
   - Options: A) Full backup (all users), B) Per-user backup, C) Both
   - Recommendation: Both options available

3. **External Trackers:**
   - Q: How to handle tracker credentials with multiple users?
   - Answer: Per-user tracker authentication (already supported in design)

---

## Conclusion

This planning document provides a comprehensive roadmap for implementing multi-user support in Suwayomi-Server. The implementation is designed to:

1. **Maintain Backward Compatibility:** Existing installations continue to work seamlessly
2. **Provide Clear Migration Path:** Step-by-step migration with rollback capability
3. **Ensure Data Isolation:** Each user's library, progress, and preferences are private
4. **Support Anonymous Access:** Public/shared library for unauthenticated users
5. **Scale Gracefully:** Performance optimization and caching strategies

**Next Steps:**

1. Review and approve this design document
2. Create detailed technical specifications for Phase 1
3. Set up development environment and testing infrastructure
4. Begin implementation of Phase 1 (Foundation)
5. Regular progress reviews and adjustments

**Contributors:**
- Community feedback welcome via GitHub issues
- Technical design review needed before implementation
- UI/UX input needed for admin interface

**References:**
- Issue #623: Multi-user support request
- Current authentication implementation: `UserType.kt`
- Database schema: `server/src/main/kotlin/suwayomi/tachidesk/manga/model/table/`
- Migration framework: `server/src/main/kotlin/suwayomi/tachidesk/server/database/migration/`

---

*This document is a living document and will be updated as the implementation progresses and new requirements emerge.*
