# Kiraflow — Backend Reference (API · DB · Tech Stack)

# Overview

**Kiraflow** is a lightweight Jira-like backend providing organizations, projects, boards, columns, tasks, epics, labels, comments, members and basic permissions.
This document lists:

* Tech stack and run instructions
* All API endpoints (MVP) with request/response shapes and permissions
* Database tables (fields + types) and relationships (ERD)
* DTOs / important shapes used in requests/responses
* Testing & Postman quick-run checklist
* Notes: Auth, migrations, best practices

---

# Tech stack

* Java 17
* Spring Boot (Spring Web, Spring Data JPA, Spring Security)
* Hibernate 6.x (JPA implementation)
* Maven build (`mvn`)
* MySQL 8.x (database) — UUIDs stored as `CHAR(36)` or `BINARY(16)` (see config)
* HikariCP connection pool
* JWT for authentication (jjwt dependency)
* Lombok for boilerplate (`@Getter`, `@Setter`, etc.)

Recommended dev-only properties:

```properties
spring.jpa.hibernate.ddl-auto=update   # dev only; use Flyway in prod
spring.jpa.show-sql=true
spring.datasource.url=jdbc:mysql://localhost:3306/kiraflow
spring.datasource.username=root
spring.datasource.password=secret
```

---

# Environment / Run

1. Ensure MySQL `kiraflow` database exists. If using `CHAR(36)` UUIDs, create DB and user.
2. Configure `src/main/resources/application.properties` or environment variables:

   * `SPRING_DATASOURCE_URL`
   * `SPRING_DATASOURCE_USERNAME`
   * `SPRING_DATASOURCE_PASSWORD`
   * `JWT_SECRET` (for token signing)
3. Build and run:

```bash
mvn clean package
mvn spring-boot:run
# or run target jar
java -jar target/kiraflow-0.0.1-SNAPSHOT.jar
```

4. Base API: `http://localhost:8081` (adjust `server.port` if changed)

---

# Authentication

* `POST /api/auth/register` — register user
  Body: `{ "name", "email", "password" }`
  Response: user DTO (id, email, name)

* `POST /api/auth/login` — login
  Body: `{ "usernameOrEmail", "password" }`
  Response: `{ "token": "<JWT>" }`
  Use header `Authorization: Bearer <token>` for protected endpoints.

`CurrentUserService` is available server-side to fetch logged-in `User` entity. Permission checks use `PermissionService` (helpers: `requireOrgMember`, `requireProjectMember`, `requireOrgAdminOrOwner`).

---

# API Endpoints (MVP)

Use JSON `Content-Type: application/json`. All IDs are UUID strings unless noted.

> Replace `{{BASE}}` with `http://localhost:8081`.

## Users

* `GET /api/users/me` — current user profile
* `GET /api/users/{id}` — get user by id

## Organization (Auth required)

* `POST /api/organizations` — Create organization
  Body: `{ "name": "OrgName" }`
  Response: `OrganizationDto { id, name, ownerId }`
  Owner automatically becomes org member (`role="owner"`).

* `GET /api/organizations` — List orgs current user is member of

* `GET /api/organizations/{orgId}` — Get org details

* `POST /api/organizations/{orgId}/members` — Add member
  Body: `{ "userId": "<uuid>", "role": "member" }`
  Permission: org owner/admin only.

* `DELETE /api/organizations/{orgId}/members/{userId}` — Remove member

## Project

* `POST /api/projects` — Create project
  Body: `{ "orgId": "<uuid>", "name": "Project", "description": "..." }`
  Permission: org member required.

* `GET /api/projects/{projectId}` — Get project

* `GET /api/projects/org/{orgId}` — List projects in org

## Board

* `POST /api/boards` — Create board `{ "projectId": "<uuid>", "name": "Board A" }`
* `GET /api/boards/{id}`, `GET /api/projects/{projectId}/boards`
* `PUT /api/boards/{id}`, `DELETE /api/boards/{id}`

## Column

* `POST /api/columns` — Create column `{ "boardId": "<uuid>", "title":"To Do", "positionIndex":0 }`
* `GET /api/boards/{boardId}/columns` — List columns
* `PUT /api/columns/{id}`, `DELETE /api/columns/{id}`

## Epic

* `POST /api/epics` — Create epic `{ "projectId","key","title","description" }`
* `GET /api/projects/{projectId}/epics`, `GET /api/epics/{id}`, `PUT /api/epics/{id}`, `DELETE /api/epics/{id}`

## Task

* `POST /api/tasks` — Create task
  Body example:

  ```json
  {
    "columnId":"<uuid>",
    "epicId":"<uuid>|null",
    "assigneeId":"<uuid>|null",
    "type":"TASK",
    "title":"Fix bug",
    "description":"details",
    "storyPoints": 3,
    "status":"TODO",
    "dueDate":"2025-09-20"
  }
  ```

  Response: `TaskDto` (see DTOs)

