# **🔐 JWT Authentication in Django REST Framework**

## **📌 Introduction**
This document provides a **detailed explanation** of how **JWT (JSON Web Token) authentication** is implemented in **Django REST Framework** using `djangorestframework-simplejwt`. It covers the **entire authentication flow**, including:
- Generating tokens (Login Process)
- How Django verifies tokens in every request
- Logging out and invalidating tokens
- Security considerations

---

# **1️⃣ How JWT Authentication Works in Django**

## **1.1 📌 The Authentication Flow**
1. **User logs in** → Sends **username & password** to `/login/`.
2. **Server validates credentials** → If correct, it generates **JWT tokens** (Access & Refresh tokens).
3. **Client stores the tokens** → In **local storage, cookies, or secure storage (mobile apps)**.
4. **Client makes authenticated requests** → By sending the **Access Token** in the `Authorization` header.
5. **Django verifies the token** → If valid, the request is processed.
6. **Access token expires** → Client uses the **Refresh Token** to get a **new Access Token**.
7. **User logs out** → The **Refresh Token** is blacklisted (cannot generate new access tokens anymore).

---

# **2️⃣ Token Generation: The Login Process**

## **2.1 📌 User Logs In**
The client sends a `POST` request to the `/login/` endpoint with **username** and **password**:
```json
{
  "username": "john_doe",
  "password": "securepassword"
}
```

## **2.2 🔄 Server Validates Credentials**
1. Extracts **username & password** from the request.
2. Queries the database:
   ```sql
   SELECT * FROM users WHERE username = 'john_doe';
   ```
3. If **user exists**, Django **verifies the password hash**:
   ```python
   user.check_password("securepassword")
   ```
4. If credentials **are valid**, Django **creates the JWT token**.

## **2.3 🔑 Creating the JWT Token**
Django generates a **JWT payload**:
```json
{
  "user_id": 1,
  "exp": 1712345678
}
```
- `user_id` → User ID from the database.
- `exp` → Expiration timestamp (1 hour later).

Django **signs the token** using `SECRET_KEY`:
```python
encoded_token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")
```

## **2.4 ✅ Response: Access & Refresh Tokens**
Django returns **two tokens**:
```json
{
  "access": "eyJhbGciOiJIUz...",
  "refresh": "eyJhbGciOiJIUz..."
}
```
- **Access Token** (valid for 1 hour) → Used for API authentication.
- **Refresh Token** (valid for 7 days) → Used to get a new access token.

---

# **3️⃣ How Django Verifies the Token in Every Request**

## **3.1 📌 Client Makes an Authenticated Request**
Every request to a **protected endpoint** includes the Access Token in the `Authorization` header:
```http
GET /users/ HTTP/1.1
Host: 127.0.0.1:8000
Authorization: Bearer eyJhbGciOiJIUz...
```

## **3.2 🔄 Django Verifies the Token**
When Django receives the request:
1. **Extracts the token** from the `Authorization` header.
2. **Decodes the token** (checks if it's valid & not tampered with).
3. **Verifies expiration (`exp`)** → Rejects if expired.
4. **Fetches the user** using `user_id`.
5. **Attaches `request.user = User(id=1, username="john_doe")`**.
6. **If token is valid** → Django allows the request.
7. **If token is invalid** → Django returns:
   ```json
   {
     "detail": "Invalid token."
   }
   ```

---

# **4️⃣ Logout Process (Blacklisting Tokens)**

## **4.1 📌 Why Can't We Just Delete the Token?**
- **JWTs are stateless** → They are not stored on the server.
- Just deleting them from the frontend **doesn't prevent reuse**.
- Solution? **Blacklist the Refresh Token** so it can't be used again.

## **4.2 🔄 Logout Process**
1. The user **sends a logout request** with the Refresh Token:
   ```json
   {
     "refresh": "eyJhbGciOiJIUz..."
   }
   ```
2. Django **adds the Refresh Token to the blacklist**.
3. If the user **tries to refresh their access token**, Django rejects it:
   ```json
   {
     "detail": "Token is blacklisted."
   }
   ```

## **4.3 ✅ Secure Logout Implementation**
```python
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class LogoutView(APIView):
    def post(self, request):
        try:
            refresh_token = request.data.get("refresh")
            token = RefreshToken(refresh_token)
            token.blacklist()
            return Response({"message": "Successfully logged out"}, status=status.HTTP_205_RESET_CONTENT)
        except Exception:
            return Response({"error": "Invalid token"}, status=status.HTTP_400_BAD_REQUEST)
```

---

# **5️⃣ Summary**

| Step | Action | Result |
|------|--------|--------|
| **1️⃣ User logs in (`/login/`)** | Sends `username` & `password` | Receives Access & Refresh Tokens |
| **2️⃣ User makes API requests** | Includes `Authorization: Bearer access_token` | Django verifies token & allows request |
| **3️⃣ Access Token expires** | Client sends Refresh Token to `/token/refresh/` | Gets a new Access Token |
| **4️⃣ User logs out (`/logout/`)** | Sends Refresh Token | Django **blacklists the token** |
| **5️⃣ Token reuse is blocked** | Blacklisted tokens **can't be used** | User must re-login |

---
