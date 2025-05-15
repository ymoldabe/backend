## General Requirements
### Does the code comply with gofumpt formatting rules?
- [ ] Yes
- [ ] No

### Does the program compile successfully with the specified build command?
- [ ] Yes
- [ ] No

### Does the program handle errors without unexpected exits or panics?
- [ ] Yes
- [ ] No

### Does the project use only allowed external packages (github.com/mattn/go-sqlite3)?
- [ ] Yes
- [ ] No

### Does the test coverage meet the minimum 20% requirement?
- [ ] Yes
- [ ] No

## Hexagonal Architecture
### Is the domain layer properly isolated from external dependencies?
- [ ] Yes
- [ ] No

### Are the infrastructure adapters (database, S3, external API) properly implemented?
- [ ] Yes
- [ ] No

### Is the user interface layer correctly separated from business logic?
- [ ] Yes
- [ ] No

## Session Management
### Does the application properly handle user sessions using cookies?
- [ ] Yes
- [ ] No

### Do sessions expire after one week as specified?
- [ ] Yes
- [ ] No

### Is user data (avatars, names) correctly associated with sessions?
- [ ] Yes
- [ ] No

## Post Management
### Does the system correctly create and store posts with all required fields?
- [ ] Yes
- [ ] No

### Are posts without comments deleted after 10 minutes?
- [ ] Yes
- [ ] No

### Are posts with comments deleted 15 minutes after the last comment?
- [ ] Yes
- [ ] No

## Data Storage
### Does the application correctly store data in PostgreSQL?
- [ ] Yes
- [ ] No

### Are images properly stored in S3-compatible storage using multiple buckets?
- [ ] Yes
- [ ] No

### Is the Rick and Morty API integration implemented correctly for avatars?
- [ ] Yes
- [ ] No

## Logging
### Is the log/slog package properly implemented throughout the application?
- [ ] Yes
- [ ] No

### Do logs include appropriate context information?
- [ ] Yes
- [ ] No

### Are different log levels (Info, Warning, Error) used appropriately?
- [ ] Yes
- [ ] No

## Frontend Integration
### Do all provided templates render correctly?
- [ ] Yes
- [ ] No

### Is the archive system functioning as specified?
- [ ] Yes
- [ ] No

### Does the comment system work properly with replies and IDs?
- [ ] Yes
- [ ] No

## Project Presentation and Code Defense
### Can the team explain their implementation of hexagonal architecture and its benefits?
- [ ] Yes
- [ ] No

### Can the team demonstrate their understanding of session management and data flow?
- [ ] Yes
- [ ] No

### Can the team explain their testing strategy and coverage choices?
- [ ] Yes
- [ ] No