* `GET /api/tasks/{taskId}` — Get task (includes many joined fields)

* `GET /api/tasks/column/{columnId}` — List tasks in column

* `PUT /api/tasks/{id}` — Update task

* `DELETE /api/tasks/{id}` — Delete

* `POST /api/tasks/{taskId}/move?targetColumnId=<id>&positionIndex=<n>` — Move task

## Labels

* `POST /api/labels` — `{ "orgId":"<uuid>", "name":"bug", "color":"#ff0000" }`
* `GET /api/organizations/{orgId}/labels` — List labels
* `PUT /api/labels/{id}`, `DELETE /api/labels/{id}`

## TaskLabel (attach/detach)

* `POST /api/task-entities/{taskId}/labels?labelId={labelId}` — attach
* `GET /api/task-entities/{taskId}/labels` — list
* `DELETE /api/task-entities/{taskId}/labels/{labelId}` — detach

## Comments

* `POST /api/comments` — `{ "taskId":"<uuid>", "content":"Nice!" }`
  Response: `CommentDto`
* `GET /api/comments/task/{taskId}` — list
* `PUT /api/comments/{id}`, `DELETE /api/comments/{id}`

## Attachments (postponed / S3)

* Endpoints planned for S3 file upload are skippable for MVP. Placeholder paths:

  * `POST /api/attachments/task/{taskId}` (multipart) — returns S3 url
  * `GET /api/attachments/task/{taskId}`
  * `DELETE /api/attachments/{id}`

## Sprints / Versions / Burndown / Activity / Notifications (optional)

* Minimal endpoints can be added later: create/list sprints, assign tasks to sprint, burndown records.

---

# DTOs (common)

Minimal set used in code:

* `OrganizationDto { UUID id, String name, UUID ownerId }`
* `CreateOrganizationRequest { String name }`
* `AddOrgMemberRequest { String userId, String role }` — (we accept String and normalize to UUID)
* `CreateProjectRequest { UUID orgId, String name, String description }`
* `ProjectDto { UUID id, UUID orgId, String name, String description }`
* `TaskDto { UUID id, UUID columnId, UUID epicId, String type, String title, String description, Integer storyPoints, String status, LocalDate dueDate, Instant createdAt, Instant updatedAt }`
* `CreateTaskRequest`, `UpdateTaskRequest` similar shapes.

---

# Database schema (tables & columns)

> This reflects entity fields used in code. Use `CHAR(36)` for UUID text representation or `BINARY(16)` if you prefer binary storage — be consistent.

## users

* `id` CHAR(36) PK
* `name` VARCHAR(255)
* `email` VARCHAR(255) UNIQUE NOT NULL
* `password` VARCHAR(255)  — hashed
* `created_at` TIMESTAMP

## organizations

* `id` CHAR(36) PK
* `name` VARCHAR(255)
* `owner_id` CHAR(36) FK -> users(id)

## org\_members

* Option A (composite):

  * `org_id` CHAR(36) FK -> organizations(id)
  * `user_id` CHAR(36) FK -> users(id)
  * `role` VARCHAR(64)
  * `joined_at` TIMESTAMP
  * PRIMARY KEY (`org_id`,`user_id`)
* Option B (surrogate id):

  * `id` CHAR(36) PK
  * `org_id` CHAR(36)
  * `user_id` CHAR(36)
  * `role` VARCHAR(64)
  * `joined_at` TIMESTAMP

## projects

* `id` CHAR(36) PK
* `org_id` CHAR(36) FK -> organizations(id)
* `name` VARCHAR(255)
* `description` TEXT

## boards

* `id` CHAR(36) PK
* `project_id` CHAR(36) FK -> projects(id)
* `name` VARCHAR(255)

## columns

* `id` CHAR(36) PK
* `board_id` CHAR(36) FK -> boards(id)
* `title` VARCHAR(255)
* `position_index` INT

## epics

* `id` CHAR(36) PK
* `project_id` CHAR(36) FK -> projects(id)
* `key` VARCHAR(64)
* `title` VARCHAR(255)
* `description` TEXT

## sprints

* `id` CHAR(36) PK
* `project_id` CHAR(36) FK -> projects(id)
* `name`, `start_date`, `end_date`, `active` boolean

## versions

* `id` CHAR(36) PK
* `project_id` CHAR(36)
* `name`, `release_date`

## tasks

* `id` CHAR(36) PK
* `column_id` CHAR(36) FK -> columns(id)
* `epic_id` CHAR(36) FK -> epics(id) NULLABLE
* `sprint_id` CHAR(36) FK -> sprints(id) NULLABLE
* `version_id` CHAR(36) FK -> versions(id) NULLABLE
* `assignee_id` CHAR(36) FK -> users(id) NULLABLE
* `parent_task_id` CHAR(36) FK -> tasks(id) NULLABLE
* `type` VARCHAR(64)
* `title` VARCHAR(255)
* `description` TEXT
* `story_points` INT
* `status` VARCHAR(64)
* `due_date` DATE
* `created_at` TIMESTAMP
* `updated_at` TIMESTAMP

