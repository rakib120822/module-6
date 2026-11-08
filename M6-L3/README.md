# Lab 03 — Nginx API Gateway with Authentication, Rate Limiting and Routing

## Lab Overview

This lab builds an API Gateway using Nginx that sits between clients and two backend microservices: orders-service (order requests) and users-service (user requests). All requests pass through Nginx, which performs three jobs: authentication, rate limiting, and request routing.

Clients never talk to the backend services directly. Nginx is the single entry point.

![Lab 03 architecture](images/lab03-architecture.png)

## What happens when a client sends a request

The client sends a request to the gateway with an API key.

Nginx checks whether the API key is valid.

If the key is invalid or missing, Nginx returns 401 Unauthorized.

If the key is valid, Nginx checks the client's request rate.

If the request exceeds the allowed rate, Nginx returns 429 Too Many Requests.

Otherwise, Nginx forwards the request to the correct backend service.

The service returns JSON data through Nginx to the client.

## Architecture

Each component has one job, and only the gateway is reachable from the host.

| Component | Responsibility |
|---|---|
| Client | Sends API requests to the gateway. |
| Nginx Gateway | Authenticates, rate-limits and routes requests. |
| Orders Service | Handles order requests on port 3001. |
| Users Service | Handles user requests on port 3002. |
| Docker Network | Lets containers talk to each other privately. |

### Request flow

![Flow chart](images/flow-chart.png)

### How rate limiting works

Nginx's limit_req module uses the leaky bucket algorithm to regulate incoming requests.

- **Allowed rate**: 5 requests per second
- **Burst allowance**: 5 requests
- **nodelay**: excess requests inside the burst allowance are forwarded immediately instead of queueing.

Think of it as a queue with a controlled processing rate: requests drip out at 5 per second, and anything beyond that is rejected with 429. The burst permits temporary spikes, but it does not raise the sustained rate above 5 requests per second.

## Prerequisites

Check every tool before you start.

| Tool | Purpose | Verification |
|---|---|---|
| Node.js v20 or v22 | Runs the backend applications. | `node -v` |
| Docker Engine v24+ | Runs applications inside containers. | `docker --version` |
| Docker Compose v2+ | Manages multiple containers together. | `docker compose version` |
| cURL | Sends HTTP requests for testing. | `curl --version` |
| jq | Formats JSON responses. | `jq --version` |

**Port requirement**: host port 8080 must be free.

> **Note:** Lab 03 and Lab 04 both use port 8080. Stop containers from previous labs first: `docker compose down -v`

## Concepts You Should Understand

These concepts underpin every step in the lab.

| Concept | Beginner-friendly explanation |
|---|---|
| API Gateway | A single entry point that receives client requests and forwards them to the right backend. |
| Reverse Proxy | A server that receives requests on behalf of another server and forwards them. |
| Authentication | Checking whether a client may access an API using valid credentials. |
| API Key | A secret value sent by a client to prove it is an authorised API consumer. |
| Rate Limiting | Restricting how frequently a client can send requests. |
| Upstream | A group or definition of backend servers Nginx can forward requests to. |
| Nginx map | Creates a mapping between an input value and an output value. |
| Docker Bridge Network | A private network that lets containers communicate with each other. |
| Container Port | The port an application listens on inside its container. |
| Port Mapping | Connects a host machine port to a container port. |

## Objectives

By the end of this lab, you will be able to:

- Create two independent Node.js microservices.
- Understand how Nginx works as an API Gateway and reverse proxy.
- Configure API key authentication using Nginx `map`.
- Apply rate limiting using `limit_req_zone` and `limit_req`.
- Connect multiple containers using a private Docker bridge network.
- Route requests dynamically based on URL paths.
- Test authentication, routing and rate limiting using cURL.

## Project Structure

You will create six files across four directories.

```text
lab03-gateway-routing/
├── docker-compose.yml
├── nginx.conf
├── services/
│   ├── orders-service/
│   │   ├── package.json
│   │   └── server.js
│   └── users-service/
│       ├── package.json
│       └── server.js
└── verify-gateway.sh
```

| File | Purpose |
|---|---|
| docker-compose.yml | Defines and runs all three containers. |
| nginx.conf | Configures authentication, rate limiting and request routing. |
| orders-service/server.js | Implements the Orders API. |
| users-service/server.js | Implements the Users API. |
| package.json | Defines dependencies and the start command for each service. |
| verify-gateway.sh | Automatically tests the gateway functionality. |

---

## Step 1 — Create the Node.js Microservices

Two independent Node.js applications sit behind the gateway. Splitting orders from users is the basic idea behind microservices architecture: one service manages customer orders, another manages customer information.

