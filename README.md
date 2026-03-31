# MyTPE API Documentation

Base URL: `https://bank.mytpe.app/api`

---

## Quick Start — Notaire Workflow

This is the step-by-step guide to go from zero to accepting payments as a notaire.

### Step 1: Register

```
POST /trader/signup
```

```json
{
  "full_name": "Maître Ahmed Benali",
  "email": "ahmed@cabinet-benali.dz",
  "password": "securePassword123",
  "password_confirmation": "securePassword123",
  "phone": "0555000000"
}
```

Save the `token` from the response. Use it as `Authorization: Bearer {token}` in all following requests.

---

### Step 2: Create Identification

This creates your notary office identity. A **brand is automatically created** for you (using your `raison_sociale` as the brand name).

```
POST /trader/RegistreCommerce/store
```

```
Content-Type: multipart/form-data

raison_sociale: "Cabinet Notarial Benali"
type_etablissement: "profession_liberale"
type_autorisation_liberale: "notary"
activite_id: 1
identification_file: (optional PDF, max 5MB)
```

> You now have an identification + a brand, both in `pending` status. No need to wait for admin approval to continue.

---

### Step 3: Get Your Identification ID

You need the identification ID to create a bank account.

```
GET /trader/RegistreCommerce
```

Save the `id` from the response (this is your `registre_commerce_id`).

---

### Step 4: Create Bank Account

Link your RIB (20-digit Algerian bank account number) to your identification.

```
POST /trader/BankAccount/store
```

```json
{
  "rib": "00299999000000001234",
  "registre_commerce_id": "your-identification-id"
}
```

Save the `id` from the response (this is your `bank_account_id`).

> You can validate your RIB first with `POST /trader/rib-bank` (optional).

---

### Step 5: Check Account Setup

Verify everything is in place before creating a MytpePay instance.

```
GET /trader/mytpe-pay/check-account
```

Expected response:

```json
{
  "has_valid_identification": true,
  "has_valid_brand": true,
  "has_valid_bank_account": true
}
```

All three must be `true`. Since the brand was auto-created in Step 2 and pending resources are accepted, this should pass right away.

---

### Step 6: Get Your Brand ID

```
GET /trader/RegistreCommerce
```

Look at the identification from Step 3. Use its `id` to find the associated brand, or list brands separately. The brand was auto-created with the same name as your `raison_sociale`.

You can also find it via:

```
GET /trader/mytpe-pay/check-account
```

Or query brands directly if an endpoint is available. The brand's `registre_commerce_id` matches your identification ID.

---

### Step 7: Create MytpePay Instance

```
POST /trader/mytpe-pay/store
```

```
Content-Type: multipart/form-data

name: "Paiements Cabinet Benali"
description: "Accepter les paiements pour les services notariaux"
logo: (optional image, jpg/png, max 2MB)
registre_commerce_id: "your-identification-id"
bank_account_id: "your-bank-account-id"
brand_id: "your-brand-id"
```

The instance is created in `pending` status. Save the instance `id`.

---

### Step 8: Pay Activation Fee

Check the price first:

```
GET /trader/mytpe-pay/price
```

Then pay using one of three methods:

**Option A — Online payment:**
```
POST /trader/mytpe-pay/{instance-id}/pay
```
→ Redirects to payment gateway. After payment, check status with `POST /trader/mytpe-pay/transaction/fetch`.

**Option B — Cheque:**
```
POST /trader/mytpe-pay/{instance-id}/pay/cheque
```
```
cheque_number: "123456"
payment_doc: (optional file)
```

**Option C — Bank transfer:**
```
POST /trader/mytpe-pay/{instance-id}/pay/bank-transfer
```
```
payment_doc: (required file, proof of transfer)
```

After payment, the instance moves to `processing` → admin reviews → `active`.

---

### Step 9: Create Payment Links

Once your instance is `active`, create payment links that your clients can use.

```
POST /trader/mytpe-pay/links/store
```

**Fixed amount link (e.g. notary fee of 5000 DA):**

```json
{
  "mytpe_pay_id": "your-instance-id",
  "title": "Frais d'authentification",
  "details": "Paiement pour service d'authentification notariale",
  "type": "dynamic",
  "amount": 5000,
  "slug": "authentification-benali",
  "payment_mode": "reusable"
}
```

**Open amount link (client enters the amount):**

```json
{
  "mytpe_pay_id": "your-instance-id",
  "title": "Paiement libre",
  "details": "Paiement pour services notariaux",
  "type": "static",
  "slug": "paiement-benali",
  "payment_mode": "reusable"
}
```

Your clients can now pay at: `https://mytpe.app/pay/{slug}`

---

### Summary

| Step | Endpoint | What happens |
|------|----------|-------------|
| 1 | `POST /trader/signup` | Create account, get token |
| 2 | `POST /trader/RegistreCommerce/store` | Create identification + brand (auto) |
| 3 | `GET /trader/RegistreCommerce` | Get identification ID |
| 4 | `POST /trader/BankAccount/store` | Link bank account |
| 5 | `GET /trader/mytpe-pay/check-account` | Verify setup (all 3 = true) |
| 6 | — | Get brand ID (auto-created in step 2) |
| 7 | `POST /trader/mytpe-pay/store` | Create MytpePay instance |
| 8 | `POST /trader/mytpe-pay/{id}/pay` | Pay activation fee |
| 9 | `POST /trader/mytpe-pay/links/store` | Create payment links |

> **No waiting required between steps 2–7.** Identification, brand, and bank account can all be pending. Admin approval happens in the background. You only need to wait for the instance to become `active` (after payment + admin review) before creating payment links.

---

## 1. Authentication

### 1.1 Login

**`POST /signin`**

Authenticate a user and receive an access token.

**Middleware:** None (public)

**Request Body:**

| Field       | Type   | Required | Rules            |
|-------------|--------|----------|------------------|
| `email`     | string | Yes      | valid email      |
| `password`  | string | Yes      |                  |

**Success Response — `200 OK`**

```json
{
  "token": "1|abc123...",
  "userInfo": {
    "id": "uuid",
    "full_name": "John Doe",
    "email": "john@example.com",
    "phone": "0555000000",
    "typable_type": "Trader",
    "typable_id": "uuid",
    "email_verified_at": "2024-01-01T00:00:00.000000Z",
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z"
  },
  "roles": ["trader"],
  "permissions": ["identification-get-all", "bank-account-get-all", "..."],
  "date_expiration": "2025-01-01T00:00:00.000000Z"
}
```

**Error Responses:**

- **`401 Unauthorized`** — Invalid credentials

```json
{
  "message": "The provided credentials are incorrect."
}
```

- **`422 Unprocessable Entity`** — Validation error

