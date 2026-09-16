`# Kudos System Specification`  
**`**Project:**`** `Datacom Internal Employee Portal`  
**`**Author:**`** `Dilgio Miguel`  
**`**Date:**`** `16 September 2026`  
**`**Version:**`** `1.0 (Approved)`

`---`

`## 1. Functional Requirements`

`### User Stories`

`1. **Authentication:** As a user, I must be logged in to give or view kudos.`  
`2. **Select Colleague:** As a user, I can select another user from a dropdown list of active employees.`  
`3. **Write Message:** As a user, I can write a message of appreciation (max 500 characters).`  
`4. **Submit Kudos:** As a user, I can submit the kudos which is stored in the database.`  
`5. **View Feed:** As a user, I can view a feed of the 20 most recent visible kudos on the dashboard.`  
`6. **Content Moderation:** As an administrator, I can hide or delete inappropriate kudos messages.`  
`7. **Audit Trail:** As an administrator, I can see a log of all moderation actions.`  
`8. **Notifications:** As a recipient, I receive an in-app notification when I get kudos.`

`### Acceptance Criteria`

**`**US-01 Authentication:**`**  
`- Only logged-in users can access kudos features.`  
`- Unauthenticated requests return HTTP 401.`

**`**US-02 Select Colleague:**`**  
`- Dropdown lists only active employees (excludes self).`  
`- List is searchable.`

**`**US-03 Write Message:**`**  
`- Message is required, max 500 characters.`  
`- Empty or whitespace-only messages are rejected.`  
`- HTML is sanitized to prevent XSS.`

**`**US-04 Submit Kudos:**`**  
`- Duplicate submissions (same sender, recipient, message within 1 minute) are blocked.`  
`- Rate limit: max 10 kudos per user per day.`  
`- Returns HTTP 201 on success.`

**`**US-05 View Feed:**`**  
``- Shows 20 most recent kudos where `is_visible = true`.``  
``- Ordered by `created_at DESC`.``  
`- Auto-refreshes every 60 seconds.`  
`- Paginated (20 per page).`

**`**US-06 Content Moderation:**`**  
``- Admin can toggle `is_visible` to hide/unhide a kudos.``  
`- Admin can permanently delete a kudos.`  
`- Hidden kudos are excluded from the public feed.`  
`- Admin-only access (role check).`

**`**US-07 Audit Trail:**`**  
``- Every moderation action is logged in `moderation_log`.``  
`- Log includes: kudos_id, admin_id, action, reason, timestamp.`

**`**US-08 Notifications:**`**  
`- Recipient receives an in-app notification.`  
`- Notification links to the kudos entry.`

`---`

`## 2. Technical Design`

`### 2.1 Database Schema`

**``**Table: `users`**``**  
`| Column | Type | Constraints |`  
`|--------|------|-------------|`  
`| id | INT | PRIMARY KEY, AUTO_INCREMENT |`  
`| name | VARCHAR(100) | NOT NULL |`  
`| email | VARCHAR(150) | UNIQUE, NOT NULL |`  
`| role | ENUM('user','admin') | DEFAULT 'user' |`  
`| is_active | BOOLEAN | DEFAULT TRUE |`  
`| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |`

**``**Table: `kudos`**``**  
`| Column | Type | Constraints |`  
`|--------|------|-------------|`  
`| id | INT | PRIMARY KEY, AUTO_INCREMENT |`  
`| sender_id | INT | FOREIGN KEY → users(id), NOT NULL |`  
`| recipient_id | INT | FOREIGN KEY → users(id), NOT NULL |`  
`| message | VARCHAR(500) | NOT NULL |`  
`| is_visible | BOOLEAN | DEFAULT TRUE |`  
`| moderated_by | INT | FOREIGN KEY → users(id), NULLABLE |`  
`| moderated_at | TIMESTAMP | NULLABLE |`  
`| reason_for_moderation | TEXT | NULLABLE |`  
`| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |`

**``**Table: `moderation_log`**``**  
`| Column | Type | Constraints |`  
`|--------|------|-------------|`  
`| id | INT | PRIMARY KEY, AUTO_INCREMENT |`  
`| kudos_id | INT | FOREIGN KEY → kudos(id) |`  
`| admin_id | INT | FOREIGN KEY → users(id) |`  
`| action | ENUM('hide','unhide','delete') | NOT NULL |`  
`| reason | TEXT | NULLABLE |`  
`| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |`

