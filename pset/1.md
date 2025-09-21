# Exercise 1: Reading a concept

1. Invariants:

- At all times, every Request in a registry must have a count $\geq 0$

- Every purchase is associated to a request, and the Item associated with the purchase should be the same as the request

The first invariant because if the count is negative, people would end up making more purchases than necessary, which defeats the intended purpose. The action affected is purchase, as this preserves sufficient numbrer of purchases of a given item for a request

2. purchase could potentially be invoked with a negative number as an argument for count, which then would increase the count when trying to decrement by a negative number. We can control the argument so that it validates we don't accept negative numbers for count.

3. Yes because you should allow users to open and close it to change the registry without having to create a brand new one.

4. Yes this could matter in practice because you can imagine storing many registries and not deleting them after a certain period would incur extra costs as it scales.

5. **Registry Owner Query:** "What items have been purchased from my registry and by whom?"

- Check the registry's requests and their associated purchases to show the owner what gifts they've received and who gave them.

**Giver Query:** "What items are still available to purchase from this registry?"

- Checks each request in an active registry and calculates the remaining needed quantities to show potential givers what items still need to be purchased.

6. In the state for a set of registries, we'll also add a new property similar
   in behavior to the open flag and call it surpriseMode, which will allow the recepient to enable/disable seeing purchases. We will also add a new action to toggle the state that we can describe as the following:

   setSurpriseMode(registry: Registry, flag: boolean)

7. SKUs allow for a quick and straightforward way to one-dimensionally lookup items fast as opposed to having to check products for properties that match all properties in your query.

# Exercise 2: Extending a familiar concept

1. The state needs to track a set of Users where each User has a username String and a password String. This allows the system to store user credentials during registration and later verify them during authentication by matching the provided username and password against the stored values.

2. register (username: String, password: String): (user: User)

   - requires no user exists with this username
   - effects create a new user with this username and password and add to Users set

   authenticate (username: String, password: String): (user: User)

   - requires a user exists with this username and password
   - effects return the matching user

3. Two users can't have the same username. This is preserved by the register action's "requires no user exists with this username", which prevents creating duplicate usernames. The authenticate action only reads the state and doesn't modify it, so it can't violate this invariant.

4. a set of Users with
   a username String
   a password String
   a confirmed Boolean
   a set of PendingRegistrations with
   a username String
   a password String
   a token String
   actions
   register (username: String, password: String): (user: User, token: String) - requires no user exists with this username and no pending registration exists with this username - effects create a new pending registration with this username, password, and a unique token
   confirm (username: String, token: String): (user: User) - requires a pending registration exists with this username and token - effects create a confirmed user with the username and password from the pending registration

# Exercise 3: Comparing concepts

## Purpose

To provide authentication for accessing GitHub's API and performing Git operations with differing customizable layers of permissions.

## Operational Principle

A user generates a random token string with specific permissions that acts as a bearer token. During authentication, the user provides their username and this token instead of their password. The system validates the token and grants access based on its the assigned permissions for that token.

## State

- tokens: set of Tokens
  - each Token:
    - value: String (randomly generated)
    - owner: User
    - scopes: set of Permission
    - created: Date
    - expires: Date (optional)
    - lastUsed: Date (optional)

## Actions

- createToken(user: User, permissions: set of Permission, expires: Date): Token
  - effects: generate new random token with specified permissions and expiration date
- authenticate(username: String, token: String): User
  - requires: token exists and is valid (not expired + belongs to user)
  - effects: return authenticated user with permissions limited to token permissions
- revokeToken(token: Token):
  - effects: delete token from system

## Differences

1. PAT allows scoped access by controlling what permissions a token may have while PasswordAuthentication gives full account access.

2. Tokens can be individually revoked without affecting other authentication methods or changing passwords.

3. Tokens have expiration dates unlike passwords which typically don't expire automatically.

## Documentation Improvements

I think the documentation as-is describes tokens almost like passwords, using phrases like "treat them like passwords". I think it confuses the user on the differences of tokens that makes them better suited for its use cases as opposed to passwords.