```json
{
  "message": "The email field is required.",
  "errors": {
    "email": ["The email field is required."],
    "password": ["The password field is required."]
  }
}
```

---

### 1.2 Register Trader

**`POST /trader/signup`**

Create a new trader account.

**Middleware:** `guest`

**Request Body:**

| Field                  | Type   | Required | Rules                          |
|------------------------|--------|----------|--------------------------------|
| `full_name`            | string | Yes      | max:255                        |
| `email`                | string | Yes      | valid email, unique in `users` |
| `password`             | string | Yes      | min:8, confirmed               |
| `password_confirmation`| string | Yes      | must match `password`          |
| `phone`                | string | Yes      | max:255, unique in `users`     |

**Success Response — `201 Created`**

```json
{
  "user": {
    "id": "uuid",
    "full_name": "John Doe",
    "email": "john@example.com",
    "phone": "0555000000",
    "typable_type": "Trader",
    "typable_id": "uuid",
    "email_verified_at": "2024-01-01T00:00:00.000000Z",
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z"
  },
  "roles": ["trader"],
  "token": "1|abc123...",
  "permission": ["identification-get-all", "bank-account-get-all", "..."]
}
```

**Error Responses:**

- **`422 Unprocessable Entity`** — Validation error

```json
{
  "message": "The email has already been taken. (and 1 more error)",
  "errors": {
    "email": ["The email has already been taken."],
    "password": ["The password confirmation does not match."],
    "phone": ["The phone has already been taken."],
    "full_name": ["The full name field is required."]
  }
}
```

---

### 1.3 Check Token

**`POST /check/signin`**

Verify if a bearer token is still valid.

**Middleware:** None (public)

**Headers:**

| Header          | Value              |
|-----------------|--------------------|
| `Authorization` | `Bearer {token}`   |

**Request Body:** None

**Success Response — `200 OK`**

```json
{
  "message": "Token is valid.",
  "userInfo": {
    "id": "uuid",
    "full_name": "John Doe",
    "email": "john@example.com",
    "phone": "0555000000",
    "typable_type": "Trader",
    "typable_id": "uuid",
    "email_verified_at": "2024-01-01T00:00:00.000000Z",
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z"
  }
}
```

**Error Responses:**

- **`401 Unauthorized`** — Missing token

```json
{
  "message": "Token is missing."
}
```

- **`401 Unauthorized`** — Invalid/expired token

```json
{
  "message": "Invalid token."
}
```

---

### 1.4 Get User Details

**`GET /user`**

Get the authenticated user's details, roles, permissions, and pack expiration status.

**Middleware:** `auth:sanctum`

**Headers:**

| Header          | Value              |
|-----------------|--------------------|
| `Authorization` | `Bearer {token}`   |

**Request Body:** None

**Success Response — `200 OK` (Trader)**

```json
{
  "expiration": "no expiré",
  "expiration_message": null,
  "userInfo": {
    "id": "uuid",
    "full_name": "John Doe",
    "email": "john@example.com",
    "phone": "0555000000",
    "typable_type": "Trader",
    "typable_id": "uuid",
    "email_verified_at": "2024-01-01T00:00:00.000000Z",
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z",
    "parent": null
  },
  "roles": ["trader"],
  "permissions": ["identification-get-all", "bank-account-get-all", "..."],
  "date_expiration": "2025-01-01T00:00:00.000000Z"
}
```

**Success Response — `200 OK` (Pack expired)**

```json
{
  "expiration": "expiré",
  "expiration_message": "Votre pack Pro a expiré",
  "userInfo": { "..." },
  "roles": ["trader"],
  "permissions": ["..."],
  "date_expiration": null
}
```

**Success Response — `200 OK` (Child user, parent pack expired)**

```json
{
  "expiration": "expiré",
  "expiration_message": "Le pack illimité de votre parent a expiré",
  "userInfo": { "..." },
  "roles": ["trader"],
  "permissions": ["..."],
  "date_expiration": null
}
```

**Success Response — `200 OK` (Child user, parent not on unlimited)**

```json
{
  "expiration": "expiré",
  "expiration_message": "Votre parent n'a pas un pack illimité, accès refusé",
  "userInfo": { "..." },
  "roles": ["trader"],
  "permissions": ["..."],
  "date_expiration": null
}
```

**Error Responses:**

- **`401 Unauthorized`** — Not authenticated

```json
{
  "message": "Unauthenticated."
}
```

---

### 1.5 Forgot Password

**`POST /forgot_password`**

Send a password reset link to the user's email.

**Middleware:** `guest`

**Request Body:**

| Field   | Type   | Required | Rules       |
|---------|--------|----------|-------------|
| `email` | string | Yes      | valid email |

**Success Response — `200 OK`**

```json
{
  "message": "We have emailed your password reset link.",
  "data": null
}
```

**Error Responses:**

- **`422 Unprocessable Entity`** — User not found or throttled

```json
{
  "message": "We can't find a user with that email address.",
  "data": null
}
```

- **`422 Unprocessable Entity`** — Validation error

```json
{
  "message": "The email field is required.",
  "errors": {
    "email": ["The email field is required."]
  }
}
```

---

### 1.6 Reset Password

**`POST /reset-password`**

Reset the user's password using the token received via email.

**Middleware:** `guest`

**Request Body:**

| Field                  | Type   | Required | Rules                 |
|------------------------|--------|----------|-----------------------|
| `token`                | string | Yes      |                       |
| `email`                | string | Yes      | valid email           |
| `password`             | string | Yes      | min:8, confirmed      |
| `password_confirmation`| string | Yes      | must match `password` |

**Success Response — `200 OK`**

```json
{
  "message": "Password reset successfully.",
  "status": true
}
```

**Error Responses:**

- **`400 Bad Request`** — Invalid/expired token

```json
{
  "message": "This password reset token is invalid.",
  "status": false
}
```

- **`422 Unprocessable Entity`** — Validation error

```json
{
  "message": "The token field is required.",
  "errors": {
    "token": ["The token field is required."],
    "email": ["The email field is required."],
    "password": ["The password confirmation does not match."]
  }
}
```

---

### 1.7 Update Password

**`POST /password/update`**

Update the authenticated user's password. Revokes all tokens and logs out all other devices.

**Middleware:** `auth:sanctum`, `verifiedEmail`

**Headers:**

| Header          | Value              |
|-----------------|--------------------|
| `Authorization` | `Bearer {token}`   |

**Request Body:**

| Field                  | Type   | Required | Rules                 |
|------------------------|--------|----------|-----------------------|
| `currentpassword`      | string | Yes      |                       |
| `password`             | string | Yes      | min:8, confirmed      |
| `password_confirmation`| string | Yes      | must match `password` |

**Success Response — `200 OK`**

