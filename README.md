# Building a Microservices Architecture on AWS
## Static Frontend · Serverless Auth · REST API · Intelligent Routing

> **Project:** AWS Microservices Architecture using S3, Lambda, EC2, and ALB
> **Author:** Ashish
> **Purpose:** Lab documentation with step-by-step setup, screenshots, and explanations

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Services Used — What & Why](#services-used--what--why)
3. [Phase 1 — S3 Frontend](#phase-1--s3-frontend)
4. [Phase 2 — Lambda Auth Service](#phase-2--lambda-auth-service)
5. [Phase 3 — EC2 API Server](#phase-3--ec2-api-server)
6. [Phase 4 — Application Load Balancer (ALB)](#phase-4--application-load-balancer-alb)
7. [Phase 5 — Update Frontend with ALB DNS](#phase-5--update-frontend-with-alb-dns)
8. [Final Testing](#final-testing)
9. [Quick Reference Table](#quick-reference-table)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Architecture Overview

The following diagram shows how all components connect together:

```
                        ┌─────────────────────────────────────────────────┐
                        │                   AWS CLOUD                      │
                        │                                                  │
   ┌──────────┐         │  ┌───────────┐      ┌────────────────────────┐  │
   │          │  HTTPS  │  │           │      │  Application Load       │  │
   │   User   │────────►│  │  S3       │      │  Balancer (ALB)        │  │
   │ (Browser)│         │  │ (Frontend)│      │                        │  │
   └──────────┘         │  │           │      │  Listener: HTTP :80    │  │
         │              │  └───────────┘      │                        │  │
         │              │        │            │  ┌────────────────────┐│  │
         │              │        │ Sends      │  │ Rule: /auth*       ││  │
         │              │        │ API calls  │  │    ▼               ││  │
         │              │        ▼            │  │ Lambda Target Grp  ││  │
         │              │  Browser fetches    │  │    ▼               ││  │
         │              │  from ALB DNS ──────┼─►│ ashish-auth-       ││  │
         │              │                     │  │ function (Lambda)  ││  │
         │              │                     │  └────────────────────┘│  │
         │              │                     │                        │  │
         │              │                     │  ┌────────────────────┐│  │
         │              │                     │  │ Rule: /users*      ││  │
         │              │                     │  │    ▼               ││  │
         │              │                     │  │ EC2 Target Group   ││  │
         │              │                     │  │    ▼               ││  │
         │              │                     │  │ api-server (EC2)   ││  │
         │              │                     │  │ Node.js :3000      ││  │
         │              │                     │  └────────────────────┘│  │
         │              │                     └────────────────────────┘  │
         │              │                                                  │
         │              └─────────────────────────────────────────────────┘
         │
         │   Flow Summary:
         │   1. User opens S3 website URL in browser
         │   2. S3 serves the static HTML frontend
         │   3. User clicks buttons → Frontend sends requests to ALB
         │   4. ALB routes /auth* → Lambda Function
         │   5. ALB routes /users* → EC2 Node.js API
         │   6. Responses returned to browser
```

### Simplified Flow Diagram

```
User Browser
     │
     ▼
S3 (Static Frontend - index.html)
     │  (JavaScript makes API calls)
     ▼
ALB (Application Load Balancer)
     │
     ├── /auth*  ──► Lambda (Auth Service) ──► Returns JWT Token
     │
     └── /users* ──► EC2 (Node.js API)    ──► Returns User List
```

---

## Services Used — What & Why

### 🪣 Amazon S3 (Simple Storage Service)
**What it is:** A cloud object storage service that can also host static websites.

**Why we use it here:**
- To host our frontend HTML file (index.html) without needing a dedicated web server
- It is highly available, cost-effective, and scales automatically
- Supports CORS so our frontend JavaScript can call the ALB from the browser

---

### ⚡ AWS Lambda
**What it is:** A serverless compute service that runs code in response to events — no servers to manage.

**Why we use it here:**
- To host the **Authentication Service** (`/auth` endpoint)
- Lambda is perfect for lightweight, stateless functions like returning an auth token
- It only runs when called and you pay per execution — very cost-efficient
- No server setup, patching, or maintenance needed

---

### 🖥️ Amazon EC2 (Elastic Compute Cloud)
**What it is:** A virtual machine (server) running in the cloud.

**Why we use it here:**
- To host the **Users API** (`/users` endpoint) as a Node.js HTTP server
- EC2 gives full control over the runtime environment — ideal for persistent, stateful apps
- Runs continuously on port 3000 serving JSON responses

---

### ⚖️ Application Load Balancer (ALB)
**What it is:** A managed load balancer that distributes incoming HTTP/HTTPS traffic based on routing rules.

**Why we use it here:**
- Acts as the **single entry point** for all API calls from the frontend
- Routes traffic intelligently:
  - `/auth*` → Lambda (serverless auth)
  - `/users*` → EC2 (API server)
- Handles cross-zone load balancing and sticky sessions
- Provides a single DNS name — the frontend only needs to know one URL

---

## Phase 1 — S3 Frontend

### What We Are Doing
We are creating an S3 bucket, enabling it to serve a static website, uploading our HTML file, and configuring access permissions so anyone can view it from a browser.

---

### Step 1.1 — Create S3 Bucket

**Navigation:** `S3 → Create bucket`

| Setting | Value |
|---|---|
| Bucket name | `your-app-frontend` (must be globally unique) |
| Block all public access | ❌ Uncheck this |

> **Why?** Bucket names are globally unique across all AWS accounts. Unchecking public access allows us to make the bucket website publicly accessible.

**Click:** `Create bucket`

---

![](Screenshot%202026-04-03%20103035.png)

![](Screenshot%202026-04-03%20103047.png)
![](Screenshot%202026-04-03%20103101.png)
![](Screenshot%202026-04-03%20103113.png)

```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #1 — S3 Bucket Created                      │
│                                                         │
│  What to capture:                                       │
│  ✅ Bucket name visible                                  │
│  ✅ Region shown                                         │
│  ✅ Creation confirmation / bucket listed in S3 console  │
│                                                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 1.2 — Enable Static Website Hosting

**Navigation:** `Bucket → Properties → Static website hosting → Edit`

| Setting | Value |
|---|---|
| Static website hosting | Enable |
| Index document | `index.html` |

> **Why?** This tells S3 to serve `index.html` when someone visits the bucket URL — turning it into a mini web server for static files (HTML, CSS, JS).

**Click:** `Save changes`

---

![](Screenshot%202026-04-03%20103533.png)
```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #2 — Static Website Hosting Enabled         │
│                                                         │
│  What to capture:                                       │
│  ✅ "Static website hosting" showing "Enabled"          │
│  ✅ The Bucket website endpoint URL visible             │
│                                                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 1.3 — Upload Frontend File

**Navigation:** `Bucket → Objects → Upload`

Upload your `index.html` file.

> **Why?** This is the actual web page users will see. It contains the HTML buttons and JavaScript that call our ALB endpoints.

---

### Step 1.4 — Configure CORS

**Navigation:** `Bucket → Permissions → Cross-origin resource sharing (CORS) → Edit`

Paste the following:

```json
[
    {
        "AllowedOrigins": ["*"],
        "AllowedMethods": ["GET", "POST", "PUT", "DELETE"],
        "AllowedHeaders": ["*"]
    }
]
```

> **Why?** Browsers block cross-origin requests by default (CORS policy). Since our frontend (S3 URL) is calling the ALB (different domain), we must explicitly allow it. This config tells the browser: "It's safe to make these API calls."

**Click:** `Save changes`

---

![](Screenshot%202026-04-03%20103546.png)

![](Screenshot%202026-04-03%20103633.png)

```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #3 — CORS Configuration Saved               │
│                                                         │
│  What to capture:                                       │
│  ✅ CORS JSON configuration visible in the console      │
│  ✅ "Successfully edited" confirmation message           │
│                                                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 1.5 — Add Bucket Policy (Public Read Access)

**Navigation:** `Bucket → Permissions → Bucket Policy → Edit`

Paste the following (replace `Bucket-Name` with your actual bucket name):

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "PublicReadGetObject",
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::Bucket-Name/*"
    }]
}
```

> **Why?** Even after disabling "Block all public access," S3 still needs an explicit bucket policy to allow public reads. This policy says: "Allow anyone (`*`) to perform `GetObject` (download/view files) from this bucket."

**Click:** `Save changes`

---

![](Screenshot%202026-04-03%20103847.png)
```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #4 — Bucket Policy Applied                  │
│                                                         │
│  What to capture:                                       │
│  ✅ Bucket policy JSON visible                          │
│  ✅ "Publicly accessible" badge shown on the bucket     │
│                                                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Phase 2 — Lambda Auth Service

### What We Are Doing
We are creating a serverless function that acts as our authentication endpoint. When called, it returns a dummy JWT token. We then give the ALB permission to invoke this function.

---

### Step 2.1 — Create Lambda Function

**Navigation:** `Lambda → Create function → Author from scratch`

| Setting | Value |
|---|---|
| Function name | `ashish-auth-function` |
| Runtime | `Node.js 18.x` |

> **Why Node.js 18.x?** It is a stable, long-term support (LTS) runtime — modern enough for ES module syntax and well-supported by AWS Lambda.

**Click:** `Create function`

---

![](Screenshot%202026-04-03%20104303.png)
![](Screenshot%202026-04-03%20104424.png)

```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #5 — Lambda Function Created                │
│                                                         │
│  What to capture:                                       │
│  ✅ Function name "ashish-auth-function" visible        │
│  ✅ Runtime showing Node.js 18.x                        │
│  ✅ "Successfully created" banner                       │
│                                                         │
│                                                         
└─────────────────────────────────────────────────────────┘
```

---

### Step 2.2 — Add Function Code

In the code editor, replace the default handler with:

```javascript
export const handler = async (event) => {
  return {
    statusCode: 200,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Headers": "*"
    },
    body: JSON.stringify({
      token: "dummy-jwt-token-12345",
      user: "demo-user"
    })
  };
};
```

> **Why these headers?**
> - `Content-Type: application/json` — tells the browser the response is JSON, not a file download
> - `Access-Control-Allow-Origin: *` — allows the browser to read the response across origins (required for CORS)
>
> **Without `Content-Type`**, the browser may try to download the response as a file instead of reading it as JSON.

**Click:** `Deploy`

---

![](Screenshot%202026-04-03%20104519.png)
```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #6 — Lambda Code Deployed                   │
│                                                         │
│  What to capture:                                       │
│  ✅ Code editor showing the handler function            │
│  ✅ "Changes deployed" confirmation                     │
│                                                         │
│  [ Paste your screenshot here ]                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 2.3 — Add ALB Permission to Invoke Lambda

**Navigation:** `Configuration → Permissions → Resource-based policy → Add permission`

| Setting | Value |
|---|---|
| AWS service | ELB |
| Action | `lambda:InvokeFunction` |
| ARN | Your ALB ARN |

> **Why?** By default, Lambda functions cannot be triggered by external services. We must explicitly grant the ALB permission to invoke (call) our Lambda function. Without this, the ALB will get an "Access Denied" error when trying to forward requests to Lambda.

---

![](Screenshot%202026-04-03%20105148.png)
```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #7 — Lambda Permission Added                │
│                                                         │
│  What to capture:                                       │
│  ✅ Resource-based policy showing ELB permission        │
│  ✅ Principal showing elasticloadbalancing.amazonaws.com │
│                                                         │
│                        │
└────────────────────────────────────────────────────────┘
```

---

### Step 2.4 — Test the Lambda Function

**Click:** `Test → Create test event → Invoke`

> **Why test?** Verify the function works correctly before connecting it to the ALB. You should see `statusCode: 200` and the JSON body with the token in the output.

---

![](Screenshot%202026-04-03%20105707.png)
![](Screenshot%202026-04-03%20110157.png)

```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #8 — Lambda Test Successful                 │
│                                                         │
│  What to capture:                                       │
│  ✅ Test result showing statusCode: 200                 │
│  ✅ Response body with token and user fields            │
│                                                         │
│                    │
└─────────────────────────────────────────────────────────┘
```

---

## Phase 3 — EC2 API Server

### What We Are Doing
We are launching a virtual machine (EC2 instance), automatically installing Node.js on it using a startup script (User Data), and running a simple HTTP server that returns a list of users on port 3000.

---

### Step 3.1 — Launch EC2 Instance

**Navigation:** `EC2 → Launch Instances`

| Setting | Value |
|---|---|
| Name | `api-server` |
| AMI | Amazon Linux 2 |
| Instance type | `t3.micro` |
| Key pair | Create new or select existing |
| VPC | Default VPC |

**Security Group Rules:**

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP |
| HTTP | 80 | Anywhere (0.0.0.0/0) |
| Custom TCP | 3000 | Anywhere (0.0.0.0/0) |

> **Why these ports?**
> - Port 22 (SSH): For connecting to the server to manage it remotely
> - Port 80 (HTTP): Standard web traffic
> - Port 3000: Our Node.js API listens here — the ALB target group will forward traffic to this port

---

### Step 3.2 — Add User Data Script (Auto-Install Node.js)

**Navigation:** `Advanced details → User data`

Paste the following script:

```bash
#!/bin/bash
yum update -y
curl -sL https://rpm.nodesource.com/setup_18.x | bash -
yum install -y nodejs
mkdir /app
cd /app
cat > index.js << 'EOF'
const http = require("http");
const server = http.createServer((req, res) => {
  res.writeHead(200, {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Headers": "*"
  });
  res.end(JSON.stringify({ users: ["user1", "user2", "user3"] }));
});
server.listen(3000, () => console.log("API running on port 3000"));
EOF
nohup node index.js > /tmp/node.log 2>&1 &
```

> **Why User Data?**
> User Data is a script that runs automatically when the EC2 instance starts for the first time. It saves us from having to SSH in and manually install software. The script:
>
> 1. Updates the OS packages
> 2. Installs Node.js 18.x
> 3. Creates an `index.js` file with our API code
> 4. Starts the server using `nohup` (so it keeps running even after the script ends)

**Click:** `Launch Instance`

---

### Step 3.3 — Verify the API is Running

Wait 2–3 minutes after launch for User Data to execute, then open a browser and visit:

```
http://<YOUR-EC2-PUBLIC-IP>:3000
```

You should see: `{"users":["user1","user2","user3"]}`

---

![](Screenshot%202026-04-03%20110426.png)
![](Screenshot%202026-04-03%20110615.png)
![](Screenshot%202026-04-03%20110630.png)

![](Screenshot%202026-04-03%20142104.png)
-------

------------------------



```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #9 — EC2 Instance Running                   │
│                                                         │
│  What to capture:                                       │
│  ✅ Instance state: "Running"                           │
│  ✅ Public IPv4 address visible                         │
│                                                         │
│                          │
└─────────────────────────────────────────────────────────┘
```


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #10 — Security Group Rules                  │
│                                                         │
│  What to capture:                                       │
│  ✅ Inbound rules showing ports 22, 80, 3000            │
│                                                         │
│                          │
└─────────────────────────────────────────────────────────┘
```


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #11 — API Response in Browser               │
│                                                         │
│  What to capture:                                       │
│  ✅ Browser showing http://<IP>:3000                    │
│  ✅ JSON response: {"users":["user1","user2","user3"]}  │
│                                                         │
                         │
└─────────────────────────────────────────────────────────┘
```

---

## Phase 4 — Application Load Balancer (ALB)

### What We Are Doing
We create two target groups (one pointing to EC2, one pointing to Lambda), create the ALB, and set up routing rules so `/auth*` goes to Lambda and `/users*` goes to EC2.

---

### Step 4.1 — Create Target Group for EC2

**Navigation:** `EC2 → Target Groups → Create target group`

| Setting | Value |
|---|---|
| Name | `ashish-api-tg` |
| Target type | Instance |
| Protocol | HTTP |
| Port | 3000 |
| Health check path | `/` |

Select your EC2 instance → `Include as pending` → `Create target group`

> **Why port 3000?** Our Node.js server on EC2 is listening on port 3000. The target group must forward traffic to that exact port.
>
> **Health check path `/`:** The ALB will periodically ping `/` to confirm the EC2 server is alive. If it stops responding, the ALB stops sending traffic to it.

---

### Step 4.2 — Create Target Group for Lambda

**Navigation:** `Target Groups → Create target group`

| Setting | Value |
|---|---|
| Name | `ashish-auth-tg` |
| Target type | Lambda function |

Select `ashish-auth-function` → `Create target group`

> **Why a Lambda target group?** ALB natively supports Lambda as a backend target. The ALB will invoke the Lambda function directly and return its response to the client — no extra infrastructure needed.

---

![](Screenshot%202026-04-03%20112632.png)![](Screenshot%202026-04-03%20112722.png)


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #12 — Both Target Groups Created            │
│                                                         │
│  What to capture:                                       │
│  ✅ ashish-api-tg (Instance type) listed                │
│  ✅ ashish-auth-tg (Lambda type) listed                 │
│  ✅ Health status of both target groups                 │
│                                                         │
│                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 4.3 — Create the Application Load Balancer

**Navigation:** `Load Balancers → Create Load Balancer → Application Load Balancer`

| Setting | Value |
|---|---|
| Name | `ashish-app-alb` |
| Scheme | Internet-facing |
| Listener | HTTP :80 |
| Security group | Allow HTTP (80) from Anywhere |

> **Why Internet-facing?** Our users are outside AWS — we need the ALB to have a public IP and DNS so browsers can reach it.
>
> **Why HTTP only (no HTTPS)?** This is a lab environment. In production, you would use HTTPS (port 443) with an SSL certificate via AWS Certificate Manager.

**Click:** `Create load balancer`

---

![](Screenshot%202026-04-03%20113326.png)

![](Screenshot%202026-04-03%20123857.png)

```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #13 — ALB Created                          │
│                                                         │
│  What to capture:                                       │
│  ✅ ALB name "ashish-app-alb" visible                   │
│  ✅ State: "Active"                                     │
│  ✅ DNS name visible (copy this for Phase 5)            │
│                                                         │
│                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 4.4 — Configure Listener Routing Rules

**Navigation:** `ALB → Listeners → HTTP:80 → View/edit rules`

Add two rules:

| Priority | Condition | Action |
|---|---|---|
| 1 | Path is `/auth*` | Forward to `ashish-auth-tg` |
| 2 | Path is `/users*` | Forward to `ashish-api-tg` |

> **Why path-based routing?** This is the core of microservices — one entry point (the ALB), but different backends handle different responsibilities. The `*` wildcard means any request starting with `/auth` (like `/auth/login`, `/auth/token`) goes to Lambda, and any starting with `/users` goes to EC2.

---

![](Screenshot%202026-04-03%20113431.png)
```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #14 — Listener Rules Configured             │
│                                                         │
│  What to capture:                                       │
│  ✅ Rule 1: /auth* → ashish-auth-tg (Lambda)           │
│  ✅ Rule 2: /users* → ashish-api-tg (EC2)              │
│                                                         │
│                          │
└─────────────────────────────────────────────────────────┘
```

---

### Step 4.5 — Enable Cross-Zone Load Balancing & Sticky Sessions

**Cross-Zone Load Balancing:**
**Navigation:** `ALB → Attributes → Edit → Enable cross-zone load balancing`

> **Why?** Cross-zone load balancing allows the ALB to distribute traffic evenly across all registered EC2 instances in **all Availability Zones**, not just the ones in the same AZ as the load balancer node. This prevents one AZ from being overloaded.

**Sticky Sessions (for EC2 Target Group):**
**Navigation:** `ashish-api-tg → Attributes → Edit → Enable stickiness`

> **Why?** Sticky sessions (session affinity) ensure a user is always routed to the **same EC2 instance** during their session. This matters if your server stores session data in memory. Without it, a user might get inconsistent results if different requests hit different instances.

---


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #15 — Cross-Zone Load Balancing Enabled     │
│                                                         │
│  What to capture:                                       │
│  ✅ Cross-zone load balancing showing "On"              │
│                                                         │
│                           │
└─────────────────────────────────────────────────────────┘
```

---


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #16 — Sticky Sessions Enabled               │
│                                                         │
│  What to capture:                                       │
│  ✅ Target group ashish-api-tg attributes               │
│  ✅ Stickiness showing "Enabled"                        │
│                                                         │
│                           │
└─────────────────────────────────────────────────────────┘
```

---

## Phase 5 — Update Frontend with ALB DNS

### What We Are Doing
Now that the ALB is created and has a DNS name, we update our `index.html` to point to the ALB, then re-upload it to S3.

---

### Step 5.1 — Copy ALB DNS Name

**Navigation:** `EC2 → Load Balancers → ashish-app-alb`

Copy the **DNS name** (looks like: `ashish-app-alb-1234567890.ap-south-1.elb.amazonaws.com`)


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #17 — ALB DNS Name Copied                   │
│                                                         │
│  What to capture:                                       │
│  ✅ ALB details page                                    │
│  ✅ DNS name highlighted / visible                      │
│                                                         │
│                         │
└─────────────────────────────────────────────────────────┘
```

---

### Step 5.2 — Update index.html

Edit the `index.html` file and update the ALB variable:

```javascript
// ⚠️ NO trailing slash after the DNS name!
const ALB = "http://ashish-app-alb-1234567890.ap-south-1.elb.amazonaws.com";
```

> **Why no trailing slash?** The button click handlers append paths like `/auth` and `/users`. If ALB already ends with `/`, the URL becomes `http://alb//users` (double slash), which causes a routing failure. Always remove the trailing slash.

---

### Step 5.3 — Re-upload index.html to S3

**Option A — AWS Console:**
`S3 → Bucket → Objects → Upload → Select updated index.html`

**Option B — AWS CLI:**
```bash
aws s3 cp index.html s3://your-bucket-name/
```


```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #18 — Updated index.html in S3              │
│                                                         │
│  What to capture:                                       │
│  ✅ S3 objects list showing index.html                  │
│  ✅ Last modified timestamp (confirms re-upload)        │
│                                                         │
│                         │
└─────────────────────────────────────────────────────────┘
```

---

## Final Testing

### What We Are Doing
Open the S3 website endpoint in the browser and verify both buttons work correctly — one calling the EC2 API, the other calling Lambda through the ALB.

---

### Test Steps

1. Open your S3 website endpoint URL in the browser
   (found in: `S3 → Bucket → Properties → Static website hosting → Bucket website endpoint`)

2. **Click "Call Users API"**
   - Expected: `{"users":["user1","user2","user3"]}`
   - Route: Browser → S3 → ALB → `/users*` → EC2 Node.js

3. **Click "Call Auth Service"**
   - Expected: `{"token":"dummy-jwt-token-12345","user":"demo-user"}`
   - Route: Browser → S3 → ALB → `/auth*` → Lambda

---

![](Screenshot%202026-04-03%20123857.png)
![](Screenshot%202026-04-03%20123906.png)

```
┌─────────────────────────────────────────────────────────┐
│  Screenshot #19 — Final Working Application             │
│                                                         │
│  What to capture:                                       │
│  ✅ Browser showing the S3 website URL                  │
│  ✅ "Call Users API" button result:                     │
│     {"users":["user1","user2","user3"]}                 │
│  ✅ "Call Auth Service" button result:                  │
│     {"token":"dummy-jwt-token-12345","user":"demo-user"}│
│                                                         │
│                           │
└─────────────────────────────────────────────────────────┘
```

---

## Quick Reference Table

| Component | Key Configuration | Purpose |
|---|---|---|
| **S3** | Static hosting ON, CORS policy, Bucket policy | Serves frontend HTML to users |
| **Lambda** | `Content-Type: application/json` header, ALB invoke permission | Handles `/auth*` requests |
| **EC2** | Node.js on port 3000 via User Data, Security Group port 3000 open | Handles `/users*` requests |
| **ALB** | Internet-facing, HTTP:80, path-based routing rules | Single entry point, routes to correct backend |
| **Target Group (EC2)** | Port 3000, sticky sessions, health check `/` | Forwards traffic to EC2 API |
| **Target Group (Lambda)** | Lambda target type | Forwards traffic to Lambda function |
| **Routing Rule 1** | `/auth*` → Lambda | Auth requests to serverless function |
| **Routing Rule 2** | `/users*` → EC2 | User data requests to Node.js API |

---

## Troubleshooting Guide

| Problem | Cause | Solution |
|---|---|---|
| Auth button triggers a file download | Lambda response missing `Content-Type` header | Add `"Content-Type": "application/json"` in Lambda response headers |
| Double slash in URL (`//users`) | Trailing slash in ALB variable | Remove trailing `/` from `const ALB = "http://..."` |
| Auth button returns error | Listener rule missing or wrong path | Check ALB listener rules — confirm `/auth*` path exists and points to Lambda target group |
| Cannot resolve ALB DNS | ALB in wrong region or not Active | Verify ALB state is "Active" and both resources are in the same AWS region |
| EC2 API not responding | Node.js not started or security group blocking | Wait 3 mins after launch; verify port 3000 is open in EC2 security group |
| S3 website shows 403 error | Bucket policy missing or incorrect | Re-check bucket policy — ensure the Resource ARN matches your bucket name exactly |
| Lambda test shows JSON but ALB gives error | ALB not authorized to invoke Lambda | Add resource-based policy granting `lambda:InvokeFunction` to ELB |

---

*Documentation generated for lab submission — all configurations performed via AWS Management Console.*
