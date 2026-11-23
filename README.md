diff --git a/docs/01_system_overview.md b/docs/01_system_overview.md
index 1f4b653..b01a760 100644
--- a/docs/01_system_overview.md
+++ b/docs/01_system_overview.md
@@ -32,6 +32,8 @@ The NUML Management System is a full-stack web application designed to centraliz
 | **Admin**      | University administrative staff | Full system control, user management, reports                     |
 | **System**     | Automated background processes  | Email notifications, challan generation, analytics updates        |
 
+*The actors above are user categories (the Admin actor is the Main Admin role, with departmental administration delegated as described in §4.4). The system's RBAC roles are listed in §4.1. In this overview "semester" means the academic term (a Fall or Spring session); a student's position in the degree is the semester level (1st to 8th).*
+
 
 ---
 
@@ -44,7 +46,8 @@ The NUML Management System is a full-stack web application designed to centraliz
 ### 4.1 Authentication & Security
 
 - JWT-based stateless authentication
-- Role-Based Access Control (RBAC) — three roles: Student, Instructor, Admin
+- Role-Based Access Control (RBAC) — six system roles: Main Admin, Instructor, HOD, Department Staff, Accounts Staff and Student. Roles are separate from department assignments (HOD, instructor and staff assignments) and from position titles such as Receptionist, which are data, not roles. Permissions come from a central catalogue; department staff receive theirs per assignment.
+- Main Admin has global authority; HODs and delegated department staff act only within the departments their own assignments authorize, and have no global account or system-role management.
 - Secure password hashing (Argon2id)
 - Access token + Refresh token pattern
 - Route-level protection on both frontend and backend
@@ -58,6 +61,7 @@ The NUML Management System is a full-stack web application designed to centraliz
 - View exam marks and calculated grades
 - Download digital fee challans (full or sub-challans)
 - View academic history across semesters
+- View their own profile and change their own phone number; names and other institutional data are managed by the university *(approved Phase 4 policy, not yet implemented)*
 
 
 
@@ -72,8 +76,10 @@ The NUML Management System is a full-stack web application designed to centraliz
 
 ### 4.4 Admin Panel
 
-- Create/update/deactivate user accounts (students, instructors, sub-admins)
-- Assign roles and permissions
+- Global account and system-role management — **Main Admin only**: global account activation and deactivation, institutional name and login-email changes, and system-role assignment (for example appointing an HOD to departments). Role grants come from the central permission catalogue. Department staff accounts are created by HODs inside their own departments (current implementation); that is departmental delegation, not global account management.
+- Student registration — Main Admin, HODs and delegated department staff within their authorized departments *(approved Phase 4 policy, not yet implemented)*.
+- Only Main Admin can globally activate or deactivate accounts. HODs cannot globally deactivate accounts or change user names, login emails, system roles or global account status.
+- Departmental permission delegation follows the central permission catalogue and the department-scoped authorization rules. An HOD delegates only permissions that are on the approved delegation allow-list and that the HOD holds in that department, and cannot create system roles. Delegated staff perform only the operations explicitly delegated to them in the authorized department; they cannot manage accounts or system roles globally.
 - Create semesters, departments, and courses
 - Enroll students into courses
 - Generate and manage fee challans
@@ -96,7 +102,7 @@ The NUML Management System is a full-stack web application designed to centraliz
 
 ### 4.6 Notifications
 
-- Email on account creation (welcome + credentials)
+- Email on account creation: staff receive a temporary password; students receive onboarding instructions only (no password is emailed) and set their password through the Forgot-password flow *(student onboarding is approved Phase 4 policy, not yet implemented)*
 - Email on challan generation
 - Email on payment status change
 - Short attendence alert
diff --git a/docs/02_requirements.md b/docs/02_requirements.md
index 549a268..71fe8af 100644
--- a/docs/02_requirements.md
+++ b/docs/02_requirements.md
@@ -65,6 +65,16 @@ The following priority levels are used throughout this document.
 
 Functional requirements define the services and behavior that the system must provide.
 
+> **Requirement status notation (Phase 4 reconciliation).** The original requirement
+> texts below are **retained** as written; IDs and wording are not changed. Where a
+> requirement is clarified, only partly delivered, or unavailable for some accounts in
+> Phase 4, its row carries a bracketed tag built from these terms: **Retained** (the
+> original text stands), **Current implementation** (what the system does today),
+> **Approved policy, not yet implemented** (decided for Phase 4 and not built until that
+> phase is delivered) and **Deferred** (the requirement stays in the SRS but is not
+> delivered in Phase 4; it belongs to a later phase or to future work). The
+> clarification blocks beneath each section explain the approved policy.
+
 ---
 
 
@@ -76,7 +86,7 @@ The Authentication Module is responsible for verifying user identity, securing a
 
 | ID       | Requirement                                                                                             | Priority    |
 | -------- | ------------------------------------------------------------------------------------------------------- | ----------- |