```json
{
  "message": "Votre mot de passe a été modifié avec succès!",
  "data": null
}
```

**Error Responses:**

- **`419`** — Current password incorrect

```json
{
  "data": null,
  "message": "Votre mot de passe actuel ne correspond pas à celui enregistré."
}
```

- **`419`** — New password same as current

```json
{
  "data": null,
  "message": "Le nouveau mot de passe doit être différent de votre mot de passe actuel."
}
```

- **`401 Unauthorized`** — Not authenticated

```json
{
  "message": "Unauthenticated."
}
```

- **`422 Unprocessable Entity`** — Validation error

```json
{
  "message": "The password field is required.",
  "errors": {
    "currentpassword": ["The currentpassword field is required."],
    "password": ["The password confirmation does not match."]
  }
}
```

---

### 1.8 Update Email

**`POST /email/update`**

Update the authenticated user's email address. A verification email is sent to the new address.

**Middleware:** `auth:sanctum`, `verifiedEmail`

**Headers:**

| Header          | Value              |
|-----------------|--------------------|
| `Authorization` | `Bearer {token}`   |

**Request Body:**

| Field                | Type   | Required | Rules                          |
|----------------------|--------|----------|--------------------------------|
| `email`              | string | Yes      | valid email, unique in `users`, confirmed |
| `email_confirmation` | string | Yes      | must match `email`             |
| `password`           | string | Yes      | min:8                          |

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": "Email changé avec succès ,verifier votre boite mail pour valider et complete la modification"
}
```

**Error Responses:**

- **`419`** — Incorrect password

```json
{
  "data": null,
  "message": "Doit etre entre un mot de passe correcte"
}
```

- **`422 Unprocessable Entity`** — Validation error

```json
{
  "message": "The email has already been taken.",
  "errors": {
    "email": ["The email has already been taken.", "The email confirmation does not match."],
    "password": ["The password field is required."]
  }
}
```

- **`401 Unauthorized`** — Not authenticated

```json
{
  "message": "Unauthenticated."
}
```

---

### 1.9 Resend Email Verification

**`POST /email/resend-verification`**

Resend the email verification notification.

**Middleware:** `auth:sanctum`, `throttle:6,1` (max 6 requests per minute)

**Headers:**

| Header          | Value              |
|-----------------|--------------------|
| `Authorization` | `Bearer {token}`   |

**Request Body:** None

**Success Response — `200 OK`**

```json
{
  "message": "email sent successfully",
  "data": null
}
```

**Error Responses:**

- **`401 Unauthorized`** — Not authenticated

```json
{
  "message": "Unauthenticated."
}
```

- **`429 Too Many Requests`** — Rate limit exceeded

```json
{
  "message": "Too Many Attempts."
}
```

---

## 2. Identification

All identification endpoints require the `Authorization: Bearer {token}` header.

---

### 2.1 List All Identifications

**`GET /trader/RegistreCommerce`**

**Success Response — `200 OK`**

```json
{
  "message": "Registers fetched successfully",
  "data": [
    {
      "id": "uuid",
      "raison_sociale": "Cabinet Notarial Doe",
      "identification_file": "uuid/abc123random.pdf",
      "type_etablissement": "profession_liberale",
      "type_autorisation_etablissement": "notary",
      "trader_id": "uuid",
      "statut_id": 1,
      "file_url": "https://bank.mytpe.app/api/registre-commerce/uuid/file-url"
    }
  ]
}
```

**Error:** `401` if not authenticated.

---

### 2.2 Show Identification

**`GET /trader/RegistreCommerce/show/{id}`**

**Success Response — `200 OK`**

```json
{
  "message": "registers fetched successfully",
  "data": {
    "id": "uuid",
    "raison_sociale": "Cabinet Notarial Doe",
    "identification_file": "uuid/abc123random.pdf",
    "type_etablissement": "profession_liberale",
    "type_autorisation_etablissement": "notary",
    "trader_id": "uuid",
    "statut_id": 1,
    "file_url": "https://bank.mytpe.app/api/registre-commerce/uuid/file-url"
  }
}
```

**Error:** `404` if not found or doesn't belong to you.

---

### 2.3 Create Identification

**`POST /trader/RegistreCommerce/store`**

Use `multipart/form-data` when uploading a file.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `raison_sociale` | string | Yes | Name of the office (must be unique) |
| `type_etablissement` | string | Yes | Always `profession_liberale` |
| `type_autorisation_liberale` | string | Yes | Always `notary` |
| `identification_file` | file | No | PDF, max 5MB |

**Example request:**

```
raison_sociale: "Cabinet Notarial Doe"
type_etablissement: "profession_liberale"
type_autorisation_liberale: "notary"
identification_file: (PDF file)
```

> **Note:** A brand (marque) is automatically created alongside the identification using the `raison_sociale` as the brand name. No need to create one separately.

**Success Response — `200 OK`**

```json
{
  "message": "message",
  "data": null
}
```

**Error Responses:**

- **`422`** — Validation error

```json
{
  "message": "The raison sociale has already been taken.",
  "errors": {
    "raison_sociale": ["The raison sociale has already been taken."]
  }
}
```

- **`403`** — Pack limit reached

```json
{
  "data": null,
  "message": "Identification limit reached for your current pack."
}
```

---

### 2.4 Update Identification

**`POST /trader/RegistreCommerce/update/{id}`**

Updates the identification and resets its status to `pending`.

**Request Body:** Same as create (see 2.3). `raison_sociale` uniqueness check excludes the current record.

**Success Response — `200 OK`**

```json
{
  "message": "L'établissement a été modifié avec succès.",
  "data": {
    "id": "uuid",
    "raison_sociale": "Cabinet Notarial Doe (updated)",
    "identification_file": "uuid/abc123random.pdf",
    "type_etablissement": "profession_liberale",
    "type_autorisation_etablissement": "notary",
    "trader_id": "uuid",
    "statut_id": 1,
    "file_url": "https://bank.mytpe.app/api/registre-commerce/uuid/file-url"
  }
}
```

**Error Responses:**

- **`419`** — Not the owner

```json
{
  "data": null,
  "message": "Vous n'êtes pas autorisé à modifier ce registre de commerce."
}
```

- **`419`** — Already accepted (cannot modify)

```json
{
  "data": null,
  "message": "Nous avons déjà accepté ce registre de commerce, vous ne pouvez pas le supprimer ou modifier."
}
```

- **`422`** — Validation error (same format as create)

- **`404`** — Not found

---

### 2.5 Get Accepted Identifications

**`GET /trader/RegistreCommerce/accept`**

Returns only identifications that have been accepted by admin (`statut_id = 2`). You need an accepted identification to create a bank account.

**Success Response — `200 OK`**

```json
{
  "message": "Registres récupérés avec succès.",
  "data": [
    {
      "id": "uuid",
      "raison_sociale": "Cabinet Notarial Doe",
      "type_etablissement": "profession_liberale",
      "type_autorisation_etablissement": "notary",
      "statut_id": 2,
      "file_url": "https://bank.mytpe.app/api/registre-commerce/uuid/file-url"
    }
  ]
}
```

---

### 2.6 Get Identification File URL

**`GET /registre-commerce/{id}/file-url`**

Returns a signed download URL (valid 1 hour) for the uploaded identification PDF.

**Success Response — `200 OK`**

```json
{
  "message": "Signed URL generated",
  "data": {
    "url": "https://bank.mytpe.app/api/registre-commerce/uuid/download?signature=abc123&expires=1234567890"
  }
}
```

**Error Responses:**

- **`403`** — Not the owner
- **`404`** — No file attached or identification not found

---

## 3. Bank Account

All bank account endpoints require the `Authorization: Bearer {token}` header.

A bank account links a **RIB** (20-digit Algerian bank account number) to one of your accepted identifications. The bank is automatically detected from the RIB.

---

### 3.1 List All Bank Accounts

**`GET /trader/BankAccount`**

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": [
    {
      "id": "uuid",
      "rib": "00299999000000001234",
      "trader_id": "uuid",
      "statut_id": 1,
      "registre_commerce_id": "uuid",
      "bank_id": 1,
      "registre_commerce": {
        "id": "uuid",
        "raison_sociale": "Cabinet Notarial Doe"
      },
      "statut": { "id": 1, "status": "pending" },
      "bank": { "id": 1, "bank_name": "CPA", "code_bank": "002" }
    }
  ]
}
```

