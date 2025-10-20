# Structure-of-JWT-Token
# Secure Software Development: Lab 2 - JWT Security (Vulnerable vs. Secure)

[cite_start]This repository demonstrates common JSON Web Token (JWT) vulnerabilities and their secure counterparts, based on Lab 2 from Alexandria National University's Cybersecurity Program (Fall 2025)[cite: 161].

[cite_start]The primary focus is on OWASP Top 10 A07:2021 - Identification and Authentication Failures[cite: 161, 163].

## Files in this Repository

* `vuln-server.js`: The **vulnerable** Express server containing critical JWT flaws.
* `secure-server.js`: The **hardened** Express server that fixes these vulnerabilities.
* `init-db.js`: A setup script to create the `users.db` SQLite database with sample users.
* `package.json`: Project dependencies (Express, jsonwebtoken, etc.).

---

## 1. Vulnerabilities in `vuln-server.js` (المشاكل)

The original `vuln-server.js` file was intentionally insecure and contained several critical flaws:

### ❌ Flaw 1: Algorithm Confusion (`alg: "none"`) Attack
The `/admin` endpoint was programmed to explicitly trust tokens that claimed to have no signature.

* **Vulnerable Code:** The server checked if `header.alg.toLowerCase() === 'none'`. If true, it **skipped signature verification** and decoded the payload, trusting any roles or data inside.
* **Impact:** An attacker could forge a token, set the algorithm to `none`, and modify the payload (e.g., `"role": "admin"`) to gain full administrative access.

### ❌ Flaw 2: Weak Signing Secret
The server used a single, hardcoded, and easily guessable secret for all tokens.

* **Vulnerable Code:** `const WEAK_SECRET = 'weak-secret';`
* **Impact:** This secret could be easily brute-forced, allowing an attacker to forge *any* valid token.

### ❌ Flaw 3: Insecure Token Storage (via `localStorage`)
The `/login` endpoint returned the long-lived token directly in the JSON response.

* **Vulnerable Code:** `return res.json({ token });`
* **Impact:** This forces the frontend to store the token in `localStorage` or `sessionStorage`, making it vulnerable to theft via Cross-Site Scripting (XSS) attacks.

### ❌ Flaw 4: Extremely Long Token Expiration
The token was set to expire in 7 days (`expiresIn: '7d'`).

* **Impact:** If a token is stolen, the attacker has a full week to use it before it expires, maximizing the damage.

---

## 2. How `secure-server.js` Fixed These Flaws (الحل)

The `secure-server.js` file implements modern security best practices to mitigate these exact flaws.

### ✅ Fix 1: Preventing `alg: "none"`
The server now **enforces** a specific algorithm and *never* trusts the token's header.

* **Secure Code:** The `authMiddleware` function uses `jwt.verify` and provides a mandatory `algorithms` list:
    ```javascript
    jwt.verify(token, ACCESS_SECRET, { algorithms: ['HS256'] }); //
    ```
* **Effect:** Any token that doesn't have a valid `HS256` signature (including `alg: "none"`) is immediately rejected.

### ✅ Fix 2: Strong, Rotated Secrets
The server uses strong, separate secrets for different token types.

* **Secure Code:** Two different secrets are defined (`ACCESS_SECRET` and `REFRESH_SECRET`).
* **Effect:** This follows the best practice of key separation.

### ✅ Fix 3: Secure Token Storage (HttpOnly Cookies)
The sensitive `refreshToken` is no longer sent in the JSON. It's stored in a secure cookie.

* **Secure Code:** The `/login` endpoint sets an `HttpOnly` cookie:
    ```javascript
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true, //
      secure: true, // (in production)
      sameSite: 'Strict'
    });
    ```
* **Effect:** `httpOnly: true` makes the cookie **inaccessible** to client-side JavaScript, completely protecting it from XSS attacks.

### ✅ Fix 4: Short-Lived Access Tokens
The `accessToken` (which is sent in JSON) is now short-lived, while the sensitive `refreshToken` (in the cookie) is long-lived.

* **Secure Code:** The `accessToken` expires in 15 minutes (`expiresIn: '15m'`).
* **Effect:** If an `accessToken` is stolen, it's only useful for 15 minutes, drastically reducing the risk. The attacker cannot get a new one without the `refreshToken`, which is securely stored in the `HttpOnly` cookie.

---

## How to Run This Lab

1.  **Clone the repository:**
    ```bash
    git clone <your-repo-url>
    cd <your-repo-name>
    ```

2.  **Install dependencies:**
    ```bash
    npm install
    ```

3.  **Initialize the database:**
    ```bash
    npm run init-db
    ```

4.  **Run the Vulnerable Server (for testing):**
    ```bash
    npm run start-vuln
    ```
    * (Server runs on `http://localhost:1234`)
    * (Perform the `alg: "none"` attack on `/admin`)

5.  **Run the Secure Server:**
    ```bash
    npm run start-secure
    ```
    * (Server runs on `http://localhost:1235`)
    * (Try the same attack – it will fail)