-| FR-01-1  | The system shall allow registered users to log in using their institutional email address and password. | Must Have   |
+| FR-01-1  | The system shall allow registered users to log in using their institutional email address and password. *[Retained. Current implementation and clarification: login uses the account's registered email address; no NUML-specific email domain is enforced]* | Must Have   |
 | FR-01-2  | The system shall authenticate user credentials before granting access.                                  | Must Have   |
 | FR-01-3  | The system shall generate a JWT Access Token and Refresh Token after successful authentication.         | Must Have   |
 | FR-01-4  | The system shall reject expired, invalid, or tampered authentication tokens.                            | Must Have   |
@@ -93,7 +103,10 @@ The Authentication Module is responsible for verifying user identity, securing a
 > **Phase 1 clarifications (accepted).**
 > - FR-01-1: every user category, **including students**, logs in with an
 >   **email address**. There is no username; a student's roll/registration
->   number is a university identifier, not a login name.
+>   number is a university identifier, not a login name. Login uses the account's
+>   registered email address. The current system does not enforce a NUML-specific
+>   email domain (this also applies where FR-02-10, FR-03-3 and FR-03-8 say
+>   "institutional email").
 > - FR-01-6: passwords use **Argon2id** (64 MiB, 3 passes, 4 lanes).
 > - FR-01-9: logout deletes the server-side session. A refresh token used a
 >   moment after rotation (several tabs refreshing together) is tolerated for
@@ -118,7 +131,7 @@ The User Management Module enables authorized administrators to create, update,
 
 | ID       | Requirement                                                                                                         | Priority    |
 | -------- | ------------------------------------------------------------------------------------------------------------------- | ----------- |
-| FR-02-1  | The system shall allow authorized administrators to create student accounts.                                        | Must Have   |
+| FR-02-1  | The system shall allow authorized administrators to create student accounts. *[Retained. Approved policy, not yet implemented (Phase 4): students are registered by Main Admin, HODs or delegated department staff within their authorized departments]*                                        | Must Have   |
 | FR-02-2  | The system shall allow authorized administrators to create instructor accounts.                                     | Must Have   |
 | FR-02-3  | The system shall allow authorized administrators to create administrative staff accounts.                           | Must Have   |
 | FR-02-4  | The system shall assign one or more system roles to each user.                                                      | Must Have   |
@@ -126,12 +139,12 @@ The User Management Module enables authorized administrators to create, update,
 | FR-02-6  | The system shall automatically generate a unique identifier for every user.                                         | Must Have   |
 | FR-02-7  | The system shall allow administrators to activate or deactivate user accounts without permanently deleting records. | Must Have   |
 | FR-02-8  | Deactivated users shall not be permitted to authenticate into the system.                                           | Must Have   |
-| FR-02-9  | The system shall allow administrators to update user information.                                                   | Must Have   |
-| FR-02-10 | The system shall prevent duplicate institutional email addresses.                                                   | Must Have   |
-| FR-02-11 | The system shall prevent duplicate student roll numbers.                                                            | Must Have   |
+| FR-02-9  | The system shall allow administrators to update user information. *[Retained. Current implementation: Main Admin (`user.update`) edits another user's names and login email; HODs and delegated staff cannot. Approved policy, not yet implemented (Phase 4): staff holding `student.update` edit a student's phone and address in their authorized department; they still cannot edit names or email]*                                                   | Must Have   |
+| FR-02-10 | The system shall prevent duplicate institutional email addresses. *[Retained. Current implementation and clarification: duplicates are decided ignoring letter case; no NUML-specific domain is enforced]*                                                   | Must Have   |
+| FR-02-11 | The system shall prevent duplicate student roll numbers. *[Retained. Approved policy, not yet implemented (Phase 4): unique ignoring letter case; roll numbers vary by department and no complete format is enforced]*                                                            | Must Have   |
 | FR-02-12 | The system shall prevent duplicate employee IDs for instructors and administrative staff.                           | Must Have   |
-| FR-02-13 | The system shall send an account activation email containing login instructions after successful account creation.  | Must Have   |
-| FR-02-14 | The system shall allow administrators to search users by name, roll number, employee ID, department, and role.      | Must Have   |
+| FR-02-13 | The system shall send an account activation email containing login instructions after successful account creation. *[Retained. Staff: current implementation (temporary-password email, Phase 1). Students: approved policy, not yet implemented — an onboarding-instructions email, no password emailed]*  | Must Have   |
+| FR-02-14 | The system shall allow administrators to search users by name, roll number, employee ID, department, and role. *[Retained. Current implementation: search by name, email, department, role and status through the user directory. Approved policy, not yet implemented: roll-number search through the student routes (Phase 4). Deferred: employee-ID search (Phase 6)]*      | Must Have   |
 | FR-02-15 | The system shall support filtering users based on their current account status.                                     | Should Have |
 | FR-02-16 | The system shall record account creation, modification, activation, and deactivation activities in the audit log.   | Must Have   |
 
@@ -143,7 +156,8 @@ The User Management Module enables authorized administrators to create, update,
 > - FR-02-9: **only Main Admin** (`user.update`) edits names and login email.
 >   HODs cannot edit names, email, status or roles; a future departmental
 >   need gets its own narrowly scoped permission. Users may edit their own
->   name only. Changing an email revokes the account's sessions, marks the new
+>   name only, **except accounts holding the `STUDENT` role, including accounts that also hold
+>   other roles** (approved Phase 4 policy below, not yet implemented). Changing an email revokes the account's sessions, marks the new
 >   address unverified, and notifies the previous address.
 > - FR-02-14/15: search by name, email, department, role and status is
 >   implemented, scoped by the caller's departments.
@@ -152,6 +166,49 @@ The User Management Module enables authorized administrators to create, update,
 >   and employee-ID search/uniqueness FR-02-12 (Phase 6); department creation
 >   (Phase 3); FR-02-13 account-activation email for those account types.
 
+> **Phase 4 clarifications (approved policy; not yet implemented).**
+> - **Identifiers.** *Internal user id* (FR-02-6): an opaque UUID created for
+>   every account. It is never a credential, and a student's profile id is the same
+>   value. *Login email* (FR-01-1, FR-02-10): the account's registered email
+>   address, unique ignoring letter case; the current system does not enforce a
+>   NUML-specific email domain. *Student roll number* (FR-02-11): a
+>   university-issued identifier supplied at registration; it is never a login
+>   name. *Employee ID* (FR-02-12): instructors and administrative staff, Phase 6.
+> - **Roll number (FR-02-11).** Roll numbers are issued by the university and their
+>   format varies by department (for example prefixes such as CS, LHR or LCM), so
+>   the system enforces no complete format. Leading and trailing ordinary ASCII spaces (U+0020) are trimmed.
+>   Tabs, newlines, non-breaking spaces and other prohibited whitespace characters
+>   are rejected wherever they occur, including at the beginning and end. The
+>   number is otherwise kept as typed (letter case and inner characters); its first
+>   character must be an ASCII Latin letter; characters of Unicode categories Cc,
+>   Cf, Zl and Zp are rejected; other printable Unicode characters, including Urdu
+>   letters, are allowed; and a **provisional technical limit of 30 characters** (a
+>   schema limit, not an official NUML rule) applies.
+>   Two roll numbers that differ only in letter case are the same roll number: each
+>   is unique across the university. A roll number is never a login name, a file
+>   name or a file path.
+> - **FR-02-1 and FR-02-13 (student onboarding).** A student is registered by Main
+>   Admin, an HOD, or delegated staff holding `student.create` for the department.
+>   The account is created with an unknown random password and **no password is
+>   emailed**. The student receives an email telling them to use *Forgot password*
+>   with their registered address; completing the one-time-code reset sets their
+>   password and verifies the email. Staff accounts are different: they receive a
+>   temporary password by email (Phase 1, current implementation).
+> - **FR-02-9 (who edits what).** **Accounts holding the `STUDENT` role, including
+>   accounts that also hold other roles, cannot change their own first or last
+>   name. Students also cannot change their own email, roll number, program,
+>   semester level, section or admission term; they can change only their own phone
+>   number.** HODs and delegated staff
+>   holding `student.update` can edit a student's phone and address within their
+>   department and nothing else. Main Admin edits names and email, roll number and
+>   admission term, with audit.
+> - **FR-02-14 (search).** Students are found by roll number through the student
+>   routes (department scope for HODs and staff, global for Main Admin), not
+>   through the general user directory.
+> - Placement movement, promotion, freeze and section merge are not part of Phase 4
+>   (roadmap item "Phase 5A"). Details:
+>   [phase4-student-registration.md](phase4-student-registration.md).
+
 
 ---
 
@@ -165,19 +222,42 @@ The Profile Management Module enables authenticated users to manage their person
 | ID       | Requirement                                                                                                                             | Priority    |
 | -------- | --------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
 | FR-03-1  | The system shall provide every authenticated user with a personal profile page.                                                         | Must Have   |
-| FR-03-2  | The system shall display the user's profile picture.                                                                                    | Must Have   |
-| FR-03-3  | The system shall display the user's full name, institutional email address, phone number, CNIC, address, department, and assigned role. | Must Have   |
-| FR-03-4  | The system shall allow users to update their phone number.                                                                              | Must Have   |
-| FR-03-5  | The system shall allow users to update their residential address.                                                                       | Must Have   |
-| FR-03-6  | The system shall allow users to upload or change their profile picture.                                                                 | Should Have |
+| FR-03-2  | The system shall display the user's profile picture. *[Retained. Deferred: profile pictures are not part of Phase 4]*                                                                                    | Must Have   |
+| FR-03-3  | The system shall display the user's full name, institutional email address, phone number, CNIC, address, department, and assigned role. *[Retained in part. Current implementation: `GET /users/me` returns name, email and roles. Approved policy, not yet implemented: a student's profile through `GET /students/me`. Deferred: CNIC (no business requirement established; not stored) and the profile picture]* | Must Have   |
+| FR-03-4  | The system shall allow users to update their phone number. *[Retained. Approved policy, not yet implemented: a student may change only their own phone number (`PATCH /students/me`). Deferred: phone self-service for other account types]*                                                                              | Must Have   |
+| FR-03-5  | The system shall allow users to update their residential address. *[Retained as a future requirement; NOT a Phase 4 requirement for students: in Phase 4 students cannot edit their own address. Approved policy, not yet implemented: departmental staff with `student.update` edit a student's address within their authorized department. Deferred: self-service address editing for other account types (future work)]*                                                                       | Must Have   |
+| FR-03-6  | The system shall allow users to upload or change their profile picture. *[Retained. Deferred]*                                                                 | Should Have |
 | FR-03-7  | The system shall allow users to change their password after verifying their current password.                                           | Must Have   |
-| FR-03-8  | The system shall not allow users to modify their institutional email address.                                                           | Must Have   |
-| FR-03-9  | The system shall not allow users to modify their CNIC information.                                                                      | Must Have   |
-| FR-03-10 | Changes to institutional information shall only be performed by authorized administrators.                                              | Must Have   |
+| FR-03-8  | The system shall not allow users to modify their institutional email address. *[Retained. Current implementation: a user cannot change their own login email; only Main Admin can, through the audited workflow that revokes the account's sessions]*                                                           | Must Have   |
+| FR-03-9  | The system shall not allow users to modify their CNIC information. *[Retained. Deferred: CNIC is not stored in Phase 4]*                                                                      | Must Have   |
+| FR-03-10 | Changes to institutional information shall only be performed by authorized administrators. *[Retained. Current implementation: another user's login email and names are changed only by Main Admin (`user.update`), but every account can currently edit its own first and last name through `PATCH /users/me`. Approved policy, not yet implemented (Phase 4): accounts holding the `STUDENT` role, including accounts that also hold other roles, cannot change their own first or last name; roll number and admission term are corrected only by Main Admin]*                                              | Must Have   |
 | FR-03-11 | The system shall validate profile information before saving updates.                                                                    | Must Have   |
 | FR-03-12 | The system shall record profile update activities in the audit log.                                                                     | Must Have   |
 
 
+> **Phase 4 clarifications (approved policy; not yet implemented).**
+> - FR-03-1 to FR-03-3: every account has `GET /users/me` today. A **student's own
+>   profile** (first and last name, registered email, roll number, admission term,
+>   phone, address and current placement) is provided by `GET /students/me`.
+>   **CNIC is excluded** (no business requirement has been established) and the
+>   **profile picture is deferred**; neither requirement is removed.
+> - FR-03-4: **a student may change their own phone number** (`PATCH /students/me`)
+>   and nothing else about their profile.
+> - FR-03-5: **in Phase 4 students cannot edit their own address.** A student's
+>   address is maintained by authorized departmental staff (`student.update` in
+>   their authorized department). Self-service address editing for other account
+>   types is **Deferred (future work)**; the original requirement is retained for
+>   that case.
+> - FR-03-8 and FR-03-10: no user can change their own email address, and
+>   institutional information (names, email, roll number, admission term, program,
+>   semester level, section) is changed only by authorized administrators.
+>   **Accounts holding the `STUDENT` role, including accounts that also hold other
+>   roles, cannot change their own first or last name.**
+> - *Current implementation (Phase 1):* `PATCH /users/me` lets every account change
+>   its own names. Phase 4 removes that for accounts holding the `STUDENT` role.
+> - FR-03-6 (picture) and FR-03-9 (CNIC) remain future requirements.
+
+
 ---
 
 
@@ -213,8 +293,16 @@ The Academic Structure Management Module enables authorized administrators to co
 >   structure. HODs manage the sections of their own departments.
 > - FR-04-10: every academic structure change is audited with old and new
 >   values; term events are visible to Main Admin only.
-> - **Deferred to Phase 4:** student placement history and section merging
->   (see `phase3-academic-organization.md`).
+> - **Student placement history** is delivered in Phase 4 (the initial
+>   placement and two Main Admin corrections only). **Placement movement,
+>   promotion, freeze and section merge** are the separately designed roadmap item
+>   "Phase 5A" (see `phase3-academic-organization.md` and
+>   `phase4-student-registration.md`).
+
+
+> **Terminology.** In FR-05 to FR-13, "semester" or "academic semester" means the
+> **academic term** (a Fall or Spring session, see FR-04). A student's position in
+> the degree is the **semester level** (1st to 8th), which is a different concept.
 
 
 ---
@@ -394,7 +482,7 @@ The Notification Management Module informs users about important academic and ad
 
 | ID       | Requirement                                                                                      | Priority    |
 | -------- | ------------------------------------------------------------------------------------------------ | ----------- |
-| FR-11-1  | The system shall send a welcome email when a new user account is created with their credentials. | Must Have   |
+| FR-11-1  | The system shall send a welcome email when a new user account is created with their credentials. *[Retained for staff accounts (current implementation: temporary password emailed, Phase 1). Approved policy, not yet implemented: students are never emailed a password or credential, only onboarding instructions]* | Must Have   |
 | FR-11-2  | The system shall notify users when their password has been successfully reset.                   | Must Have   |
 | FR-11-3  | The system shall notify students when new course material is uploaded.                           | Should Have |
 | FR-11-4  | The system shall notify students when new assessment marks are published.                        | Must Have   |
@@ -407,6 +495,16 @@ The Notification Management Module informs users about important academic and ad
 | FR-11-11 | The system shall retain notification history for future reference.                               | Should Have |
 
 
+> **Phase 4 clarification (approved policy; not yet implemented).** FR-11-1: the
+> welcome email carries a **temporary password only for staff accounts** created by
+> an HOD (Phase 1, current implementation). **Students are never emailed a
+> password.** Their onboarding email contains instructions to use the *Forgot
+> password* flow with their registered email address; completing the one-time-code
+> reset sets the password and verifies the email. *Current implementation:* email is
+> sent with FastAPI `BackgroundTasks` (in-process, at-most-once); queue-based
+> delivery and the in-app notification centre (FR-11-9 to FR-11-11) remain planned.
+
+
 ---
 
 
diff --git a/docs/03_system_acrchitecture.md b/docs/03_system_acrchitecture.md
index ce2b416..854ef02 100644
--- a/docs/03_system_acrchitecture.md
+++ b/docs/03_system_acrchitecture.md
@@ -18,6 +18,13 @@ The system is designed as a modern web-based university management platform that
 
 Rather than describing implementation details, this document focuses on the architectural design decisions that ensure the system remains maintainable, scalable, secure, and modular throughout its development lifecycle.
 
+> **Terminology and implemented modules.** In this document "semester" (and the
+> `semesters` module in the planned structure) means the **academic term**, a Fall or
+> Spring session; a student's position in the degree is the **semester level** (1st to
+> 8th). The implemented backend names that module `terms`, and keeps departments and
+> their HOD and staff assignments in `academics`; see `implementation-plan.md` for the
+> real layout.
+
 ---
 
 # 2. Architectural Goals
@@ -210,7 +217,7 @@ The following technologies have been selected based on project requirements, sca
 | Authentication | JWT + Refresh Tokens | Secure Authentication |
 | Authorization | RBAC | Role-Based Access Control |
 | Cache | Redis | Caching & Background Queue |
-| Background Tasks | Redis Queue (RQ) | Email Processing |
+| Background Tasks | Redis Queue (RQ) *(planned; see the note below the table)* | Email Processing |
 | Email Service | Google SMTP | Email Notifications |
 | File Storage | Local Uploads | Profile Pictures, Course Materials, Challans, Transcripts |
 | Assignment Storage | Google Drive | Student Assignment Submissions |
@@ -218,6 +225,18 @@ The following technologies have been selected based on project requirements, sca
 | Containerization | Docker | Deployment |
 | Version Control | Git & GitHub | Source Code Management |
 
+> **Current implementation (as of Phase 3).** Email is sent with FastAPI
+> `BackgroundTasks` through `schedule_email` (`app/core/email.py`): in-process, after
+> the response and the database commit, at-most-once, with no retry or
+> persistence. Redis is currently used for sessions and refresh tokens, login and
+> IP rate limits and password-reset one-time codes; it is **not yet used as a job
+> queue**, and no RQ library or worker process exists. The Redis Queue (RQ) design
+> in this document remains the **intended architecture** for later phases
+> (notification retries, imports, reports). `schedule_email` is the primary
+> application integration point for transitioning email delivery to the planned
+> queue-based architecture. The transition will also require the corresponding worker
+> implementation, dependencies, configuration, job registration and deployment setup.
+
 ---
 
 # 6. Architecture Decision Records (ADR)
@@ -349,7 +368,7 @@ Google SMTP is responsible for delivering email notifications.
 
 Examples include:
 
-- Welcome Emails
+- Account onboarding emails (staff: temporary password; students: onboarding instructions, no password)
 - Password Reset Emails
 - Attendance Shortage Alerts
 - Fee Notifications
@@ -824,6 +843,18 @@ Characteristics:
 - Stored securely
 - Can be revoked during logout
 
+> **Current implementation (Phase 1, session policy "Option A").** The access token lasts
+> 15 minutes and carries **no roles or permissions**: authorization is re-read from the
+> database on every request (so "contains assigned role" above is the original design, not
+> the implementation). The refresh token is an opaque random value kept only as a hash in
+> Redis, rotated on every use, and sent in an HttpOnly, Secure, SameSite=Lax cookie that is
+> **not persistent** (no `Max-Age` or `Expires`). A page refresh, switching portal views and
+> closing one tab keep the login; closing the browser normally requires logging in again,
+> subject to the browser's own session restoration. Redis keeps a session for at most 7 days
+> (a server-side cap). Explicit logout deletes the server-side session. A token used within
+> 10 seconds of rotation, at most 5 times, receives an access token only and no new refresh
+> cookie. See `phase1-identity-and-security.md`, section 3.
+
 ---
 
 ## Password Security
@@ -1011,6 +1042,18 @@ Certain operations do not require an immediate response to the user.
 
 To improve responsiveness, the system processes long-running tasks asynchronously using Redis Queue (RQ).
 
+> **Current implementation (as of Phase 3).** Email is sent with FastAPI
+> `BackgroundTasks` through `schedule_email` (`app/core/email.py`): in-process, after
+> the response and the database commit, at-most-once, with no retry or
+> persistence. Redis is currently used for sessions and refresh tokens, login and
+> IP rate limits and password-reset one-time codes; it is **not yet used as a job
+> queue**, and no RQ library or worker process exists. The Redis Queue (RQ) design
+> in this document remains the **intended architecture** for later phases
+> (notification retries, imports, reports). `schedule_email` is the primary
+> application integration point for transitioning email delivery to the planned
+> queue-based architecture. The transition will also require the corresponding worker
+> implementation, dependencies, configuration, job registration and deployment setup.
+
 ---
 
 ## Background Tasks
@@ -1061,9 +1104,11 @@ Whenever an event requires email delivery, the application adds an email job to
 
 The worker retrieves the queued job and sends the email using Google SMTP.
 
+*Planned architecture. Today the email is sent by FastAPI `BackgroundTasks` (see the current-implementation note at the start of this section).*
+
 Examples include:
 
-- Welcome Emails
+- Account onboarding emails (staff: temporary password; students: onboarding instructions, no password)
 - Password Reset Emails
 - Attendance Alerts
 - Fee Notifications
@@ -1103,6 +1148,11 @@ Redis serves multiple purposes within the system.
 - Rate Limiting
 - Frequently Accessed Reference Data
 
+> **Current versus planned.** Today Redis provides sessions and refresh-token hashes, login and
+> IP rate limiting and password-reset OTP storage. The background job queue and the dashboard,
+> notification and reference-data caches listed here are **planned** (see the
+> current-implementation note under the technology stack table).
+
 ---
 
 ## 16.2 Cached Data
@@ -1139,6 +1189,10 @@ During logout, refresh tokens are added to Redis until their natural expiration
 
 This prevents reuse of revoked tokens without requiring additional database storage.
 
+> *Current implementation:* logout deletes the server-side session (the refresh token is only
+> ever stored as a hash in Redis) instead of adding the token to a blacklist; see the session
+> note in the authentication section.
+
 ---
 
 ## 16.5 Rate Limiting
@@ -1174,7 +1228,7 @@ Delivered using Google SMTP.
 
 Examples include:
 
-- Welcome Email
+- Account onboarding email (staff: temporary password; students: onboarding instructions, no password)
 - Password Reset OTP
 - Attendance Shortage Alert
 - Fee Challan Upload
diff --git a/docs/implementation-plan.md b/docs/implementation-plan.md
index 4c66a70..6101b04 100644
--- a/docs/implementation-plan.md
+++ b/docs/implementation-plan.md
@@ -4,7 +4,8 @@
 **Architecture:** Modular Monolith\
 **Backend:** FastAPI\
 **Primary Database:** PostgreSQL + SQLAlchemy\
-**Cache / Queue / Temporary State:** Redis\
+**Cache / Queue / Temporary State:** Redis (today: sessions, refresh tokens,
+rate limiting and OTP storage; caching and the job queue are planned)\
 **File Storage (FYP):** Local `uploads/` directory, except student
 assignment submissions\
 **Assignment Storage:** Google Drive\
@@ -54,64 +55,49 @@ include:
 The backend remains one deployable FastAPI application, but business
 functionality is separated into modules.
 
-Recommended structure:
+Layout of the repository as it is today (Phases 0-3), with the modules planned
+for later phases marked:
 
 ``` text
 backend/
+├── pyproject.toml          # dependencies and tool configuration, managed with uv
+├── uv.lock
+├── alembic.ini
+├── docker-compose.yaml     # PostgreSQL and Redis for development
+├── .env / .env.example     # configuration (.env is never committed)
 ├── app/
 │   ├── main.py
-│   │
-│   ├── core/
-│   │   ├── config.py
-│   │   ├── database.py
-│   │   ├── redis.py
-│   │   ├── security.py
-│   │   ├── exceptions.py
-│   │   ├── logging.py
-│   │   └── permissions.py
-│   │
+│   ├── core/               # config, database, redis, security, exceptions,
+│   │                       # logging, permissions, email
 │   ├── modules/
-│   │   ├── auth/
-│   │   ├── users/
-│   │   ├── access_control/
-│   │   ├── departments/
-│   │   ├── programs/
-│   │   ├── sections/
-│   │   ├── students/
-│   │   ├── instructors/
-│   │   ├── courses/
-│   │   ├── enrollments/
-│   │   ├── attendance/
-│   │   ├── course_materials/
-│   │   ├── assignments/
-│   │   ├── marks/
-│   │   ├── results/
-│   │   ├── fees/
-│   │   ├── notifications/
-│   │   ├── reports/
-│   │   └── audit/
-│   │
-│   ├── shared/
-│   │   ├── models/
-│   │   ├── schemas/
-│   │   ├── constants/
-│   │   └── utils/
-│   │
-│   └── workers/
-│       ├── email_tasks.py
-│       ├── notification_tasks.py
-│       └── import_tasks.py
-│
-├── migrations/
-├── uploads/
+│   │   │   implemented:
+│   │   ├── auth/           # login, sessions, password flows, RBAC, department scope
+│   │   ├── users/          # account status, directory, profile
+│   │   ├── academics/      # departments, HOD assignments, staff assignments
+│   │   ├── staff/          # department staff onboarding and delegation
+│   │   ├── programs/       # programs of a department (Phase 3)
+│   │   ├── terms/          # academic terms, Fall / Spring (Phase 3)
+│   │   ├── sections/       # program + semester level + name (Phase 3)
+│   │   ├── audit/          # append-only audit log (built in Phase 1)
+│   │   │   planned:
+│   │   ├── students/       # Phase 4 (with student placement history)
+│   │   ├── instructors/ courses/ enrollments/ attendance/ course_materials/
+│   │   └── assignments/ marks/ results/ fees/ notifications/ reports/
+│   ├── shared/             # base model; schemas, constants and utils as needed
+│   └── workers/            # empty placeholder; the planned home of queue workers
+├── migrations/             # Alembic revisions
+├── scripts/                # seed_rbac, bootstrap_admin, reset_seed_admin,
+│                           # provision_db_roles
 ├── tests/
-├── .env
-├── alembic.ini
-├── requirements.txt
-├── Dockerfile
-└── docker-compose.yml
+└── uploads/                # local file storage (git-ignored)
 ```
 
+There is no `requirements.txt` and no application `Dockerfile` yet: dependencies are
+declared in `pyproject.toml` and installed with `uv`. Earlier drafts of this plan
+showed a `departments/` module and an `access_control/` module; departments live in
+`academics/`, and the access-control code lives in `auth/` and
+`core/permissions.py`.
+
 A normal business module should follow:
 
 ``` text
@@ -197,6 +183,14 @@ Return
 Whenever authoritative data changes, the related cache should be
 invalidated or refreshed.
 
+> **Current implementation (as of Phase 3).** Redis holds sessions and refresh
+> tokens, login and IP rate-limit counters and password-reset one-time codes (and,
+> in the approved Phase 4 design, the onboarding-email resend limits). It is **not**
+> yet used as a cache or as a job queue, and no worker exists: email is sent with
+> FastAPI `BackgroundTasks` through `schedule_email`. The cache and the Redis
+> background queue listed above remain **planned**; reference data is deliberately
+> not cached yet.
+
 ## 3.3 MongoDB --- Decision
 
 MongoDB is technically possible, but it should **not be introduced
@@ -357,6 +351,20 @@ Access token: 15 minutes
 Refresh token: 7 days
 ```
 
+Session policy ("Option A", *current implementation*; see
+[phase1-identity-and-security.md](phase1-identity-and-security.md) section 3):
+
+-   The 7 days is the **server-side session lifetime cap**, not the life of the
+    cookie. The refresh-token cookie is **not persistent** (no `Max-Age` or
+    `Expires`; HttpOnly, Secure, SameSite=Lax, path `/api/v1/auth`), so closing the
+    browser normally ends the login, subject to browser session restoration.
+-   Refreshing a page, switching portal views and closing one tab keep the login.
+-   Explicit logout deletes the server-side session.
+-   Refresh tokens rotate. A token used within 10 seconds of rotation (several tabs
+    refreshing together), at most 5 times, yields an access token only and no new
+    refresh cookie; any other reuse destroys the session.
+-   No long-lived persistent refresh-token cookie is introduced.
+
 Redis can store:
 
 -   OTPs
@@ -443,6 +451,12 @@ audit.view
 
 The complete permission catalogue should be maintained centrally.
 
+Notes (current implementation and approved policy): `section.move_student` is
+reserved to Main Admin and is never delegated to HODs or department staff (Phase 4
+uses it only for the Main Admin correction of a wrongly entered initial placement).
+`student.bulk_import` is not on the staff delegation allow-list today; whether it
+should be is a Phase 5 decision.
+
 ## 5.5 Department Scope
 
 Never assume:
@@ -529,7 +543,10 @@ summary:
 -   **User directory:** department-scoped search and profile reads. `user.update`
     stays **Main Admin only**; HODs cannot edit names, email, status or roles.
     Changing a login email notifies the old address, revokes sessions and marks
-    the new address unverified.
+    the new address unverified. Every account can currently edit its own name;
+    **approved Phase 4 policy (not yet implemented): accounts holding the `STUDENT`
+    role cannot change their own first or last name** (they may change only their
+    own phone number).
 -   The **permission catalogue** in 5.4 is fully registered in
     `app/core/permissions.py`. Registering a code grants nothing; codes for
     unbuilt modules are held by Main Admin only. Three names were aligned with
@@ -614,6 +631,14 @@ Deactivating a staff account must not delete historical audit records.
 
 Reusable audit service that future modules can call.
 
+> **Status: delivered in Phase 1.** The reusable audit service, the append-only
+> protection of the audit table and department-scoped audit search were built in
+> Phase 1 because the identity features needed them (see
+> [phase1-identity-and-security.md](phase1-identity-and-security.md) section 9).
+> Phase 2 therefore has no separate implementation work. What remains is a
+> standing requirement: every new module records its sensitive operations through
+> that service, in the same transaction as the change.
+
 ------------------------------------------------------------------------
 
 # 7. Phase 3 --- Academic Organization Foundation
@@ -657,9 +682,9 @@ Support:
 
 -   Create section
 -   Rename section
--   Merge sections
+-   Merge sections *(not built in Phase 3; deferred to Phase 5A)*
 -   Activate/deactivate section
--   View section history
+-   View section history *(built: served from the audit log)*
 
 A student can later be moved between academic semester/section
 placements without deleting historical academic records.
@@ -691,10 +716,15 @@ BSCS 3/A
 
 The department controls such movement.
 
+> **As built and roadmap.** Phase 3 builds **no** placement table. A student's
+> initial placement and its history table arrive in **Phase 4** (the initial
+> placement and two Main Admin corrections only). Movement, promotion and semester
+> freeze are the separately designed roadmap item **Phase 5A**.
+
 ## Deliverable
 
-Department, program, semester/term, section, and student-placement
-foundations.
+Department, program, semester/term and section foundations. *(Student-placement
+history was planned here but is delivered in Phase 4; see the amendments below.)*
 
 ## Phase 3 amendments (as built)
 
@@ -714,11 +744,11 @@ The authoritative description is
     staff receive no new permission.
 -   Deactivation is blocked by active dependents (never cascaded); nothing is
     hard-deleted. Department names are unique case-insensitively.
--   **Deferred to Phase 4 by decision:** the `student_placements` table (it is
-    created together with the students table so it has a real foreign key),
-    placement-history protection, section merging (it moves placements) and all
-    promotion/freeze/movement endpoints. The intended placement design and its
-    integration requirements are recorded in the Phase 3 document, §9.
+-   **Deferred by decision:** the `student_placements` table and its history
+    protection are built in **Phase 4** (it is created together with the students
+    table so it has a real foreign key; see the Phase 4 design). Section merging (it
+    moves placements) and all promotion, freeze and movement endpoints are the
+    separately designed **Phase 5A**.
 
 ------------------------------------------------------------------------
 
@@ -746,7 +776,9 @@ Student information includes the agreed registration fields:
 
 -   First name and last name (required)
 -   Email (required; it is the login identity)
--   Roll/Registration number (required; an identifier, never a login name)
+-   Roll/Registration number (required; a university-issued identifier whose
+    format varies by department, never a login name; see the roll-number rules
+    below)
 -   Program (required)
 -   Semester level (required; the student's level in the degree, for example
     1st to 8th. It is **not** an academic term)
@@ -762,10 +794,45 @@ Not part of manual registration in Phase 4:
     (current placement, then program, then department).
 -   **CNIC** is excluded. The business requirement for storing it has not been
     established, so there is no column, field or validation for it until it is
-    separately approved (see the Phase 4 design document, section 12).
+    separately approved (see the CNIC section of the Phase 4 design document).
 -   **Picture** is deferred.
 -   No other personal fields (gender, date of birth, guardian) without approval.
 
+### Roll numbers
+
+NUML does not use one roll-number format: it varies by department and begins with
+an alphabetic prefix (for example CS, LHR or LCM; these are examples only). The
+system therefore enforces **no complete format** and builds no department-specific
+patterns or prefix mapping. Leading and trailing ordinary ASCII spaces (U+0020) are trimmed. Tabs, newlines,
+non-breaking spaces and other prohibited whitespace characters are rejected wherever
+they occur, including at the beginning and end. The number is otherwise kept as typed
+(letter case and inner characters); its first character must be an ASCII Latin letter;
+characters of Unicode categories Cc, Cf, Zl and Zp are rejected; other printable
+Unicode characters, including Urdu letters, are allowed; and a **provisional technical
+limit of 30 characters** (a schema limit, not an
+official NUML rule). Roll numbers that differ only in letter case are the same
+number, unique across the university, decided by one PostgreSQL expression,
+`upper(roll_number)`, which every duplicate check, correction and lookup also uses.
+A roll number is never a login, a file path or a file name.
+
+### Reference data for registration
+
+Delegated department staff cannot read programs, sections or terms through the
+Phase 3 endpoints (those permissions are not delegable). Registration therefore has
+a dedicated read-only endpoint, `GET /departments/{d}/students/registration-options`,
+authorized by `student.create` in the department. It returns only the department's
+active programs, their active sections grouped by semester level, and non-archived
+academic terms. The delegation allow-list is not expanded.
+
+### Student self-service
+
+Students can view their own profile (`GET /students/me`) and change **only their
+own phone number** (`PATCH /students/me`). They cannot change their first or last
+name (`PATCH /users/me` is refused for any account holding the `STUDENT` role),
+email, roll number, program, semester level, section or admission term. Departmental
+staff holding `student.update` edit a student's phone and address within their
+department; names and email stay Main Admin-only.
+
 ### Initial Credentials
 
 > **Superseded by the Phase 1 authentication decisions.** The original text of
@@ -797,7 +864,8 @@ Students can later change their password.
 
 Validate at minimum:
 
--   Roll number uniqueness (global, case-insensitive, after normalization)
+-   Roll number validity and uniqueness (the rules above: global and
+    case-insensitive through one PostgreSQL expression)
 -   Email uniqueness (an email already used by any account is refused; an
     existing account is never reused)
 -   Required fields
@@ -817,8 +885,9 @@ Validate at minimum:
 
 User account creation, student profile creation, the initial placement and the
 audit record occur in **one database transaction** so partial student accounts
-are never left behind. Row locks are taken in the fixed order department,
-program, section.
+are never left behind. Row locks follow one global order, `term → department (when required) →
+program → section → student → placement`; registration takes shared locks on the
+admission term, the program and the section (see the Phase 4 design).
 
 ### After Successful Registration
 
@@ -858,14 +927,33 @@ The authoritative design is
     are Main Admin only. Main Admin also gets a global student search.
 -   **Movement, promotion and freeze are not part of this phase.** Only two
     Main Admin-only corrections of data-entry mistakes exist: the initial
-    placement (existing `section.move_student`, allowed only while no later
-    placement exists) and the roll number / admission term (existing
+    placement (existing `section.move_student`, allowed while every placement is the
+    initial one or a correction, so repeated corrections are possible; a superseded
+    row is preserved permanently) and the roll number / admission term (existing
     `user.update`). Both are audited with old and new values.
 -   Account deactivation stays Main Admin-only through the existing user
     endpoint; there is no student deactivation endpoint and no withdrawal or
     graduation lifecycle.
--   Not in Phase 4: CNIC, profile pictures, `/students/me`, accounts-staff
-    access, bulk registration, and any frontend work.
+-   **Student self-service:** students can view their own profile
+    (`GET /students/me`) and change only their own phone (`PATCH /students/me`);
+    they cannot change their names (`PATCH /users/me` is refused for any account
+    holding the `STUDENT` role). Staff with `student.update` edit phone and address in
+    their department.
+-   **Roll numbers** vary by department; no complete format is enforced (see 8.1).
+    The 30-character limit is a provisional technical limit.
+-   **Registration reference data:** `GET /departments/{d}/students/registration-options`,
+    authorized by `student.create`; the delegation allow-list is unchanged.
+-   **Locking:** one global order, `term → department (when required) → program →
+    section → student → placement`.
+-   **Database:** column-level `UPDATE` privileges for the runtime role (`students`:
+    roll number, admission term, phone, address and `updated_at`;
+    `student_placements`: `ended_at` only) and no `DELETE` on either table.
+-   **Email:** students are never emailed a password. Delivery uses FastAPI
+    `BackgroundTasks` today (current implementation); a Redis queue worker remains
+    the planned architecture.
+-   Not in Phase 4: CNIC, profile pictures, accounts-staff access, bulk
+    registration, placement movement, promotion, freeze and section merge
+    (Phase 5A), and any frontend work.
 
 ------------------------------------------------------------------------
 
@@ -873,7 +961,9 @@ The authoritative design is
 
 ## Objective
 
-Allow departmental staff to upload exported Google Form data.
+Allow departmental staff to upload exported Google Form data. *(Whether staff or
+only Main Admin may run an import is a Phase 5 decision; see the note at the end of
+this section.)*
 
 ## Workflow
 
@@ -946,6 +1036,14 @@ should be reused by bulk registration. Phase 4 builds that service with plain
 inputs and one transaction per student, so Phase 5 can call it once per valid
 row; the onboarding emails are queued after each successful row, never before.
 
+Duplicate roll numbers inside an uploaded file must be detected with the same
+PostgreSQL expression (`upper(roll_number)`) that the database uses, not with
+separate Python case-folding. Whether `student.bulk_import` may be delegated to
+department staff is a **Phase 5 decision**: today it is Main Admin-only and not on
+the delegation allow-list, and the demonstration scenario in section 31 does not show
+it being granted. This plan authorizes no change to the permission catalogue or the
+allow-list.
+
 ## Deliverable
 
 Production-style student registration module with manual and bulk
@@ -953,6 +1051,44 @@ workflows.
 
 ------------------------------------------------------------------------
 
+# 9A. Phase 5A --- Student Placement Movement, Promotion, Freeze and Section Merge
+
+> **Roadmap item. It requires its own design and approval before any
+> implementation, and it is not part of Phase 4.**
+
+## Objective
+
+Deliver the placement functionality that Phase 4 deliberately excludes. It sits after
+Phase 5 and before course enrollment (Phase 8), because enrollment depends on each
+student's correct current semester level and section.
+
+## Scope (to be designed and approved separately)
+
+-   Department-controlled movement of a student between sections.
+-   Semester progression after final results, decided per student and never assumed
+    to be automatic.
+-   Semester freeze and similar movement to a lower level.
+-   Section merge (moves current placements; atomic; fixed lock order; the merge
+    lineage is preserved).
+-   The placement reasons these add (the Phase 4 reason check allows only `INITIAL`
+    and `CORRECTION`), and the permissions and delegation they need.
+    `section.move_student` remains Main Admin-only until a separate decision says
+    otherwise.
+
+## Rules inherited from Phase 4
+
+-   Placement history is never edited or deleted; a new row supersedes the previous
+    one, which is preserved.
+-   One current placement per student; program, semester level and section stay
+    consistent in the database.
+-   The Phase 4 correction window closes automatically for a student once any other
+    placement reason exists.
+-   The global lock order applies.
+
+No design, API or permission decisions have been made for this item yet.
+
+------------------------------------------------------------------------
+
 # 10. Phase 6 --- Instructor Management and Teaching Assignments
 
 ## Objective
@@ -984,6 +1120,12 @@ Instructor
 
 The same course may have different instructors in different sections.
 
+> *Sections here are the Phase 3 sections: a group of students identified by
+> program, semester level and name. A course offering is associated with such a
+> section (the course-section association of Phase 7); it does not create a second
+> kind of section, and it must follow a section's merge lineage once section merge
+> exists (Phase 5A).*
+
 ------------------------------------------------------------------------
 
 # 11. Phase 7 --- Course Management and Prerequisites
@@ -1301,6 +1443,11 @@ Accounts staff:
 Store actual FYP challan files in `uploads/` and file metadata in
 PostgreSQL.
 
+A roll number is only a **lookup key** when matching uploaded records or file names
+to students (through the same case-insensitive PostgreSQL expression used
+everywhere else). It is never used as a stored file name or path: stored challan
+files use generated identifiers.
+
 ## Student
 
 Student can:
@@ -1414,6 +1561,10 @@ Background Queue
 Google SMTP
 ```
 
+> **Current implementation:** emails are sent with FastAPI `BackgroundTasks`
+> through `schedule_email`; the background queue shown above is the planned
+> architecture.
+
 In-app notifications initially use polling.
 
 Possible future upgrade:
@@ -1569,7 +1720,7 @@ Example:
 Admin assigns HOD
 → HOD creates receptionist
 → Receptionist logs in
-→ Receptionist imports students
+→ Receptionist imports students   (Phase 5; only if import becomes delegable, see section 31)
 → HOD views receptionist activity
 ```
 
@@ -1663,7 +1814,8 @@ Complete:
 -   Department staff accounts
 -   Manual student registration
 -   Bulk student registration
--   Welcome emails
+-   Onboarding emails (staff: temporary password; students: instructions only, no
+    password)
 
 **Result:** Admin can appoint HOD → HOD can create receptionist →
 receptionist can register students → HOD can audit actions.
@@ -1751,21 +1903,21 @@ HOD grants:
     student.view
     student.create
     student.update
-    student.bulk_import
     ↓
-System emails credentials
+System emails a temporary password (staff account)
     ↓
 Receptionist logs in
     ↓
 Receptionist manually registers a student
     ↓
 Receptionist bulk imports students
+    (Phase 5; who may import is undecided, see the note below)
     ↓
 System validates records
     ↓
 Receptionist confirms import
     ↓
-Students receive credentials
+Students receive onboarding instructions
     (email login; onboarding via the forgot-password email-OTP flow, no
     password emailed, never CNIC-based; see Phase 4)
     ↓
@@ -1774,6 +1926,13 @@ HOD opens Audit Logs
 HOD sees actions performed by Receptionist
 ```
 
+> **Delegation note.** The grants shown above are the ones an HOD can delegate
+> today. `student.bulk_import` is **not** part of the current staff delegation
+> allow-list (it is Main Admin-only), so the bulk-import steps are Phase 5 steps.
+> Whether `student.bulk_import` becomes delegable to department staff is a **Phase 5
+> decision**. This document authorizes no change to the permission catalogue or the
+> allow-list.
+
 If this workflow works correctly, many important architectural concerns
 have already been proven:
 
@@ -1924,5 +2083,11 @@ Optional Future Data Store
 └── MongoDB only if a justified document/event workload appears
 ```
 
+> **Redis today (current implementation):** sessions, refresh tokens, rate
+> limiting and password-reset OTP storage. Redis caching and job-queue processing
+> (RQ workers, as in the architecture documents) remain **planned**, and that
+> architecture is unchanged. Email is currently sent with FastAPI
+> `BackgroundTasks`.
+
 This stack should remain the baseline unless an implementation
 requirement demonstrates a concrete need to change it.
diff --git a/docs/phase1-identity-and-security.md b/docs/phase1-identity-and-security.md
index 0f16f6d..f9b63f5 100644
--- a/docs/phase1-identity-and-security.md
+++ b/docs/phase1-identity-and-security.md
@@ -19,7 +19,9 @@ Production database roles are in [database-roles.md](database-roles.md).
   current one.
 * **Temporary passwords** (staff onboarding): `secrets.token_urlsafe(16)`
   (16 random bytes, 128 bits), hashed immediately, emailed once, never stored, logged or
-  returned by an API.
+  returned by an API. **Students are not onboarded this way:** under the approved
+  Phase 4 design (not yet implemented) no password is ever emailed to a student; see
+  `phase4-student-registration.md`.
 
 ## 2. Email verification and login
 
@@ -78,12 +80,14 @@ separate piece of work.
   assignment or deactivating an account takes effect immediately.
 * **Refresh token:** opaque random value, SHA-256-hashed in Redis, rotated on
   every use, stored in an `HttpOnly; Secure; SameSite=Lax` cookie scoped to
-  `/api/v1/auth` with **no `Max-Age`/`Expires`** (a browser-session cookie —
-  closing the whole browser requires logging in again). Redis keeps the session
+  `/api/v1/auth` with **no `Max-Age`/`Expires`** (a browser-session cookie; see
+  the next bullet for what closing the browser means). Redis keeps the session
   at most 7 days as a server-side cap.
-* Page refresh and new tabs re-establish the session from the cookie; closing a
-  tab does not affect other tabs; explicit logout deletes the server session;
-  deactivating an account revokes all of its sessions.
+* Page refresh, switching portal views and new tabs keep or re-establish the session
+  from the cookie; closing one tab does not affect other tabs; closing the whole
+  browser normally requires logging in again, subject to the browser's own session
+  restoration; explicit logout deletes the server session; deactivating an account
+  revokes all of its sessions.
 
 ### Concurrent refresh (several tabs at once)
 
@@ -237,7 +241,7 @@ the allow-list. Removing a permission is always allowed.
 |---|---|
 | `GET /users` (q, role, is_active, department_id, limit ≤ 100, offset) | Main Admin: all. Others: only users attached (HOD / instructor / staff assignment) to a department where the caller's **own** assignment grants `user.view`. No such department ⇒ 403. Filtering by a department outside that scope ⇒ 403. |
 | `GET /users/{id}` | self, Main Admin, or in scope. Anything else is a **404** identical to "does not exist". A scoped viewer sees only the departments within their own scope. |
-| `GET/PATCH /users/me` | own profile; names only |
+| `GET/PATCH /users/me` | own profile; names only. *Current implementation:* every account, students included, may change its own names. *Approved Phase 4 policy (not yet implemented):* an account holding the `STUDENT` role, including a multi-role account, gets **403** on `PATCH /users/me` and cannot change its name (malformed, empty or unknown fields still return 422 first). A student can change only their own phone number, through `PATCH /students/me`. Every other account keeps its existing behaviour. |
 | `PATCH /users/{id}` | needs `user.update`, which **only Main Admin holds and which must stay that way**. Main Admin: names and email. (The code also supports a department-scoped holder — names only, inside their departments, never a Main Admin — but no role has that grant.) |
 
 Search matches names, full name and email case-insensitively (LIKE wildcards are
@@ -311,7 +315,7 @@ uv run alembic upgrade head && uv run alembic check
 | Refresh tokens: any reuse is theft | Reuse of the immediately-previous token within 10 s (max 5) is a concurrent retry; otherwise theft (§3) |
 | "Deactivate access" for staff (§5.7) | Revoke one assignment; account stays active; global deactivation is Main Admin only (§7) |
 | Staff authority from role permissions | Staff have none; authority is per assignment (§6) |
-| Login by username/roll number for students (§8.1) | **Email login for all users, students included.** The roll number is an identifier, not a login name. The CNIC-as-temporary-password idea is withdrawn; Phase 4 designs a secure credential and onboarding workflow (§8.1 of the plan) |
+| Login by username/roll number for students (§8.1) | **Email login for all users, students included.** The roll number is an identifier, not a login name. The CNIC-as-temporary-password idea is withdrawn; the Phase 4 design (approved, not yet implemented) creates the student with an unknown random password, emails **no password**, and has the student set one through the existing Forgot-password OTP flow (see `phase4-student-registration.md`) |
 | Legacy permission placeholders | Kept, with grants, until their modules are designed |
 | `attendance.update`, `course.instructor.assign`, `result.view` | Renamed to the plan's `attendance.correct`, `course.assign_instructor`, `results.view` |
 
@@ -322,8 +326,11 @@ Accepted as intentional future-phase work; none blocks Phase 1:
 * **Department creation / editing** (Phase 3): until then HOD assignment and
   staff flows are exercised with database-seeded departments.
 * **Student account creation** and **roll-number search/uniqueness** (Phase 4;
-  FR-02-1, 02-11). Students log in by email; credential generation is designed
-  there, not CNIC-based.
+  FR-02-1, 02-11). Students log in by email. The approved Phase 4 design (not yet
+  implemented) creates the account with an unknown random password, emails **no
+  password**, and has the student use the Forgot-password OTP flow. Roll numbers
+  vary by department, no complete format is enforced, and they are unique ignoring
+  letter case. Nothing is CNIC-based.
 * **Instructor account creation** and **employee-ID search/uniqueness**
   (Phase 6; FR-02-2, 02-12).
 * **Re-assigning an existing account to a new departmental responsibility**
@@ -337,10 +344,13 @@ Accepted as intentional future-phase work; none blocks Phase 1:
   …): reconcile when their modules are designed. Nothing may be silently removed;
   any rename follows the in-place rename mechanism so ids and grants survive, and
   is documented.
-* **Account-activation email** after student/instructor creation (FR-02-13).
+* **Account-activation email** after student/instructor creation (FR-02-13): for
+  students this is the Phase 4 onboarding email (instructions only, no password);
+  for instructors it is Phase 6.
 * **Permission grants for future modules** (`section.*`, `enrollment.*`,
   `marks.*`, `results.import`, `fee.*`, `student.bulk_import` …): codes are
   registered; HOD/instructor/staff grants are decided module by module.
 * **Email delivery** uses FastAPI `BackgroundTasks`: at-most-once, no retry,
-  in-process. `schedule_email` in `app/core/email.py` is the single seam to swap
-  in a Redis-backed worker.
+  in-process. `schedule_email` in `app/core/email.py` is the primary integration point for moving
+  to the planned Redis-backed worker; that move also needs the worker, dependencies,
+  configuration and deployment setup.
diff --git a/docs/phase3-academic-organization.md b/docs/phase3-academic-organization.md
index d706fe7..5956dd3 100644
--- a/docs/phase3-academic-organization.md
+++ b/docs/phase3-academic-organization.md
@@ -19,7 +19,10 @@ functionality, and no frontend.
 *Fall 2026* is **one** term row, however many semester levels are being taught
 during it. Admission batches progress through semester levels term by term, and
 the department decides each student's progression after the final results; that
-workflow (promotion, freeze, movement) belongs to Phase 4 and later.
+workflow (promotion, freeze, movement) is **not** part of Phase 4 either. Phase 4
+only creates a student's *initial* placement (plus two Main Admin corrections of
+data-entry mistakes). Placement movement, promotion, freeze and section merge are
+the separately designed roadmap item **"Phase 5A"** in `implementation-plan.md`.
 
 ## 2. Entities
 
@@ -124,6 +127,13 @@ be active. Locks are always taken **department → program → section**, so
 lifecycle changes cannot deadlock, and concurrency tests cover creation racing
 a parent's deactivation, duplicate creation, and simultaneous term activation.
 
+Phase 4 extends this to **one global order** that also covers terms, students and
+placements: `term → department (when required) → program → section → student →
+placement` (see "Lock order" in `phase4-student-registration.md`). The Phase 3
+operations already follow it. The one same-level pair is term activation, which
+locks its own term and then the single ACTIVE term; it is deadlock-free because
+that second row is always the one ACTIVE row.
+
 ## 4. Authorization
 
 The established model is unchanged: **the same role assignment must grant both
@@ -141,7 +151,7 @@ permission, and the delegation allow-list is untouched.
 | View sections and section history | yes | yes | `section.view` |
 | Create / edit / lifecycle of **terms** | yes | no | **new** `term.create`, `term.update`, `term.activate` |
 | View terms | yes | yes | **new** `term.view` |
-| Merge sections, move students | reserved | reserved | `section.merge`, `section.move_student` |
+| Merge sections, move students | reserved (Main Admin only) | **not granted** | `section.merge`, `section.move_student` |
 
 Notes:
 
@@ -155,6 +165,9 @@ Notes:
   role-bound check, which refuses inactive departments.
 * Out-of-department child ids are indistinguishable from missing ones (404).
 * Terms are global, so `term.view` is a role-level read permission.
+* `section.move_student` is **Main Admin-only and is never delegated** to HODs or
+  department staff. Phase 4 uses it only for the Main Admin correction of a wrongly
+  entered *initial* placement; general movement is Phase 5A.
 
 ## 5. API (prefix `/api/v1`)
 
@@ -239,13 +252,16 @@ These were considered and **deferred by decision**; none exists in Phase 3.
   integrity is more important than an early table.
 * **No placement-history triggers or provisioning categories.**
 * **No section merge.** Its correctness depends on transferring student
-  placements, so it belongs to the phase where placements exist.
+  placements, so it is deferred to the separately designed roadmap item
+  **Phase 5A** (placement movement, promotion, freeze and section merge).
   `section.merge` and `section.move_student` stay reserved to Main Admin.
-* **No promotion, freeze or semester-movement endpoints.**
+* **No promotion, freeze or semester-movement endpoints** (Phase 5A).
 
-### Intended placement-history design (for Phase 4)
+### Placement-history notes recorded when Phase 3 closed
 
-Recorded here so Phase 4 builds on it rather than rediscovering it.
+These notes were written for Phase 4 at the end of Phase 3. The **authoritative
+design is now `phase4-student-registration.md`**; where they differ, that document
+wins. Nothing below is implemented in Phase 3.
 
 * A student's placement is **program + semester level + (optional) section**.
   Section is nullable: a student with no section has no assignment.
@@ -253,8 +269,9 @@ Recorded here so Phase 4 builds on it rather than rediscovering it.
   `ended_at IS NULL` (a partial unique index on the student enforces one
   current placement). A move closes that row and inserts a new one linked by
   `previous_placement_id`, with the `effective term` (an `academic_terms` row),
-  the actor, a reason (initial, promotion, freeze, movement, correction) and
-  timestamps. Fall→Spring progression therefore creates a new placement; it
+  the actor, a reason and timestamps (Phase 4 implements only the reasons
+  `INITIAL` and `CORRECTION`; promotion, freeze and movement belong to Phase 5A).
+  Fall→Spring progression therefore creates a new placement; it
   never edits the old one and never creates term rows per semester level.
 * The table is created **with** the students table, with a real foreign key to
   it, in one migration.
@@ -265,23 +282,48 @@ Recorded here so Phase 4 builds on it rather than rediscovering it.
   of a section's identity, this guarantees that a placement's section always
   belongs to the placement's program **and** semester level.
 * Placement history should be protected like the audit log (no delete/truncate
-  for the runtime role, edits limited to closing a row). That is an extension
-  of `provision_db_roles.py` to design with the table.
+  for the runtime role, edits limited to closing a row). That is now designed
+  in the Phase 4 document: a history trigger plus a column-level
+  `UPDATE (ended_at)` privilege for the runtime role.
 * `SectionService.deactivate_section` **must also refuse while the section has
   current placements** (the Phase 3 service carries a note); likewise a
   program's duration change must consider placed semester levels.
 * Student registration (Phase 4) must offer only sections of an **active**
   program in the student's department, and validate the semester level against
   `duration_semesters`.