### 1.1 Create the project directories

```bash
mkdir -p ~/lab03-gateway-routing/services/orders-service
mkdir -p ~/lab03-gateway-routing/services/users-service
cd ~/lab03-gateway-routing
```

`mkdir` creates a directory and `-p` creates parent directories if needed. `cd` moves into the project directory.

### 1.2 Create the Orders Service

```bash
cd services/orders-service
```

Create `package.json`:

```json
{
  "name": "orders-service",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

Create `server.js`:

```javascript
const express = require('express');
const app = express();

app.use(express.json());

const orders = [
  { id: 1, customer: 'Alice', item: 'Laptop', total: 1299.99, status: 'shipped' },
  { id: 2, customer: 'Bob', item: 'Phone', total: 799.99, status: 'processing' },
  { id: 3, customer: 'Charlie', item: 'Tablet', total: 499.99, status: 'delivered' }
];

app.get('/api/v1/health', (req, res) => {
  res.json({ service: 'orders-service', status: 'healthy' });
});

app.get('/api/v1/orders', (req, res) => {
  res.json({ service: 'orders-service', count: orders.length, data: orders });
});

app.get('/api/v1/orders/:id', (req, res) => {
  const order = orders.find(o => o.id === Number(req.params.id));
  if (!order) {
    return res.status(404).json({ service: 'orders-service', error: 'Order not found' });
  }
  res.json({ service: 'orders-service', data: order });
});

app.post('/api/v1/orders', (req, res) => {
  const newOrder = { id: orders.length + 1, status: 'created', ...req.body };
  orders.push(newOrder);
  res.status(201).json({ service: 'orders-service', data: newOrder });
});

const PORT = process.env.PORT || 3001;

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Orders service listening on port ${PORT}`);
});
```

### 1.3 Create the Users Service

```bash
cd ../users-service
```

Create `package.json`:

```json
{
  "name": "users-service",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

Create `server.js`:

```javascript
const express = require('express');
const app = express();

app.use(express.json());

const users = [
  { id: 1, name: 'Alice', email: 'alice@example.com', role: 'customer' },
  { id: 2, name: 'Bob', email: 'bob@example.com', role: 'admin' },
  { id: 3, name: 'Charlie', email: 'charlie@example.com', role: 'customer' }
];

app.get('/api/v1/health', (req, res) => {
  res.json({ service: 'users-service', status: 'healthy' });
});

app.get('/api/v1/users', (req, res) => {
  res.json({ service: 'users-service', count: users.length, data: users });
});

app.get('/api/v1/users/:id', (req, res) => {
  const user = users.find(u => u.id === Number(req.params.id));
  if (!user) {
    return res.status(404).json({ service: 'users-service', error: 'User not found' });
  }
  res.json({ service: 'users-service', data: user });
});

app.post('/api/v1/users', (req, res) => {
  const newUser = { id: users.length + 1, ...req.body };
  users.push(newUser);
  res.status(201).json({ service: 'users-service', data: newUser });
});

const PORT = process.env.PORT || 3002;

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Users service listening on port ${PORT}`);
});
```

Return to the project root:

```bash
cd ~/lab03-gateway-routing
```

---

## Step 2 — Configure the Nginx API Gateway

The gateway config combines three mechanisms: `map` for authentication, `limit_req_zone` for rate limiting, and `location` blocks for routing.

Create `nginx.conf` in the project root:

```nginx
events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/json;

    # Rate Limiting: 5 requests per second per IP with burst allowance of 5
    limit_req_zone $binary_remote_addr zone=api_quota:10m rate=5r/s;
    limit_req_status 429;

    # API Key Authentication Mapping
    map $http_x_api_key $api_client_authorized {
        default 0;
        "valid-secret-key-123" 1;
        "prod-partner-key-999" 1;
    }

    upstream orders_backend {
        server orders_service:3001;
    }

    upstream users_backend {
        server users_service:3002;
    }

    server {
        listen 80;

        # Standard Proxy Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Gateway Health Endpoint (Bypasses Auth & Rate Limits)
        location /health {
            access_log off;
            return 200 '{"status":"UP","gateway":"Nginx","version":"1.25"}';
        }

        # Compatible /healthz alias
        location /healthz {
            access_log off;
            return 200 '{"status":"UP","gateway":"Nginx","version":"1.25"}';
        }

        # Protected Orders Route
        location /api/v1/orders {
            if ($api_client_authorized = 0) {
                return 401 '{"type":"https://api.example.com/errors/unauthorized","title":"Unauthorized","status":401,"detail":"Missing or invalid X-API-Key header."}';
            }

            limit_req zone=api_quota burst=5 nodelay;
            proxy_pass http://orders_backend;
        }

        # Protected Users Route
        location /api/v1/users {
            if ($api_client_authorized = 0) {
                return 401 '{"type":"https://api.example.com/errors/unauthorized","title":"Unauthorized","status":401,"detail":"Missing or invalid X-API-Key header."}';
            }

            limit_req zone=api_quota burst=5 nodelay;
            proxy_pass http://users_backend;
        }
    }
}
```

