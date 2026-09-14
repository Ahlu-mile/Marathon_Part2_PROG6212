# RaceDay - Part 2: RESTful API Development

**Student Number:** ST10488271
**Module:** PROG6212
**Part:** 2 of 3 - RESTful API Development

## System Description

RaceDay is a full-stack, API-driven event management platform for the South African road running, walking, and cycling community. Part 1 planned the system (ERD, endpoint plan, SQL script). This Part 2 submission builds the RESTful API in ASP.NET Core that powers the entire system, exactly following the Part 1 plan (see `docs/Part2_Endpoint_Plan_Deviations.md` for the two minor, explained deviations).

### Roles
- **Organiser** - creates, edits, and deletes events; manages event categories; captures participant results; and views all enrolments for events they own.
- **Participant** - registers an account, browses events, enters events by selecting a category, views their own enrolments, and tracks their own result history.

## Project Structure

```
Marathon_Part2_ST10488271_PROG6212/
├── RaceDay.sln
├── RaceDay.API/              <- the Web API project
│   ├── Controllers/          <- Auth, Users, Events, Categories, Enrolments, Results
│   ├── Models/                <- EF Core entities (match the Part 1 ERD exactly)
│   ├── Data/ApplicationDbContext.cs
│   ├── DTOs/                  <- request/response shapes
│   ├── Security/              <- RequireAuthAttribute, RequireRoleAttribute, SessionKeys
│   ├── Services/PasswordHasher.cs
│   └── Program.cs
├── RaceDay.Tests/             <- xUnit test project
├── .github/workflows/ci-part2.yml
└── docs/                      <- Part 1 artefacts (ERD, endpoint plan, SQL) + Part 2 deviation notes
```

## How Authentication & Role-Based Access Work

This API uses **server-side session state**, not JWT tokens, per the brief's explicit requirement to maintain the user's authenticated state and role via session management:

1. `POST /api/auth/register` creates a user with a hashed password (PBKDF2-SHA256, random salt per user - see `Services/PasswordHasher.cs`). The password is never stored or logged in its original form.
2. `POST /api/auth/login` verifies the password hash and, on success, writes `UserId` and `Role` into `HttpContext.Session`. ASP.NET Core issues a session cookie to the caller.
3. Every subsequent request on that session automatically carries the cookie. Two action filters read the session on each request:
   - `RequireAuthAttribute` - returns `401 Unauthorized` if there is no active session.
   - `RequireRoleAttribute("Organiser")` / `RequireRoleAttribute("Participant")` - returns `403 Forbidden` if the session's role doesn't match.
4. Ownership checks (e.g. an organiser can only edit/delete their own events) are enforced in the controller itself by comparing the session's `UserId` to the resource's owner ID.

This means Organiser functionality is never reachable by a Participant session and vice versa, and no endpoint other than register/login/public browsing is reachable without a session at all.

## Database (EF Core Code-First)

`ApplicationDbContext` defines the same six entities as the Part 1 ERD - `Users`, `Events`, `Categories`, `Enrolments`, `Results`, `RouteInfo` - with the same primary keys, foreign keys, and constraints (unique email, CHECK constraints on `Role`/`EventType`/`Status`, unique participant+category enrolment, unique one-result-per-enrolment).