-* Section merging, when built, must move *current* placements (close and
+* Section merging, when built (Phase 5A), must move *current* placements (close and
   re-open, never edit), leave historical ones, be atomic and lock sections in a
   fixed order. Later course-section links must follow the merge lineage.
-* Permissions to hand out then: `section.move_student` (HOD, and staff the HOD
-  explicitly delegates it to), `student.*` as planned. The delegation
-  allow-list is unchanged until a phase needs it.
-* Department staff registering students will need read access to programs,
-  terms and sections; that means extending the delegation allow-list with
-  `program.view`, `term.view`, `section.view`, which is a Phase 4 decision.
+* Permissions: `student.create`, `student.view` and `student.update` are reused
+  as they are. **`section.move_student` is Main Admin-only and is never granted to
+  HODs or delegated to staff**; Phase 4 uses it only for the Main Admin correction
+  of a wrongly entered *initial* placement. The delegation allow-list is
+  unchanged.
+* Department staff registering students need the program, section and term
+  ids. This is **not** solved by extending the delegation allow-list (it stays
+  unchanged, and the Phase 3 endpoints keep their authorization). Phase 4 adds a
+  dedicated read-only `GET /departments/{d}/students/registration-options`
+  endpoint, authorized by `student.create` in the department, that returns only
+  active programs, their active sections by semester level, and non-archived terms.
+
+### Phase 4 policies that build on Phase 3 data
+
+Recorded here so the two documents agree. They are **approved policy for Phase 4**
+and are not implemented in Phase 3:
+
+* **Roll numbers** are university-issued and vary by department; no complete
+  format is enforced. Leading and trailing ordinary ASCII spaces (U+0020) are
+  trimmed and other whitespace is rejected wherever it occurs; it is kept as typed, must start with an
+  ASCII Latin letter, is unique ignoring letter case through one PostgreSQL
+  expression, and has a provisional technical limit of 30 characters. It is
+  never a login name or a file name.
+* **Student names and phone.** Students cannot change their own first or last
+  name; they can change only their own phone number. Departmental staff with
+  `student.update` edit a student's phone and address in their department. Names
+  and email stay Main Admin-only.
+* **Programs, sections and terms** are not changed by these policies. A student's
+  placement refers to a program, a semester level and an optional section of that
+  program and level, and an admission term; the section is never tied to a term.
 
 ## 10. Testing summary
 
diff --git a/docs/phase4-student-registration.md b/docs/phase4-student-registration.md
index 45a1513..02168a9 100644
--- a/docs/phase4-student-registration.md
+++ b/docs/phase4-student-registration.md
@@ -1,61 +1,72 @@
 # Phase 4 — Student Registration (design)
 
-**Status:** FINAL DESIGN FOR SIGN-OFF (revised after the owner's answers to the
-nine open questions). Nothing described here is implemented. No migration has
-been written, and the development database is untouched. Implementation starts
-only after separate, explicit approval. Assumptions and points where this design
-touches existing decisions are listed in §19.
+**Status:** DESIGN, reconciled across the project documents and approved for
+implementation planning. **Nothing in this document is implemented yet.** No
+migration has been written and the development database is untouched.
+Implementation starts only after separate, explicit authorization.
 
 **Scope:** manual registration of **one** student at a time: user account,
-student profile and initial placement, in one transaction. Bulk registration
-(Excel/CSV) is Phase 5. No frontend work.
+student profile and initial placement in one transaction, plus the narrow
+student, staff and Main Admin operations listed below. Bulk registration is
+Phase 5. No frontend work.
+
+**How to read the statements.** Where it matters, a statement is labelled:
+
+* **Current implementation:** exists in the code today (Phases 0–3).
+* **Approved policy:** decided; to be implemented in Phase 4.
+* **Planned:** intended for a later phase.
+* **Deferred:** deliberately not built yet.
 
 ## 1. Approved decisions
 
 | Area | Decision |
 |---|---|
 | Identity | `students` has a one-to-one relation with `users` and **shares its primary key** (`students.id` is also a foreign key to `users.id`). Email, names, password, roles and activation stay in `users`. There is no second authentication system. |
-| Login | By email only. The roll number is an identifier, never a username or credential. |
+| Login | By the account's registered email address. The current system does not enforce a NUML-specific email domain. The roll number is an identifier, never a login. |
 | Department | Never stored on the student. Derived: current placement → program → department. |
 | Required fields | first name, last name, email, roll number, program, semester level, admission academic term. |
 | Optional fields | section, phone, address. |
-| **CNIC** | **Completely excluded from Phase 4**: no database column, no API field, no registration support, no encryption implementation. The unanswered business questions are retained (§12). |
+| **CNIC** | **Excluded from Phase 4**: no column, no API field, no registration support, no encryption code. The unanswered business questions are kept (§16). |
 | Profile picture | Deferred. No other personal fields (gender, date of birth, guardian). |
-| Roll number | Globally unique, case-insensitive; trimmed and uppercased; 3–30 characters; no NUML-specific pattern invented; never used for authentication; corrections are Main Admin-only and audited (§13.2). |
-| Phone | E.164 normalization; valid international numbers accepted; Pakistani `03XXXXXXXXX` converted to `+923XXXXXXXXX`; invalid or ambiguous input is rejected, never guessed (§9). |
+| Roll number | University-issued; **the format varies by department and no complete format is enforced.** Leading and trailing ordinary ASCII spaces (U+0020) are trimmed; tabs, newlines, non-breaking spaces and other prohibited whitespace are rejected wherever they occur. Rules in §4. The 30-character limit is a provisional technical limit. |
+| Phone | E.164 normalization; valid international numbers accepted; Pakistani `03XXXXXXXXX` converted to `+923XXXXXXXXX`; invalid or ambiguous input rejected, never guessed. |
 | Address | Optional free text, trimmed, at most 500 characters. |
-| Placement | History preserved; one current placement per student; nullable section; program/level/section consistency enforced by the database; admission term and effective term separate; initial placement created at registration; no hard deletion; no in-place edits. |
-| Movement | **No** general movement, promotion or freeze. Two narrow Main Admin-only correction operations are approved (§13). `section.move_student` is not granted to HODs or staff. |
-| Permissions | Reuse `student.create`, `student.view`, `student.update`. No new codes, no grant changes. Role-bound department authorization is unchanged. |
-| Field visibility | Basic fields to every `student.view` holder; contact fields to `student.create`/`student.update` holders in the department; **account flags to Main Admin only** (§7). |
+| Placement | History preserved; one current placement per student; nullable section; program/level/section consistency enforced by the database; admission term and effective term separate; initial placement created at registration; no hard deletion; no in-place edits; a correction supersedes but never removes the earlier row. |
+| Movement | **No** general movement, promotion or freeze, and no section merge: these are the separately designed roadmap item "Phase 5A" (see the implementation plan). Only two Main Admin-only corrections are in Phase 4 (§17). `section.move_student` stays Main Admin-only and is not granted to HODs or staff. |
+| Permissions | Reuse `student.create`, `student.view`, `student.update`. No new permission codes and no grant changes. The delegation allow-list is **not** expanded. Role-bound department authorization is unchanged. |
+| Reference data for registration | A dedicated read-only endpoint authorized by `student.create` (§11). |
+| Field visibility | Basic fields to every `student.view` holder in the department; contact fields to `student.create`/`student.update` holders in the department; **account flags to Main Admin only** (§9). |
+| Student self-service | **Students cannot change their first or last name. Students can view their own profile and change only their own phone number** through `GET` and `PATCH /students/me` (§13). Supersedes the earlier deferral of `/students/me`. |
 | Instructors | Name, roll number and current placement only. No unrestricted search. |
 | Main Admin search | `GET /api/v1/students`, global, with roll-number lookup. HODs and staff stay department-scoped. |
 | Onboarding | **Option B:** unknown random Argon2id-hashed password; nothing secret is emailed; the email points the student to the existing forgot-password OTP flow, which proves mailbox ownership. Rate-limited, audited resend. |
-| Profile edits | HOD/delegated staff edit phone and address in their department. Names and email stay Main Admin-only (`PATCH /users/{id}`). |
+| Staff editing | HOD/delegated staff with `student.update` edit phone and address in their department. Names and email stay Main Admin-only (`PATCH /users/{id}`). |
 | Deactivation | Only Main Admin, through the existing `POST /users/{id}/deactivate`. No student deactivation endpoint. |
 | Admission term | Required; must exist and must not be ARCHIVED; need not be ACTIVE; separate from the semester level. |
-| Database | Runtime role may update only `ended_at` of `student_placements`; trigger and application protections stay; migration downgrades refuse to drop populated student tables. |
-| Other | First and last name required. Current department controls live access; old departments keep historical audit visibility only. `/students/me` and accounts staff are deferred. |
+| Database | Column-level `UPDATE` privileges (§8); trigger and application protections; downgrades refuse to drop populated student tables. |
+| Locking | One global lock order (§6). |
+| Other | First and last name required. Current department controls live access; old departments keep historical audit visibility only. Accounts-staff access is deferred. |
 
 ## 2. Deliberately out of scope
 
-Bulk import (Phase 5); promotion, freeze and semester-progression automation;
-any general placement-movement endpoint; section merge; student withdrawal or
-graduation lifecycle; **CNIC in any form**; profile pictures; `/students/me`;
-accounts-staff access; courses, enrollments, fees; frontend work. (Real
-onboarding needs a forgot-password screen in the frontend, which does not exist
-yet; the API works without it.)
+Bulk import (Phase 5); promotion, freeze, semester progression, general
+placement movement and section merge (**Phase 5A**, separate design and
+approval); student withdrawal or graduation lifecycle; **CNIC in any form**;
+profile pictures; accounts-staff access; courses, enrollments, fees; frontend
+work. (Real onboarding needs a forgot-password screen in the frontend, which
+does not exist yet; the API works without it.)
 
 ## 3. Data model
 
-Both tables use `RESTRICT` foreign keys. Nothing is hard-deleted.
+*Approved policy.* Both tables use `RESTRICT` foreign keys. Nothing is
+hard-deleted.
 
 ### `students`
 
 | Column | Type | Notes |
 |---|---|---|
 | `id` | UUID, PK | also FK → `users.id` (`fk_students_user_id`), no default: set to the new user's id |
-| `roll_number` | varchar(30), NOT NULL | stored normalized (trim + uppercase) |
+| `roll_number` | varchar(30), NOT NULL | display value with leading and trailing ordinary spaces trimmed, original case preserved (§4). The 30 is a **provisional technical limit**. |
 | `admission_term_id` | UUID, NOT NULL | FK → `academic_terms.id`; the admission batch (for example Fall 2023) |
 | `phone` | varchar(16), NULL | E.164 (`+` and up to 15 digits) |
 | `address` | varchar(500), NULL | trimmed free text |
@@ -63,10 +74,12 @@ Both tables use `RESTRICT` foreign keys. Nothing is hard-deleted.
 
 Constraints and indexes:
 
-* `uq_students_roll_number_lower` — unique on `lower(roll_number)`.
-* `ck_students_roll_number` — `roll_number = upper(roll_number)` and
-  `roll_number ~ '^[A-Z0-9/-]{3,30}$'`: the database refuses an unnormalized or
-  malformed value even if a service bug lets one through.
+* `uq_students_roll_number_upper` — **unique expression index on
+  `upper(roll_number)`**, the single definition of "the same roll number" (§4).
+* `ck_students_roll_number` — database backstop for the rules that PostgreSQL
+  can express: the value has no leading or trailing space, its first character
+  is an ASCII Latin letter, and it contains no ASCII or C1 control character.
+  (The remaining Unicode-category rules are enforced by the application; see §4.)
 * `ck_students_phone` — `phone IS NULL OR phone ~ '^\+[1-9][0-9]{7,14}$'`.
 * `ix_students_admission_term_id`.
 * Account status is `users.is_active` only; there is no `is_active` on `students`.
@@ -85,9 +98,9 @@ history is immutable and the only permitted change is closing a row.
 | `semester_level` | smallint, NOT NULL | CHECK 1–16 |
 | `section_id` | UUID, NULL | no "No Section" rows |
 | `effective_term_id` | UUID, NOT NULL | FK → `academic_terms.id`; the term from which this placement applies. At registration it equals the admission term; afterwards the two are independent. |
-| `change_reason` | varchar(20), NOT NULL | `INITIAL` or `CORRECTION` (check constraint; later phases extend it by migration) |
+| `change_reason` | varchar(20), NOT NULL | `INITIAL` or `CORRECTION` (check constraint). Any other reason needs a later, separately designed migration (Phase 5A). |
 | `changed_by_user_id` | UUID, NOT NULL | FK → `users.id` (who registered or corrected) |
-| `previous_placement_id` | UUID, NULL | self FK; links the chain |
+| `previous_placement_id` | UUID, NULL | the row this one supersedes (§5) |
 | `note` | varchar(500), NULL | required for corrections |
 | `started_at` | timestamptz, NOT NULL | default `now()` (when it was recorded) |
 | `ended_at` | timestamptz, NULL | `NULL` means current |
@@ -101,6 +114,12 @@ Constraints:
   with no section is valid and a placement with a section is fully verified:
   the section must belong to the placement's program **and** semester level.
   Tests prove both behaviours.
+* `uq_student_placements_id_student` — `UNIQUE (id, student_id)` (trivially true;
+  it is the target of the next constraint).
+* `fk_student_placements_previous_same_student` — composite FK
+  `(previous_placement_id, student_id)` → `student_placements(id, student_id)`.
+  A placement can only supersede **a placement of the same student**. It is not
+  checked when `previous_placement_id` is NULL (initial rows).
 * `uq_student_placements_one_current` — partial unique `(student_id) WHERE
   ended_at IS NULL`.
 * `uq_student_placements_one_initial` — partial unique `(student_id) WHERE
@@ -122,59 +141,211 @@ Additive: `uq_sections_id_program_level UNIQUE (id, program_id,
 semester_level)`. It is trivially true (the id is already unique) but is
 required as the target of the composite foreign key. No data changes.
 
-## 4. Placement integrity, history protection and concurrency
+## 4. Roll-number rules
+
+A roll number is a **university-issued identifier supplied at registration**. It
+is not generated, and it is never a login name. NUML roll numbers are not in one
+format: they vary by department and begin with an alphabetic prefix (for example
+`CS…`, `LHR…`, `LCM…`; these are examples, not a specification). The system
+therefore **does not enforce a complete format**, invents no department-specific
+patterns, and builds no prefix-to-department mapping. (If the university later
+supplies an authoritative prefix mapping, a check against the student's
+department could be considered; none exists now.)
+
+| # | Rule (Approved policy) |
+|---|---|
+| 1 | The **first character is an ASCII Latin letter** (`A–Z` or `a–z`). |
+| 2 | **Leading and trailing ordinary ASCII spaces (U+0020) are trimmed.** Tabs, newlines, non-breaking spaces and other prohibited whitespace characters are **rejected wherever they occur, including at the beginning and end**; they are never trimmed. Trimming happens before the other checks. |
+| 3 | After trimming, the value is **not empty** and has **at most 30 characters**. The 30 is a **provisional technical/schema limit pending confirmation, not an official NUML rule**; it can be raised by a migration. |
+| 4 | The value is **preserved as typed**, including letter case and inner characters. It is never uppercased or otherwise rewritten for display or storage. Inner ordinary spaces are kept as typed. |
+| 5 | **Rejected characters:** any character in Unicode categories **Cc** (controls, including tab, newline, NUL and the C1 controls), **Cf** (format characters such as zero-width space, zero-width joiner and non-joiner, and bidirectional formatting controls), **Zl** and **Zp** (line and paragraph separators), and **any whitespace other than the ordinary space U+0020** (for example the non-breaking space). These are rejected **wherever they occur, including at the beginning and the end of the value.** |
+| 6 | **Allowed:** every other printable Unicode character in positions 2 onward, including letters of other scripts such as Urdu. There is **no ASCII-only rule.** |
+| 7 | **Uniqueness is university-wide and case-insensitive**, defined once, in PostgreSQL, as `upper(roll_number)`. |
+| 8 | **Every comparison uses that same SQL expression**: registration duplicate checks, roll-number corrections (§17), lookups and searches (`upper(roll_number) = upper(:value)`). **Python never case-folds a roll number for comparison.** Phase 5 duplicate detection within an uploaded file must also compute its keys in PostgreSQL. |
+| 9 | There are no department-specific regular expressions and no prefix mapping. |
+| 10 | A roll number is **never used as a filesystem path or file name.** Anything stored for a student uses a generated identifier. Reading a roll number out of an uploaded file name (planned, fees phase) is a lookup key only, matched through the same expression. |
+
+**Validation order.** (1) Remove ordinary spaces (U+0020) from both ends of the typed
+value, and nothing else. (2) Reject the result if it is empty. (3) Reject it if it is
+longer than 30 characters (provisional limit). (4) Reject it if its first character is
+not an ASCII Latin letter. (5) Reject it if it contains, **anywhere, including the
+first and last positions**, a character of category Cc, Cf, Zl or Zp, or a whitespace
+character other than an inner ordinary space. (6) Otherwise accept it and store it
+exactly as it now stands. A tab, newline or non-breaking space at either end survives
+step 1, so it is rejected by step 4 or step 5; it is **never silently removed**.
+
+**Where each rule is enforced.** The **application's input validation is the authoritative gate
+for every rule**, including the Unicode-category checks (Cc, Cf, Zl, Zp) and the rejection of
+whitespace other than an ordinary space. PostgreSQL regular expressions cannot classify Unicode
+categories, so the **database does not independently enforce every Unicode-category
+restriction.** As a backstop it enforces only what it can express: the value has no leading or
+trailing ordinary space (PostgreSQL's `btrim` removes only ordinary spaces, which matches rule 2),
+is not empty, fits the 30-character limit, starts with an ASCII Latin letter, and contains no ASCII
+or C1 control character. A zero-width character, bidirectional control, non-breaking space, U+2028
+or U+2029 is therefore stopped by the application only. Uniqueness does not depend on any of this:
+rule 7 is a pure PostgreSQL expression, so the index and every comparison share one definition and
+cannot disagree with each other. A change of the database's locale affects `upper()` and would
+require the index to be rebuilt.
+
+**Tests (when implemented):** a letter-first requirement (digit, symbol and
+non-ASCII first characters rejected); trimming (ordinary spaces at both ends removed,
+inner spaces kept as typed); a leading or trailing tab, newline, non-breaking space
+or other prohibited whitespace **rejected, not trimmed**; each rejected category
+inside the value (zero-width space, zero-width joiner and non-joiner,
+bidirectional override, non-breaking space, U+2028, U+2029, tab, newline, NUL);
+allowed values (an Urdu letter in a later position, digits, `-`, `/`, `.`, an
+inner space); the 30-character boundary (30 accepted, 31 rejected); case
+variants of one roll number rejected as duplicates through both registration and
+correction; display value returned exactly as typed; and a lookup that is
+case-insensitive.
+
+## 5. Placement integrity, history protection and concurrency
 
 **Trigger** (`trg_student_placements_history`, `BEFORE UPDATE OR DELETE`, per
 row), modelled on the audit-log trigger:
 
 * `DELETE` is always rejected.
 * `UPDATE` is allowed **only** when `OLD.ended_at IS NULL`, `NEW.ended_at IS
-  NOT NULL`, and every other column is `IS NOT DISTINCT FROM` its old value.
-  So the current row's `ended_at` can be set exactly once; no historical field
-  can ever change; a closed row cannot be re-opened or edited.
-* `TRUNCATE` is not fired by row triggers; it is prevented by privileges (§6).
+  NOT NULL`, and every other column is `IS NOT DISTINCT FROM` its old value. So
+  the current row's `ended_at` can be set exactly once; no historical field can
+  ever change; a closed row cannot be re-opened or edited.
+* `TRUNCATE` is not fired by row triggers; it is prevented by privileges (§8).
 
 **Three independent layers protect the history:** the trigger, the runtime
-role's column-level `UPDATE (ended_at)` privilege (§6) and the application,
-which only ever closes and inserts.
+role's column-level `UPDATE (ended_at)` privilege (§8) and the application, which
+only ever closes a row and inserts a new one.
+
+**Which kind of row is which.**
+
+| Row | Created by | Previous row | Meaning |
+|---|---|---|---|
+| `INITIAL` | registration only | none | where the student was first placed. Exactly one per student. |
+| `CORRECTION` | the Main Admin placement-correction operation only | required (the row it supersedes) | the corrected version of an entry-error. |
+| any other reason | **nothing in Phase 4** | — | genuine academic movement, promotion and freeze need a separately designed phase (Phase 5A) and a migration that extends the reason check. |
+
+**Supersession.** A correction **supersedes** the row it replaces: the
+correction row determines the student's effective placement from then on. The
+superseded row is **preserved permanently** (closed, never edited, never
+deleted) and stays available for history views and audit; its successor is
+recorded by the correction row's `previous_placement_id`. Nothing is erased or
+ignored.
+
+**The correction window is stated positively.** A correction is allowed only if
+**every** placement of the student is `INITIAL` or `CORRECTION`. Repeated
+corrections are therefore allowed, and the moment a later phase creates any other
+kind of placement the window closes automatically, with no further code change.
 
 **Concurrency: no two current placements for one student.**
 
-1. The partial unique index is the final guard: whatever the application does,
-   a second current row for the same student fails with a unique violation.
-2. Every operation that changes a student's placement first takes the
-   **student row** `FOR UPDATE`. That row is the serialization point, so two
-   concurrent operations queue up instead of racing.
-3. Inside the transaction the order is always *close the current row, then
-   insert the new one* (the unique index is checked immediately).
-4. Correction requests carry the `current_placement_id` the caller saw; if it
-   no longer matches, the answer is 409, so an accidental double submission is
-   never applied twice.
-5. Registration creates the student and its first placement in one
-   transaction, so the only things it can race with are a duplicate roll number
-   or email, which unique constraints turn into a clean 409.
+1. The partial unique index is the final guard: whatever the application does, a
+   second current row for the same student fails with a unique violation.
+2. Every operation that changes a student's placement locks the **student row**
+   `FOR UPDATE`, then the current placement row (§6). That serializes concurrent
+   operations instead of letting them race.
+3. Inside the transaction the order is always *close the current row, then insert
+   the new one* (the unique index is checked immediately).
+4. Correction requests carry the `current_placement_id` the caller saw; if it no
+   longer matches, the answer is 409, so an accidental double submission is never
+   applied twice.
+5. Registration creates the student and its first placement in one transaction,
+   so the only things it can race with are a duplicate roll number or email,
+   which unique constraints turn into a clean 409.
 
 Tests cover: simultaneous corrections of one student (exactly one wins),
-simultaneous registrations with the same roll number or email (one student),
-and raw SQL attempts to create two current placements or edit history (rejected
-by the database).
+simultaneous registrations with the same roll number or email (one student), and
+raw SQL attempts to create two current placements, edit history, or point a
+placement at another student's placement (all rejected by the database).
+
+## 6. Lock order
 
-## 5. Migration design
+**One global acquisition order for every operation:**
 
-Three additive revisions on top of `e9b6f637ed78`, each tested on a scratch
-database before anything is applied to development:
+```text
+term → department (when required) → program → section → student → placement
+```
+
+* *Approved policy for Phase 4.* No operation takes a lock at a later level and
+  then one at an earlier level. A term lock comes first simply because
+  registration and correction take it first; terms are not special.
+* If several rows **at the same level** must be locked, they are taken in
+  **ascending id order**.
+* *Current implementation (Phase 3), consistent with this order:* department,
+  program and section operations already follow department → program → section.
+  The one same-level pair is term activation, which locks its own term and then
+  the single ACTIVE term, without id ordering. It is deadlock-free because the
+  second row is always the one ACTIVE row and no operation holds it while waiting
+  for another term; any future multi-term operation must use ascending id.
+* `SELECT … FOR SHARE` (`S`) lets many registrations proceed together. Any
+  operation that changes the state a registration depends on takes the row
+  `FOR UPDATE` (`X`) and therefore waits for, or is waited for by, the
+  registration.
+
+| Operation | Locks, in order |
+|---|---|
+| Register a student | term (S) → program (S) → section (S, if given) → inserts |
+| Correct a placement | changed effective term (S, if it changes) → new program (S) → new section (S, if given) → student (X) → current placement (X) |
+| Correct admission term | term (S) → student (X) |
+| Correct roll number; edit phone/address; self phone update | the student row (X) |
+| Term lifecycle (update, activate, complete, archive) | term (X) (activation also locks the ACTIVE term, see above) |
+| Department update, activate, deactivate | department (X) |
+| Program create, activate | department (X) → program (X) |
+| Program update, deactivate, duration change | program (X) |
+| Section create | program (X) |
+| Section activate | program (X) → section (X) |
+| Section rename, deactivate | section (X) |
+
+Correction locks only the **new** placement's rows. The old placement's program
+and section are not locked: closing a row cannot invalidate them, so a
+cross-department correction never needs several rows at one level.
+
+**How this prevents invalid concurrent states**
+
+* **Term archived during registration or correction.** Archiving needs the term
+  `X`; registration holds it `S`. Either registration commits first (the student
+  was registered while the term was not archived) or the archive commits first
+  and registration sees ARCHIVED and refuses. A student can never be registered
+  *after* the archive.
+* **Program or section deactivated, or duration shortened, during registration.**
+  Each takes the program or section `X`, then counts current placements (§19).
+  It waits for the registration's `S` lock and then sees the new placement and
+  refuses, or it runs first and registration sees the inactive state or the
+  shorter duration and refuses. The shared lock on the program is also what stops
+  a duration change from invalidating the chosen semester level.
+* **Department deactivated during registration (no department lock).** The
+  department lock may be omitted from registration **only because** two
+  invariants hold: *an inactive department has no active program*, and *an
+  inactive program has no active section*. Department deactivation holds the
+  department `X` and counts active programs; registration's `S` lock keeps its
+  program active, so that count is at least one and the deactivation is refused.
+  The invariants are maintained by the Phase 3 services (not by database
+  constraints), so registration also **validates** the department's and program's
+  `is_active` flags inside its own transaction, and a concurrency test
+  (deactivating a department while a registration is in flight) pins the
+  behaviour.
+* **Two corrections of one student, or a correction racing a section
+  deactivation.** The student row `X` serializes the first; the loser fails the
+  `current_placement_id` check (409). A section's deactivation needs the section
+  `X` and waits for the correction's `S`.
+* **Deadlock freedom.** Shared locks do not conflict with each other, every
+  chain follows the order above, and no operation waits on a lock that a holder
+  of a later level could be waiting to release.
+
+## 7. Migration design
+
+*Approved policy.* Three additive revisions on top of `e9b6f637ed78`, each tested
+on a scratch database before anything is applied to development:
 
 | # | Purpose | Contents |
 |---|---|---|
 | M1 | composite-key target on sections | `uq_sections_id_program_level` |
-| M2 | students | table, roll-number index and check, phone check, FKs |
-| M3 | student placements | table, all constraints and indexes, the history function and trigger |
+| M2 | students | table, roll-number index and checks, phone check, FKs |
+| M3 | student placements | table, all constraints and indexes (including both composite foreign keys and `uq_student_placements_id_student`), the history function and trigger |
 
 * Constraints are named explicitly; `downgrade()` is complete.
-* **Downgrade guard (approved):** because these tables hold personal data,
-  `downgrade()` of M2 and M3 **refuses to run** (explicit error, nothing
-  changed) if the tables contain rows, instead of silently destroying them.
-  Empty tables downgrade normally.
+* **Downgrade guard:** because these tables hold personal data, `downgrade()` of
+  M2 and M3 **refuses to run** (explicit error, nothing changed) if the tables
+  contain rows, instead of silently destroying them. Empty tables downgrade
+  normally.
 * Both models are registered in `migrations/env.py` and `tests/conftest.py`
   (otherwise test truncation would silently skip them).
 * **Test plan (`test_phase4_migrations.py`):** build a scratch database at
@@ -184,47 +355,66 @@ database before anything is applied to development:
   Also: a populated downgrade is refused and leaves everything intact; and a
   fresh-database round trip to base and back.
 * No RBAC seed change (no new permission codes).
+* The existing regression baseline of **547 tests** is a minimum gate after every
+  task.
 
-## 6. Runtime database permissions and no-delete protection
+## 8. Runtime database permissions and no-delete protection
 
-Applied by `scripts/provision_db_roles.py` (a deployment step; **never** run
-against the development database without separate approval):
+*Approved policy*, applied by `scripts/provision_db_roles.py` at deployment
+(**never** to the development database without separate approval):
 
 | Table | Runtime role | Owner/migration role |
 |---|---|---|
-| `students` | `SELECT, INSERT, UPDATE`; **no `DELETE`**, no `TRUNCATE` | all |
-| `student_placements` | `SELECT, INSERT`, and `UPDATE` on the single column **`ended_at`** only; **no `DELETE`**, no `TRUNCATE` | all |
-
-* Both tables are added to `NO_DELETE_TABLES`.
-* For `student_placements` the script revokes table-level `UPDATE` and grants
-  `UPDATE (ended_at)` only. `--check` verifies the table-level privilege is
-  absent and the column privilege present (PostgreSQL's
-  `has_table_privilege` ignores column-only grants; a test confirms this).
-* Consequence for code: the repository closes a placement with a statement that
-  sets only `ended_at`; the ORM must never flush any other attribute of a
-  placement (tests under the restricted role prove it).
+| `students` | `SELECT`, `INSERT`, and `UPDATE` on **only** `roll_number`, `admission_term_id`, `phone`, `address`, `updated_at`; **no `DELETE`**, no `TRUNCATE`. The shared `id` and `created_at` cannot be updated. | all |
+| `student_placements` | `SELECT`, `INSERT`, and `UPDATE` on **only `ended_at`**; **no `DELETE`**, no `TRUNCATE` | all |
+
+* Both tables join `NO_DELETE_TABLES`.
+* **Script design.** *Current implementation:* the script's categories cannot
+  express "update only some columns" (`APPEND_ONLY_TABLES` grants `SELECT` and
+  `INSERT` only, and the default privileges grant table-level `UPDATE`). The
+  design adds a category for **column-restricted tables**, a mapping from table
+  to its updatable columns: the script revokes table-level `UPDATE` and grants
+  `UPDATE (columns)`; `--check` verifies that the table-level privilege is absent
+  and that each listed column privilege is present and no other is. It must be
+  re-run after the migration that creates the tables, because the default
+  privileges grant table-level `UPDATE` first. (PostgreSQL's
+  `has_table_privilege` ignores column-only grants; a test confirms this.)
+* **Row locks still work.** `SELECT … FOR UPDATE/SHARE` needs `UPDATE` on at
+  least one column, which both tables provide.
+* **What the database cannot enforce.** The application connects with one shared
+  runtime role, so the database cannot tell Main Admin from an HOD. "Main Admin
+  only" for roll-number and admission-term corrections remains enforced by
+  application authorization and the audit trail, not by privileges.
+* **Consequence for code:** the repository closes a placement with a statement
+  that sets only `ended_at`; the ORM must never flush any other attribute of a
+  placement, and never touch `students.id` or `created_at`. Tests under the
+  restricted role prove it.
 * Tests (scratch database, restricted role): real registration, a real
-  correction, closing a placement, and failing `DELETE`, `TRUNCATE` and
-  `UPDATE` of any other column; the owner role retains full rights.
-* Phase 1 assignment revocations are unaffected.
+  correction, closing a placement, and failing `DELETE`, `TRUNCATE`, and `UPDATE`
+  of any non-listed column; the owner role retains full rights.
+* Phase 1 assignment revocations are unaffected. `docs/database-roles.md` is
+  updated when the script changes, at implementation.
 
-## 7. Authorization and field visibility
+## 9. Authorization and field visibility
 
 The rule is unchanged: **the same role assignment must grant the permission and
 the target department.** Nothing is combined across roles.
 
 | Operation | Main Admin | HOD (own departments) | Delegated staff | Instructor | Student |
 |---|---|---|---|---|---|
+| Registration reference data | yes | `student.create` | if delegated `student.create` | no | no |
 | Register a student | yes | `student.create` | only if delegated `student.create` | no | no |
 | List / read **basic** fields | yes | `student.view` | if delegated `student.view` | `student.view`, basic only | no |
-| Read **contact** fields | yes | `student.create` or `student.update` | if delegated `student.create` or `student.update` | **no** | no |
+| Read **contact** fields | yes | `student.create` or `student.update` | if delegated `student.create` or `student.update` | **no** | own record only |
 | Read **account flags** | **yes** | **no** | **no** | no | no |
-| Edit phone / address | yes | `student.update` | if delegated `student.update` | no | no |
+| Edit phone / address of a student | yes | `student.update` | if delegated `student.update` | no | no |
+| Edit **own** phone | yes | n/a | n/a | n/a | **yes, own record only** (§13) |
+| Edit own first/last name | yes (existing) | yes (existing) | yes (existing) | yes (existing) | **no** (§13) |
 | Resend onboarding email | yes | `student.create` | if delegated `student.create` | no | no |
 | Global search (`GET /students`) | yes | no (department route only) | no | no | no |
 | Correct roll number / admission term | yes (`user.update`) | no | no | no | no |
 | Correct initial placement | yes (`section.move_student`) | **no** | **no** | no | no |
-| Edit name or email; deactivate account | yes (existing `/users` endpoints) | no | no | no | no |
+| Edit another user's name or email; deactivate an account | yes (existing `/users` endpoints) | no | no | no | no |
 
 **Field tiers** (computed once per request, role-bound):
 
@@ -238,45 +428,51 @@ the target department.** Nothing is combined across roles.
 * **Account flags:** `is_active`, `is_email_verified`, `must_change_password`.
   **Main Admin only.** The department routes never return them to anyone else,
   and the department list has **no `is_active` filter** (it would leak the same
-  information).
-* Search text `q` for basic-tier callers matches name and roll number only, so
-  an instructor cannot probe for emails or phone numbers. Contact-tier callers
-  may also match email.
+  information). Main Admin's global search has it.
+* Search text `q` for basic-tier callers matches name and roll number only, so an
+  instructor cannot probe for emails or phone numbers. Contact-tier callers may
+  also match email. Roll-number matching uses the §4 expression.
 
