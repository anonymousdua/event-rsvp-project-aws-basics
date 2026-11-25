# 🎉 Event RSVP: Serverless Full-Stack App on AWS

A beginner-friendly, serverless full-stack web app built entirely on **AWS Free Tier** services. Users can browse upcoming events, open an event to see its details, view **live RSVP counts**, see who's attending, and RSVP **Yes** or **No**. Each email can RSVP only once per event.

```
Browser ─► CloudFront ─► S3 (static frontend)
   │
   └──► API Gateway (HTTP API) ─► Lambda (Node.js) ─┬─► RDS MySQL   (event details)
                                                    └─► DynamoDB    (RSVPs + live counts)
```

> Based on the YouTube tutorial **[AWS FULL STACK TUTORIAL | Build and Deploy Your First App on AWS](https://www.youtube.com/watch?v=ADDG7LLS5IM)** by Darla David. Original source: [darladvd/event-rsvp-aws-tutorial](https://github.com/darladvd/event-rsvp-aws-tutorial).

---

## 📑 Table of Contents

1. [Setup & Run](#-1-setup--run)
   - [Prerequisites](#prerequisites)
   - [Step 0: Budget alarm](#step-0-create-a-budget-alarm)
   - [Step 1: RDS MySQL](#step-1-amazon-rds-mysql--event-details)
   - [Step 2: DynamoDB](#step-2-amazon-dynamodb--rsvps--counts)
   - [Step 3: Lambda](#step-3-aws-lambda--backend)
   - [Step 4: API Gateway](#step-4-amazon-api-gateway--routes)
   - [Step 5: Test with Postman](#step-5-test-the-api-with-postman)
   - [Step 6: Run the frontend locally](#step-6-run-the-frontend-locally)
   - [Step 7: Deploy to S3 + CloudFront](#step-7-deploy-the-frontend-to-s3--cloudfront)
2. [Tools & Tech Stack](#-2-tools--tech-stack)
3. [Project Details](#-3-project-details)
   - [Features](#features)
   - [Project structure](#project-structure)
   - [Architecture & request flow](#architecture--request-flow)
   - [Data model](#data-model)
   - [API reference](#api-reference)
   - [Frontend](#frontend)
   - [Networking & security](#networking--security)
   - [Cost](#cost)
   - [Troubleshooting](#troubleshooting)
   - [Pausing & cleanup](#pausing--cleanup)
   - [Ideas for extending](#ideas-for-extending)

---

## 🚀 1. Setup & Run

### Prerequisites

| Requirement | Why you need it |
|---|---|
| **AWS account** (Free Tier) | Hosts every part of the app |
| **Node.js** + npm | Installs the backend's dependencies so they can be zipped and uploaded to Lambda |
| **Code editor** (e.g. VS Code) | Editing the Lambda and frontend code |
| **Database client** (e.g. DBeaver, MySQL Workbench) | Creating the table and viewing data in RDS |
| **API tester** (e.g. Postman, Thunder Client) | Testing the API endpoints before wiring up the frontend |
| **Python 3** (or any static server) | Serving the frontend locally |
| **Git + GitHub** *(optional)* | Version control and portfolio |

Clone the repo:

```bash
git clone https://github.com/<your-username>/event-rsvp-aws-tutorial.git
cd event-rsvp-aws-tutorial
```

> 🌏 **Pick one AWS region and use it for everything.** The tutorial uses **`ap-southeast-1` (Singapore)**. Most "it can't connect" problems come from resources being created in different regions.

---

### Step 0: Create a budget alarm

Do this first so AWS warns you before anything gets expensive.

1. AWS Console → **Billing and Cost Management** → **Budgets** → **Create budget**.
2. Choose the **Monthly cost budget** template (or the **Zero spend budget** to be alerted at $0.01).
3. Set an amount (e.g. `$10`) and add your email address.
4. You'll be alerted when actual spend reaches 85% or 100%, or forecasted spend is expected to reach 100%.

---

### Step 1: Amazon RDS (MySQL): event details

1. **RDS** → **Create database** → **Standard create**.
2. Engine: **MySQL**. Template: **Free tier**.
3. DB instance identifier: e.g. `event-rsvp-db`. Set a **master username** (e.g. `admin`) and a **password**.
4. Instance class: **`db.t3.micro`** (Free Tier eligible). Availability: **Single-AZ**.
5. **Connectivity**
   - Don't connect to an EC2 compute resource.
   - VPC: **default VPC** (or let AWS create one).
   - DB subnet group: let AWS create it automatically.
   - **Public access: Yes** (so you can connect from your machine during development).
   - VPC security group: **Choose existing → `default`**.
6. **Additional configuration → Initial database name:** `eventsdb` ⚠️ *Don't skip this.*
7. Click **Create database** and wait for the status to become **Available**.

**Allow your computer to connect.** Open the instance's VPC security group → **Edit inbound rules** → add a rule:

| Type | Port | Source |
|---|---|---|
| MySQL/Aurora | 3306 | **My IP** |

> Never open port 3306 to `0.0.0.0/0`.

**Connect with your DB client** (e.g. DBeaver → New connection → MySQL) using the RDS **endpoint**, port `3306`, database `eventsdb`, and your master credentials.

**Create the table and add sample data** (also in [`database-notes.txt`](database-notes.txt)):

```sql
CREATE TABLE events (
  event_id      VARCHAR(64) PRIMARY KEY,
  title         VARCHAR(200) NOT NULL,
  description   TEXT,
  start_at      DATETIME NOT NULL,
  venue         VARCHAR(200),
  banner_url    VARCHAR(500),
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO events (event_id, title, description, start_at, venue, banner_url) VALUES
('aws-meetup-manila-2025', 'AWS User Group Manila Monthly Meetup 2025',
 'Join fellow builders for lightning talks, live demos, and networking with AWS enthusiasts in Manila.',
 '2025-10-18 18:00:00', 'AWS PH Office, Bonifacio Global City', ''),
('aws-buildercards-tournament-2025', 'AWS BuilderCards Tournament 2025',
 'Learn all about AWS BuilderCards.',
 '2025-11-08 17:00:00', 'AWS PH Office, Bonifacio Global City', ''),
('aws-community-day-ph-2025', 'AWS Community Day Philippines 2025',
 'The biggest gathering of AWS builders, user group leaders, and cloud enthusiasts in the Philippines.',
 '2025-11-30 09:00:00', 'AWS PH Office, Bonifacio Global City', '');

SELECT * FROM events;
```

> 💤 Not continuing for a few days? **Stop** (don't delete) the RDS instance. A running instance is billed for compute even when idle.

---

### Step 2: Amazon DynamoDB: RSVPs and counts

1. **DynamoDB** → **Create table**.
2. Table name: **`event-rsvp-counts`** *(this is the name hard-coded in `index.js`; if you use another name, see the note below)*.
3. Partition key: **`pk`** (String)
4. Sort key: **`sk`** (String)
5. Keep the default settings → **Create table**.

> ⚠️ **Key names are case-sensitive.** The code uses lowercase `pk` / `sk`. If you name them `PK` / `SK`, every DynamoDB call will fail.
>
> ⚠️ **Using a different table name?** Replace every `"event-rsvp-counts"` in `index.js` (there are 5 occurrences, all marked with a comment).

Don't add any items. The Lambda creates them when the first RSVP comes in.

---

### Step 3: AWS Lambda: backend

#### 3a. Create the function

1. **Lambda** → **Create function** → **Author from scratch**.
2. Name: `event-rsvp-handler`. Runtime: **Node.js** (latest). Architecture: default.
3. Execution role: **Create a new role with basic Lambda permissions**.
4. **Create function**.

#### 3b. Configure it

**Configuration → General configuration → Edit → Timeout:** `15–30 seconds` (the default 3 s is too short for RDS connections).

**Configuration → Environment variables:**

| Key | Example value |
|---|---|
| `DB_HOST` | `event-rsvp-db.xxxxxxxx.ap-southeast-1.rds.amazonaws.com` |
| `DB_USER` | `admin` |
| `DB_PASS` | `your-password` |
| `DB_NAME` | `eventsdb` |
| `REGION` | `ap-southeast-1` |

**Configuration → Permissions → click the role name → Add permissions → Attach policies:**

- `AWSLambdaVPCAccessExecutionRole`: lets Lambda join your VPC to reach RDS
- `AmazonDynamoDBFullAccess` (or `AmazonDynamoDBFullAccess_v2`): lets Lambda read and write the RSVP table

**Configuration → VPC → Edit:** select the **same VPC, subnets, and security group** as your RDS instance.

#### 3c. Allow Lambda to reach RDS

Sharing a security group is **not** enough on its own. Edit the RDS security group's **inbound rules** and add:

| Type | Port | Source |
|---|---|---|
| MySQL/Aurora | 3306 | **the security group itself** (`sg-xxxxxxxx`) |

#### 3d. Allow Lambda (inside the VPC) to reach DynamoDB

DynamoDB is a public AWS service, and a Lambda inside a VPC has no route to it by default. Add a free **gateway VPC endpoint**:

1. **VPC** → **Endpoints** → **Create endpoint**.
2. Type: **AWS services**. Service: **`com.amazonaws.ap-southeast-1.dynamodb`** (type **Gateway**). Check that the console is in the right region.
3. VPC: the one your Lambda uses. Route tables: select them.
4. Policy: **Full access** → **Create endpoint**.

> A NAT Gateway would also work, but it is **not** free. The VPC endpoint is the cheaper option.

#### 3e. Package and upload the code

Lambda doesn't install npm packages for you, so bundle `node_modules` with the code:

```bash
npm install          # installs mysql2 and @aws-sdk/client-dynamodb

# macOS / Linux
zip -r rsvp-backend.zip index.js package.json package-lock.json node_modules

# Windows (PowerShell)
Compress-Archive -Path index.js,package.json,package-lock.json,node_modules -DestinationPath rsvp-backend.zip
```

In the Lambda console: **Code → Upload from → .zip file** → select `rsvp-backend.zip` → **Save**.

Then check **Runtime settings → Handler** is **`index.handler`**.

> 💡 For small fixes that don't add dependencies, you can edit the code directly in the Lambda console and click **Deploy**. You don't need to re-zip.

---

### Step 4: Amazon API Gateway: routes

1. **API Gateway** → **Create API** → **HTTP API** → **Build**.
2. Integration: **Lambda** → select `event-rsvp-handler`. API name: `event-rsvp-api`.
3. Configure routes (all pointing to the Lambda integration):

   | Method | Route |
   |---|---|
   | `GET` | `/events` |
   | `GET` | `/event/{event_id}` |
   | `POST` | `/rsvp` |
   | `GET` | `/stats/{event_id}` |
   | `GET` | `/attendees/{event_id}` |

4. Stage: keep `$default` with auto-deploy → **Create**.
5. **CORS** (left menu) → **Configure**:
   - Access-Control-Allow-Origin: `*`
   - Access-Control-Allow-Methods: `GET, POST, OPTIONS` (or `*`)
   - Access-Control-Allow-Headers: `content-type`
6. Copy the **Invoke URL / Default endpoint**, e.g. `https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com`.

> ⚠️ Use the path parameter name **`{event_id}`** exactly. The Lambda reads `event.pathParameters.event_id`.

---

### Step 5: Test the API with Postman

Replace `$API` with your invoke URL.

```bash
# List all events
curl $API/events

# One event
curl $API/event/aws-meetup-manila-2025

# Submit an RSVP
curl -X POST $API/rsvp \
  -H "Content-Type: application/json" \
  -d '{"event_id":"aws-meetup-manila-2025","full_name":"Jane Doe","email":"jane@example.com","response":"Yes"}'
# → {"message":"RSVP recorded!"}

# Submit again with the same email → 409
# → {"message":"You have already RSVP'd for this event with this email!","code":"DUPLICATE_RSVP"}

# Live counts
curl $API/stats/aws-meetup-manila-2025
# → {"Yes":1,"No":0}

# Attendee list (optionally filter: ?response=Yes)
curl $API/attendees/aws-meetup-manila-2025
```

If something fails, check **CloudWatch → Log groups → `/aws/lambda/event-rsvp-handler`**. Every request is logged with its full event payload. See [Troubleshooting](#troubleshooting).

---

### Step 6: Run the frontend locally

1. Open **`utils.js`** and set your API URL (no trailing slash):

   ```js
   const API_BASE_URL = 'https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com';
   ```

2. Start a local static server from the project folder:

   ```bash
   python3 -m http.server 8000
   # or: npx serve .
   ```

3. Open **http://localhost:8000**. You should see the event cards. Click one to open the modal, view stats and attendees, and submit an RSVP.

> Opening `index.html` directly with `file://` can break `fetch` calls, so use a local server.

---

### Step 7: Deploy the frontend to S3 + CloudFront

#### 7a. S3 bucket

1. **S3** → **Create bucket** (same region). Use a globally unique name.
2. Leave **Block all public access ✅ ON**. CloudFront will serve the files and the bucket stays private.
3. Upload `index.html`, `style.css`, `utils.js`, `events.js`, `app.js`.
4. *(Optional)* Create a `banners/` folder and upload your event banner images.

> Don't upload `index.js`, `package*.json`, or `node_modules`. Those are backend files.

#### 7b. CloudFront distribution

1. **CloudFront** → **Create distribution**.
2. Origin: **Amazon S3** → choose your bucket → **Allow private S3 bucket access to CloudFront** (Origin Access Control). This adds the bucket policy for you.
3. Keep the recommended settings. WAF is optional for this demo but recommended for real projects.
4. After it's created: **Settings → Edit → Default root object:** `index.html`.
5. Wait for deployment, then open the **Distribution domain name** (e.g. `https://d1234abcd.cloudfront.net`). 🎉 Your app is live.

#### 7c. Point event banners to CloudFront *(optional)*

```sql
UPDATE events
SET banner_url = 'https://d1234abcd.cloudfront.net/banners/builder-cards-tournament.jpg'
WHERE event_id = 'aws-buildercards-tournament-2025';
```

Events without a `banner_url` fall back to a default Unsplash image.

#### 7d. Updating the frontend later

CloudFront caches files. After re-uploading changed files to S3:

1. **CloudFront → your distribution → Invalidations → Create invalidation** → path `/*`.
2. Wait for **Completed**, then hard-refresh (`Ctrl + F5` on Windows / `Cmd + Shift + R` on macOS).

---

## 🧰 2. Tools & Tech Stack

### Application

| Layer | Technology |
|---|---|
| Frontend | HTML5, CSS3, vanilla JavaScript (no framework, no build step) |
| Backend | Node.js (ES modules) on **AWS Lambda** |
| Backend libraries | [`mysql2`](https://www.npmjs.com/package/mysql2) `^3.11.0`, [`@aws-sdk/client-dynamodb`](https://www.npmjs.com/package/@aws-sdk/client-dynamodb) `^3.679.0` |
| Relational DB | **MySQL** on Amazon RDS |
| NoSQL DB | **Amazon DynamoDB** |

### AWS services

| Service | Role in this project |
|---|---|
| **AWS Budgets** | Spending alerts |
| **IAM** | Execution role and policies that grant Lambda access to the VPC and DynamoDB |
| **Amazon RDS (MySQL)** | Stores structured event details |
| **Amazon DynamoDB** | Stores RSVPs and atomic Yes/No counters |
| **AWS Lambda** | Runs the backend logic (one function, five routes) |
| **Amazon API Gateway (HTTP API)** | Public REST endpoints and CORS, triggers Lambda |
| **Amazon VPC** | Private network, security groups, DynamoDB gateway endpoint |
| **Amazon CloudWatch Logs** | Lambda logs for debugging |
| **Amazon S3** | Stores the static frontend files and banner images (private bucket) |
| **Amazon CloudFront** | Global CDN, HTTPS, serves the S3 site via Origin Access Control |

### Developer tools

| Tool | Used for |
|---|---|
| VS Code | Editing code |
| DBeaver / MySQL Workbench | Connecting to RDS, running SQL |
| Postman / Thunder Client | Testing API endpoints |
| Node.js + npm | Installing backend dependencies before zipping |
| Python 3 `http.server` | Local frontend server |
| Browser DevTools (Network tab) | Inspecting API requests, payloads and responses |
| Git + GitHub | Version control |

---

## 📚 3. Project Details

### Features

- 📅 Event list loaded from MySQL, sorted by date, shown as a responsive card grid with hover effects
- 🪟 Event details modal with banner, description, date and time, and venue
- ✅ RSVP form (name, email, Yes/No)
- 🔢 Live Yes/No counts that refresh right after you submit
- 👥 Attendee table that scrolls when the list gets long
- 🚫 Duplicate protection: one RSVP per email per event (HTTP `409 DUPLICATE_RSVP`)
- ⚡ Fully serverless: no servers to manage, and you pay only when code runs
- 💸 Runs entirely within the AWS Free Tier

### Project structure

```
.
├── index.html           # Page skeleton: header, events grid, modal
├── style.css            # Layout, cards, modal, stats bar, attendee table, form
├── utils.js             # API_BASE_URL, apiCall() fetch wrapper, date/time formatters, closeModal()
├── events.js            # API calls, RSVP submission, card rendering, modal, stats
├── app.js               # Entry point: waits for DOMContentLoaded, then loads events
├── index.js             # AWS Lambda handler (backend; all 5 routes)
├── package.json         # Backend dependencies (mysql2, @aws-sdk/client-dynamodb)
├── package-lock.json
├── database-notes.txt   # SQL: CREATE TABLE + sample INSERTs
└── .gitignore           # node_modules/, .DS_Store
```

### Architecture & request flow

```
┌─────────┐   HTTPS    ┌────────────┐  OAC  ┌──────────┐
│ Browser │──────────►│ CloudFront │──────►│ S3 bucket│   (HTML/CSS/JS, banners)
└────┬────┘            └────────────┘       └──────────┘
     │ fetch() from utils.js
     ▼
┌──────────────────┐  invoke  ┌───────────────────────── VPC ──────────────────────────┐
│ API Gateway      │─────────►│  ┌────────────┐   3306   ┌───────────────┐             │
│ (HTTP API + CORS)│          │  │  Lambda    │─────────►│ RDS MySQL     │ events      │
└──────────────────┘          │  │  Node.js   │          └───────────────┘             │
                              │  │            │ gateway  ┌───────────────┐             │
                              │  │            │─endpoint►│ DynamoDB      │ RSVPs/counts│
                              │  └────────────┘          └───────────────┘             │
                              └────────────────────────────────────────────────────────┘
```

1. The browser loads static files from **CloudFront**, which pulls them from the private **S3** bucket.
2. The JavaScript calls the **API Gateway** URL. That URL is separate from the CloudFront domain, which is why the Network tab shows a different host for API calls.
3. API Gateway invokes **Lambda**, which routes on HTTP method and path.
4. Lambda reads event data from **RDS** and reads or writes RSVPs in **DynamoDB**.
5. Lambda returns JSON with CORS headers, and the frontend renders it.

**Why two databases?**

- **MySQL (SQL)** suits structured, planned data with a fixed schema: event details.
- **DynamoDB (NoSQL)** suits fast, high-volume, constantly changing writes: RSVPs and counters. It handles many simultaneous updates without the lock contention a single SQL row would face.

**Why serverless?** Compared with running Node.js on an EC2 server, Lambda runs only when a request arrives, scales automatically, and bills by the millisecond. You don't have to patch an OS or manage uptime.

### Data model

#### MySQL: `eventsdb.events`

| Column | Type | Notes |
|---|---|---|
| `event_id` | `VARCHAR(64)` **PK** | Slug, e.g. `aws-meetup-manila-2025` |
| `title` | `VARCHAR(200)` NOT NULL | |
| `description` | `TEXT` | |
| `start_at` | `DATETIME` NOT NULL | Used for sorting |
| `venue` | `VARCHAR(200)` | |
| `banner_url` | `VARCHAR(500)` | Falls back to a default image if empty |
| `created_at` | `TIMESTAMP` | Defaults to `CURRENT_TIMESTAMP` |

#### DynamoDB: `event-rsvp-counts` (single-table design)

Think of `pk` as a **folder** (one per event) and `sk` as a **file name** inside it. Prefixes create logical sub-folders.

| `pk` | `sk` | Other attributes | Purpose |
|---|---|---|---|
| `EVENT#<event_id>` | `RESPONDENT#<email>` | `full_name`, `email`, `response`, `timestamp` | One item per attendee |
| `EVENT#<event_id>` | `RESPONSE#Yes` | `count` (Number) | Running "Yes" counter |
| `EVENT#<event_id>` | `RESPONSE#No` | `count` (Number) | Running "No" counter |

**Access patterns**

| Need | Operation |
|---|---|
| Record an RSVP | `TransactWriteItems`: **Put** the respondent item (with `attribute_not_exists` condition) **and** `ADD count 1` on the matching counter, atomically. Both succeed or both fail. |
| Get counts | `BatchGetItem` on `RESPONSE#Yes` and `RESPONSE#No` |
| List attendees | `Query` where `pk = EVENT#<id>` and `begins_with(sk, "RESPONDENT#")` |

The condition on the Put is what blocks duplicate RSVPs from the same email.

### API reference

Base URL: your API Gateway invoke URL. All responses are JSON and include CORS headers.

| Method | Path | Source | Success response | Errors |
|---|---|---|---|---|
| `GET` | `/events` | RDS | Array of events sorted by `start_at` ascending | `500` |
| `GET` | `/event/{event_id}` | RDS | Single event object | `404 Event not found` |
| `GET` | `/stats/{event_id}` | DynamoDB | `{ "Yes": 3, "No": 1 }` | `500` |
| `GET` | `/attendees/{event_id}` | DynamoDB | `[{ full_name, email, response, timestamp }]`. Optional `?response=Yes` or `?response=No` filter | `500` |
| `POST` | `/rsvp` | DynamoDB | `200 { "message": "RSVP recorded!" }` | `400` missing fields · `409 DUPLICATE_RSVP` · `500` |
| `OPTIONS` | any | — | `200` CORS preflight | — |
| any | unknown | — | — | `404 Route not found` |

**`POST /rsvp` body**

```json
{
  "event_id": "aws-meetup-manila-2025",
  "full_name": "Jane Doe",
  "email": "jane@example.com",
  "response": "Yes"
}
```

All four fields are required. `response` must be exactly `"Yes"` or `"No"` (case-sensitive) for the counters to line up.

### Frontend

| File | Responsibility |
|---|---|
| `index.html` | Header, empty `#eventsGrid` (filled by JS), hidden `#eventModal` popup, and script tags in load order: `utils.js` → `events.js` → `app.js` |
| `style.css` | Responsive auto-fit grid, card hover lift, full-screen modal overlay with slide-in animation, 50/50 stats bar, scrollable attendee table, styled form with an orange submit button |
| `utils.js` | **`API_BASE_URL`** (the only config to change), `apiCall()` fetch wrapper (keeps API calls DRY and turns non-2xx responses into errors), `formatDate()` / `formatTime()`, `closeModal()`, and click-outside-to-close |
| `events.js` | `fetchEvents` / `fetchEventDetails` / `fetchEventStats` / `fetchEventAttendees`, `submitRSVP` (POST, then shows a message and refreshes stats and attendees), `displayEvents`, `loadEventStats`, `openEventModal` |
| `app.js` | Waits for `DOMContentLoaded`, then runs `initApp()` → `loadEvents()` → `displayEvents()` |

### Networking & security

- **VPC**: your private network ("gated neighbourhood"). RDS and Lambda both live inside it.
- **Security group**: a firewall per resource. Inbound rules on the RDS security group:
  - `3306` from **My IP**, for your DB client during development
  - `3306` from **the same security group**, so Lambda can connect
- **DynamoDB gateway endpoint**: gives the in-VPC Lambda a private route to DynamoDB without a NAT Gateway.
- **IAM role**: Lambda only gets what it needs: VPC networking and DynamoDB access.
- **S3 + CloudFront OAC**: the bucket stays private, and only CloudFront can read it.
- **CORS**: set to `*` for simplicity. For production, restrict `Access-Control-Allow-Origin` to your CloudFront domain.

**Hardening ideas for production**

- Set RDS **Public access: No** and remove the My-IP rule once setup is done.
- Store DB credentials in **AWS Secrets Manager** instead of plain environment variables.
- Replace `AmazonDynamoDBFullAccess` with a least-privilege policy scoped to the one table.
- Enable **AWS WAF** on CloudFront.
- Add input validation (email format, `response` must be `Yes`/`No`).

### Cost

Everything here is Free Tier eligible, within monthly limits.

| Service | Notes |
|---|---|
| RDS `db.t3.micro` | The main thing to watch. Running 24/7 can cost roughly **$0–5**. **Stop it** between sessions. |
| DynamoDB | On-demand. Effectively free at this scale. |
| Lambda / API Gateway | Pay per request. Effectively free for a demo. |
| S3 | Free Tier includes 5 GB storage and 20,000 GET requests per month. Your files are only a few MB. |
| CloudFront | Generous free data-transfer allowance. A demo won't come close to it. |
| VPC, subnets, security groups, gateway endpoint | No charge on their own |
| NAT Gateway | **Not used.** It costs money, which is why this project uses the VPC endpoint. |

### Troubleshooting

| Symptom | Likely cause → fix |
|---|---|
| DB client can't connect to RDS | Wrong region · **Public access** isn't *Yes* · security group lacks the **My IP** rule on 3306 · database name missing from the JDBC URL |
| `Cannot find package 'mysql2'` in CloudWatch | Code was pasted into the console without dependencies → run `npm install`, zip **with `node_modules`**, and upload |
| `Task timed out after 3.00 seconds` | Increase the Lambda **timeout** to 15–30 s |
| `/events` times out, or Lambda can't reach RDS | Lambda isn't in the same VPC/subnets/SG as RDS, or the SG lacks the **self-referencing 3306 rule** |
| `/events` works but `/rsvp`, `/stats`, `/attendees` time out | No route from the VPC to DynamoDB → create the **DynamoDB gateway VPC endpoint** in the correct region and attach the route tables |
| `ResourceNotFoundException` | DynamoDB table name in `index.js` doesn't match your table, or the table is in a different region from `REGION` |
| `ValidationException ... key element does not match the schema` | Table keys aren't named exactly lowercase **`pk`** / **`sk`** |
| `Runtime.HandlerNotFound` | Handler setting must be `index.handler` |
| `Route not found` / 404 | Check the path spelling (`/event/` singular vs `/events` plural) and the `{event_id}` parameter name |
| Works in Postman, but the browser says **"Failed to fetch"** | CORS isn't configured on API Gateway, or `API_BASE_URL` in `utils.js` is wrong (no trailing slash) |
| Page stuck on "Loading events..." | Open DevTools → Console/Network. Usually an empty or incorrect `API_BASE_URL` |
| CloudFront root URL shows `AccessDenied` | Set **Default root object** to `index.html` |
| Site still shows old files after an update | Create a CloudFront **invalidation** for `/*` and hard-refresh |

### Pausing & cleanup

**Option A: Pause (keep everything, near-zero cost)**

| Service | Action |
|---|---|
| RDS | **Actions → Stop temporarily**. Data is kept. AWS auto-restarts stopped instances after 7 days, so stop it again if needed. |
| CloudFront | **Disable** the distribution |
| API Gateway, Lambda, DynamoDB | No action. They're idle and cost nothing. |
| S3 | Keep it private |

**Option B: Archive, then delete everything (zero ongoing cost).** Follow this order:

1. **Back up:** DynamoDB → *Export to S3* (then download it) · push code to GitHub · *(optional)* RDS snapshot, which has a small storage cost.
2. **CloudFront:** disable → wait → delete.
3. **API Gateway:** delete the API.
4. **Lambda:** delete the function.
5. **IAM:** delete the Lambda execution role.
6. **RDS:** delete the instance (skip the final snapshot if you don't need it).
7. **S3:** download anything you need → empty the bucket → delete it.
8. **DynamoDB:** delete the table.
9. **VPC:** delete the DynamoDB endpoint, plus any custom security groups or DB subnet groups you created. The default SG can't be deleted.
10. **CloudWatch:** delete leftover log groups.
11. **Keep your budget alarm active.**

### Ideas for extending

- Admin page to create, edit, and delete events (`PUT` / `DELETE` routes)
- Authentication with **Amazon Cognito**
- Email confirmations with **Amazon SES**
- Custom domain with **Route 53** + **ACM** certificate
- Infrastructure as code (**AWS SAM**, **CDK**, or **Terraform**)
- CI/CD with **GitHub Actions** to auto-deploy Lambda and sync S3
- Reuse the architecture for a booking site, a club dashboard, or event registrations

---

## 🙏 Credits

- Tutorial and original code: **Darla David** ([YouTube video](https://www.youtube.com/watch?v=ADDG7LLS5IM) · [GitHub](https://github.com/darladvd/event-rsvp-aws-tutorial))

> *"Be the senior you needed when you were a junior."*
