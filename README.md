# Authentication Backend Specification

This document outlines the standard API endpoints, required payloads, and internal logic required to build a secure user authentication system. **This specification is entirely technology-agnostic.**

---

## 1. Architectural Assumptions

* **Stateless or Stateful:** The flows below work whether you choose **JWTs (JSON Web Tokens)** or **Session Cookies**.
* **Security:** All endpoints must be served over HTTPS. Passwords must *never* be stored in plain text (always use a strong cryptographic hashing function like bcrypt, Argon2, or PBKDF2 with a unique salt before saving to the database).

---

## 2. Endpoint Reference Table

| Method | Endpoint | Description | Auth Required |
| --- | --- | --- | --- |
| `POST` | `/api/auth/register` | Creates a new user account | No |
| `POST` | `/api/auth/login` | Authenticates credentials and returns a token/session | No |
| `POST` | `/api/auth/logout` | Invalidates the current user session/token | **Yes** |
| `POST` | `/api/auth/forgot-password` | Generates a temporary reset token and sends it via email | No |
| `POST` | `/api/auth/reset-password` | Verifies the reset token and updates the password | No |
| `GET` | `/api/auth/me` | Returns the profile of the currently logged-in user | **Yes** |

---

## 3. Detailed Endpoint Flows

### 🔑 POST `/api/auth/register`

Registers a new user in the system.

* **Expected Input Payload:**
* `email` (String, required, must be validated as a proper email format)
* `password` (String, required, should enforce minimum strength requirements)
* *Optional:* `name`, `username`, etc.


* **Backend Logic:**
1. Validate that the input fields are present and meet formatting rules.
2. Query the database to check if the `email` already exists.
* *If exists:* Return an error (`409 Conflict`).


3. Hash the `password` using a secure hashing algorithm.
4. Save the new user record (Email + Hashed Password) to the database.
5. *Optional:* Generate an authentication token immediately, or require them to log in.


* **Success Response (`201 Created`):**
```json
{
  "success": true,
  "message": "User registered successfully",
  "token": "access_token_string_here", 
  "user": { "id": "unique_user_id", "email": "user@example.com" }
}

```



---

### 🔓 POST `/api/auth/login`

Authenticates a user and establishes a session.

* **Expected Input Payload:**
* `email` (String, required)
* `password` (String, required)


* **Backend Logic:**
1. Find the user record in the database by `email`.
* *If not found:* Return a generic error (`401 Unauthorized`). *Note: Do not explicitly say "Email not found" for security reasons to prevent user enumeration.*


2. Compare the incoming plain-text `password` against the stored `hashed password` using your hashing library's built-in verification function.
* *If mismatch:* Return a generic error (`401 Unauthorized`).


3. Generate the session mechanism:
* **If using Tokens (JWT):** Generate an Access Token (short-lived) and optionally a Refresh Token (long-lived).
* **If using Cookies:** Generate a session ID, store it in your cache/database, and set it in the response header as an `HttpOnly, Secure` cookie.




* **Success Response (`200 OK`):**
```json
{
  "success": true,
  "token": "access_token_string_here", 
  "user": { "id": "unique_user_id", "email": "user@example.com" }
}

```



---

### 🔒 POST `/api/auth/logout`

Terminates the user's active session.

* **Expected Input Payload:** None (Relies on the Auth Header or Cookie sent with the request).
* **Backend Logic:**
* **If using Cookies:** Clear the session cookie from the client's browser and delete the session ID from your backend store.
* **If using Tokens:** (Optional but recommended) Add the current token to a "blacklist" database until its natural expiration time, or simply delete the Refresh Token from the database.


* **Success Response (`200 OK`):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}

```



---

### 📧 POST `/api/auth/forgot-password`

Initiates the password recovery workflow.

* **Expected Input Payload:**
* `email` (String, required)


* **Backend Logic:**
1. Look up the user by `email`.
2. *Security Best Practice:* Regardless of whether the email exists or not, return a `200 OK` success message to prevent attackers from guessing registered emails.
3. If the user *does* exist:
* Generate a unique, random, short-lived **Reset Token** (e.g., valid for 15 minutes).
* Hash and save this token in the database alongside an expiration timestamp tied to the user.
* Trigger an external email service to send a link to the user (e.g., `https://frontend.com/reset-password?token=XYZ`).




* **Success Response (`200 OK`):**
```json
{
  "success": true,
  "message": "If the email exists, a password reset link has been sent."
}

```



---

### 🔄 POST `/api/auth/reset-password`

Consumes the reset token to update the user's password.

* **Expected Input Payload:**
* `token` (String, required)
* `newPassword` (String, required)


* **Backend Logic:**
1. Find the user record associated with the provided `token`.
2. Verify if the token has expired.
* *If invalid or expired:* Return an error (`400 Bad Request`).


3. Hash the `newPassword`.
4. Update the user record with the new hashed password and **delete/invalidate** the reset token so it can't be reused.


* **Success Response (`200 OK`):**
```json
{
  "success": true,
  "message": "Password updated successfully. You can now log in."
}

```



---

### 👤 GET `/api/auth/me`

An example of a protected route. Fetches the current user's profile information.

* **Expected Input Payload:** None (Requires active Auth Token/Session Cookie in request headers).
* **Backend Logic (Middleware):**
1. Intercept the request to read the Authorization header or cookie.
2. Validate the session/token. If invalid, block the request and return `401 Unauthorized`.
3. Extract the `user_id` from the valid token/session.
4. Fetch the user details from the database using that ID.
5. Return the profile data (excluding sensitive fields like the password hash).


* **Success Response (`200 OK`):**
```json
{
  "success": true,
  "user": {
    "id": "unique_user_id",
    "email": "user@example.com",
    "createdAt": "timestamp"
  }
}

```