**Error:** `401` if not authenticated.

---

### 3.2 Show Bank Account

**`GET /trader/BankAccount/show/{id}`**

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": {
    "id": "uuid",
    "rib": "00299999000000001234",
    "trader_id": "uuid",
    "statut_id": 1,
    "registre_commerce_id": "uuid",
    "bank_id": 1,
    "registre_commerce": {
      "id": "uuid",
      "raison_sociale": "Cabinet Notarial Doe"
    },
    "statut": { "id": 1, "status": "pending" },
    "bank": { "id": 1, "bank_name": "CPA", "code_bank": "002" }
  }
}
```

**Error:** `404` if not found or doesn't belong to you.

---

### 3.3 Create Bank Account

**`POST /trader/BankAccount/store`**

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `rib` | string | Yes | Exactly 20 digits, valid RIB checksum, unique |
| `registre_commerce_id` | string | Yes | Must be one of your accepted identifications |

**Example request:**

```json
{
  "rib": "00299999000000001234",
  "registre_commerce_id": "uuid-of-your-identification"
}
```

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": {
    "id": "uuid",
    "rib": "00299999000000001234",
    "trader_id": "uuid",
    "bank_id": 1,
    "registre_commerce_id": "uuid",
    "statut_id": 1
  }
}
```

**Error Responses:**

- **`422`** — Validation error

```json
{
  "message": "The rib has already been taken. (and 1 more error)",
  "errors": {
    "rib": ["The rib has already been taken.", "The rib is invalid."],
    "registre_commerce_id": ["The registre commerce id field is required."]
  }
}
```

- **`404`** — Identification not found or doesn't belong to you

```json
{
  "data": null,
  "message": "Identification introuvable."
}
```

- **`403`** — Pack limit reached

```json
{
  "data": null,
  "message": "Bank account limit reached for your current pack."
}
```

---

### 3.4 Update Bank Account

**`POST /trader/BankAccount/update/{id}`**

**Request Body:** Same as create (see 3.3). RIB uniqueness check excludes the current record.

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": {
    "id": "uuid",
    "rib": "00299999000000005678",
    "trader_id": "uuid",
    "bank_id": 2,
    "registre_commerce_id": "uuid",
    "statut_id": 1
  }
}
```

**Error Responses:**

- **`419`** — Not the owner

```json
{
  "data": null,
  "message": "Vous n'êtes pas autorisé à modifier ce compte bancaire ."
}
```

- **`404`** — Identification not found

```json
{
  "data": null,
  "message": "Identification introuvable."
}
```

- **`422`** — Validation error (same format as create)

- **`404`** — Bank account not found

---

### 3.5 List Bank Accounts by Identification

**`GET /trader/RegistreCommerce/{id}/BankAccount`**

Returns all bank accounts linked to a specific identification.

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": [
    {
      "id": "uuid",
      "rib": "00299999000000001234",
      "trader_id": "uuid",
      "statut_id": 1,
      "registre_commerce_id": "uuid",
      "bank_id": 1,
      "bank": { "id": 1, "bank_name": "CPA", "code_bank": "002" }
    }
  ]
}
```

**Error:** `404` if identification not found.

---

### 3.6 Validate RIB

**`POST /trader/rib-bank`**

Validate a RIB and get the associated bank info. Useful to check before creating a bank account.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `rib_initials` | string | No | 1-3 digits (bank code prefix) |
| `rib` | string | No | 18-20 digits (full or partial RIB) |

**Example request:**

```json
{
  "rib_initials": "002"
}
```

**Success Response — `200 OK`**

```json
{
  "message": "success",
  "data": {
    "bank": { "id": 1, "bank_name": "CPA", "code_bank": "002" },
    "validRib": true
  }
}
```

**Error Responses:**

- **`421`** — RIB initials don't match full RIB

```json
{
  "data": null,
  "message": "not the same initials"
}
```

- **`421`** — Invalid bank code

```json
{
  "data": null,
  "message": "error"
}
```

---

## 4. MytpePay

MytpePay lets you create payment instances — each instance is linked to one of your accepted identifications, a bank account, and a brand. Once created, you pay the activation fee (online, cheque, or bank transfer), and after approval your instance becomes active. You can then create payment links under it.

All MytpePay endpoints require the `Authorization: Bearer {token}` header.

**Instance statuses:** `pending` → `processing` → `active` (or `refused` / `inactive`)

---

### 4.1 Check Account Setup

**`GET /trader/mytpe-pay/check-account`**

Check if your account has the prerequisites to create a MytpePay instance: an identification, a brand, and a bank account. Resources can be in any status (including pending) — they do not need to be accepted first.

**Success Response — `200 OK`**

```json
{
  "has_valid_identification": true,
  "has_valid_brand": true,
  "has_valid_bank_account": true
}
```

All three must be `true` before you can create an instance. Since a brand is auto-created with each identification, you only need to create an identification and a bank account.

**Error:** `401` if not authenticated.

---

### 4.2 Get Price

**`GET /trader/mytpe-pay/price`**

Get the activation price for a MytpePay instance (may vary by user/pack).