-**Instructors:** they hold `student.view` role-wide and pass the department
-check for departments they are assigned to, so they can list those students'
-basic fields (name, roll number, placement) and nothing else; they get no global
-search and no contact or flag fields, by construction (separate response
-models, not post-hoc filtering). See §19 A6.
+**Instructors:** they hold `student.view` role-wide and pass the department check
+for departments they are assigned to, so they can list those students' basic
+fields (name, roll number, placement) and nothing else; they get no global search
+and no contact or flag fields, by construction (separate response models, not
+post-hoc filtering).
 
 **No new permission codes and no grant changes.** `student.create`,
 `student.view` and `student.update` are already held by HOD role-wide and are on
 the delegation allow-list, so a receptionist delegated `student.create` in a
-department can register there and only there. `section.move_student` stays Main
-Admin-only. Accounts-staff access is deferred to the fees phase.
-
-**Justifications:** `student.create` authorizes the resend because resending is
-exactly the "give this person access" step of registration; it needs the same
-authority and no more. `user.update` authorizes roll-number/admission-term
+department can register there and only there. **Students are not granted
+`student.update`** or any permission: their self-service is a separate, narrowly
+scoped service (§13). `section.move_student` stays Main Admin-only. Accounts-staff
+access is deferred to the fees phase.
+
+**Justifications:** `student.create` authorizes the resend and the registration
+reference data because both are part of "give this person access"; they need the
+same authority and no more. `user.update` authorizes roll-number/admission-term
 correction because it is the existing Main Admin-only "edit identity data"
 permission that already guards name and email changes. `section.move_student`
 authorizes the placement correction because it is the existing Main Admin-only
 placement permission.
 
