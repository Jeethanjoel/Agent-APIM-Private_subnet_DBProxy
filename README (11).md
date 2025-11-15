# APIM → Private DB Proxy Architecture

## Request/Response Flow

1. **AI Agent (Copilot Studio) → APIM (Public HTTPS)**
   - Our custom AI agent, built in **Microsoft Copilot Studio**, sends an HTTPS request to the **public APIM endpoint on port 443**.
   - The agent uses this public endpoint for both **GET** and **POST** operations.

2. **APIM Authentication & Validation**
   - APIM validates the request using:
     - Required **subscription key**
     - Optional **Azure AD token** issued specifically for our AI agent
   - Any request missing these credentials is **immediately rejected**.

3. **APIM → Private Network (Azure VNet)**
   - After successful authentication, APIM forwards the request **privately** into the Azure VNet using **private endpoints**.
   - The request is routed securely into to DB Proxy present in Private Subnet
   - APIM may also **inject internal headers** if needed.

4. **DB Proxy → Azure SQL (Private Connection)**
   - The DB Proxy receives the call inside the VNet and validates it.
   - It connects to **Azure SQL via the SQL Private Endpoint** located in other Private Subnet .
   - Authentication used by DB Proxy is NTLM (Managed Identity or Key Vault).

5. **Azure SQL Query Execution**
   - Azure SQL runs the query and returns the output to the DB Proxy.

6. **Proxy → APIM (Private Path)**
   - The DB Proxy formats the response and sends it back to APIM using the same private network path.

7. **APIM → AI Agent (Public HTTPS)**
   - APIM returns the final response to the external AI agent over HTTPS.
   - This completes the secure, end-to-end request cycle.

---

## ✅ Why This Architecture Works

Our custom AI agent built using Microsoft Copilot Studio can call the publicly exposed HTTPS endpoint of APIM on port 443 and use that public URL to make both GET and POST requests. APIM then acts as the secure bridge between the external AI agent (running outside our network) and the database proxy that we built and deployed inside our Azure VNet in a private subnet. After receiving a request, APIM validates authentication and securely forwards it through private networking to the proxy. The proxy interacts with our database using NTLM authentication and returns the results back through APIM to the AI agent via the same secure path. This keeps both the database and proxy fully private while still allowing our external AI agent to access them safely through APIM.

Although APIM is publicly reachable (0.0.0.0/0), only our AI agent can successfully access it because APIM enforces strict authentication. Every request must include our unique subscription key and, optionally, a valid Azure AD token issued specifically to our AI agent. Any call without the correct keys or tokens is blocked instantly. Once APIM verifies the caller, it forwards the request privately into the company VNet where the proxy is located, using private endpoints. The proxy then performs the required NTLM authentication with the database. This ensures the database and proxy remain fully private while allowing only our external AI agent to securely reach them through APIM, even though the APIM endpoint itself is publicly accessible.

---


## 📌 High-Level Architecture

### **Architecture Reference I**
![Architecture Reference I](https://github.com/Jeethanjoel/Agent-APIM-Private_subnet_DBProxy/blob/main/reference_I.png)

### **Architecture Reference II**
![Architecture Reference II](https://github.com/Jeethanjoel/Agent-APIM-Private_subnet_DBProxy/blob/main/reference_II.png)

---

## Sample HTTP Requests

### ✅ **GET — List Items**

```
curl -i -X GET "https://my-api.azure-api.net/items?filter=x" \
  -H "Ocp-Apim-Subscription-Key: <APIM_SUBSCRIPTION_KEY>" \
  -H "Accept: application/json"
```

---

### ✅ **POST — Insert / Run Query**

```
curl -i -X POST "https://my-api.azure-api.net/query" \
  -H "Ocp-Apim-Subscription-Key: <APIM_SUBSCRIPTION_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"sql":"SELECT TOP 10 * FROM customers WHERE name=@name","params":{"name":"Alice"}}'
```

---

### 🔄 **Using OAuth Instead of Subscription Key**

Replace the header with:

```
-H "Authorization: Bearer <ACCESS_TOKEN>"
```

---

### 🔐 **Internal Header Added by APIM for DB Proxy**

(External callers **never** see this.)

```
x-internal-auth: <signed-token>
```

APIM injects this using policy so that only trusted APIM-originated calls reach the proxy.

---

## 🔐 Request Using Subscription Key/ OAuth2 Bearer Token

### **1️⃣ With Subscription Key**
```
GET https://my-api.azure-api.net/items
Headers:
  Ocp-Apim-Subscription-Key: <SUBSCRIPTION_KEY>
  Accept: application/json
```

### **2️⃣ With OAuth2 Bearer Token (Azure AD JWT)**
```
GET https://my-api.azure-api.net/items
Headers:
  Authorization: Bearer <ACCESS_TOKEN>
  Accept: application/json
```

---