**Success Response — `200 OK`**

```json
{
  "amount": 100
}
```

**Error:** `401` if not authenticated.

---

### 4.3 Create Instance

**`POST /trader/mytpe-pay/store`**

Use `multipart/form-data` when uploading a logo.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `name` | string | Yes | max:255 |
| `description` | string | No | |
| `logo` | file | No | jpg/png/jpeg, max 2MB |
| `registre_commerce_id` | string | Yes | must be an accepted identification |
| `bank_account_id` | string | Yes | must be an accepted bank account |
| `brand_id` | string | Yes | must be an accepted brand |

**Example request:**

```
name: "My Notary Office Payments"
description: "Accept payments for notary services"
logo: (image file)
registre_commerce_id: "uuid-of-identification"
bank_account_id: "uuid-of-bank-account"
brand_id: "uuid-of-brand"
```

**Success Response — `201 Created`**

```json
{
  "message": "MytpePay instance created successfully",
  "data": {
    "id": "uuid",
    "name": "My Notary Office Payments",
    "description": "Accept payments for notary services",
    "logo": "trader/mytpe-pay/logos/random.png",
    "application_id": "ext-app-id",
    "status": "pending",
    "registre_commerce_id": "uuid",
    "bank_account_id": "uuid",
    "brand_id": "uuid",
    "user_id": "uuid",
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z"
  }
}
```

**Error Responses:**

- **`422`** — Validation error

```json
{
  "message": "The name field is required. (and 2 more errors)",
  "errors": {
    "name": ["The name field is required."],
    "registre_commerce_id": ["The selected registre commerce id is invalid."],
    "bank_account_id": ["The selected bank account id is invalid."],
    "brand_id": ["The selected brand id is invalid."]
  }
}
```

- **`500`** — Instance creation failed (license/application error)

```json
{
  "error": "Failed to create license"
}
```

---

### 4.4 Update Instance

**`POST /trader/mytpe-pay/update`**

Update the name, description, or logo of an existing instance. You cannot change the identification, bank account, or brand.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `id` | string | Yes | must be a valid instance ID |
| `name` | string | No | max:255 |
| `description` | string | No | |
| `logo` | file | No | jpg/png/jpeg, max 2MB |

**Success Response — `200 OK`**

```json
{
  "message": "MytpePay instance updated successfully",
  "data": {
    "id": "uuid",
    "name": "Updated Name",
    "description": "Updated description",
    "logo": "trader/mytpe-pay/logos/new-random.png",
    "status": "pending",
    "registre_commerce_id": "uuid",
    "bank_account_id": "uuid",
    "brand_id": "uuid",
    "user_id": "uuid"
  }
}
```

**Error Responses:**

- **`422`** — Validation error
- **`403`** — Not the owner

```json
{
  "error": "Unauthorized"
}
```

---

### 4.5 List Instances

**`GET /trader/mytpe-pay`**

Returns a paginated list of your MytpePay instances with `collected_amount` (total revenue) and your `member_role`.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `per_page` | integer | Items per page (default: 10) |
| `status` | string | Filter: `pending`, `processing`, `active`, `inactive`, `refused` |
| `search` | string | Search by name or description |
| `registre_commerce_id` | string | Filter by identification |
| `bank_account_id` | string | Filter by bank account |
| `brand_id` | string | Filter by brand |

**Success Response — `200 OK`**

```json
{
  "current_page": 1,
  "data": [
    {
      "id": "uuid",
      "name": "My Notary Office Payments",
      "description": "Accept payments for notary services",
      "logo": "trader/mytpe-pay/logos/random.png",
      "status": "active",
      "registre_commerce_id": "uuid",
      "bank_account_id": "uuid",
      "brand_id": "uuid",
      "user_id": "uuid",
      "collected_amount": "15000.00",
      "member_role": "owner",
      "registre_commerce": { "id": "uuid", "raison_sociale": "Cabinet Notarial Doe" },
      "bank_account": { "id": "uuid", "rib": "00299999000000001234" },
      "brand": { "id": "uuid", "name": "My Brand" }
    }
  ],
  "last_page": 1,
  "per_page": 10,
  "total": 1
}
```

---

### 4.6 List Active Instances

**`GET /trader/mytpe-pay/active`**

Returns only your active instances (status = `active`). Useful for dropdowns when creating payment links.

**Success Response — `200 OK`**

```json
{
  "message": "Active instances retrieved successfully",
  "data": {
    "current_page": 1,
    "data": [
      {
        "id": "uuid",
        "name": "My Notary Office Payments",
        "status": "active",
        "registre_commerce": { "id": "uuid", "raison_sociale": "Cabinet Notarial Doe" },
        "bank_account": { "id": "uuid", "rib": "00299999000000001234" }
      }
    ],
    "total": 1
  }
}
```

---

### 4.7 Delete Instance

**`DELETE /trader/mytpe-pay/{id}`**

Delete an instance. All payment links under it must be deleted first.

**Success Response — `200 OK`**

```json
{
  "message": "MytpePay instance and all associated data deleted successfully"
}
```

**Error Responses:**

- **`403`** — Not the owner

```json
{
  "error": "Unauthorized"
}
```

- **`500`** — Instance has payment links

```json
{
  "error": "CANNOT_DELETE_INSTANCE_HAS_PAYLINKS|This instance has 3 payment links that must be deleted first."
}
```

---

### 4.8 Pay Online

**`POST /trader/mytpe-pay/{id}/pay`**

Initiate online payment for instance activation. Returns a payment URL to redirect the user to.

**Request Body:** None

**Success Response — `200 OK`**

```json
{
  "message": "Payment initiated. Please complete payment.",
  "payment_url": "https://payment-gateway.example.com/pay/abc123"
}
```

**Error Responses:**

- **`400`** — Already paid or not pending

```json
{
  "error": "This instance is already paid or inactive."
}
```

---

### 4.9 Pay by Cheque

**`POST /trader/mytpe-pay/{id}/pay/cheque`**

Submit cheque payment. Provide either a cheque number or a payment document (or both).

Use `multipart/form-data` when uploading a file.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `cheque_number` | string | No | max:255 |
| `payment_doc` | file | No | pdf/jpg/jpeg/png, max 2MB |

At least one of `cheque_number` or `payment_doc` is required.

**Success Response — `200 OK`**

```json
{
  "message": "Cheque payment submitted successfully"
}
```

**Error Responses:**

- **`400`** — Already paid or not pending
- **`422`** — Neither cheque number nor document provided

```json
{
  "error": "Either cheque number or payment document is required."
}
```

---

### 4.10 Pay by Bank Transfer

**`POST /trader/mytpe-pay/{id}/pay/bank-transfer`**

Submit bank transfer proof. A payment document is required.