## task\_comments

* `id` CHAR(36) PK
* `task_id` CHAR(36) FK -> tasks(id)
* `user_id` CHAR(36) FK -> users(id)
* `content` TEXT
* `created_at` TIMESTAMP

## labels

* `id` CHAR(36) PK
* `org_id` CHAR(36) FK -> organizations(id)
* `name`, `color`

## task\_labels

* `task_id` CHAR(36) FK
* `label_id` CHAR(36) FK
* composite PK (`task_id`,`label_id`)

## attachments

* `id` CHAR(36) PK
* `task_id` CHAR(36) FK
* `uploaded_by` CHAR(36) FK -> users(id)
* `url` TEXT
* `uploaded_at` TIMESTAMP

## activity\_logs

* `id` CHAR(36) PK
* `org_id` CHAR(36)
* `user_id` CHAR(36)
* `task_id` CHAR(36)
* `action_type` VARCHAR(128)
* `metadata` JSON
* `created_at` TIMESTAMP

## notifications

* `id` CHAR(36) PK
* `user_id` CHAR(36)
* `task_id` CHAR(36)
* `payload` JSON
* `read` BOOLEAN
* `created_at` TIMESTAMP

## sprint\_assignments, burndown\_records, task\_dependency — follow ERD shapes

---

# Entity relationships (ERD summary)

* Organization `1..*` Projects
* Project `1..*` Boards, Versions, Epics, Sprints
* Board `1..*` Columns
* Column `1..*` Tasks
* Epic `1..*` Tasks
* Task `1..*` TaskComment, Attachment, TaskLabel
* User `1..*` Task (assignee), Attachment (uploader), TaskComment (author), OrgMember
* Label `1..*` TaskLabel (join)
* OrgMember connects User and Organization (composite or surrogate)

(See top-level textual ERD in the repo for precise fields & FKs.)

---

# Permissions & Security flow 

* Authentication: JWT token -> `JwtAuthFilter` -> set `SecurityContext`.
* `PermissionService` used in services to assert:

  * `requireOrgMember(orgId)` — current user must be in `org_members`.
  * `requireOrgAdminOrOwner(orgId)` — role must be `admin` or `owner`.
  * `requireProjectMember(projectId)` — check membership via org/project chain.
* Enforce permissions at **service** layer (not controller) so any call path is protected.

---

# Postman / Tests

Import the Postman-ready requests (we created earlier) or create the 10 single tests. Run in this order:

1. Register user1
2. Login user1 (save `TOKEN`)
3. Create organization (save `ORG_ID`)
4. List organizations
5. Create project (save `PROJECT_ID`)
6. Create board (save `BOARD_ID`)
7. Create column (save `COLUMN_ID`)
8. Create task (save `TASK_ID`)
9. Create target column + move task
10. (Optional) comments / labels / attach label / comment list

Use the `UuidUtil.parseFlexibleUuid` method in controllers when accepting UUID-like inputs (helps if you accidentally pasted DB hex forms).

---

# Migrations / Production notes

* For production, switch from `spring.jpa.hibernate.ddl-auto=update` to **Flyway** or **Liquibase** migrations. Create SQL migration files for tables described above.
* Be consistent with UUID storage (`CHAR(36)` or `BINARY(16)`) across all tables and MySQL queries.
* Turn off `show-sql` in production.
* Store JWT secret securely (KMS / environment variable).
* Secure S3 access for attachments and serve pre-signed URLs to frontend (do not expose credentials).

---

# Troubleshooting (common issues & fixes)

* **`Association ... targets the type 'org.springframework.scheduling.config.Task'`**
  — likely you named your entity `Task` and Spring imported wrong `Task` class. Rename entity to `TaskEntity` or fully-qualify type in mappings. Avoid class name collisions with `org.springframework` packages.

* **UUID parsing errors (`InvalidFormatException`)**
  — accept `String` in DTO and normalize to UUID in controller using `UuidUtil.parseFlexibleUuid`.

* **Missing table errors (`Table 'kiraflow.epics' doesn't exist`)**
  — run migrations or enable `spring.jpa.hibernate.ddl-auto=update` in dev to auto-create tables.

* **Composite key NPE / `Could not set value of type [java.util.UUID]`**
  — initialize `@EmbeddedId` before persist OR switch to surrogate `@Id` UUID field. If using `@MapsId`, set `embeddedId` explicitly (`new OrgMemberId(orgId,userId)`) before save.

* **White-label error pages / login page on root**
  — ensure security config permits public endpoints (e.g., `/api/auth/**`, `/api/smoke`) and that default Spring Security login page is disabled in API-only app.

---