### How the config works

| Directive | Purpose |
|---|---|
| `include mime.types` / `default_type` | Serves static assets with correct MIME types and defaults everything else to `application/json`. |
| `map $http_x_api_key $api_client_authorized` | Turns the X-API-Key header into 1 (valid) or 0 (invalid). Two keys are accepted: `valid-secret-key-123` and `prod-partner-key-999`. |
| `if ($api_client_authorized = 0)` | Returns 401 before any request reaches a backend. |
| `limit_req_zone $binary_remote_addr zone=api_quota:10m rate=5r/s` | Defines the shared-memory zone and the 5r/s rate, keyed by client IP. |
| `limit_req zone=api_quota burst=5 nodelay` | Applies the limit with a burst of 5 and no queueing delay. |
| `limit_req_status 429` | Returns 429 instead of the default 503 when a request is rejected. |
| `upstream orders_backend` / `upstream users_backend` | Names the backend server groups Nginx can proxy to. |
| `proxy_pass` | Forwards the request to the matching upstream. |
| `location /health` and `location /healthz` | Gateway health endpoints that bypass both authentication and rate limiting. |

> **Note:** `orders_service:3001` is a Docker network service name, not localhost. Inside the Nginx container, localhost means the Nginx container itself.

---

## Step 3 — Docker Compose Configuration

Compose defines three containers on one private bridge network. Only the gateway publishes a host port.

Create `docker-compose.yml`:

```yaml
services:
  orders-service:
    build: ./services/orders-service
    container_name: orders-service
    environment:
      PORT: "3001"
    networks:
      - gateway-network
    restart: unless-stopped

  users-service:
    build: ./services/users-service
    container_name: users-service
    environment:
      PORT: "3002"
    networks:
      - gateway-network
    restart: unless-stopped

  gateway:
    image: nginx:1.25-alpine
    container_name: api-gateway
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "8080:80"
    depends_on:
      - orders-service
      - users-service
    networks:
      - gateway-network
    restart: unless-stopped

networks:
  gateway-network:
    driver: bridge
```

- `ports: "8080:80"` maps host port 8080 to container port 80.
- The backends have no `ports` section, so they are reachable only inside `gateway-network`.
- The `nginx.conf` mount is read-only (`:ro`) so the container cannot modify it.

---

## Step 4 — Verification Script

The script checks the gateway health endpoint, authentication, routing and rate limiting in one pass.

Create `verify-gateway.sh`:

```bash
#!/usr/bin/env bash
set -e

GATEWAY_URL="http://localhost:8080"
VALID_KEY="valid-secret-key-123"

echo "=================================================="
echo "  Lab 03: API Gateway & Rate-Aware Routing Verification  "
echo "=================================================="
echo ""

echo "1. Checking Gateway Health (Unauthenticated)..."
curl -s -i "$GATEWAY_URL/health" | head -n 8
echo ""

echo "2. Calling /api/v1/orders WITHOUT API Key (Expect 401 Unauthorized)..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$GATEWAY_URL/api/v1/orders")
echo "HTTP Status Code: $HTTP_CODE"
if [ "$HTTP_CODE" -eq 401 ]; then
  echo " PASS: 401 Unauthorized enforced at Gateway edge."
else
  echo " FAIL: Expected 401, got $HTTP_CODE"
fi
echo ""

echo "3. Calling /api/v1/orders WITH valid API Key (Expect 200 OK)..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $VALID_KEY" "$GATEWAY_URL/api/v1/orders")
echo "HTTP Status Code: $HTTP_CODE"
if [ "$HTTP_CODE" -eq 200 ]; then
  echo " PASS: Orders service upstream routed successfully."
else
  echo " FAIL: Expected 200, got $HTTP_CODE"
fi
echo ""

echo "4. Calling /api/v1/users WITH valid API Key (Expect 200 OK)..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $VALID_KEY" "$GATEWAY_URL/api/v1/users")
echo "HTTP Status Code: $HTTP_CODE"
if [ "$HTTP_CODE" -eq 200 ]; then
  echo " PASS: Users service upstream routed successfully."
else
  echo " FAIL: Expected 200, got $HTTP_CODE"
fi
echo ""

echo "5. Flooding Gateway to Test Rate Limiting (Expect 429)..."
for i in {1..15}; do
  CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $VALID_KEY" "$GATEWAY_URL/api/v1/orders")
  echo "Request #$i -> Status $CODE"
done
```