# Exercise 4

## URL Shortener

### Purpose

Create shorter links for long URLs, making them easier to share and track and allowing for supporting automatic generation of URL suffixes and allowing user customization of short codes.

### Operational Principle

When a user provides a long URL, the system will generates or take a short suffix and creates a mapping between the short URL and the original URL. When someone navigates to the short URL, they are redirected to the original long URL.

### State

- mappings: set of Mapping
  - Mapping:
    - shortCode: String
    - originalUrl: String
    - creator: User (optional)
    - created: Date
    - isCustom: Boolean

### Actions

- createShortUrl(originalUrl: String, customCode: String optional): Mapping
  - requires: if customCode provided, no mapping exists with that shortCode
  - effects: create new mapping with either customCode or auto-generated shortCode
- redirect(shortCode: String): String
  - requires: mapping exists for shortCode
  - effects: increment clickCount and return originalUrl
- deleteMapping(shortCode: String, user: User)
  - requires: mapping exists and user is creator
  - effects: remove mapping from system

### Notes

The short codes need to have unique suffixes: for auto-generated codes we cannot generate an existing suffix that is already mapping to something and we also need to prevent users from claiming already-taken shortcodes. The creator field being optional is to allow for the case where people just want to create a shortlink, and we don't care about who the link is associated to.

## Billable Hours Tracking

### Purpose

Records time spent on client projects for billing purposes, tracking work sessions from start to finish with descriptions of work performed. Also handles users forgetting to end the work sessions.

### Operational Principle

An employee starts a session by selecting a project and describing the work, which starts the time tracking. When they end the session, the elapsed time is recorded. If a session is forgotten and left open, it can be manually closed after a specified duration.

### State

- sessions: set of Session
  - Session:
    - employee: User
    - project: Project
    - description: String
    - startTime: DateTime
    - endTime: DateTime (optional)
    - status: SessionStatus (active, completed, corrected)
- projects: set of Project
  - each Project has:
    - name: String
    - client: String
    - isActive: Boolean

### Actions

- startSession(employee: User, project: Project, description: String): Session
  - requires: project is active, employee is not already assigned to another active session
  - effects: create new session with current time as startTime
- endSession(session: Session):
  - requires: session is active
  - effects: set endTime to current time, mark status as completed
- timeoutSession(session: Session):
  - requires: session exists, actualEndTime > session.startTime
  - effects: set endTime to current time, mark status as corrected

### Notes

We don't allow for more than one work session assigned per employee. Also, we create the timeoutSession function to allow the code implementer to essentially determine what the timeout duration is before the timeoutSession function gets called. For example, if 48 hours elapses and the session hasn't ended, it will call the timeoutSession.

## Conference Room Booking

### Purpose

Manage reservations of rooms in a building while preventing conflicts

### Operational Principle

Users make a request to book a room for some time period. The system checks for conflicts and, if none exist, creates a reservation. Users can view, modify, or cancel their bookings.

### State

- bookings: set of Booking
  - Booking:
    - room: Room
    - user: User
    - startTime: DateTime
    - endTime: DateTime
    - purpose: String
    - status: BookingStatus (confirmed, cancelled)
- rooms: set of Room
  - Room:
    - name: String
    - isAvailable: Boolean

### Actions

- createBooking(user: User, room: Room, startTime: DateTime, endTime: DateTime, purpose: String): Booking
  - requires: room is available, no conflicting bookings exist for room during time period, startTime < endTime
  - effects: create new booking
- cancelBooking(booking: Booking, user: User):
  - requires: booking exists, user is booking owner
  - effects: change booking status to cancelled
- checkAvailability(room: Room, startTime: DateTime, endTime: DateTime): Boolean
  - effects: return true or false depending on if room is free during specified time period
- getBookings(room: Room, date: Date): set of Booking
  - effects: returns all bookings for a given room on specified date

### Notes

My definition doesn't delete bookings, rather keeps them as cancelled, allowing this to be used for other data/analytics purposes.
