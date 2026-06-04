
## 1. Introduction

This document defines the requirements for **Blog Agent**, a web-based blogging platform that enables authors to create, manage, and publish content, while providing readers with a clean, searchable browsing experience. It is the authoritative reference for design, development, testing, and acceptance decisions throughout the project lifecycle.

---

## 2. Goals & Objectives

### 2.1 Primary Goals
- Provide a secure, role-based authoring environment for creating and publishing blog posts
- Deliver a fast, accessible public-facing blog for readers
- Support rich-media content including images and formatted text
- Ensure maintainability and extensibility for future enhancements

### 2.2 Non-Goals (v1)
- No real-time collaboration or live editing
- No newsletter or subscription delivery system
- No multi-language (i18n) support
- No third-party social login

---

## 3. Target Audience

### 3.1 Primary Users
- **Authors / Admins**: Content creators who write, edit, and publish posts; manage categories, tags, and media
- **Readers**: Public visitors who browse, search, and read published posts

### 3.2 Secondary Users
- **Platform Administrators**: Responsible for deployment, monitoring, and user management

### 3.3 Assumptions About Users
- Authors are familiar with basic web-based content management
- Readers access the platform using modern browsers on desktop and mobile devices

---

## 4. Functional Requirements

### 4.1 Authentication & User Management
- Users can register with email and password
- Email verification required before first login
- Users can log in and log out securely
- Password reset via email link
- Admins can manage user accounts (create, deactivate, change roles)

### 4.2 Post Management
- Authors can create, edit, save as draft, publish, and archive posts
- Posts support rich-text formatting (headings, bold, italic, lists, code blocks, links)
- Posts can be assigned to one or more categories and tags
- Posts have a slug auto-generated from the title (editable)
- Soft delete: archived posts are retained and restorable by admins
- `reading_time` is computed server-side at ~200 words per minute and stored in minutes as an integer on save

### 4.3 Media Management
- Authors can upload images (JPEG, PNG, WebP, GIF)
- On upload, a thumbnail is automatically generated (resized to 300×200 px)
- Uploaded media is stored with metadata: filename, MIME type, size, URL, thumbnail URL, alt text
- Authors can browse and reuse previously uploaded media

### 4.4 Categories & Tags
- Admins can create, edit, and delete categories (hierarchical, with optional parent)
- Authors can create and assign tags to posts
- Public visitors can browse posts by category or tag

### 4.5 Search
- Public visitors can search posts by keyword (title and content)
- Search results are ranked by relevance
- Search accuracy target: > 90% (see §10.1)

### 4.6 Comments
- Visitors can post comments on published posts (anonymous or authenticated)
- Comments require moderation (pending → approved / spam)
- Authors and admins can approve, reject, or delete comments

### 4.7 Public Blog
- Published posts are visible to all visitors without login
- Posts display: title, author name, published date, reading time, categories, tags, and content
- Pagination or infinite scroll for post listings

---

## 5. Non-Functional Requirements

### 5.1 Performance
- Page load time < 2 seconds (see §10.1)
- API responses < 500 ms at p95 under 100 concurrent users (see §8.4)

### 5.2 Security
- All passwords stored as bcrypt hashes
- CSRF protection on all state-changing endpoints
- XSS prevention via input sanitisation and output escaping
- JWT-based stateless authentication with short-lived access tokens
- HTTPS enforced in all environments

### 5.3 Accessibility
- WCAG 2.1 AA compliance
- Mobile-responsive layout supporting screens ≥ 320 px wide

### 5.4 Reliability
- Target uptime: 99.9% (see §10.1)

### 5.5 Scalability
- Stateless API design to support horizontal scaling
- CDN-ready static asset delivery

---

## 6. Technology Stack

> [TODO: stakeholder input needed — confirm final stack choices before implementation begins]

### 6.1 Proposed Stack
| Layer | Technology |
|-------|-----------|
| Backend API | Node.js / Express.js (see `ARCHITECTURE.md`) |
| Frontend | React SPA (see `ARCHITECTURE.md`) |
| Database | PostgreSQL (see `ARCHITECTURE.md`) |
| File Storage | Azure Blob Storage + Azure CDN (see `ARCHITECTURE.md`) |
| Authentication | JWT (short-lived access token + httpOnly refresh cookie) |
| Cache | [TODO: stakeholder input needed — e.g. Redis / in-memory] |
| Hosting | Azure (App Service, Static Web Apps, Bicep IaC — see `ARCHITECTURE.md`) |

---

## 7. Database Schema (High-Level)

### 7.1 Users Table
- id (PK)
- username
- email
- password_hash
- display_name
- profile_image
- bio
- role (admin, author, subscriber)
- email_verified_at
- created_at
- updated_at

### 7.2 Posts Table
- id (PK)
- user_id (FK)
- title
- slug
- content
- excerpt
- featured_image
- status (draft, published, archived)
- reading_time (integer, minutes — computed at ~200 wpm on save)
- view_count
- published_at
- created_at
- updated_at
- deleted_at (soft delete — Posts only; all other tables use hard delete)

### 7.3 Categories Table
- id (PK)
- name
- slug
- description
- parent_id (self-referencing FK)
- created_at

