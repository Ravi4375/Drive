# Code Review: student-management-backend

Date: 2026-02-10  
Scope reviewed: `src/main/java`, `src/main/resources`, `src/test/java`

## Executive Summary

The repository contains a minimal Spring Boot file-management backend with upload, download, list, and delete endpoints. The implementation works for basic local development, but it has **critical security and architectural gaps** that should be addressed before production use.

### Overall assessment
- **Code quality:** Fair for prototype, weak for maintainability.
- **Security:** High risk (missing authz/authn, unsafe file handling, exposed metadata, hardcoded credentials).
- **Performance:** Acceptable only for very small datasets; several scaling bottlenecks.
- **Best practices:** Multiple deviations from REST and Spring production conventions.

## Findings by Category

## 1) Security

### 1.1 Missing authentication and authorization (**Critical**)
All file operations are publicly accessible (`/upload`, `/download/{id}`, `/list`, `/delete/{id}`), enabling any caller to read/write/delete files.

**Where:** `FileController` endpoints.

**Recommendation:**
- Add Spring Security with JWT/session auth.
- Enforce ownership/role checks per file record.
- Add method-level authorization guards.

### 1.2 Path traversal and unsafe filename usage (**High**)
`saveFile` directly uses `MultipartFile.getOriginalFilename()` and resolves it into the upload directory. A crafted filename (e.g., `../../etc/passwd`) may attempt directory escape depending on platform/path normalization.

**Where:** `FileServiceStorage.saveFile`.

**Recommendation:**
- Normalize and validate path (`normalize()`, `startsWith(uploadPath)`).
- Replace user-provided name with generated storage key (UUID), keep original as metadata.
- Restrict allowed extensions/content-types.

### 1.3 Hardcoded DB credentials (**High**)
`application.properties` contains plain-text local DB credentials (`root/root`).

**Where:** `src/main/resources/application.properties`.

**Recommendation:**
- Move credentials to environment variables or secrets manager.
- Use profile-specific config (`application-dev.properties`, etc.).

### 1.4 Overly permissive CORS setup (**Medium**)
CORS is fixed to localhost frontend, which is development-specific and not environment-driven.

**Where:** `@CrossOrigin(origins = "http://localhost:3000")`.

**Recommendation:**
- Externalize allowed origins to configuration.
- Prefer global CORS config with env-based policy.

### 1.5 Information leakage through entity exposure (**Medium**)
`list` returns `FileEntity` directly, exposing internal storage path.

**Where:** `FileController.listfiles` and `FileEntity.path` field.

**Recommendation:**
- Use DTOs for API responses.
- Do not return internal filesystem paths.

## 2) Code Quality & Maintainability

### 2.1 Generic exception handling masks real failures (**Medium**)
Controller catches broad `Exception` and returns coarse status codes/messages.

**Where:** upload/download/delete handlers.

**Recommendation:**
- Use domain-specific exceptions.
- Add centralized `@ControllerAdvice` with consistent error schema.

### 2.2 Service contains TODO-level comments and ad-hoc logic (**Low**)
Prototype comments and incomplete exception strategy indicate unfinished production hardening.

**Where:** `FileServiceStorage.saveFile`.

**Recommendation:**
- Remove placeholder comments.
- Convert to explicit validation + exception mapping.

### 2.3 Missing validation constraints (**Medium**)
No validation for empty files, null filenames, huge files, or unsupported content types.

**Recommendation:**
- Add bean validation and multipart limits.
- Reject invalid uploads early.

## 3) Performance & Scalability

### 3.1 In-memory filtering over full table scans (**High**)
`getFilesInFolder` loads all rows via `findAll()` and filters in Java streams. This scales poorly.

**Where:** `FileServiceStorage.getFilesInFolder`.

**Recommendation:**
- Add repository query: `findByParentFolderId(Long parentFolderId)`.
- Add pagination (`Pageable`) for list endpoint.

### 3.2 Blocking file I/O on request thread (**Medium**)
Large uploads/downloads use synchronous local filesystem operations.

**Recommendation:**
- Add size limits and timeouts.
- Consider object storage (S3/MinIO) and streaming patterns.

## 4) Data Integrity & Reliability

### 4.1 Non-transactional file+DB operations can leave inconsistent state (**High**)
If disk write succeeds and DB save fails (or vice versa for delete), metadata and file content diverge.

**Where:** upload/delete flows across service/controller.

**Recommendation:**
- Move delete/upload orchestration into service layer.
- Use transactional semantics where possible and compensating cleanup logic.

### 4.2 `ddl-auto=update` unsafe for production schema control (**Medium**)
Automatic schema mutation risks drift and unexpected changes.

**Where:** `spring.jpa.hibernate.ddl-auto=update`.

**Recommendation:**
- Use Flyway/Liquibase and set safer production policy.

## 5) Testing & DevEx

### 5.1 Test coverage is effectively absent (**High**)
Only a context load test exists.

**Where:** `DriveBeApplicationTests`.

**Recommendation:**
- Add controller tests for upload/download/list/delete (success + failure).
- Add service unit tests for path validation and repository interaction.
- Add integration tests for transactional consistency and error paths.

## Prioritized Action Plan

1. Add authentication/authorization and ownership checks.
2. Fix filename/path handling (sanitize + UUID storage keys + path boundary checks).
3. Replace `findAll` filtering with repository queries + pagination.
4. Introduce DTOs and centralized exception handling.
5. Externalize secrets and adopt environment profiles.
6. Add robust test suite (unit, web, integration).
7. Add migration tooling (Flyway/Liquibase) and review `ddl-auto` usage.

## Suggested API/Design Improvements

- Introduce layered DTO model:
  - `FileUploadResponseDto`
  - `FileListItemDto` (without `path`)
  - `ApiErrorDto`
- Move disk and DB orchestration to service only; keep controller thin.
- Add structured logging with correlation IDs.
- Add rate limiting and request size controls.

## Conclusion

The current code is a workable learning/prototype baseline but not production-ready due to several critical and high-risk findings, especially in security and scalability. Addressing the prioritized items above will significantly improve safety, reliability, and maintainability.