Use `multipart/form-data`.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `payment_doc` | file | Yes | pdf/jpg/jpeg/png, max 2MB |

**Success Response — `200 OK`**

```json
{
  "message": "Bank transfer submitted. Waiting for approval."
}
```

**Error Responses:**

- **`400`** — Already paid or not pending
- **`422`** — Missing payment document

```json
{
  "message": "The payment doc field is required.",
  "errors": {
    "payment_doc": ["The payment doc field is required."]
  }
}
```

---

### 4.11 Fetch Transaction Status

**`POST /trader/mytpe-pay/transaction/fetch`**

Check the payment status of an instance activation transaction. Use the `order_number` returned after initiating online payment.

**Request Body:**

| Field | Type | Required |
|-------|------|----------|
| `order_number` | string | Yes |

**Success Response — `200 OK` (completed)**

```json
{
  "message": "Transaction Found",
  "transaction": {
    "id": 1,
    "transactionable_id": "uuid",
    "transactionable_type": "App\\Models\\MytpePay",
    "order_number": "abc123",
    "amount": 100,
    "status": "completed",
    "payment_method": "online",
    "gateway_message": "Approved",
    "order_id": "order-id",
    "approval_code": "123456",
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z"
  }
}
```

**Success Response — `200 OK` (status updated)**

```json
{
  "message": "Transaction status updated.",
  "transaction": { "..." }
}
```

**Error Responses:**

- **`422`** — Missing order_number
- **`404`** — Transaction not found

---

### 4.12 Get Refusal Motif

**`GET /trader/mytpe-pay/motif/{id}`**

Get the reason why a payment was refused (for instances with `pending` status after a refused payment).

**Success Response — `200 OK`**

```json
{
  "motif": "Invalid cheque number",
  "transaction_date": "2024-01-01T00:00:00.000000Z"
}
```

**Error Responses:**

- **`400`** — Instance is not pending

```json
{
  "error": "This instance is not in pending state."
}
```

- **`404`** — No refused transaction found

```json
{
  "error": "No refused transaction found."
}
```

---

### 4.13 Get Pay Links for Instance

**`GET /trader/mytpe-pay/{id}/links`**

Returns a paginated list of payment links under a specific instance, with `collected_amount` per link.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `per_page` | integer | Items per page (default: 10) |
| `search` | string | Search by title, details, or slug |
| `status` | string | Filter: `active`, `inactive` |
| `type` | string | Filter by link type |
| `sort_by` | string | `collected_amount` (default) or `title` |

**Success Response — `200 OK`**

```json
{
  "message": "Pay links retrieved successfully",
  "data": {
    "current_page": 1,
    "data": [
      {
        "id": "uuid",
        "title": "Notary Service Fee",
        "slug": "notary-service-fee",
        "amount": 5000,
        "status": "active",
        "type": "fixed",
        "collected_amount": "25000.00",
        "created_at": "2024-01-01T00:00:00.000000Z"
      }
    ],
    "total": 1
  }
}
```

---

### 4.14 Global Stats

**`GET /trader/mytpe-pay/links/global-stats`**

Get high-level statistics across all your instances.

**Success Response — `200 OK`**

```json
{
  "totalInstances": 3,
  "totalActiveInstances": 2,
  "totalInactiveInstances": 1,
  "totalActiveLinks": 8
}
```

---

### 4.15 Transaction Stats

**`GET /trader/mytpe-pay/stats`**

Get transaction statistics (completed transactions only).

**Success Response — `200 OK`**

```json
{
  "transaction_count": 45,
  "total_amount": "150000.00",
  "mytpe_pay_count": 3
}
```

---

### 4.16 Profile Stats

**`GET /trader/mytpe-pay/profile-stats`**

Combined overview stats for the dashboard.

**Success Response — `200 OK`**

```json
{
  "total_instances": 3,
  "active_instances": 2,
  "total_links": 8,
  "total_transactions": 45,
  "total_revenue": "150000.00"
}
```

---

### 4.17 Recent Transactions

**`GET /trader/mytpe-pay/transaction/recent`**

Paginated list of recent transactions across all your payment links.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `per_page` | integer | Items per page (default: 10) |

**Success Response — `200 OK`**

```json
{
  "current_page": 1,
  "data": [
    {
      "id": 1,
      "transactionable_id": "uuid",
      "transactionable_type": "App\\Models\\MytpePayLink",
      "amount": 5000,
      "status": "completed",
      "payment_method": "online",
      "environment": "production",
      "name": "John Doe",
      "email": "john@example.com",
      "phone_number": "0555000000",
      "gateway_message": "Approved",
      "order_number": "abc123",
      "order_id": "order-id",
      "approval_code": "123456",
      "created_at": "2024-01-01T00:00:00.000000Z",
      "updated_at": "2024-01-01T00:00:00.000000Z",
      "transactionable": {
        "id": "uuid",
        "title": "Notary Service Fee"
      }
    }
  ],
  "last_page": 5,
  "per_page": 10,
  "total": 45
}
```

---

### 4.18 Download Receipt

**`POST /trader/mytpe-pay/receipt/download`**

**Middleware:** None (public)

Get a download URL for a transaction receipt.

**Request Body:**

| Field | Type | Required |
|-------|------|----------|
| `order_number` | string | Yes |

**Success Response — `200 OK`**

```json
{
  "message": "Receipt URL generated successfully",
  "data": {
    "download_url": "https://payment-gateway.example.com/receipts/abc123.pdf",
    "order_number": "abc123"
  }
}
```

**Error Responses:**

- **`404`** — Transaction not found

```json
{
  "error": "Transaction not found",
  "details": "No completed transaction found with the provided order number"
}
```

- **`404`** — MytpePay instance not found

```json
{
  "error": "MytpePay instance not found",
  "details": "The associated payment instance could not be found"
}
```

---

### 4.19 Email Receipt

**`POST /trader/mytpe-pay/receipt/email`**

**Middleware:** None (public)

Email a transaction receipt to a specified address.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `order_number` | string | Yes | |
| `email` | string | Yes | valid email |

**Success Response — `200 OK`**

```json
{
  "message": "Receipt sent successfully",
  "details": {
    "email": "client@example.com",
    "order_number": "abc123"
  }
}
```

**Error Responses:**

- **`404`** — Transaction or instance not found (same as download receipt)
- **`500`** — Failed to send

```json
{
  "error": "Failed to send receipt",
  "details": "Payment service failed to process email request"
}
```

---

### 4.20 Export Transactions

**`GET /trader/mytpe-pay/{id}/export-transactions`**

Download a CSV file of all transactions for a specific instance.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status: `completed`, `failed`, `pending` |

**Success Response:** Returns a CSV file download (`transactions-{id}.csv`)

**Error:** `403` if not the owner