-## 8. API (prefix `/api/v1`)
+## 10. API (prefix `/api/v1`)
 
 | Method and path | Permission and scope | Purpose |
 |---|---|---|
+| `GET /departments/{d}/students/registration-options` | `student.create`, department | reference data for the registration form (§11) |
 | `POST /departments/{d}/students` | `student.create`, department | register one student |
 | `GET /departments/{d}/students` | `student.view`, department | list and search (`q`, `roll_number`, `program_id`, `semester_level`, `section_id`; `limit`, `offset`, `total`); fields per tier |
 | `GET /departments/{d}/students/{id}` | `student.view`, department | profile with current placement (fields per tier) |
-| `GET /departments/{d}/students/{id}/placements` | `student.view`, department | placement history, newest first |
+| `GET /departments/{d}/students/{id}/placements` | `student.view`, department | placement history, newest first, including superseded rows |
 | `PATCH /departments/{d}/students/{id}` | `student.update`, department | edit phone and address only (`null` clears) |
-| `POST /departments/{d}/students/{id}/onboarding-email` | `student.create`, department | resend; uniform `202` (§10) |
+| `POST /departments/{d}/students/{id}/onboarding-email` | `student.create`, department | resend; uniform `202` (§14) |
+| `GET /students/me` | authenticated account **with a student profile** | own profile (§13) |
+| `PATCH /students/me` | authenticated account **with a student profile** | own phone only (§13) |
 | `GET /students` | `student.view`, **Main Admin role only** | global search (`q`, `roll_number`, `department_id`, `program_id`, `semester_level`, `section_id`, `is_active`; paginated); all fields |