`### 2.2 API Endpoints`

`| Method | Endpoint | Description | Auth |`  
`|--------|----------|-------------|------|`  
``| POST | `/api/kudos` | Create a new kudos | User |``  
``| GET | `/api/kudos/feed?page=1` | Get public feed (visible only) | User |``  
``| GET | `/api/kudos/all` | Get all kudos (incl. hidden) | Admin |``  
``| PATCH | `/api/kudos/:id/hide` | Hide a kudos | Admin |``  
``| PATCH | `/api/kudos/:id/unhide` | Unhide a kudos | Admin |``  
``| DELETE | `/api/kudos/:id` | Delete a kudos | Admin |``  
``| GET | `/api/users` | List active employees | User |``  
``| GET | `/api/moderation/log` | View moderation log | Admin |``

`### 2.3 Frontend Components`

`| Component | Purpose |`  
`|-----------|---------|`  
``| `KudosForm` | Form to select recipient, write message, submit. |``  
``| `KudosFeed` | Public feed on dashboard, shows visible kudos. |``  
``| `KudosCard` | Individual kudos entry (sender, recipient, message, time). |``  
``| `UserDropdown` | Searchable dropdown of active employees. |``  
``| `AdminModerationPanel` | Admin-only view to hide/delete kudos. |``  
``| `NotificationBell` | In-app notifications for recipients. |``

`### 2.4 Security Considerations`

`- **Authentication:** JWT-based sessions.`  
`- **Authorization:** Role-based (user vs. admin).`  
`- **Input Validation:** Message length (500 chars), HTML sanitization.`  
`- **Rate Limiting:** Max 10 kudos/user/day.`  
`- **CSRF Protection:** Tokens on all POST/PATCH/DELETE.`  
`- **Audit Trail:** All moderation actions logged.`

`### 2.5 Performance Considerations`

`- **Pagination:** Feed returns 20 entries per page.`  
`- **Caching:** Feed cached for 30 seconds.`  
``- **Indexing:** Index on `kudos.created_at` and `kudos.is_visible`.``  
`- **Lazy Loading:** Notifications loaded on demand.`

`### 2.6 Error Handling & Logging`

``- All API errors return structured JSON: `{ "error": "message", "code": 400 }`.``  
`- Errors logged with timestamp, user_id, endpoint.`  
``- Moderation actions logged separately in `moderation_log`.``

`---`

`## 3. Implementation Plan`

`### Phase 1: Backend Setup`  
``1. Create database migrations (`users`, `kudos`, `moderation_log`).``  
`2. Set up authentication middleware (JWT).`  
``3. Implement `/api/users` endpoint.``

`### Phase 2: Kudos API`  
``1. Implement `POST /api/kudos` (create).``  
``2. Implement `GET /api/kudos/feed` (public feed).``  
`3. Add input validation and rate limiting.`

`### Phase 3: Moderation API`  
``1. Implement `PATCH /api/kudos/:id/hide` and `/unhide`.``  
``2. Implement `DELETE /api/kudos/:id`.``  
``3. Implement `GET /api/moderation/log`.``  
`4. Add admin role check.`

`### Phase 4: Frontend`  
``1. Build `KudosForm` and `UserDropdown`.``  
``2. Build `KudosFeed` and `KudosCard`.``  
``3. Build `AdminModerationPanel`.``  
`4. Integrate feed into main dashboard.`

`### Phase 5: Notifications`  
``1. Implement `NotificationBell`.``  
`2. Trigger notification on kudos creation.`

`### Phase 6: Testing`  
`1. Unit tests for API endpoints.`  
`2. Integration tests for moderation flow.`  
`3. UI tests for form submission and feed display.`

`### Phase 7: Deployment`  
`1. Run migrations on staging.`  
`2. Deploy backend and frontend.`  
`3. Monitor for errors and performance.`

`---`

`## 4. Reflection`

**`**Why spec-driven development?**`**  
`By defining requirements and design *before* implementation, we reduce rework and ensure the AI agent builds the right feature. The architect's role is to think strategically — the AI handles the tactical coding.`

**`**Key additions made:**`**  
``- Content moderation (US-06, US-07) with `is_visible`, `moderated_by`, `moderated_at`, `reason_for_moderation`.``  
`- Rate limiting and duplicate prevention to handle spam edge cases.`  
`- Audit trail for accountability.`

**`**Approved by:**`** `Dilgio Miguel`  
**`**Date:**`** `16 September 2026`  