---

## 5. MytpePay Links

Payment links are created under an active MytpePay instance. Each link has a unique slug and a public payment page at `https://mytpe.app/pay/{slug}`.

All authenticated endpoints require the `Authorization: Bearer {token}` header.
Routes are under `/trader/mytpe-pay/links/` (authenticated) and public routes have no auth.

**Link types:**
- `static` — customer enters their own amount
- `dynamic` — fixed amount set by you

**Payment modes:**
- `reusable` — unlimited payments (default)
- `one_shot` — single use, closes after one successful payment
- `limited` — limited number of payments (set `max_payments`)

**Link statuses:** `active`, `inactive`, `closed`

---

### 5.1 List All Links

**`GET /trader/mytpe-pay/links`**

Returns a paginated list of all your payment links across all instances.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `per_page` | integer | Items per page (default: 10) |
| `status` | string | Filter: `active`, `inactive`, `closed` |
| `type` | string | Filter: `static`, `dynamic` |
| `search` | string | Search by title, details, or slug |

**Success Response — `200 OK`**

```json
{
  "message": "Payment links retrieved successfully.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "id": "uuid",
        "mytpe_pay_id": "uuid",
        "title": "Notary Service Fee",
        "details": "Payment for notary authentication service",
        "slug": "notary-service-fee",
        "type": "dynamic",
        "amount": 5000,
        "status": "active",
        "logo": "trader/mytpe-pay/logos/random.png",
        "payment_mode": "reusable",
        "max_payments": null,
        "successful_payments_count": 12,
        "metadata": null,
        "settings": { "show_retry_button": true },
        "payment_link": "https://mytpe.app/pay/notary-service-fee",
        "payment_status": "active_receiving_payments",
        "created_at": "2024-01-01T00:00:00.000000Z"
      }
    ],
    "total": 1
  }
}
```

---

### 5.2 Show Link

**`GET /trader/mytpe-pay/links/{id}`**

**Success Response — `200 OK`**

```json
{
  "id": "uuid",
  "mytpe_pay_id": "uuid",
  "title": "Notary Service Fee",
  "details": "Payment for notary authentication service",
  "slug": "notary-service-fee",
  "type": "dynamic",
  "amount": 5000,
  "status": "active",
  "logo": "trader/mytpe-pay/logos/random.png",
  "logo_url": "https://bank.mytpe.app/storage/public/trader/mytpe-pay/logos/random.png",
  "payment_mode": "reusable",
  "max_payments": null,
  "successful_payments_count": 12,
  "metadata": null,
  "settings": { "show_retry_button": true },
  "payment_link": "https://mytpe.app/pay/notary-service-fee",
  "payment_status": "active_receiving_payments",
  "mytpe_pay": {
    "id": "uuid",
    "name": "My Notary Office Payments"
  }
}
```

**Error:** `404` if not found or doesn't belong to you.

---

### 5.3 Create Link

**`POST /trader/mytpe-pay/links/store`**

Use `multipart/form-data` when uploading a logo.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `mytpe_pay_id` | string | Yes | must be an active instance |
| `title` | string | Yes | max:255 |
| `details` | string | Yes | |
| `type` | string | Yes | `static` or `dynamic` |
| `amount` | number | If `dynamic` | min:0, required when type is `dynamic` |
| `slug` | string | Yes | unique, used in the payment URL |
| `logo` | file | No | jpg/png/jpeg, max 2MB |
| `payment_mode` | string | No | `reusable` (default), `one_shot`, or `limited` |
| `max_payments` | integer | If `limited` | min:1, required when payment_mode is `limited` |
| `metadata` | object | No | custom key-value data |
| `settings` | object | No | e.g. `{ "show_retry_button": true }` |

**Example request:**

```json
{
  "mytpe_pay_id": "uuid-of-instance",
  "title": "Notary Service Fee",
  "details": "Payment for notary authentication service",
  "type": "dynamic",
  "amount": 5000,
  "slug": "notary-service-fee",
  "payment_mode": "reusable"
}
```

**Success Response — `201 Created`**

```json
{
  "message": "Created successfully.",
  "data": {
    "id": "uuid",
    "mytpe_pay_id": "uuid",
    "title": "Notary Service Fee",
    "details": "Payment for notary authentication service",
    "slug": "notary-service-fee",
    "type": "dynamic",
    "amount": 5000,
    "status": "active",
    "payment_mode": "reusable",
    "max_payments": null,
    "successful_payments_count": 0,
    "payment_link": "https://mytpe.app/pay/notary-service-fee",
    "payment_status": "active"
  }
}
```

**Error Responses:**

- **`422`** — Validation error

```json
{
  "message": "The title field is required. (and 2 more errors)",
  "errors": {
    "title": ["The title field is required."],
    "slug": ["The slug has already been taken."],
    "type": ["The selected type is invalid."]
  }
}
```

---

### 5.4 Update Link

**`POST /trader/mytpe-pay/links/update/{id}`**

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `title` | string | No | max:255 |
| `details` | string | No | |
| `type` | string | No | `static` or `dynamic` |
| `amount` | number | If `dynamic` | min:0 |
| `logo` | file | No | jpg/png/jpeg, max 2MB |
| `payment_mode` | string | No | `reusable`, `one_shot`, or `limited` |
| `max_payments` | integer | No | min:1 (cannot reduce below current `successful_payments_count`) |
| `metadata` | object | No | |
| `settings` | object | No | |

**Success Response — `200 OK`**

```json
{
  "message": "Lien mis à jour avec succès.",
  "data": {
    "id": "uuid",
    "title": "Updated Title",
    "slug": "notary-service-fee",
    "type": "dynamic",
    "amount": 7500,
    "status": "active",
    "payment_mode": "reusable",
    "payment_link": "https://mytpe.app/pay/notary-service-fee",
    "payment_status": "active"
  }
}
```

**Error Responses:**