-| `PATCH /students/{id}/institutional-data` | `user.update`, Main Admin | correct roll number and/or admission term (§13.2) |
-| `POST /students/{id}/placement-corrections` | `section.move_student`, Main Admin | correct the initial placement (§13.1) |
+| `PATCH /students/{id}/institutional-data` | `user.update`, Main Admin | correct roll number and/or admission term (§17.2) |
+| `POST /students/{id}/placement-corrections` | `section.move_student`, Main Admin | correct the initial placement (§17.1) |
 
+* `/students/me` routes are declared before any `/students/{id}` route.
 * Global routes use the existing global-permission dependency, which checks the
   MAIN_ADMIN role specifically (not "any role that holds the code"), so HODs and
   instructors, who hold `student.view`, never reach `GET /students`.
@@ -284,9 +480,38 @@ placement permission.
   `roles`, `is_active`, `department_id`, `status`, `picture` and the like are
   422s. Errors: 403 permission, 404 not found or out of scope, 409 conflicts,
   422 validation, 429 rate limit (with `Retry-After`).
-* No student deactivation endpoint; no `/students/me`.
+* No student deactivation endpoint.
+
+## 11. Registration reference data (registration-options)
+
+`GET /api/v1/departments/{d}/students/registration-options` (*Approved policy*).
+
+**Why it exists.** *Current implementation:* delegated department staff cannot
+call the Phase 3 programs, sections or terms endpoints, because `program.view`,
+`section.view` and `term.view` are not on the delegation allow-list. Without this
+endpoint a receptionist who may register students could not discover the ids the
+registration form needs. The allow-list is **not** expanded and the Phase 3
+endpoints keep their authorization.
+
+**Authorization.** `student.create` in the requested department, role-bound and
+through the existing department dependency (Main Admin passes, including for an
+inactive department; anyone else must hold the permission through an assignment
+that covers the department).
 