Make it executable:

```bash
chmod +x verify-gateway.sh
```

---

## Step 5 — Run and Test the Gateway

Build and start the stack, then confirm every container is healthy.

### 5.1 Build and start

```bash
docker compose build
docker compose up -d
```

![Docker container build](images/docker-container-build.png)

Check status:

```bash
docker compose ps
```

![Docker container list](images/docker-container-list.png)

All three containers should show Up.

### 5.2 Confirm backend isolation

The backends must not be reachable from the host:

```bash
curl -s http://localhost:3001/api/v1/orders
curl -s http://localhost:3002/api/v1/users
```

Both commands should fail to connect. Only port 8080 is published.

### 5.3 Test the gateway health endpoint

The health endpoint bypasses authentication and rate limiting:

```bash
curl -i http://localhost:8080/health
curl -i http://localhost:8080/healthz
```

Both should return `200 OK` with `{"status":"UP","gateway":"Nginx","version":"1.25"}`.

### 5.4 Test authentication

Missing API key -> 401

```bash
curl -i http://localhost:8080/api/v1/orders
```

![Invalid request](images/invalid-request.png)

Valid API key -> 200

```bash
curl -i -H "X-API-Key: valid-secret-key-123" http://localhost:8080/api/v1/orders
```

![Get request](images/get-request.png)

The first response must contain `"service": "orders-service"`. The second must contain `"service": "users-service"`.

### 5.5 Run the automated verification

```bash
./verify-gateway.sh
```

All four checks should print `PASS`, and the flood loop should show `429` once the rate limit is reached.

![Verify gateway](images/verify-gateway.png)

---

## Troubleshooting

Work through these symptoms in order.

| Symptom | Likely cause | Fix |
|---|---|---|
| 401 even with the right key | Header name or value mismatch | Check `map` uses `$http_x_api_key` and that the key is exactly `valid-secret-key-123` or `prod-partner-key-999`. |
| 502 Bad Gateway | Nginx cannot reach the backend | Check service names and ports match `nginx.conf` and `docker-compose.yml`. |
| 503 instead of 429 | `limit_req_status` not set | Add `limit_req_status 429;` in the `http` block. |
| 404 on a protected route | The backend does not serve the `/api/v1` prefix | Confirm the service defines `/api/v1/orders` and `/api/v1/users`, matching the `location` blocks. |
| Connection refused on 8080 | Gateway container not running | Run `docker compose ps` and `docker compose logs gateway`. |
| Rate limit does not trigger | Burst is too large for the request count | Lower `burst` or send more requests in the loop. |
| Backend reachable from host | Port mapping exposed the service | Remove `ports` from the backend services. |
| nginx.conf changes ignored | Config cached in the container | Run `docker compose restart gateway`. |

### Useful diagnostic commands

```bash
docker compose ps
docker compose logs gateway
docker compose logs orders-service
docker compose logs users-service
docker compose exec gateway nginx -t
```

These commands tell you whether the problem lies with Docker, Nginx or a backend application.

---

## Teardown & Cleanup

Stop the containers and remove the network when the lab is done.

```bash
docker compose down -v
```

This stops and removes the containers, removes the Compose-created network, and removes associated named volumes.

Confirm they have stopped:

```bash
docker compose ps
```

The output should be empty. Port 8080 is now free for Lab 04.

---

## Final Review

You built a gateway with three distinct controls.

| Feature | Implementation | Expected result |
|---|---|---|
| API Gateway | Nginx | Single public entry point |
| Authentication | Nginx `map` | 401 for invalid keys |
| Rate limiting | `limit_req_zone` and `limit_req` | 429 when limits are exceeded |
| Request routing | Nginx upstreams | Requests reach the correct service |
| Health endpoints | `location /health` and `/healthz` | 200 without authentication or rate limiting |
| Container isolation | Docker bridge network | Private backend communication |
| Port exposure | Docker Compose | Only the gateway exposes host port 8080 |

### Key Takeaways

Completing this lab taught you seven core ideas:

- An API Gateway provides a centralised entry point for multiple backend services.
- Nginx can authenticate API consumers before requests reach backend applications.
- Rate limiting protects services from excessive incoming traffic.
- Reverse proxying lets Nginx forward requests to different backend services.
- Docker Compose makes it easier to manage multiple containers.
- Docker bridge networks allow private communication between containers.
- Backend services do not need to expose their ports publicly.

### Conclusion
Congratulations! You have built an Nginx API Gateway with authentication, rate limiting and dynamic routing using Docker Compose.