### 7.4 Tags Table
- id (PK)
- name
- slug
- created_at

### 7.5a Post_Category (Pivot Table)
- post_id (FK → posts)
- category_id (FK → categories)

### 7.5b Post_Tag (Pivot Table)
- post_id (FK → posts)
- tag_id (FK → tags)

### 7.6 Media Table
- id (PK)
- user_id (FK)
- filename
- original_name
- mime_type
- size
- url
- thumbnail_url (auto-generated on upload, 300×200 px — see §4.3)
- alt_text
- created_at

### 7.7 Comments Table
- id (PK)
- post_id (FK)
- user_id (FK → users, nullable — null indicates anonymous comment)
- parent_id (self-referencing FK)
- author_name
- author_email
- content
- status (pending, approved, spam)
- created_at

---

## 8. Testing Requirements

### 8.1 Unit Testing
- Minimum 80% code coverage
- Test all business logic functions
- Test utility functions
- Test data transformation functions

### 8.2 Integration Testing
- API endpoint testing
- Database interaction testing
- Authentication flow testing
- File upload testing

### 8.3 End-to-End Testing
- User registration and login flows
- Post creation and publishing
- Image upload and management
- Public browsing experience
- Search functionality

### 8.4 Performance Testing
- Load testing with 100 concurrent users — acceptance criteria: p95 response time < 500 ms, error rate < 1%
- Image upload stress testing — acceptance criteria: 95% of uploads complete within 10 s for files ≤ 5 MB
- Database query performance testing — all queries must complete in < 100 ms under normal load

### 8.5 Security Testing
- Penetration testing for common vulnerabilities
- OWASP Top 10 security checks
- Authentication and authorization testing
- Input validation testing

---

## 9. Project Milestones

### Phase 1: Foundation (Weeks 1-2)
- Project setup and configuration
- Database schema implementation
- User authentication system
- Basic API structure

### Phase 2: Core Features (Weeks 3-4)
- Blog post CRUD operations
- Rich text editor integration
- Image upload and management
- Public blog viewing

### Phase 3: Enhancement (Weeks 5-6)
- Search functionality
- Categories and tags
- Comments system
- Performance optimization

### Phase 4: Polish (Weeks 7-8)
- UI/UX refinement
- Accessibility improvements
- Testing and bug fixes
- Documentation
- Deployment

---

## 10. Success Metrics

### 10.1 Technical Metrics
- Page load time < 2 seconds
- 99.9% uptime
- Zero critical security vulnerabilities
- Automated test coverage > 80%

### 10.2 User Experience Metrics
- Post creation time < 5 minutes for average post (baseline: ~500 words, 1–2 images)
- Successful image upload rate > 95%
- Search accuracy rate > 90%
- Mobile responsiveness score > 90/100

### 10.3 Business Metrics
- User registration completion rate > 70%
- Average session duration > 3 minutes
- Bounce rate < 50%
- Return visitor rate > 30%

---

## 11. Assumptions and Constraints

### 11.1 Assumptions
- Users have modern browsers with JavaScript enabled
- Minimum screen width of 320px for mobile support
- Users have internet connectivity for all features
- Content is primarily text with supporting images
- Single language support (English); i18n scaffolding is excluded from the v1 codebase
- Deployment targets Belgium (Azure West Europe, single-region) — see `ARCHITECTURE.md`

### 11.2 Constraints
- Initial release focuses on core functionality
- No real-time features in v1
- No multi-language support or i18n scaffolding in v1
- Limited to single author per post
- No subscription or newsletter system in v1
- Budget constraints for third-party services

### 11.3 GDPR Compliance
- User personal data must be deleted within 30 days of account deletion (right to erasure)
- A right-to-erasure API endpoint must be provided for account self-deletion
- Cookie consent banner required before setting any non-essential cookies
- Data processing activities must be documented in a privacy policy accessible from all pages
- No personal data may be transferred outside the EU without appropriate safeguards

---

## 12. Future Enhancements (v2+)
- Social media login integration
- Newsletter subscription
- RSS feed support
- Multi-author collaboration
- Analytics dashboard
- AMP support
- Dark mode
- Multi-language support
- Custom themes
- API for third-party integrations
- Podcast/video content support
- Email notifications for subscribers
- Advanced SEO tools

---

## 13. Appendix

### 13.1 Glossary
- **JWT**: JSON Web Token - A compact, URL-safe means of representing claims between two parties
- **CSRF**: Cross-Site Request Forgery - An attack that forces authenticated users to submit unwanted requests
- **XSS**: Cross-Site Scripting - An injection attack where malicious scripts are injected into trusted websites
- **CDN**: Content Delivery Network - Geographically distributed network of proxy servers
- **WCAG**: Web Content Accessibility Guidelines

### 13.2 References
- OWASP Security Guidelines
- WCAG 2.1 Standards
- GDPR Compliance Requirements
- RESTful API Design Best Practices

---

**Document Version**: 1.1  
**Last Updated**: 2026-06-04  
**Approved By**: [TODO: stakeholder input needed]  
**Next Review Date**: [TODO: stakeholder input needed]