- **`422`** — Validation error (e.g. can't reduce max_payments below sales)
- **`403`** — Not the owner
- **`404`** — Link not found

---

### 5.5 Delete Link

**`DELETE /trader/mytpe-pay/links/delete/{id}`**

**Success Response — `200 OK`**

```json
{
  "message": "Deleted successfully"
}
```

**Error:** `500` if not the owner or link not found.

---

### 5.6 Check Slug Availability

**`POST /trader/mytpe-pay/links/check-slug`**

Check if a slug is available before creating a link.

**Request Body:**

| Field | Type | Required |
|-------|------|----------|
| `slug` | string | Yes |

**Success Response — `200 OK`**

```json
{
  "available": true,
  "message": "This slug is available."
}
```

```json
{
  "available": false,
  "message": "This slug is already taken."
}
```

---

### 5.7 Toggle Link Status

**`PATCH /trader/mytpe-pay/links/{id}/toggle-status`**

Toggle a link between `active` and `inactive`.

**Request Body:** None

**Success Response — `200 OK`**

```json
{
  "message": "Status updated.",
  "data": {
    "id": "uuid",
    "status": "inactive",
    "payment_status": "inactive"
  }
}
```

**Error:** `500` if not the owner or link not found.

---

### 5.8 Update Link Settings

**`PATCH /trader/mytpe-pay/links/{id}/settings`**

Merge new settings into the link's existing settings.

**Request Body:**

| Field | Type | Required |
|-------|------|----------|
| `settings` | object | Yes |
| `settings.show_retry_button` | boolean | No |

**Success Response — `200 OK`**

```json
{
  "message": "Settings updated.",
  "data": {
    "id": "uuid",
    "settings": { "show_retry_button": false }
  }
}
```

**Error:** `404` if link not found or doesn't belong to you.

---

### 5.9 Link Transactions

**`GET /trader/mytpe-pay/links/{id}/transactions`**

Paginated list of transactions for a specific payment link.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `per_page` | integer | Items per page (default: 10) |
| `status` | string | Filter: `completed`, `failed`, `pending` |
| `payment_category` | string | Filter by category (e.g. `zakat_fitr`, `global`) |

**Success Response — `200 OK`**

```json
{
  "current_page": 1,
  "data": [
    {
      "id": 1,
      "transactionable_id": "uuid",
      "transactionable_type": "App\\Models\\MytpePayLink",
      "amount": 5000,
      "status": "completed",
      "payment_method": "online",
      "environment": "production",
      "name": "John Doe",
      "email": "john@example.com",
      "phone_number": "0555000000",
      "gateway_message": "Approved",
      "order_number": "abc123",
      "order_id": "order-id",
      "approval_code": "123456",
      "payment_category": null,
      "created_at": "2024-01-01T00:00:00.000000Z"
    }
  ],
  "total": 12
}
```

**Error:** `404` if link not found.

---

### 5.10 Link Statistics

**`GET /trader/mytpe-pay/links/{id}/statistics`**

Get detailed statistics with time-series data for a specific link.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `start_date` | date | Start of period (e.g. `2024-01-01`) |
| `end_date` | date | End of period |
| `time_range` | string | Preset: `last_7_days`, `last_30_days`, `last_90_days` |
| `aggregation` | string | `daily`, `weekly` (default), `monthly`, `yearly` |

**Success Response — `200 OK`**

```json
{
  "data": {
    "pay_link_id": "uuid",
    "pay_link_slug": "notary-service-fee",
    "total_completed_amount": "60000.00",
    "time_series_data": [
      {
        "date": "2024-W01",
        "pending": 0,
        "processing": 0,
        "completed": 5,
        "failed": 1,
        "refunded": 0,
        "refused": 0
      },
      {
        "date": "2024-W02",
        "pending": 0,
        "processing": 0,
        "completed": 7,
        "failed": 0,
        "refunded": 0,
        "refused": 0
      }
    ]
  }
}
```

---

### 5.11 Export Link Transactions

**`GET /trader/mytpe-pay/links/{id}/export-transactions`**

Download a CSV file of transactions for a specific link.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter: `completed`, `failed`, `pending` |

**Success Response:** Returns a CSV file download (`link-transactions-{id}.csv`)

**Error:** `404` if link not found or doesn't belong to you.

---

### 5.12 Get Payment Link by Slug (Public)

**`GET /trader/mytpe-pay/links/public/{slug}`**

**Middleware:** None (public)

Retrieve public payment link data for rendering the payment page.

**Success Response — `200 OK`**

```json
{
  "message": "Payment link retrieved successfully.",
  "data": {
    "slug": "notary-service-fee",
    "title": "Notary Service Fee",
    "details": "Payment for notary authentication service",
    "type": "dynamic",
    "amount": 5000,
    "logo": "https://bank.mytpe.app/storage/public/trader/mytpe-pay/logos/random.png",
    "status": "active",
    "payment_mode": "reusable",
    "remaining_payments": null,
    "merchant_name": "My Notary Office Payments",
    "merchant_website": null,
    "settings": { "show_retry_button": true },
    "created_at": "2024-01-01T00:00:00.000000Z"
  }
}
```

**Error:** `404` if slug not found.

---

### 5.13 Initiate Payment (Public)

**`POST /trader/mytpe-pay/links/{slug}/pay`**

**Middleware:** None (public)

Start an online payment for a payment link. Returns a URL to redirect the payer to the payment gateway.

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `name` | string | Yes | max:255 |
| `email` | string | Yes | valid email |
| `phone_number` | string | Yes | max:20 |
| `amount` | number | If `static` type | min:0 (only for static links where the payer chooses the amount) |

**Example request:**

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "phone_number": "0555000000"
}
```

**Success Response — `200 OK`**

```json
{
  "message": "Payment initiated.",
  "form_url": "https://payment-gateway.example.com/pay/abc123"
}
```

**Error Responses:**

- **`422`** — Validation error
- **`403`** — Instance is inactive
- **`404`** — Link not found or not active
- **`500`** — Payment initiation failed

---

### 5.14 Fetch Link Transaction (Public)

**`POST /trader/mytpe-pay/links/transaction/fetch`**

**Middleware:** None (public)

Fetch the status of a payment link transaction after payment completion/redirect.

**Request Body:**

| Field | Type | Required |
|-------|------|----------|
| `order_number` | string | Yes |

**Success Response — `200 OK`**

```json
{
  "message": "Payment status updated",
  "transaction": {
    "id": 1,
    "order_number": "abc123",
    "amount": 5000,
    "status": "completed",
    "payment_method": "online",
    "environment": "production",
    "name": "John Doe",
    "email": "john@example.com",
    "phone_number": "0555000000",
    "gateway_message": "Approved",
    "created_at": "2024-01-01T00:00:00.000000Z"
  },
  "link_settings": { "show_retry_button": true }
}
```

**Error:** `404` if transaction not found.

---

### 5.15 Check Transaction (Public)

**`POST /trader/mytpe-pay/links/transaction/check`**

**Middleware:** None (public)

Check and update a transaction's status directly from the payment gateway.

**Request Body:**

| Field | Type | Required |
|-------|------|----------|
| `order_number` | string | Yes |

**Success Response — `200 OK`**

```json
{
  "transaction": {
    "id": 1,
    "order_number": "abc123",
    "amount": 5000,
    "status": "completed",
    "gateway_message": "Approved",
    "order_id": "order-id",
    "approval_code": "123456"
  },
  "gateway_response": { "..." }
}
```

**Error Responses:**

- **`400`** — Missing order_number
- **`404`** — Transaction or payment link not found
