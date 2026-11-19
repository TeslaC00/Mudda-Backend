# Mudda

Backend for a civic complaint platform. Citizens report local issues (broken roads, garbage, water problems and so on), others can comment and like, and government accounts handle them.

I built this as my final-year project. The team had a frontend side too, but this repo is only the backend, and the backend is the part I wrote. There is no UI here. You use it through the REST API.

## What it does

- Citizens create issues with media attachments, comment on them, reply, and like comments.
- Government users see and update issues for their area. Administrators manage users and roles. They can update issue status and mark as Resolved.
- Uploaded issue is checked by an AI moderation step before it is accepted. If flagged it sends you notification to edit it.
- Users get email for email authentication.

## Stack

| Area                 | What I used                                              |
| -------------------- | -------------------------------------------------------- |
| Language / framework | Java 21, Spring Boot 4                                   |
| Database             | PostgreSQL (10 tables)                                   |
| Security             | Spring Security, JWT and session auth, role-based access |
| AWS                  | S3 (media), SQS + Lambda (moderation), SES (email)       |
| Push notifications   | Firebase Cloud Messaging                                 |
| Testing              | JUnit, Spring Boot Test, MockMvc, k6 for load tests      |
| Build                | Maven                                                    |

## API

Main groups (47 endpoints)

- Auth: login, refresh, logout
- Account: get and update account info
- Issues: create, list, get, delete
- Comments and replies: post, like
- Media: upload, remove and link media

## How the main pieces work

### Authentication

The app supports two ways to authenticate: session-based and JWT. JWT for mobile app and session for webapp. It uses an extra flag in header to identify the client type. Accounts get locked after repeated failed logins, and requests are rate limited.

### Post and moderation

Post(called issues in project) goes to PostgreSQL and media to S3. A message is then put on SQS and a Lambda function runs the AI moderation check. An event is generated and sent back to server if the Issue was flagged, that sends a notification to the user.

### Events

Some data is derived: complaint statistics and comment counts. Instead of updating these inside every service method, the services publish Spring Application Events and listeners update the aggregates. This keeps the core logic from knowing about the bookkeeping.

## Tests

Most of my tests are integration tests that go through a controller and check the whole flow, for example: log in, create an issue, comment, like, delete.

```bash
./mvnw test
```

There is no coverage report. I would not claim a coverage figure.

## Load testing

I ran k6 against a flow that logs in, reads account info, creates an issue, posts a comment, likes it, posts a reply, lists the user's issues, refreshes the token and deletes the issue. Every check passed in all runs.

| Run | Peak users | Duration | Requests | Failed | Median | p95    | Throughput |
| --- | ---------- | -------- | -------- | ------ | ------ | ------ | ---------- |
| 1   | 50         | 5 min    | 18,290   | 0%     | 12 ms  | 867 ms | ~58 req/s  |
| 2   | 50         | 5 min    | 18,540   | 0%     | 9.8 ms | 782 ms | ~61 req/s  |
| 3   | 150        | 6 min    | 30,560   | 0%     | 95 ms  | 3.72 s | ~81 req/s  |

My threshold was p95 under 500 ms, and all three runs missed it. Most requests are fast, but a slice of them is slow. When I tripled the users, throughput only went from about 61 to 81 req/s while p95 went up several times, so something is saturating. Nothing failed, it just got slower.

I have not found the bottleneck yet. Suspects: password hashing on login, the database connection pool, the deduplication check.

## Known limitations

- Backend only, no frontend in this repo.
- Not deployed anywhere. It runs locally.
- The load-test bottleneck is unidentified (see above).
- Unit test coverage is thin, as described in Tests.
