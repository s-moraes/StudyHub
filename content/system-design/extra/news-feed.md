# Design a News Feed

Assemble a personalised, ranked timeline for hundreds of millions of users.

## Architecture
- **Fan-out-on-write (Push):** Posts are pushed to followers' inboxes upon publishing. Fast reads, but slow for celebrity accounts.
- **Fan-out-on-read (Pull):** Timeline is computed when the user opens the app.
