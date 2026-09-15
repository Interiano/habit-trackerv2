# Serverless Habit Tracker — Correct-Path Build Guide (HTTP API + Cognito Auth)

> This is the from-scratch build reflecting the **corrected architecture**: API Gateway
> **HTTP API** (not REST) with a native **Cognito JWT authorizer**, multi-user from day one.
> It supersedes the original single-user REST build. Follow the order top to bottom — each
> step depends on the one before it.

---

## Target architecture

```
Client (browser)
  │  1. logs in via Cognito → receives JWT
  │  2. calls API with  Authorization: Bearer <JWT>
  ▼
Route 53 → CloudFront → S3            (static frontend)
                        │
                        └── API calls ──►  API Gateway (HTTP API)
                                              │  JWT authorizer validates token
                                              ▼
                                           Lambda (CRUD)
                                              │  reads userId from token claim
                                              ▼
                                           DynamoDB  (partition key = userId)
```

| Layer | Service | Notes |
|---|---|---|
| DNS | Route 53 | Custom domain (optional) |
| CDN / TLS | CloudFront | HTTPS, caches static assets |
| Frontend | S3 | Static site (HTML/JS) |
| **Auth** | **Cognito User Pool** | User directory; issues JWTs |
| **API** | **API Gateway HTTP API** | Native JWT authorizer |
| Compute | Lambda | CRUD functions |
| Data | DynamoDB | One table, per-user partitioning |

---

## Why HTTP API (not REST API)

- **71% cheaper** — $1.00 vs $3.50 per million requests.
- **Native JWT authorizer** — validates Cognito tokens with no custom Lambda authorizer.
- Supports everything this app needs: Lambda proxy integration, JWT auth, CORS,
  custom domains, route-level throttling.
- REST API is only worth its premium for **caching, usage plans / API keys, request
  validation, or native WAF** — none of which this app uses.

> HTTP API is a **separate resource type**, not a setting on a REST API. This build
> creates it fresh.

---

## Key design decision: userId comes from the token, never the request body

The original build hardcoded `userId = "user-001"` and trusted the request body.
In the correct path:

- Each Cognito user has a unique **`sub`** claim (a UUID).
- The JWT authorizer verifies the token **before** Lambda runs.
- Lambda reads the verified `sub` and uses it as the DynamoDB **partition key**.
- The client **cannot** set or spoof another user's id — it isn't taken from the body.

**HTTP API claims path (payload format 2.0):**
```
event["requestContext"]["authorizer"]["jwt"]["claims"]["sub"]
```
(REST API used `...authorizer.claims.sub` — the HTTP API path has an extra `jwt` level.)

---

## Build order

### Step 1 — DynamoDB table
- Table name: `habits`
- Partition key: `userId` (String)  ← will hold the Cognito `sub`
- Sort key: `habitId` (String)       ← lets one user own many habits
- On-demand capacity is fine for a portfolio app.

**Why:** Build the data layer first so Lambda has something to talk to.

### Step 2 — Cognito User Pool (auth foundation)
- Create a **User Pool** (sign-in with email; enable email verification).
- Create an **App Client**:
  - Public client, no client secret (browser-based SPA).
  - Enable the **Hosted UI** and set callback/sign-out URLs to your CloudFront domain.
- Note these for later:
  - **User Pool ID** — e.g. `us-east-1_XXXXXXXXX`
  - **App Client ID**
  - **Issuer URL** — `https://cognito-idp.<region>.amazonaws.com/<userPoolId>`

**Why:** The API's authorizer and the frontend login both point at these values.

### Step 3 — Lambda CRUD functions
Create functions (Python example below): `create_habit`, `list_habits`,
`update_habit`, `delete_habit`.

- Attach an IAM role allowing `dynamodb:PutItem`, `GetItem`, `Query`, `UpdateItem`,
  `DeleteItem` on the `habits` table only (least privilege).
- Each function derives `userId` from the token claim, not the body.

```python
import json, boto3, uuid
ddb = boto3.resource("dynamodb").Table("habits")

def _user_id(event):
    # HTTP API (payload v2) JWT authorizer claims
    return event["requestContext"]["authorizer"]["jwt"]["claims"]["sub"]

def create_habit(event, context):
    user_id = _user_id(event)
    body = json.loads(event.get("body") or "{}")
    item = {
        "userId": user_id,
        "habitId": str(uuid.uuid4()),
        "habitName": body["habitName"],
        "frequency": body.get("frequency", "daily"),
    }
    ddb.put_item(Item=item)
    return {"statusCode": 201, "body": json.dumps(item)}

def list_habits(event, context):
    user_id = _user_id(event)
    resp = ddb.query(
        KeyConditionExpression=boto3.dynamodb.conditions.Key("userId").eq(user_id)
    )
    return {"statusCode": 200, "body": json.dumps(resp["Items"])}
```

**Why:** With auth wired later, `_user_id()` guarantees a user only ever touches
their own rows.

### Step 4 — HTTP API + JWT authorizer
- Create an **HTTP API**.
- Add a **JWT authorizer**:
  - Identity source: `$request.header.Authorization`
  - Issuer URL: `https://cognito-idp.<region>.amazonaws.com/<userPoolId>`
  - Audience: your **App Client ID**
- Create routes, integrate each with its Lambda, and **attach the authorizer**:
  - `POST /habits`            → `create_habit`
  - `GET /habits`            → `list_habits`
  - `PUT /habits/{habitId}`   → `update_habit`
  - `DELETE /habits/{habitId}`→ `delete_habit`
- Enable **CORS** for your CloudFront origin.
- Deploy to a stage (e.g. `$default` or `dev`).

**Why:** The authorizer rejects any request without a valid Cognito token *before*
it reaches Lambda — no auth code in your functions.

### Step 5 — Frontend, CloudFront, Route 53
- Upload the static site to **S3** (private bucket).
- Put **CloudFront** in front (Origin Access Control; HTTPS).
- Optional: point a **Route 53** record at the CloudFront distribution.

### Step 6 — Wire login → token → API
- Frontend redirects to the **Cognito Hosted UI** (or use the Amplify SDK for a
  custom form).
- After login, grab the **ID token** (JWT) from the Cognito response.
- Send it on every API call:
  ```
  Authorization: Bearer <ID token>
  ```
- The HTTP API authorizer validates it; Lambda reads `sub`; DynamoDB returns only
  that user's habits.

---

## Verify the correct path end-to-end
- [ ] Call `GET /habits` **without** a token → `401 Unauthorized`.
- [ ] Log in as User A, create a habit, list → see only A's habits.
- [ ] Log in as User B → see **none** of A's habits.
- [ ] Confirm no `userId` is ever read from the request body.

---

## What changed vs the original REST build
| | Original | Correct path |
|---|---|---|
| API type | REST API | **HTTP API** |
| Request cost | $3.50 / M | **$1.00 / M** |
| Auth | none (hardcoded `user-001`) | **Cognito User Pool + JWT authorizer** |
| userId source | request body | **verified `sub` claim** |
| Claims path | `authorizer.claims.sub` | `authorizer.jwt.claims.sub` |
| Multi-user | no | **yes** |

---

## Cert tie-in (SAA-C03)
- HTTP API vs REST API trade-offs (cost, features, authorizer types).
- Cognito **User Pool = authentication**; add an **Identity Pool** only if clients
  need direct AWS resource access (this app does not).
- JWT authorizer validates issuer + audience + signature + expiry.
- DynamoDB partition-key design for per-tenant isolation.

*Pricing figures: AWS API Gateway pricing, US East (N. Virginia), verified Sep 2026.*
