---
layout: post
title: "Web API Security Notes"
date: 2022-12-22 10:35:23 -0500
published: true
---

### Foreword
These are my notes from my reading of the book: ["Hacking APIs: Breaking Web Application Programming Interfaces"](https://nostarch.com/hacking-apis). Amazon [link](https://www.amazon.com/Hacking-APIs-Application-Programming-Interfaces-ebook/dp/B09M82N4B4).
They are rough, that's the intention.

### URL
- protocol://hostname{:port number}/{path}/{?query}{parameters}

### HTTP requests

```
POST(1) /sessions(2) HTTP/1.1(3)  
Host: twitter.com(4)  
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8 Accept-Language: en-US,en;q=0.5  
Accept-Encoding: gzip, deflate  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 444  
Cookie: _personalization_id=GA1.2.1451399206.1606701545; dnt=1;

username_or_email%5D=hAPI_hacker&(5)password%5D=NotMyPassword(6)%21(7)
```

(1) method  
(2) path  
(3) protocol version  
(4) host  
(5) username  
(6) password  
(7) encoded exclamation mark (%21)  

### HTTP responses

```
HTTP/1.1(1) 302 Found(2)  
content-security-policy: default-src 'none'; connect-src 'self'  
location: https://twitter.com/  
pragma: no-cache  
server: tsa_a  
set-cookie: auth_token=8ff3f2424f8ac1c4ec635b4adb52cddf28ec18b8; Max-Age=157680000; Expires=Mon, 01 Dec 2025 16:42:40 GMT; Path=/; Domain=.twitter.com; Secure; HTTPOnly; SameSite=None

<html><body>You are being <a href="https://twitter.com/">redirected</a>.</body></html>
```

(1) protocol version  
(2) status code/status message  

### Authentication/authorization
- **authentication** - prove you are who you say you are  
- **authorization** - the privilege level you're granted once authenticated

### HTTP response codes
[md table generator](https://www.tablesgenerator.com/markdown_tables)

### HTTP response methods

### Stateless vs stateful communications
- **stateless** - the server maintains no persistent data about a user session, any data will be stored client-side (cookies)  
	- scales well, but introduces security concerns, storing data client-side means it has to be inherently mistrusted   

- **stateful** - the server stores cache information about users  
	- scales directly with available computer resources, so not very well, but can better ensure user data integrity  

### SQL (relational) vs NoSQL (nonrelational) databases
- **relational** - data is organized in tables (rows/columns)  
- **nonrelational** - data is organized in documents (closer to a key/value store)  

### Anatomy of web APIs
- client -> api gateway -> requested microservice  
- the primary interaction methods for an API are usually CRUD  

### Standardized web API types
#### 1. REST
- representational state transfer  
- entirely HTTP  
- more of a style than an officially standardized representation, lots of "shoulds" and not many "musts"
- kinda brittle, as endpoints only return exactly what they are programmed to return with no input from the client

example request:
```
GET /api/v3/inventory/item/pillow HTTP/1.1
HOST: rest-shop.com
User-Agent: Mozilla/5.0
Accept: application/json
```

example response:
```
HTTP/1.1 200 OK
Server: RESTfulServer/0.1
Cache-Control: no-store
Content-Type: application/json

{  
"item": {
    "id": "00101",
    "name": "pillow",
    "count": 25
    "price": {
		"currency": "USD",
		"value": "19.99"
		}
	},
}
```

Common REST headers
- `Authorization: <type> <token/credentials> `
	- `Bearer Ab4dtok3n`
	- `Basic b64payload`
- `Content-Type: <type>`
	- `application/json`
	- `application/xml`
	- `application/x-www-form-urlencoded`
- Middleware (X) headers
	- anything with an `X-<anything>` are middleware headers
	- can be added by anything that forwards traffic between the client and server (i.e. load balancers, WAFs, application servers themselves, etc)

#### 2. SOAP
- old, replaced by REST
- Simple Object Access Protocol (SOAP)
- action-oriented, xml-rpc based

example request:
```
POST /Inventory HTTP/1.1
Host: www.soap-shop.com
Content-Type: application/soap+xml; charset=utf-8
Content-Length: nnn

<?xml version="1.0"?>

(1)<soap:Envelope 
(2)xmlns:soap="http://www.w3.org/2003/05/soap-envelope/" soap:encodingStyle="http://www.w3.org/2003/05/soap-encoding">

(3)<soap:Body xmlns:m="http://www.soap-shop.com/inventory"> 
<m:GetInventoryPrice>
    <m:InventoryName>ThebestSOAP</m:InventoryName>
  </m:GetInventoryPrice>
</soap:Body>

</soap:Envelope>
```

(1) evelope - XML tag at the beginning signifying the start of a SOAP message  
(2) header - used to standardize communications machine-to-machine  
(3) body - primary payload of XML message  
(4) fault - optional, used for error messaging  

example response:
```
HTTP/1.1 200 OK
Content-Type: application/soap+xml; charset=utf-8
Content-Length: nnn

<?xml version="1.0"?>

<soap:Envelope
xmlns:soap="http://www.w3.org/2003/05/soap-envelope/"
soap:encodingStyle="http://www.w3.org/2003/05/soap-encoding">

<soap:Body xmlns:m="http://www.soap-shop.com/inventory"> 
(4)<soap:Fault> <faultcode>soap:VersionMismatch</faultcode>
		<faultstring, xml:lang='en">
            Name does not match Inventory record
		</faultstring>
</soap:Fault>
</soap:Body>
</soap:Envelope>
```


#### 3. GraphQL
- fits the REST 6 constraints, but is closer to a query language like SQL, i.e. *"query-centric"*
- resources are stored in a graph data structure
- differs from traditional REST APIs
	- you can query only specifically for fields/data that you (the client) want
	- in REST you have to filter out anything extra that you don't want since API endpoints just spit back what they're told to, whereas with GraphQL you can pick and choose the data that gets returned to you
	- this just changes where the filter processing happens (i.e. REST clearly depends on more action/resources from the client/application-side where GraphQL handles this server-side)
- only needs a single endpoint for all queries

example request:
```
POST /graphql HTTP/1.1
HOST: graphql-shop.com
Authorization: Bearer ab4dt0k3n

{query(1) {  
inventory(2) (item:"Graphics Card", id: 00101) {

name fields (3){ price quantity  
}  
}  
}  
}
```

(1) query operation  
(2) graphql node (aka root query type)  
(3) field (key:value pairs which comprise a graphql node)  
(4) response field  

example response:
```
HTTP/1.1 200 OK
Content-Type: application/json
Server: GraphqlServer

{"data": {
"inventory": {
"name": "Graphics Card", "fields": (4)[  
{  
"price":"999.99" "quantity": 25
} ] } } }
```

CRUD in GraphQL
- ***query*** - operation to retrieve data (read)
- ***mutation*** - operation to submit & write data (create, update, delete)
- ***subscription*** - operation to send data when an event occurs (read)
- ***schema*** - collections of data queryable by a given service (API collection)
	- gives you info for querying the API

### API Data Interchange Formats
#### 1. JSON
- Javascript Object Notation (JSON)
- key/value pairs separated by commas, within a pair of curly braces

example data:
```
{
    "firstName": "James",
    "lastName": "Lovell",
    "tripsToTheMoon": 2,
    "isAstronaut": true,
    "walkedOnMoon": false,
    "comment" : "This is a comment",
    "spacecrafts": ["Gemini 7", "Gemini 12", "Apollo 8", "Apollo 13"],
    "book": [
		{
	        "title": "Lost Moon",
	        "genre": "Non-fiction"
		} 
	]
}
```

#### 2. XML 
- Extensible Markup Language (XML)
- tagged data, similar to HTML
- almost exclusvie to SOAP, which relies on XML

example ***prolog***: ```<?xml version="1.0" encoding="UTF-8" ?>```
- ***prolog*** - contains information about XML version and encoding used
- ***elements*** - tag, or information surrounded by tags

#### 3. YAML
- YAML Ain't Markup Language (YAML)
- key/value based with colons
- minimalistic JSON feel, simple root elements: `---` / `...`

### API Authentication
#### 1. Basic Authentication
- HTTP basic authentication
- username/password in header or body of a request
- sometimes b64 encoded

#### 2. API Keys
- unique strings API providers generate for authorized access of approved consumers
- can be passed in query string parameters, request headers, body data, or as a cookie
- considered more secure than basic auth since it can be hard to brute-force
- still a myriad of struggles like disparate generation mechanisms creating guessable API keys (generating keys directly based on user data), can be exposed online, left in Git repos, intercepted over unencrypted communications, stolen through phishing

#### 3. JSON Web Tokens
- JSON Web Token (JWT)
- b64'd three part, period-separated structure
	- ***header*** - info about algorithm used to sign the payload
	- ***payload*** - data; username, time-stamp, issuer
	- ***signature*** - encoded+encrypted message used to validate token

example JWT:
```
Header:
{
  "alg": "HS512",
  "typ": "JWT"
}

Payload:
{
"sub": "1234567890",
"name": "hAPI Hacker",
"iat": 1516239022
}

Signature:
HMACSHA512(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
SuperSecretPassword
)

JWT:
eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6ImhBUEkgSGFja2VyIiwiaWF0IjoxNTE2MjM5MDIyfQ.zsUjGDbBjqI-bJbaUmvUdKaGSEvROKfNjy9K6TckK55sd97AMdPDLxUZwsneff4O1ZWQikhgPm7HHlXYn4jm0Q
```

- generally pretty good but can have implementation-based issues
	- no encryption
	- null alg
	- alg collision

#### 4. HMAC
- Hash-Based Message Authentication Code (HMAC)
- provider create secret key -> consumer 
- consumer makes request -> client-side hashing of request data+secret key = message digest
- message digest attached to subsequent requests
- server-side also calculates message digest (again, this is the hash of request data+secret key), if they match then the request is authorized
- security of message digest largely depends on the security of the cryptographic hash algorithm itself and the strength of the secret key

example hmac algorithms/hashes:
```
Algorithm    Hash output
HMAC-MD5    f37438341e3d22aa11b4b2e838120dcf
HMAC-SHA1    4c2de361ba8958558de3d049ed1fb5c115656e65
```

- hash security is a performance trade-off for HMAC since strong cryptographic hash functions are slower

#### 5. OAuth 2.0 (OAuth)
- Open Authorization Standard (OAuth)
- allows different services to access each other's data 
- service-to-service communciation
- limited time-based access token

#### 6. No Auth
- sometimes the case can be made for public APIs that handle no sensitive data

### Common API Vulnerabilties
#### 1. Information Disclosure
- sharing sensitive information with underprivleged users

#### 2. Broken Object-Level Authorization (BOLA)
- User A can successfully request resources of User B
- Isn't this just IDOR?

#### 3. Broken User Authentication
- The stateless nature of REST conflicts with the interests of security
- Usually a token distribution system (you register with provider, they give you token, you send token with every request)
Potential sub-issues:
	- registration process: password reset, multifactor (request brute forcing multifactor codes if there's no rate limiting; 4 digit=10,000 requests)
	- token handling: storage, tranmission method, prescence of hardcoded/debug tokens
	- token generation: collect a sample and analyze for similarity
	- key disclosure (github, etc): osint

#### 4. Excessive Data Exposure
- When an API endpoint responds with significantly more data than requested assuming the user will filter the results

#### 5. Lack of Resources & Rate Limiting
- You should be rate limiting users, at least behind some sort of a paywall to avoid uneccessary increases in infrastructure costs

#### 6. Broken Function Level Authorization (BFLA)
- User of one role/group can access API functionality of another role/group
- Can be lateral or privilege escalation
- BFLA vs BOLA: BOLA is unauthorized access, BFLA is unauthorized actions
- Switching out HTTP verbs if they aren't restricted could indicate a BFLA
- When looking for BFLA look for functionality that could help (i.e. altering user accounts, accessing user resources, gaining access to restricted endpoints)
- If there's documentation - focus on the admin functions and try them as an underprivileged user
- No documentation = discover or reverse the admin functions or endpoints

#### 7. Mass Assignment
- Occurs when an API consumer is able to take advantage of hidden but existent request parameters (i.e. ?isAdmin=True attached to a valid request like a password reset)
- Occur when the API isn't checking user action against a whitelist of permitted actions
- Discovery:
	- documentation
	- parameters seen in (possibly overly verbose) API responses
	- guess/fuzz

#### 8. Security Misconfigurations
- Mistakes or security oversights in implementation/development
- Examples:
	- misconfigured headers
	- misconfigured transit encryption
	- default accounts
	- uneccessary HTTP methods
	- lack of input sanitization
	- verbose errors
- `X-Response-Time` is a middleware header for usage metrics
	- Can be unintentionally user enumeration if valid vs invalid requests have a noticable time skew
example:
```
HTTP/UserA 404 Not Found --snip-- X-Response-Time: 25.5
HTTP/UserB 404 Not Found --snip-- X-Response-Time: 25.5
HTTP/UserC 404 Not Found --snip-- X-Response-Time: 510.00
```

#### 9. Injections
- Occur when there's either no or lacking input sanitization
- When user input is fed directly to backend systems which execute the user input as code
- Discovered by diligently testing every API endpoint and field while paying attention to how the API responds

#### 10. Improper Asset Management
- When organizations expose retired or development APIs, these are usually much more unfit for production
- Found by paying close attention to documentation, changelogs, version history on repos, guessing, brute-force, predictable path/API naming schemes

#### 11. Business Logic Vulnerabilities
- Also known as business logic flaws (BLFs)
- Intended application features/functions that an attacker can use maliciously
- Think: What did the developers intend for me to do? What assumptions or trusts did they make along the way?
- Things to look for in documentation:
	- “Only use feature X to perform function Y.”
	- “Do not do X with endpoint Y.”  
	- “Only admins should perform request X.”