-## 9. Registration workflow, validation and transaction
+**It returns only:**
+
+* the **active programs** of the requested department (id, name, code, duration
+  in semesters);
+* for each such program, its **active sections grouped by semester level** (id,
+  name);
+* the **non-archived academic terms** (id, name, term type, academic year,
+  status).
+
+It is read-only, takes no body, and exposes nothing about students, other
+departments, inactive programs or sections, or archived terms. For an inactive
+department the program list is empty.
+
+## 12. Registration workflow, validation and transaction
 
 ### Request
 
@@ -300,8 +525,7 @@ section_id, phone, address                           (optional)
 
 * **Names:** trimmed, inner whitespace collapsed, both required.
 * **Email:** trimmed and lowercased (the login identity); any valid address.
-* **Roll number:** trimmed and uppercased, then `^[A-Z0-9/-]{3,30}$`. No other
-  format is imposed.
+* **Roll number:** the §4 rules.
 * **Phone (optional), E.164, nothing guessed.** After removing spaces, hyphens
   and parentheses (formatting only), the value must be exactly one of:
   * Pakistani local notation `03` followed by nine digits (eleven digits in
@@ -311,40 +535,40 @@ section_id, phone, address                           (optional)
 
   Everything else is rejected with a clear message rather than interpreted:
   digits without `+` (`923001234567`, `3001234567`), the `00` international
-  prefix, any other number beginning with `0` (area-code ambiguity), letters, or
-  a wrong length. This is a *format* check; it does not prove a number exists or
-  belongs to a region, because no phone-numbers library is added.
-* **Address (optional):** trimmed, 1–500 characters; an empty string is treated
-  as absent.
+  prefix, any other number beginning with `0` (area-code ambiguity), letters, or a
+  wrong length. This is a *format* check; it does not prove a number exists,
+  because no phone-numbers library is added. The same rules apply to every
+  operation that sets a phone (registration, staff edit, student self-service).
+* **Address (optional):** trimmed, 1–500 characters; an empty string is treated as
+  absent.
 
 ### Sequence
 
 1. **Authorize** (router dependency): role-bound `student.create` for the path
    department. Unauthorized callers never reach step 2.
 2. **Validate the body** with the strict schema and normalize.
-3. **Lock**, always in the order *department → program → section → (student)*:
-   department `FOR SHARE`, program `FOR SHARE`, section `FOR SHARE` if given.
-   Shared locks let many registrations proceed together; deactivation of any
-   of them takes an exclusive lock and therefore waits for registrations.
+3. **Lock**, in the §6 order: the admission term `FOR SHARE`, the program `FOR
+   SHARE`, and the section `FOR SHARE` if given. There is no department lock; the
+   invariants in §6 are validated instead.
 4. **Validate relationships:** the department exists and is active; the program
    belongs to the path department and is active; `semester_level` ≤
    `duration_semesters`; a given section belongs to that program **and that
    level** and is active; the admission term exists and is not ARCHIVED.
-5. **Validate uniqueness:** roll number, email. An email already used by **any**
-   account is a 409: an existing account is never reused or given the student
-   role here (attaching a role to an existing account is the separately
-   deferred, controlled workflow). Messages are generic ("already registered")
-   and never name a department or person.
-6. **Create**, in the one transaction: the `User` (email, names, random
-   Argon2id password, `is_active=True`, `is_email_verified=False`,
-   `must_change_password=True`), the `STUDENT` role, the `students` row (same
-   id), and the `INITIAL` placement (`effective_term_id` = admission term,
+5. **Validate uniqueness:** roll number (the §4 expression) and email. An email
+   already used by **any** account is a 409: an existing account is never reused
+   or given the student role here (attaching a role to an existing account is the
+   separately deferred, controlled workflow). Messages are generic ("already
+   registered") and never name a department or person.
+6. **Create**, in the one transaction: the `User` (email, names, random Argon2id
+   password, `is_active=True`, `is_email_verified=False`,
+   `must_change_password=True`), the `STUDENT` role, the `students` row (same id),
+   and the `INITIAL` placement (`effective_term_id` = admission term,
    `changed_by_user_id` = actor).
 7. **Audit** (`student.registered`) in the same transaction; **commit**.
 8. **After commit**, the router queues the onboarding email through
    `schedule_email` and returns `201` with the student: basic and contact fields
-   (the caller holds `student.create`), **never account flags** unless the
-   caller is Main Admin.
+   (the caller holds `student.create`), **never account flags** unless the caller
+   is Main Admin.
 
 ### Transactional consistency
 
@@ -354,28 +578,105 @@ section_id, phone, address                           (optional)
   every pre-check, so a concurrent duplicate becomes a 409 and a full rollback,
   never a half-created student.
 * The email is **never** queued before the commit, and an email failure never
-  undoes a committed registration (at-most-once delivery as in Phase 1; the
-  resend and the forgot-password flow are the recovery paths).
+  undoes a committed registration. *Current implementation:* delivery uses FastAPI
+  `BackgroundTasks` through `schedule_email` (in-process, at-most-once, no retry);
+  a Redis-queue (RQ) worker is the planned future mechanism, and `schedule_email` is the
+  primary application integration point for that transition (the worker, dependencies,
+  configuration and deployment setup are needed as well). The resend and the forgot-password flow are the
+  recovery paths.
 * The service is written for Phase 5 reuse: plain inputs, a result object, no
-  HTTP types, one transaction per call; scheduling the email stays in the
-  router layer.
-
-## 10. Onboarding (Option B) and the resend
-
-**Initial credential:** a cryptographically random value (`secrets.
-token_urlsafe(32)`, via a small new helper next to the existing
+  HTTP types, one transaction per call; scheduling the email stays in the router
+  layer.
+
+## 13. Student self-service and the name restriction
+
+*Approved policy.* Students can **view their own profile** and **change only
+their own phone number**. Students cannot change their first name, last name,
+email, roll number, program, semester level, section or admission term, nor their
+roles, permissions or account status. Departmental staff editing (phone and
+address, `student.update`, within the department) is separate and unchanged.
+
+### `GET /api/v1/students/me`
+
+* Requires an authenticated account **with an actual student profile**. A valid
+  account without a student profile (for example a staff member who somehow holds
+  no `students` row) gets 404 and receives no student data.
+* Identity comes **exclusively from the authenticated account**. The endpoint has
+  no student id in the path and none as a query parameter, and it never uses a
+  client-supplied selector. *Following the project's general API convention,
+  unknown query parameters are ignored and cannot change whose data is returned;
+  there is no special query-string rejection.* A test proves that
+  `GET /students/me?student_id=<another student>` still returns the caller's own
+  profile.
+* Returns: first name, last name, registered email, roll number, admission term,
+  phone, address and current placement (program, semester level, section,
+  effective term). It exposes **no account-security flags** (the account's own
+  flags are already available through `/auth/me`).
+
+### `PATCH /api/v1/students/me`
+
+* Same requirement as above (authenticated, with a student profile).
+* Body: exactly `{ "phone": "<number>" }` or `{ "phone": null }`;
+  `extra="forbid"`, and `phone` is required (so `{}` is a 422).
+* Names, email, address, roll number, program, level, section, admission term,
+  roles, permissions, account status and every other field are rejected with 422.
+* Phone follows the §12 rules; `null` clears the number.
+* It updates **only the authenticated student's own record**, in a dedicated,
+  narrowly scoped service operation. It does not use, and students are not
+  granted, `student.update`.
+* Audited as `student.updated` with the changed field **name** only
+  (`fields: ["phone"]`, `self_service: true`), the student as actor, and the
+  student's current department; no value is recorded.
+
+### `PATCH /api/v1/users/me`
+
+* *Current implementation:* any authenticated account, including students, can
+  change its own first and last name (names only; extra fields are rejected).
+* *Approved policy (not yet implemented):* **any account holding the `STUDENT`
+  role, including a multi-role account, gets 403** and nothing changes ("Student
+  names are managed by the university"). Request validation is unchanged and runs
+  first, so malformed, empty or unknown fields remain 422. Accounts **without**
+  the `STUDENT` role keep their existing behaviour. The restriction is enforced in
+  the backend (`UserService.update_own_profile`), not only in the frontend. It
+  checks the role, so the users module needs no dependency on the students module.
+* Institutional name and email changes remain with Main Admin through
+  `PATCH /users/{id}` (audited, as in Phase 1).
+
+### Tests (when implemented)
+
+* A student changing a name through `/users/me` gets 403, the name is unchanged,
+  and no audit row is written; the same for a multi-role account holding `STUDENT`;
+  non-student accounts are unaffected; malformed or extra fields are still 422.
+* `/students/me`: returns only the caller's data; `?student_id=<another>` is
+  ignored and returns the caller's own profile; an account without a student
+  profile gets 404.
+* `PATCH /students/me`: valid and normalized phone, `null` clears, invalid and
+  ambiguous input rejected, every other field rejected (including names, email,
+  address, roll number, program, section, `is_active`, roles), an empty body is a
+  422, and the audit event holds no value.
+* A student cannot reach the department student routes or another student; the
+  staff phone/address edit still requires `student.update` in the department.
+
+## 14. Onboarding (Option B) and the resend
+
+**Initial secret (never delivered):** a cryptographically random value
+(`secrets.token_urlsafe(32)`, via a small new helper next to the existing
 `generate_temporary_password`) hashed with the shared Argon2id `hash_password`.
 Nobody ever sees it; it is never stored in plaintext, returned or logged. The
 account cannot be signed into until the student sets a password.
 
-**Onboarding email** (no password, code, link, CNIC or secret): tells the
-student that an account exists, for which program and department, that the
-sign-in address is the email it was sent to, and that to choose a password they
-should use **Forgot password** on the sign-in page with that address and enter
-the one-time code they receive. It contains no link (there is no verified
-frontend route yet, and plain instructions are harder to imitate).
+This **differs from staff onboarding**, where an HOD creates the account and a
+temporary password is emailed with a mandatory first-login change (*current
+implementation*, Phase 1). A student is **never** emailed a password.
+
+**Onboarding email** (no password, code, link, CNIC or secret): tells the student
+that an account exists, for which program and department, that the sign-in
+address is the email it was sent to, and that to choose a password they should use
+**Forgot password** on the sign-in page with that address and enter the one-time
+code they receive. It contains no link (there is no verified frontend route yet,
+and plain instructions are harder to imitate).
 
-**What the student's action does (existing code, unchanged):** the reset
+**What the student's action does (current implementation, unchanged):** the reset
 request issues a 6-digit OTP (10 minutes, 5 attempts, 60 s cooldown, 5/hour per
 mailbox, per-IP limits); completing it sets the password, clears
 `must_change_password`, sets `is_email_verified = true`, revokes sessions and
@@ -385,51 +686,51 @@ audits the reset. Receiving the OTP is the proof of mailbox ownership.
 
 * **Authorization and scope first.** The caller must hold `student.create` for
   the path department, and the student must currently belong to it; otherwise
-  403/404 exactly as for any student route. The route takes a student id, never
-  an email, so it cannot be used to ask whether an arbitrary address belongs to
-  an account.
+  403/404 exactly as for any student route. The route takes a student id, never an
+  email, so it cannot be used to ask whether an arbitrary address belongs to an
+  account.
 * **Uniform response.** For every authorized, in-scope request the answer is
-  `202 Accepted` with the same body ("If the student is still waiting to
-  complete onboarding, the email has been queued"), whether or not an email is
-  actually sent. The email is sent only if the account is active and
-  `is_email_verified` is false (**resending after verification is refused**
-  internally). This avoids revealing the account flags, which only Main Admin
-  may see; the audit entry records whether it was sent.
-* **Limits, enforced server-side in Redis** (one atomic Lua script that checks
-  all three and consumes them together; keys are SHA-256 digests, never raw
-  ids):
+  `202 Accepted` with the same body ("If the student is still waiting to complete
+  onboarding, the email has been queued"), whether or not an email is actually
+  sent. The email is sent only if the account is active and `is_email_verified`
+  is false (**resending after verification is refused** internally). This avoids
+  revealing the account flags, which only Main Admin may see; the audit entry
+  records whether it was sent.
+* **Limits, enforced server-side in Redis** (one atomic Lua script that checks all
+  three and consumes them together; keys are SHA-256 digests, never raw ids):
   * 1 request per student per 5 minutes;
   * at most 3 per student per 24 hours;
   * at most 20 per initiating user per hour.
 
   Every authorized, in-scope request counts, so the limits cannot be used as an
-  oracle either. A limit returns `429` with `Retry-After`. If Redis is
-  unavailable the request fails and nothing is sent (fail closed).
+  oracle either. A limit returns `429` with `Retry-After`. If Redis is unavailable
+  the request fails and nothing is sent (fail closed).
 * Each request is audited (`student.onboarding_email.resent`, department scope,
   `sent` true/false, no secrets) and the email is queued only after that audit
   commit. Delivery is at-most-once; the endpoint cannot guarantee delivery.
 
-## 11. Audit and email behaviour
+## 15. Audit and email behaviour
 
 | Action | When | Metadata |
 |---|---|---|
 | `student.registered` | registration | roll number, program id, semester level, section id, admission term id |
-| `student.updated` | phone/address edit | the **names of the fields changed only**; never old or new values, and never email, phone or address values |
+| `student.updated` | phone/address edit by staff, or phone edit by the student | the **names of the fields changed only** (and `self_service: true` for the student's own edit); never old or new values, and never email, phone or address values |
 | `student.onboarding_email.resent` | resend request | whether an email was sent; no addresses |
-| `student.institutional_data.corrected` | roll number / admission term correction | **old and new values** of the corrected fields (not sensitive) |
+| `student.institutional_data.corrected` | roll number / admission term correction | **old and new values** of the corrected fields (the display roll number and the term ids) |
 | `student.placement.corrected` | initial-placement correction | old and new placement ids and values, effective term if changed, the written reason |
 
 * Every event carries the student's department (`department_id`) so an HOD sees
   their department's events; entity types `student` and `student_placement`.
+* Audit and the business change are one transaction; the audit service and its
+  append-only design are the Phase 1 infrastructure and are not rebuilt.
 * The existing sanitizer redacts metadata keys containing `cnic`, `password`,
   `token`, `otp`, `secret` or `hash`; audit keys avoid those words.
-* After a department change the old department keeps its historical audit
-  entries (they carry the department at the time of the event), but not live
-  access.
+* After a department change the old department keeps its historical audit entries
+  (they carry the department at the time of the event), but not live access.
 * Emails are built by pure functions (like the Phase 1 staff email), contain no
   secret, are queued only after commit, and failures are logged without the body.
 
-## 12. CNIC — excluded; business questions retained
+## 16. CNIC — excluded; business questions retained
 
 **Decision: CNIC is completely excluded from Phase 4.** There is no database
 column, no API or schema field, no registration support, no validation helper and
@@ -437,8 +738,8 @@ no encryption code. The registration schema rejects a `cnic` field
 (`extra="forbid"`), and the audit sanitizer keeps redacting any metadata key that
 mentions it.
 
-**What the project documents:** the original field list names CNIC; FR-03-3
-lists it among profile fields and FR-03-9 forbids users from editing it; the
+**What the project documents:** the original field list names CNIC; FR-03-3 lists
+it among profile fields and FR-03-9 forbids users from editing it; the
 bulk-import example mentions an "Invalid CNIC" row. **No document says why the
 system must store it.** The following remain **unanswered** and must be settled
 before anyone designs CNIC storage:
@@ -457,44 +758,43 @@ before anyone designs CNIC storage:
 If CNIC is later approved it needs its own design and approval (at minimum
 encryption at rest, a keyed blind index for uniqueness, key management and
 rotation, a dedicated restricted permission). Nothing in the Phase 4 schema
-prevents adding it later as an additive migration. The Phase 5 plan's "Invalid
-CNIC" example row is **not** a Phase 4/5 requirement until this is resolved.
+prevents adding it later as an additive migration. The "Invalid CNIC" example row
+in the Phase 5 plan is **not** a Phase 4/5 requirement until this is resolved.
 
-## 13. Approved corrections (Main Admin only)
+## 17. Approved corrections (Main Admin only)
 
 Both are corrections of data-entry mistakes. They are **not** general placement
-movement, promotion or freeze functionality, and they add no permission codes
-and change no grants.
+movement, promotion or freeze functionality (Phase 5A), and they add no
+permission codes and change no grants.
 
-### 13.1 Initial-placement correction
+### 17.1 Initial-placement correction
 
 `POST /api/v1/students/{id}/placement-corrections`, Main Admin only, using the
 existing `section.move_student`. HODs and departmental staff cannot do it; they
-ask a Main Admin. It is a global route because the corrected program may belong
-to a different department; the service checks both the old and the new
-department.
+ask a Main Admin. It is a global route because the corrected program may belong to
+a different department; the service checks both the old and the new department.
 
-**Request:** `current_placement_id`, `program_id`, `semester_level`,
-`section_id` (nullable), optional `effective_term_id` (default: unchanged), and a
-written `reason` of 10–500 characters.
+**Request:** `current_placement_id`, `program_id`, `semester_level`, `section_id`
+(nullable), optional `effective_term_id` (default: unchanged), and a written
+`reason` of 10–500 characters.
 
 **Preconditions (otherwise 409):**
 
-* **no subsequent placement exists**: every placement the student has is of type
-  `INITIAL` or `CORRECTION` (no other placement type exists in Phase 4; a later
-  phase that adds movement, promotion or freeze rows ends the correction window);
+* **the correction window is open** (§5): every placement the student has is of
+  type `INITIAL` or `CORRECTION`, i.e. there is no genuine academic movement.
+  Repeated corrections are therefore allowed;
 * `current_placement_id` equals the student's current placement (stale or
   repeated submissions are refused);
 * the new program, level and section pass the same validation as registration
-  (active program and section, level ≤ duration, section consistent with
-  program and level), and a given effective term exists and is not ARCHIVED.
+  (active program and section, level ≤ duration, section consistent with program
+  and level), and a given effective term exists and is not ARCHIVED.
 
-**Data effect (history is never modified or deleted):** one transaction, locks
-in the order department → program → section → student → placement: close the
-current row (`ended_at = now()`), insert a new row with `change_reason =
-'CORRECTION'`, `previous_placement_id` = the closed row, `started_at` = now, the
-acting Main Admin, and the reason in `note`. The wrong entry stays in the
-history with its audit trail.
+**Data effect (history is never modified or deleted):** one transaction, locks in
+the §6 order: close the current row (`ended_at = now()`), insert a new row with
+`change_reason = 'CORRECTION'`, `previous_placement_id` = the row it supersedes,
+`started_at` = now, the acting Main Admin, and the reason in `note`. The
+**superseded row stays in the history permanently**; the correction row now
+determines the effective placement.
 
 **Audit:** `student.placement.corrected` with old and new values and the reason.
 If the department changed, one entry is written for each affected department.
@@ -502,23 +802,23 @@ If the department changed, one entry is written for each affected department.
 **Consequence:** the student's department follows the new current placement
 automatically; the old department loses live access and keeps its audit history.
 
-### 13.2 Roll-number and admission-term correction
+### 17.2 Roll-number and admission-term correction
 
 `PATCH /api/v1/students/{id}/institutional-data`, Main Admin only, using
 `user.update`. Body: `roll_number` and/or `admission_term_id`.
 
-* The roll number is normalized (trim, uppercase) and validated against the same
-  rule and the same case-insensitive global uniqueness as at registration
-  (409 on conflict).
+* The roll number is validated by the same §4 rules and the same PostgreSQL
+  uniqueness expression as at registration (409 on conflict); the new display
+  value is stored as typed (after leading and trailing ordinary spaces are trimmed).
 * The admission term must exist and must not be ARCHIVED.
 * The audit event `student.institutional_data.corrected` records **old and new
   values**.
 * This operation changes only `students`. It does not rewrite the initial
-  placement (history is immutable), so `effective_term_id` of the placement can
-  differ from the corrected admission term; the placement correction (§13.1)
+  placement (history is immutable), so the placement's `effective_term_id` can
+  differ from the corrected admission term; the placement correction (§17.1)
   accepts an optional `effective_term_id` for when both must change.
 
-## 14. Account deactivation versus academic status
+## 18. Account deactivation versus academic status
 
 * **Account deactivation** is `users.is_active`. A deactivated account cannot
   authenticate and its sessions end. Only Main Admin may do it, through the
@@ -531,145 +831,134 @@ automatically; the old department loses live access and keeps its audit history.
   current placement; until then every registered student has exactly one current
   placement.
 
-## 15. Phase 3 follow-ups completed in Phase 4
+## 19. Phase 3 follow-ups completed in Phase 4
 
 * `SectionService.deactivate_section` refuses while the section has current
   placements; `ProgramService.deactivate_program` refuses while any current
   placement is in the program; a program's duration cannot drop below the highest
   semester level used by any placement (current or historical), in addition to
   the existing section rule. The counts are made while holding the same row
-  locks, so they cannot race with a registration.
+  locks (§6), so they cannot race with a registration.
 * The user directory (`GET /users`) is intentionally **not** extended to cover
   students; student search lives in the students routes. Main Admin's `/users`
   already lists students as ordinary users.
+* `UserService.update_own_profile` gains the `STUDENT`-role restriction (§13).
 
-## 16. Security notes
+## 20. Security notes
 
 * No public registration endpoint; every route needs authentication and a
-  role-bound permission.
+  role-bound permission (or, for `/students/me`, an authenticated student).
 * No password, code or secret in any response, email, log or audit row; no CNIC
   exists anywhere.
-* The random initial password makes the account unusable until the OTP flow is
+* The random initial secret makes the account unusable until the OTP flow is
   completed; deactivated accounts cannot request or use an OTP.
 * Account flags are never returned to anyone but Main Admin, and the resend
   answers uniformly so it cannot be used to read them.
 * Registration conflicts are generic, to avoid revealing students of other
-  departments (see §19 A5 for the unavoidable remainder).
+  departments. An authorized registrar still learns that an email or roll number
+  is already registered; that cannot be hidden without making registration
+  silently succeed, and it matches Phase 1 staff creation.
 * Mass assignment is prevented by strict schemas; instructors cannot see contact
-  fields or flags by construction (tiered response models).
+  fields or flags by construction (tiered response models); a student cannot
+  reach another student's data because identity comes only from the token.
+* Roll numbers reject invisible and formatting characters (§4) so that two
+  look-alike numbers cannot coexist; a roll number is never a path or file name.
 * Reads of contact data are not audited in Phase 4 (writes are).
 
-## 17. Testing and regression strategy
+## 21. Testing and regression strategy
 
-* **Gate:** the existing **547 tests** stay unchanged and green as a minimum
-  after every task, together with all new Phase 4 tests, ruff, ruff-format, mypy
-  and a single Alembic head.
+* **Gate:** the existing **547 tests** stay unchanged and green as a minimum after
+  every task, together with all new Phase 4 tests, ruff, ruff-format, mypy and a
+  single Alembic head.
 * **Registration:** every validation rule, positive and negative (student without
   a section, level above duration, section/level/program mismatch, inactive
-  program or section, archived admission term, duplicate roll number and email,
-  existing-account email, rejected `cnic`/`password`/`roles`/`is_active` fields);
-  phone cases (valid local, valid international, and every rejected shape in §9).
+  program or section, archived admission term, duplicate roll number (including a
+  case-variant) and email, existing-account email, rejected
+  `cnic`/`password`/`roles`/`is_active` fields); phone cases (valid local, valid
+  international, every rejected shape); the roll-number cases in §4.
+* **Registration reference data:** staff with delegated `student.create` receive
+  only the allowed data; a holder of `student.view` only, an instructor, a student
+  and other-department users are refused; archived terms, inactive programs and
+  sections are absent; the Phase 3 endpoints still refuse that staff member.
+* **Self-service and names:** the cases in §13.
 * **Authorization:** Main Admin, HOD, delegated staff, instructor, student and
-  other-department users; multiple HOD assignments; multiple roles on one
-  account; role-bound pairing (instructor of A plus HOD of B); inactive
-  department; out-of-scope ids returning 404; field tiers (instructor never gets
-  contact fields or flags; HOD and staff never get flags; Main Admin gets all);
+  other-department users; multiple HOD assignments; multiple roles on one account;
+  role-bound pairing (instructor of A plus HOD of B); inactive department;
+  out-of-scope ids returning 404; field tiers (instructor never gets contact
+  fields or flags; HOD and staff never get flags; Main Admin gets all);
   `GET /students` unreachable by HOD, staff and instructor.
-* **Transactions and concurrency:** audit failure rolls back account, profile and
-  placement; email failure does not; registration races; correction races;
-  deactivation racing registration; two raw current placements rejected.
-* **Database level:** all placement constraints, the composite foreign key with
-  and without a section, the trigger (delete and every edit rejected, `ended_at`
-  settable once), restricted-role privileges including `UPDATE (ended_at)` only,
-  no `DELETE`/`TRUNCATE`.
-* **Onboarding and resend:** the initial password is not guessable, not stored in
+* **Transactions, locking and concurrency:** audit failure rolls back account,
+  profile and placement; email failure does not; registration races (same roll
+  number and email); correction races; registration against term archive,
+  section/program deactivation, duration change and **department deactivation**;
+  two raw current placements rejected.
+* **Database level:** all placement constraints, both composite foreign keys (with
+  and without a section; a placement cannot supersede another student's), the
+  trigger (delete and every edit rejected, `ended_at` settable once), restricted
+  role privileges (`UPDATE` of listed columns only, no `DELETE`/`TRUNCATE`).
+* **Onboarding and resend:** the initial secret is not guessable, not stored in
   plaintext, in no response, log, audit row or email; the forgot-password flow
   completes onboarding (verifies the email, clears `must_change_password`); the
   uniform 202; all three limits and their windows under real Redis; refusal after
   verification; fail-closed behaviour; audit entries.
 * **Corrections:** Main Admin only (HOD, staff, instructor refused); preconditions;
-  history preserved (old row closed, never altered or deleted); department change
-  and both audit entries; roll-number uniqueness, normalization and old/new audit.
-* **Migrations:** §5, including populated Phase 1 + Phase 3 data and the refused
+  repeated corrections allowed; history preserved (the superseded row is closed,
+  never altered or deleted, and remains in the placement history); department
+  change and both audit entries; roll-number uniqueness through `upper()`, display
+  value preserved and old/new audit.
+* **Migrations:** §7, including populated Phase 1 + Phase 3 data and the refused
   populated downgrade.
 
-## 18. Implementation sequence
+## 22. Implementation sequence
 
-1. **Documentation:** the `implementation-plan.md` statements are corrected in
-   this revision (see the change list in the review report). Remaining:
-   Phase 4 clarifications in `02_requirements.md`, and updating
-   `phase3-academic-organization.md` §9 to the approved design (it currently
-   says `section.move_student` may be given to HODs and staff, which is no longer
-   the plan; see §19 C1).
+1. **Documentation:** reconciled across the project documents (this document, the
+   implementation plan, the Phase 1 and Phase 3 documents, the SRS and the
+   overview and architecture documents).
 2. **M1:** the `sections` composite unique constraint, with tests.
 3. **M2/M3 and models:** `students`, `student_placements`, trigger, constraints,
-   provisioning (`NO_DELETE_TABLES`, column-level `UPDATE`), model registration,
-   database-level and restricted-role tests, migration tests (with the downgrade
-   guard).
-4. **Phase 3 follow-ups** (§15), so nothing can be broken once placements exist.
-5. **Registration service** (no HTTP): validation, phone normalization, locking,
-   credential, audit; unit and concurrency tests.
-6. **Registration API** and onboarding email; transaction and authorization tests.
+   provisioning (`NO_DELETE_TABLES`, the column-restricted category),
+   model registration, database-level and restricted-role tests, migration tests
+   (with the downgrade guard); update `docs/database-roles.md`.
+4. **Phase 3 follow-ups** (§19), so nothing can be broken once placements exist.
+5. **Registration service** (no HTTP): validation, roll-number and phone rules,
+   locking, initial secret, audit; unit and concurrency tests.
+6. **Registration reference data** and the **registration API** with the
+   onboarding email; transaction and authorization tests.
 7. **Read APIs:** department list/search, profile, placement history, field
    tiers, and the Main Admin global search.
-8. **Profile edit** (phone, address) and the **resend** with its Redis limiter.
-9. **Main Admin corrections:** roll number / admission term (§13.2) and the
-   initial-placement correction (§13.1).
-10. **Close-out:** docs, full verification, scratch-database migration round
-    trips. Development migration, provisioning and commits each wait for
-    separate approval, as in Phase 3.
-
-## 19. Assumptions, and interactions with existing decisions
-
-### Assumptions I made (please confirm or correct)
-
-* **A1. Repeated corrections.** "No subsequent placement exists" is read as "no
-  placement other than the initial one and corrections of it". So a corrected
-  placement can itself be corrected, otherwise a mistaken correction could never
-  be fixed. If a correction row should count as "subsequent", only one correction
-  per student is possible.
-* **A2. Uniform resend response.** The resend returns `202` for every authorized,
-  in-scope request (it does not say whether an email was sent), so it cannot leak
-  the account flags that only Main Admin may see. The audit entry records the
-  outcome.
-* **A3. Redis failure.** If Redis is unavailable the resend fails closed
-  (nothing sent).
-* **A4. Phone strictness.** Anything that is not `03XXXXXXXXX` or a `+`-prefixed
-  E.164 number is rejected (including `00…` and digits without `+`). The check is
-  format-only; no phone-numbers library is added.
-* **A5. Registration conflicts reveal existence to the registrar.** An authorized
-  registrar learns that an email or roll number is already registered (generic
-  message, no department or person). This cannot be hidden without making
-  registration silently succeed, and it matches Phase 1 staff creation.
-* **A6. Instructors can list basic fields.** An instructor assigned to a
-  department can list that department's students' name, roll number and
-  placement through the department route, because `student.view` is role-wide for
-  INSTRUCTOR. They get no contact fields, flags or global search. If you want
-  instructors to see students only through their courses (Phase 6–8), the
-  alternative is to deny them the list route until then.
-* **A7. Admission-term correction.** It changes `students` only and does not
-  rewrite the immutable initial placement (§13.2).
-* **A8. Clearing optional fields.** `PATCH` with `null` clears phone or address.
-* **A9. Department list has no `is_active` filter.** It would expose the account
-  flag; the Main Admin global search has it.
-
-### Where this touches existing decisions (no outright conflict found)
-
-* **C1. `phase3-academic-organization.md` §9** says `section.move_student` could
-  be handed to HODs and delegated staff. The approved decision keeps it Main
-  Admin-only. That document is not changed in this step and must be corrected
-  before implementation (step 1 above).
-* **C2. `user.update` and `section.move_student` get a wider meaning** (student
-  institutional data and initial-placement corrections) but stay Main Admin-only
-  with no grant changes. Phase 1's rule that HODs must not hold `user.update` is
-  untouched, and a departmental need would still require a new narrowly scoped
-  permission.
-* **C3. Existing user directory.** HODs see account flags of staff and instructors
-  in `GET /users` (Phase 1). Students are not in that directory for them, and the
-  student routes hide flags, so the rules do not collide; it is a deliberate
-  difference between the two modules.
-* **C4. Phase 3 "Deferred to Phase 4" items** (placement table, deactivation
-  guards, composite foreign key) are all covered here; the Phase 3 note that the
-  composite key includes the semester level is applied (§3).
-* **C5. Phase 5's shared registration service.** Bulk will call the same service
-  once per row in its own transaction; Phase 4 does not build any bulk behaviour.
+8. **Staff profile edit** (phone, address) and the **resend** with its Redis limiter.
+9. **Student self-service:** `GET`/`PATCH /students/me` and the `STUDENT`-role
+   restriction on `PATCH /users/me`, with their tests.
+10. **Main Admin corrections:** roll number / admission term (§17.2) and the
+    initial-placement correction (§17.1).
+11. **Close-out:** docs, full verification, scratch-database migration round
+    trips. Development migration, provisioning and commits each wait for separate
+    approval, as in Phase 3.
+
+## 23. Consistency with the other documents, and open items
+
+**Decisions that every document must agree on** (checked at reconciliation):
+roll numbers vary by department with no complete format (§4); students cannot
+change their names but can change their own phone (§13); staff edit phone and
+address with `student.update`; Main Admin owns names, email, roll number and
+admission term; `section.move_student` is Main Admin-only and never delegated;
+repeated corrections while every placement is `INITIAL` or `CORRECTION`;
+superseded placements are preserved; the §6 lock order; no password is emailed to
+students; current email delivery is FastAPI `BackgroundTasks` with RQ as the
+planned future architecture; movement, promotion, freeze and section merge are
+Phase 5A.
+
+**Open items (none blocks Phase 4 design):**
+
+* **Roll-number limit.** The 30-character limit is provisional until the
+  university confirms a real maximum.
+* **CNIC.** The six business questions in §16.
+* **Phase 5.** `student.bulk_import` is **not** on the staff delegation allow-list
+  today (it is Main Admin-only and non-delegable), and the plan's demonstration
+  scenario no longer shows it being granted. Whether it becomes delegable is a
+  Phase 5 decision; this design authorizes no change to the permission catalogue
+  or the allow-list.
+* **Phase 5A.** Movement, promotion, freeze and section merge need their own
+  design and approval, including the placement reasons they add.
+* **Database-role documentation.** `docs/database-roles.md` documents the script
+  as it behaves today and is updated when the script changes at implementation.
