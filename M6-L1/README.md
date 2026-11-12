# Lab 01 — Contract-First API Design with OpenAPI 3

> **Module 06 — API Design & Networking Essentials** · Lab 1
> **Difficulty:** Intermediate · **Estimated Time:** 30–35 minutes

You will build a REST service using contract-first design: the API's interface is written down and machine-validated before a single line of server logic exists. The deliverable is a validated OpenAPI 3.0.3 contract covering cursor pagination, idempotency keys, and URI versioning, plus server stubs generated directly from that contract.

The stack is OpenAPI 3.0.3, the Apidevtools Swagger tooling (`swagger-cli` for validation, `swagger-parser` for reading the contract), and Node.js with Express.

<p align="center">
  <img src="images/lab01-architecture.png" alt="Contract-First API Architecture Overview" width="750">
</p>

## Why these tools?

OpenAPI is a language-agnostic description of an HTTP API. Writing it first forces you to agree on the request shape, the response shape, and the error format before any implementation detail can leak into the design. `swagger-cli` proves the document is valid. `swagger-parser` lets a program read the contract and emit code from it. Express supplies the running HTTP server. `curl` and `jq` let you check the result by hand.

| Tool | Role in this lab |
| --- | --- |
| OpenAPI 3.0.3 | The written contract: endpoints, headers, schemas, status codes |
| `swagger-cli` | Validates the contract against the OpenAPI 3 schema |
| `swagger-parser` | Loads and dereferences the contract so a generator can read it |
| Node.js | Runs the Express server and the stub generator |
| Express | Handles HTTP routing, parsing, and responses |
| `curl` | Sends test requests from the terminal |
| `jq` | Formats JSON responses so they are readable |

---

## Prerequisites

Before starting this lab, ensure you have:

- **Node.js:** v20.x or v22.x LTS installed (`node -v`)
- **npm:** comes bundled with Node.js (`npm -v`)
- **CLI Utilities:** `curl` and `jq` installed (`curl --version`, `jq --version`)
- **Port Availability:** Host port `5000` must be free. Check with `lsof -i :5000` or `netstat -tuln | grep 5000`

> [!NOTE]
> **Architectural Callout: Header Requirement vs. Distributed De-duplication (H-02)**
> In this lab the server enforces the *presence* and *UUID format* of the `Idempotency-Key` header as part of the OpenAPI interface contract. True de-duplication, atomic mutual exclusion locks (`SETNX`), and replay caching require an external distributed state store (Redis), which is implemented in **[Lab 02: Idempotent Write Endpoint](../M6-L2/README.md)**. Submitting two requests with the same key in this lab will validate successfully and insert two distinct records.

> [!IMPORTANT]
> **Why this lab does not use OpenAPI Generator CLI**
> The official `openapi-generator-cli` package runs on the JVM and requires a Java runtime, which is not installed in the Poridhi VM. This lab uses `swagger-parser` to read the contract and a small generator script to emit Express stubs. The result is the same idea — code derived from the specification — without a Java dependency.

---

## Concepts

| Term | Definition |
| :--- | :--- |
| **Contract-First Design** | An API architecture approach where the interface specification is defined and reviewed before implementation code is written. |
| **OpenAPI 3.0.3** | A standardized specification format used to describe endpoints, headers, payload schemas, and response codes. |
| **Spec Validation** | The process of checking a document against the OpenAPI 3 meta-schema to catch malformed YAML, broken `$ref` links, and missing required fields before the document is used. |
| **Dereferencing** | The process of resolving every `$ref` pointer in a specification into a single self-contained document, so a consumer can walk the schemas directly. |
| **Stub Generation** | Producing compilable server scaffolding from a contract, so route paths, parameter names, and validation rules originate in the specification rather than in hand-written code. |
| **Cursor Pagination** | A pagination method that retrieves records sequentially after a unique pointer (`WHERE id > cursor LIMIT limit`), preventing duplicate items during concurrent writes. |
| **Idempotency Key** | A unique client-provided token supplied via the `Idempotency-Key` HTTP header that identifies a write mutation across retries. |
| **RFC 7807 Problem Details** | An IETF standard (`application/problem+json`) defining machine-readable HTTP error payloads containing `type`, `title`, `status`, `detail`, and `instance`. |
| **URI Versioning** | Explicit versioning within the URL path (`/api/v1/...`) that preserves backward compatibility for existing consumers. |

In a contract-first workflow, the specification is authored and validated before any server code is written. When the contract changes, the stubs are regenerated to match. Every request that arrives is checked against the same rules the contract declares — a missing header, a malformed UUID, a `limit` above the documented maximum, or a body field outside its documented range all produce an RFC 7807 Problem Details response rather than a stack trace.

---

## Objectives

By the end of this lab, you will be able to:

- Configure a project workspace with Express, YAML parsing, and Swagger tooling dependencies.
- Author an OpenAPI 3.0.3 specification defining order endpoints, write idempotency headers, cursor pagination bounds, single-order lookups, and RFC 7807 error models.
- Validate the specification with `swagger-cli` and read it programmatically with `swagger-parser`.
- Generate an Express server stub whose routes and validation rules are derived from the contract.
- Implement a contract-compliant Express server with in-memory order data and cursor pagination.
- Expose the application using the Poridhi Load Balancer on port 5000.
- Verify contract enforcement, boundary restrictions, cursor pagination, and automated unit tests.

---

## What You Will Build

```text
lab01-contract-first-api/
├── openapi.yaml          # OpenAPI 3.0.3 contract specification
├── package.json          # Node.js dependencies and script definitions
├── generate-stub.js      # Reads the contract and emits an Express stub
├── server.generated.js   # Auto-generated stub (do not hand-edit)
├── server.js             # Hand-written contract-compliant server
└── test.js               # Automated contract validation test suite
```

The contract is the single source of truth. `generate-stub.js` reads it and emits `server.generated.js`, which proves the contract is machine-usable. `server.js` is the version you carry forward, with real persistence and real pagination logic.

---

## Step 1: Initialize the Project Workspace

**What you are doing:** creating an empty folder for the lab and declaring which packages the project needs.

**Why it matters:** keeping the contract, the generator, and the server together makes it obvious that the contract drives everything else.

Start the Poridhi VM and open VS Code, then create the project directory:

```bash
mkdir -p ~/lab01-contract-first-api
cd ~/lab01-contract-first-api
```

<p align="center">
  <img src="images/package.png" alt="Create Project Directory" width="700">
</p>

Create a file named `package.json` with the following contents:

```json
{
  "name": "lab01-contract-first-api",
  "version": "1.0.0",
  "description": "Contract-First API with OpenAPI 3",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "test": "node test.js",
    "validate": "swagger-cli validate openapi.yaml",
    "generate": "node generate-stub.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "yaml": "^2.4.2",
    "@apidevtools/swagger-parser": "^13.1.0"
  },
  "devDependencies": {
    "@apidevtools/swagger-cli": "^4.0.4"
  }
}
```

**Explanation:**

- `main` — points to `server.js` as the entrypoint file of the application.
- `scripts.start` — defines the command `node server.js` to run the Express server.
- `scripts.test` — defines the command `node test.js` to execute the automated verification test suite.
- `scripts.validate` — validates `openapi.yaml` against the OpenAPI 3 meta-schema.
- `scripts.generate` — regenerates `server.generated.js` from the contract.
- `dependencies.express` — installs the Express web framework for routing and HTTP handling.
- `dependencies.yaml` — installs the YAML parser library to load and inspect `openapi.yaml` at runtime.
- `dependencies.@apidevtools/swagger-parser` — loads the contract and resolves every `$ref` so the generator can read the resolved schemas.
- `devDependencies.@apidevtools/swagger-cli` — provides the spec validator as a local binary, so nothing needs to be installed globally.

Install the project dependencies:

```bash
npm install
```

<p align="center">
  <img src="images/package.png" alt="Install Dependencies" width="700">
</p>

**Expected output:**

```text
added 96 packages in 3s

found 0 vulnerabilities
```

---

## Step 2: Define the OpenAPI 3.0.3 Contract

**What you are doing:** writing the API's agreement in YAML before implementing any routes.

**Why it matters:** the contract gives frontend developers, backend developers, and testers one shared description of the API. When it is wrong, everyone is wrong in the same documented way.

A useful way to read the file:

| Key | Purpose |
| :--- | :--- |
| `openapi` | Identifies the OpenAPI specification version. |
| `info` | Describes the API: title, version, description. |
| `servers` | Gives the base URL, including the version prefix. |
| `paths` | Lists the endpoints and their HTTP methods. |
| `components.schemas` | Holds reusable object definitions referenced by `$ref`. |
| `components.responses` | Holds reusable error responses referenced by `$ref`. |

Take your time with indentation: YAML uses spaces to show which fields belong inside other fields. A single stray space breaks the document.

Create a file named `openapi.yaml` with the following contents:

```yaml
openapi: 3.0.3

info:
  title: Orders Service API
  version: 1.0.0
  description: Contract-first API specification supporting cursor
    pagination, idempotency keys, and RFC 7807 error formats.

servers:
  - url: http://localhost:5000/api/v1
    description: Local Development Server

paths:
  /orders:
    post:
      summary: Create an order (Idempotent write)
      description: Submits a new order mutation. The client MUST include an
        Idempotency-Key header to prevent duplicate orders upon retries.
      operationId: createOrder
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          description: Unique client-generated token (UUID v4) guaranteeing
            idempotent execution.
          schema:
            type: string
            format: uuid
          example: "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order successfully placed and committed.
          headers:
            Location:
              schema:
                type: string
              example: "/api/v1/orders/101"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '400':
          $ref: '#/components/responses/ProblemDetails400'

    get:
      summary: List orders with cursor-based pagination
      description: Retrieves a chronological stream of orders using
        high-performance cursor pagination.
      operationId: listOrders
      parameters:
        - name: cursor
          in: query
          required: false
          description: Opaque cursor identifier representing the last seen order
            ID.
          schema:
            type: integer
          example: 100
        - name: limit
          in: query
          required: false
          description: Number of records to return per page.
          schema:
            type: integer
            minimum: 1
            maximum: 50
            default: 10
          example: 10
      responses:
        '200':
          description: Paginated collection of orders.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaginatedOrdersResponse'
        '400':
          $ref: '#/components/responses/ProblemDetails400'

  /orders/{id}:
    get:
      summary: Retrieve order by ID
      description: Returns the order details matching the provided order ID.
      operationId: getOrderById
      parameters:
        - name: id
          in: path
          required: true
          description: Numeric identifier of the order.
          schema:
            type: integer
          example: 101
      responses:
        '200':
          description: Order found and returned.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '404':
          $ref: '#/components/responses/ProblemDetails404'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required:
        - item_id
        - quantity
        - amount
      properties:
        item_id:
          type: integer
          example: 402
        quantity:
          type: integer
          minimum: 1
          example: 2
        amount:
          type: number
          format: float
          minimum: 0.01
          example: 150.50

    OrderResponse:
      type: object
      required:
        - id
        - item_id
        - quantity
        - amount
        - status
        - created_at
      properties:
        id:
          type: integer
          example: 101
        item_id:
          type: integer
          example: 402
        quantity:
          type: integer
          example: 2
        amount:
          type: number
          example: 150.50
        status:
          type: string
          enum: [PENDING, CONFIRMED, COMPLETED]
          example: CONFIRMED
        created_at:
          type: string
          format: date-time
          example: "2026-10-02T10:00:00Z"

    PaginationMetadata:
      type: object
      required:
        - next_cursor
        - has_more
        - count
      properties:
        next_cursor:
          type: integer
          nullable: true
          example: 110
        has_more:
          type: boolean
          example: true
        count:
          type: integer
          example: 10

    PaginatedOrdersResponse:
      type: object
      required:
        - data
        - pagination
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/OrderResponse'
        pagination:
          $ref: '#/components/schemas/PaginationMetadata'

    ProblemDetails:
      type: object
      description: RFC 7807 standard error structure.
      required:
        - type
        - title
        - status
        - detail
      properties:
        type:
          type: string
          format: uri
          example: "https://api.example.com/errors/invalid-parameter"
        title:
          type: string
          example: "Invalid Request Parameter"
        status:
          type: integer
          example: 400
        detail:
          type: string
          example: "The limit parameter exceeds maximum allowed of 50."
        instance:
          type: string
          example: "/api/v1/orders"

  responses:
    ProblemDetails400:
      description: Bad Request (RFC 7807 Problem Details).
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
    ProblemDetails404:
      description: Not Found (RFC 7807 Problem Details).
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
```

**Explanation:**

- `servers.url` — specifies `http://localhost:5000/api/v1` as the base URI path to enforce explicit API versioning.
- `paths./orders.post.parameters` — declares the `Idempotency-Key` header with `required: true` and UUID formatting.
- `paths./orders.get.parameters` — defines `cursor` and `limit` query parameters with explicit validation boundaries (`minimum: 1`, `maximum: 50`, `default: 10`).
- `paths./orders/{id}.get` — specifies the resource retrieval endpoint matching the `Location` header emitted on successful creation.
- `components.schemas.CreateOrderRequest` — enforces mandatory fields (`item_id`, `quantity`, `amount`) and minimum values (`quantity >= 1`, `amount >= 0.01`).
- `components.schemas.ProblemDetails` — declares the RFC 7807 error format with `type`, `title`, `status`, `detail`, and `instance` properties.
- `components.responses.ProblemDetails400` and `ProblemDetails404` — map standard HTTP client errors to `application/problem+json`.

---

## Step 3: Validate the Contract with Swagger CLI

**What you are doing:** running the specification through a validator that checks it against the OpenAPI 3 meta-schema.

**Why it matters:** a malformed contract is worse than no contract, because every tool that reads it will produce a confusing error. Validation catches broken indentation, dangling `$ref` pointers, and missing required fields in seconds.

Validate the specification:

```bash
npm run validate
```

<p align="center">
  <img src="images/test.png" alt="Validate OpenAPI Specification" width="700">
</p>

**Expected output:**

```text
> lab01-contract-first-api@1.0.0 validate
> swagger-cli validate openapi.yaml

openapi.yaml is valid
```

Now try to see what a failure looks like. Copy the contract and point one `$ref` at a schema that does not exist:

```bash
cp openapi.yaml broken.yaml
sed -i "s|#/components/schemas/CreateOrderRequest|#/components/schemas/DoesNotExist|" broken.yaml
npx swagger-cli validate broken.yaml
```

**Expected output:**

```text
Token "DoesNotExist" does not exist.
```

The command exits with a non-zero status, so you can gate a build on it.

Delete the broken copy so it does not confuse later steps:

```bash
rm broken.yaml
```

**Explanation:**

- `swagger-cli validate` — checks the document structure, the field names, and every `$ref` against the OpenAPI 3 meta-schema.
- The `Token "DoesNotExist" does not exist.` message — names the missing schema directly, so you know which `$ref` to repair without reading the whole file.
- `npm run validate` — resolves the local `swagger-cli` binary installed as a dev dependency; no global install is needed.

---

## Step 4: Generate Server Stubs from the Contract

**What you are doing:** writing a generator that reads the contract and emits Express route code from it.

**Why it matters:** this is the payoff of contract-first design. Change `maximum: 50` to `maximum: 100` in `openapi.yaml`, regenerate, and the enforcement code changes with it. The contract stops being documentation and becomes the source.

Create a file named `generate-stub.js` with the following contents:

```javascript
const fs = require('fs');
const path = require('path');
const SwaggerParser = require('@apidevtools/swagger-parser');

const SPEC_PATH = path.join(__dirname, 'openapi.yaml');
const OUT_PATH = path.join(__dirname, 'server.generated.js');

// Build a plausible example value for a schema, used to seed stub responses.
function pickExample(schema, depth = 0) {
  if (!schema || depth > 4) return null;
  if (schema.example !== undefined) return schema.example;
  if (schema.default !== undefined) return schema.default;
  if (schema.$ref) return null;
  if (schema.type === 'object' && schema.properties) {
    const obj = {};
    for (const [key, value] of Object.entries(schema.properties)) {
      obj[key] = pickExample(value, depth + 1);
    }
    return obj;
  }
  if (schema.type === 'array') return [];
  switch (schema.type) {
    case 'integer':
    case 'number':
      return 0;
    case 'boolean':
      return false;
    case 'string':
      return schema.format === 'date-time' ? new Date().toISOString() : 'string';
    default:
      return null;
  }
}

// Emit validation statements for one request-body property.
function renderValidation(propName, schema, indent) {
  const pad = ' '.repeat(indent);
  const lines = [];

  if (schema.type === 'integer' || schema.type === 'number') {
    lines.push(`${pad}if (typeof body.${propName} !== 'number') {`);
    lines.push(
      `${pad}  return problems(res, 400, 'Invalid Request Body', \`${propName} must be a number.\`, instance);`
    );
    lines.push(`${pad}}`);
  }

  if (schema.minimum !== undefined) {
    lines.push(`${pad}if (body.${propName} < ${schema.minimum}) {`);
    lines.push(
      `${pad}  return problems(res, 400, 'Value Out of Bounds', \`${propName} must be >= ${schema.minimum}.\`, instance);`
    );
    lines.push(`${pad}}`);
  }

  if (schema.maximum !== undefined) {
    lines.push(`${pad}if (body.${propName} > ${schema.maximum}) {`);
    lines.push(
      `${pad}  return problems(res, 400, 'Value Out of Bounds', \`${propName} must be <= ${schema.maximum}.\`, instance);`
    );
    lines.push(`${pad}}`);
  }

  return lines;
}

async function main() {
  const spec = await SwaggerParser.dereference(SPEC_PATH);
  const basePath = new URL(spec.servers[0].url).pathname.replace(/\/$/, '');

  const out = [];

  out.push('// AUTO-GENERATED FROM openapi.yaml');
  out.push('// Regenerate with: npm run generate');
  out.push('// Do not edit by hand. Your changes will be overwritten.');
  out.push('');
  out.push("const express = require('express');");
  out.push('');
  out.push('const app = express();');
  out.push('const PORT = process.env.PORT || 5000;');
  out.push('');
  out.push('app.use(express.json());');
  out.push('');
  out.push('// ---- Shared helpers ----');
  out.push('function problems(res, status, title, detail, instance) {');
  out.push("  res.setHeader('Content-Type', 'application/problem+json');");
  out.push('  return res.status(status).json({');
  out.push("    type: 'https://api.example.com/errors/' + status,");
  out.push('    title,');
  out.push('    status,');
  out.push('    detail,');
  out.push('    instance');
  out.push('  });');
  out.push('}');
  out.push('');

  for (const [routePath, pathItem] of Object.entries(spec.paths)) {
    for (const [method, operation] of Object.entries(pathItem)) {
      const parameters = operation.parameters || [];
      const headerParams = parameters.filter((p) => p.in === 'header');
      const queryParams = parameters.filter((p) => p.in === 'query');
      const pathParams = parameters.filter((p) => p.in === 'path');
      const expressPath = routePath.replace(/\{(\w+)\}/g, ':$1');
      const fullPath = `${basePath}${expressPath}`;

      out.push(`// ${method.toUpperCase()} ${fullPath} — ${operation.summary || operation.operationId}`);
      out.push(`app.${method}('${fullPath}', (req, res) => {`);
      out.push(`  const instance = '${fullPath}';`);

      // Required headers, with format checks pulled from the schema.
      for (const header of headerParams) {
        if (!header.required) continue;
        const variable = header.name.toLowerCase().replace(/-/g, '_');
        out.push(`  const ${variable} = req.header('${header.name}');`);
        out.push(`  if (!${variable}) {`);
        out.push(
          `    return problems(res, 400, 'Missing Required Header', 'The ${header.name} header is required for this operation.', instance);`
        );
        out.push('  }');
        if (header.schema && header.schema.format === 'uuid') {
          out.push(
            '  const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;'
          );
          out.push(`  if (!UUID_RE.test(${variable})) {`);
          out.push(
            `    return problems(res, 400, 'Invalid Header Format', 'The ${header.name} header must be a valid UUID string.', instance);`
          );
          out.push('  }');
        }
      }

      // Path parameters.
      for (const param of pathParams) {
        out.push(`  const ${param.name} = parseInt(req.params.${param.name}, 10);`);
        out.push(`  if (isNaN(${param.name})) {`);
        out.push(
          `    return problems(res, 400, 'Invalid Parameter', 'The ${param.name} path parameter must be an integer.', instance);`
        );
        out.push('  }');
      }

      // Query parameters, including their documented bounds.
      for (const param of queryParams) {
        const name = param.name;
        const hasDefault = param.schema.default !== undefined;
        const fallback = hasDefault ? param.schema.default : 'null';
        out.push(
          `  let ${name} = req.query.${name} !== undefined ? parseInt(req.query.${name}, 10) : ${fallback};`
        );
        if (!hasDefault) {
          out.push(`  if (${name} !== null && isNaN(${name})) {`);
          out.push(
            `    return problems(res, 400, 'Invalid Query Parameter', 'The ${name} query parameter must be an integer.', instance);`
          );
          out.push('  }');
        }
        if (param.schema.minimum !== undefined) {
          out.push(`  if (${name} < ${param.schema.minimum}) {`);
          out.push(
            `    return problems(res, 400, 'Invalid Query Parameter', \`The ${name} parameter must be >= ${param.schema.minimum}. Received: \${req.query.${name}}\`, instance);`
          );
          out.push('  }');
        }
        if (param.schema.maximum !== undefined) {
          out.push(`  if (${name} > ${param.schema.maximum}) {`);
          out.push(
            `    return problems(res, 400, 'Invalid Query Parameter', \`The ${name} parameter must be <= ${param.schema.maximum}. Received: \${req.query.${name}}\`, instance);`
          );
          out.push('  }');
        }
      }

      // Request body: required fields, types, and documented ranges.
      const requestBody = operation.requestBody;
      if (requestBody && requestBody.content && requestBody.content['application/json']) {
        const bodySchema = requestBody.content['application/json'].schema;
        const requiredFields = bodySchema.required || [];
        const properties = bodySchema.properties || {};

        out.push('  const body = req.body || {};');
        for (const field of requiredFields) {
          out.push(`  if (body.${field} === undefined) {`);
          out.push(
            `    return problems(res, 400, 'Invalid Request Body', '${field} is required.', instance);`
          );
          out.push('  }');
        }
        for (const [propName, propSchema] of Object.entries(properties)) {
          if (!requiredFields.includes(propName)) continue;
          out.push(...renderValidation(propName, propSchema, 2));
        }
      }

      // Success response, seeded from the documented schema example.
      const successCode =
        Object.keys(operation.responses).find((code) => code.startsWith('2')) || '200';
      const successResponse = operation.responses[successCode];
      const successSchema =
        successResponse && successResponse.content && successResponse.content['application/json']
          ? successResponse.content['application/json'].schema
          : null;

      if (successSchema && successSchema.type === 'object' && successSchema.properties) {
        const exampleObject = {};
        for (const [key, value] of Object.entries(successSchema.properties)) {
          exampleObject[key] = pickExample(value, 0);
        }
        out.push(
          `  const payload = ${JSON.stringify(exampleObject, null, 2).split('\n').join('\n  ')};`
        );
      } else if (successSchema && successSchema.type === 'array') {
        out.push('  const payload = [];');
      } else {
        out.push('  const payload = {};');
      }

      if (
        operation.responses['201'] &&
        operation.responses['201'].headers &&
        operation.responses['201'].headers.Location
      ) {
        out.push('  // TODO: replace with a real persistence layer');
        out.push("  res.setHeader('Location', '/api/v1/orders/1');");
      }

      out.push(`  return res.status(${successCode}).json(payload);`);
      out.push('});');
      out.push('');
    }
  }

  out.push('if (require.main === module) {');
  out.push('  app.listen(PORT, () => {');
  out.push("    console.log('[Generated Stub] Running on port ' + PORT);");
  out.push('  });');
  out.push('}');
  out.push('');
  out.push('module.exports = app;');
  out.push('');

  fs.writeFileSync(OUT_PATH, out.join('\n'));

  const routeCount = Object.values(spec.paths).reduce(
    (total, pathItem) => total + Object.keys(pathItem).length,
    0
  );
  console.log(`Generated ${path.basename(OUT_PATH)} from ${path.basename(SPEC_PATH)}`);
  console.log(`Routes generated: ${routeCount}`);
}

main().catch((err) => {
  console.error('Stub generation failed:', err.message);
  process.exit(1);
});
```

**Explanation:**

- `SwaggerParser.dereference(SPEC_PATH)` — loads `openapi.yaml` and resolves every `$ref` pointer into a single document, so `$ref: '#/components/schemas/CreateOrderRequest'` becomes a readable object with `required` and `properties`.
- `new URL(spec.servers[0].url).pathname` — extracts the `/api/v1` prefix straight from the contract, so URI versioning is defined in exactly one place.
- `routePath.replace(/\{(\w+)\}/g, ':$1')` — converts OpenAPI path templates like `/orders/{id}` into Express syntax like `/orders/:id`.
- `header.required` and `header.schema.format` — the generator reads `required: true` and `format: uuid` and emits both a presence check and a UUID format check, without either being hardcoded in the generator.
- `param.schema.minimum` and `param.schema.maximum` — the `limit` bounds of 1 and 50 come from the contract. Change them in `openapi.yaml` and regenerate.
- `pickExample()` — walks the response schema and builds a sample payload from the documented `example` values, so the stub returns a body that already matches the contract shape.
- `operation.responses['201'].headers.Location` — the presence of a `Location` header in the contract is what causes the generator to emit the `res.setHeader('Location', ...)` line.

Run the generator:

```bash
npm run generate
```

<p align="center">
  <img src="images/server start.png" alt="Generate Server Stubs" width="700">
</p>

**Expected output:**

```text
> lab01-contract-first-api@1.0.0 generate
> node generate-stub.js

Generated server.generated.js from openapi.yaml
Routes generated: 3
```

Inspect what was produced:

```bash
head -40 server.generated.js
```

**Expected output:**

```text
// AUTO-GENERATED FROM openapi.yaml
// Regenerate with: npm run generate
// Do not edit by hand. Your changes will be overwritten.

const express = require('express');

const app = express();
const PORT = process.env.PORT || 5000;

app.use(express.json());

// ---- Shared helpers ----
function problems(res, status, title, detail, instance) {
  res.setHeader('Content-Type', 'application/problem+json');
  return res.status(status).json({
    type: 'https://api.example.com/errors/' + status,
    title,
    status,
    detail,
    instance
  });
}

// POST /api/v1/orders — Create an order (Idempotent write)
app.post('/api/v1/orders', (req, res) => {
  const instance = '/api/v1/orders';
  const idempotency_key = req.header('Idempotency-Key');
  if (!idempotency_key) {
    return problems(res, 400, 'Missing Required Header', 'The Idempotency-Key header is required for this operation.', instance);
  }
  const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
  if (!UUID_RE.test(idempotency_key)) {
    return problems(res, 400, 'Invalid Header Format', 'The Idempotency-Key header must be a valid UUID string.', instance);
  }
```

Notice what is present in the generated file: the versioned route path `/api/v1/orders`, the required-header check, the UUID format check, and the `limit` bounds check. None of those were typed into a route by hand.

**To see the contract really drive the output,** change the maximum page size in the contract and regenerate:

```bash
sed -i 's/maximum: 50/maximum: 25/' openapi.yaml
npm run generate
grep "must be <= " server.generated.js
```

**Expected output:**

```text
  if (limit > 25) {
    return problems(res, 400, 'Invalid Query Parameter', `The limit parameter must be <= 25. Received: ${req.query.limit}`, instance);
```

Change it back so the rest of the lab matches the contract:

```bash
sed -i 's/maximum: 25/maximum: 50/' openapi.yaml
npm run generate
```

---

## Step 5: Implement the Contract-Compliant Server

**What you are doing:** writing the Express application that serves the contract with real behaviour — in-memory storage, genuine cursor pagination, and RFC 7807 error responses.

**Why it matters:** the generated stub proves the contract is machine-readable. This file is what you carry into a real project, because it does the work the contract describes.

The generated stub always returns example data. The server below keeps the same routes and the same validation rules, but stores orders, advances cursors, and returns real records.

Before reading the code, know these basics:

| Concept | Meaning |
| :--- | :--- |
| Route | Connects an HTTP method and URL to a function. |
| `req` | Contains information sent by the client. |
| `res` | Used to send the response. |
| `req.body` | Contains JSON sent in the request body. |
| `req.query` | Contains query parameters such as `limit` and `cursor`. |
| `req.params` | Contains values from a path such as `:id`. |

Create a file named `server.js` with the following contents:

```javascript
const express = require('express');
const fs = require('fs');
const path = require('path');
const yaml = require('yaml');

const app = express();
const PORT = process.env.PORT || 5000;

app.use(express.json());

// Load the OpenAPI 3 contract. This is the single source of truth.
const openapiPath = path.join(__dirname, 'openapi.yaml');
const openapiSpec = yaml.parse(fs.readFileSync(openapiPath, 'utf8'));

// Helper: RFC 7807 Problem Details formatter.
function sendProblem(res, status, title, detail, instance, type = 'about:blank') {
  res.setHeader('Content-Type', 'application/problem+json');
  return res.status(status).json({
    type,
    title,
    status,
    detail,
    instance
  });
}

// In-memory orders database, seeded with 25 records for pagination testing.
let ordersDb = [];
let nextOrderId = 101;

for (let i = 1; i <= 25; i++) {
  ordersDb.push({
    id: nextOrderId++,
    item_id: 400 + (i % 5),
    quantity: (i % 3) + 1,
    amount: parseFloat((25.5 * i).toFixed(2)),
    status: 'CONFIRMED',
    created_at: new Date(Date.now() - (25 - i) * 60000).toISOString()
  });
}

// Route: expose the raw OpenAPI specification.
app.get('/openapi.yaml', (req, res) => {
  res.setHeader('Content-Type', 'text/yaml');
  res.sendFile(openapiPath);
});

// UUID v4 validation regex, matching the `format: uuid` in the contract.
const UUID_V4_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

// Route: POST /api/v1/orders — contract-driven mutation endpoint.
app.post('/api/v1/orders', (req, res) => {
  const idempotencyKey = req.header('Idempotency-Key');

  // Contract check 1: header presence.
  if (!idempotencyKey) {
    return sendProblem(
      res,
      400,
      'Missing Required Header',
      'The Idempotency-Key header is required for this write operation.',
      '/api/v1/orders',
      'https://api.example.com/errors/missing-header'
    );
  }

  // Contract check 2: header UUID format compliance.
  if (!UUID_V4_REGEX.test(idempotencyKey)) {
    return sendProblem(
      res,
      400,
      'Invalid Header Format',
      'The Idempotency-Key header must be a valid UUID string.',
      '/api/v1/orders',
      'https://api.example.com/errors/invalid-header'
    );
  }

  // Contract check 3: request body fields and types.
  const { item_id, quantity, amount } = req.body || {};

  if (typeof item_id !== 'number' || typeof quantity !== 'number' || typeof amount !== 'number') {
    return sendProblem(
      res,
      400,
      'Invalid Request Body',
      'item_id, quantity, and amount must all be numbers.',
      '/api/v1/orders',
      'https://api.example.com/errors/invalid-body'
    );
  }

  if (quantity < 1 || amount <= 0) {
    return sendProblem(
      res,
      400,
      'Value Out of Bounds',
      'quantity must be >= 1 and amount must be > 0.',
      '/api/v1/orders',
      'https://api.example.com/errors/out-of-bounds'
    );
  }

  // Create the order entity.
  const newOrder = {
    id: nextOrderId++,
    item_id,
    quantity,
    amount: parseFloat(amount.toFixed(2)),
    status: 'CONFIRMED',
    created_at: new Date().toISOString()
  };

  ordersDb.push(newOrder);

  // RFC 9110 compliant Location header pointing to the resource URL.
  res.setHeader('Location', `/api/v1/orders/${newOrder.id}`);

  return res.status(201).json(newOrder);
});

// Route: GET /api/v1/orders/:id — retrieve a single order by ID.
app.get('/api/v1/orders/:id', (req, res) => {
  const orderId = parseInt(req.params.id, 10);

  if (isNaN(orderId)) {
    return sendProblem(
      res,
      400,
      'Invalid Parameter',
      'Order ID path parameter must be an integer.',
      req.originalUrl,
      'https://api.example.com/errors/invalid-parameter'
    );
  }

  const order = ordersDb.find((o) => o.id === orderId);

  if (!order) {
    return sendProblem(
      res,
      404,
      'Order Not Found',
      `No order found with ID ${orderId}.`,
      req.originalUrl,
      'https://api.example.com/errors/not-found'
    );
  }

  return res.status(200).json(order);
});

// Route: GET /api/v1/orders — cursor-based pagination endpoint.
app.get('/api/v1/orders', (req, res) => {
  let limit = parseInt(req.query.limit, 10);

  if (isNaN(limit)) {
    limit = 10; // Default, matching `default: 10` in the contract.
  }

  // Contract check: limit bounds [1..50].
  if (limit < 1 || limit > 50) {
    return sendProblem(
      res,
      400,
      'Invalid Query Parameter',
      `The limit parameter must be between 1 and 50. Received: ${req.query.limit}`,
      '/api/v1/orders',
      'https://api.example.com/errors/invalid-parameter'
    );
  }

  const cursor = req.query.cursor ? parseInt(req.query.cursor, 10) : 0;

  // Select records strictly after the cursor ID, then slice to the limit.
  const filtered = ordersDb.filter((o) => o.id > cursor);
  const pageItems = filtered.slice(0, limit);

  const nextCursor = pageItems.length > 0 ? pageItems[pageItems.length - 1].id : null;
  const hasMore = filtered.length > limit;

  return res.status(200).json({
    data: pageItems,
    pagination: {
      next_cursor: hasMore ? nextCursor : null,
      has_more: hasMore,
      count: pageItems.length
    }
  });
});

if (require.main === module) {
  app.listen(PORT, () => {
    console.log(`[Contract-First Server] Running on port ${PORT}`);
    console.log(`OpenAPI Spec accessible at http://localhost:${PORT}/openapi.yaml`);
  });
}

module.exports = app;
```

**Explanation:**

- `yaml.parse(fs.readFileSync(...))` — reads and parses `openapi.yaml` into memory upon server startup, so the contract travels with the service.
- `sendProblem()` — helper function setting `Content-Type: application/problem+json` and returning RFC 7807 compliant error payloads.
- `ordersDb` — in-memory array pre-seeded with 25 records so cursor pagination can be verified immediately.
- `UUID_V4_REGEX` — validates the `Idempotency-Key` header against the UUID format declared in `openapi.yaml`.
- `res.setHeader('Location', ...)` — returns the resource URI for newly created orders alongside HTTP 201 Created.
- `app.get('/api/v1/orders/:id')` — provides the endpoint backing the `Location` header so consumers can immediately resolve created resources.
- `ordersDb.filter((o) => o.id > cursor)` — executes cursor-based pagination by selecting items strictly greater than the cursor ID and slicing up to the validated `limit`.

---

## Step 6: Expose the Server via Poridhi Load Balancer

**What you are doing:** starting your API and making it reachable through the Poridhi Load Balancer.

**Why it matters:** your server listens inside the lab environment. The load balancer provides a public URL that forwards requests to that server, so the API can be tested from any browser.

Open a terminal and start the server process in the background on port 5000:

```bash
node server.js &
```

<p align="center">
  <img src="images/server start.png" alt="Start Server" width="700">
</p>

Retrieve the private IP address of your virtual machine:

```bash
hostname -I | awk '{print $1}'
```

<p align="center">
  <img src="images/IP address.png" alt="Get Private IP" width="700">
</p>

Configure the Poridhi Load Balancer:

1. Open the **Load Balancer** modal in the Poridhi interface.
2. Enter the IP address obtained from `hostname -I`.
3. Set the target port to **5000**.
4. Click **Expose** to generate an active public URL.

<p align="center">
  <img src="images/step4-loadbalancer-modal.png" alt="Load Balancer Modal" width="700">
</p>

<p align="center">
  <img src="images/step4-loadbalancer-exposed.png" alt="Load Balancer Exposed" width="700">
</p>

Open the generated Load Balancer URL in your web browser to confirm connectivity:

<p align="center">
  <img src="images/step4-browser-status.png" alt="Browser Status" width="700">
</p>

---

## Verification

You will verify contract validation, mutation handling, query parameter limits, single-item retrieval, and cursor pagination using `curl` and the automated test runner.

### How to read each test

1. Run the command.
2. Look at the HTTP status code.
3. Look at the response headers and JSON body.
4. Compare the actual result with the expected output.
5. Read the "What to Observe" note to understand what the test proves.

### Scenario 1: Missing Idempotency-Key Header (Expected 400 Bad Request)

**What are we testing?** The contract declares `Idempotency-Key` as a required header on `POST /orders`. This request intentionally leaves it out, so the server should reject it before touching the database.

Execute a `POST` request without the `Idempotency-Key` header:

```bash
curl -i -X POST http://localhost:5000/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"item_id": 402, "quantity": 1, "amount": 99.50}'
```

<p align="center">
  <img src="images/bad-request.png" alt="Test 1 Missing Idempotency Key" width="700">
</p>

**Expected output:**

```text
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json; charset=utf-8

{
  "type": "https://api.example.com/errors/missing-header",
  "title": "Missing Required Header",
  "status": 400,
  "detail": "The Idempotency-Key header is required for this write operation.",
  "instance": "/api/v1/orders"
}
```

> **What to Observe:** The server returns standard RFC 7807 `application/problem+json` specifying `status: 400` and `title: Missing Required Header`.

---

### Scenario 2: Valid Order Placement and Resource Lookup (Expected 201 Created and 200 OK)

**What are we testing?** First, send a valid order. Then use the URL returned in the `Location` response header to retrieve that same order.

- **201 Created** means a new resource was created.
- **Location** tells the client where the new resource can be retrieved.
- **200 OK** means the follow-up retrieval succeeded.

Execute a `POST` request supplying a valid `Idempotency-Key` UUID header:

```bash
curl -i -X POST http://localhost:5000/api/v1/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d" \
  -d '{"item_id": 402, "quantity": 2, "amount": 199.00}'
```

<p align="center">
  <img src="images/post request.png" alt="Test 2 Valid Order" width="700">
</p>

**Expected output:**

```text
HTTP/1.1 201 Created
Location: /api/v1/orders/126
Content-Type: application/json; charset=utf-8

{
  "id": 126,
  "item_id": 402,
  "quantity": 2,
  "amount": 199,
  "status": "CONFIRMED",
  "created_at": "2026-10-02T02:00:00.000Z"
}
```

Now retrieve the created resource using the URI returned in the `Location` header:

```bash
curl -i http://localhost:5000/api/v1/orders/126
```

<p align="center">
  <img src="images/get request.png" alt="Retrieve Created Order" width="700">
</p>

**Expected output:**

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "id": 126,
  "item_id": 402,
  "quantity": 2,
  "amount": 199,
  "status": "CONFIRMED",
  "created_at": "2026-10-02T02:00:00.000Z"
}
```

> **What to Observe:** The `Location: /api/v1/orders/126` header points to a resolvable resource that returns `200 OK` with the exact committed state.

> [!NOTE]
> **Observation: Header Presence vs. Deduplication**
> If you repeat the `POST` request using the identical `Idempotency-Key`, the server returns `201 Created` with a new `id` (`127`). This is expected in Lab 01: the OpenAPI specification validates header presence and syntax, but true distributed deduplication requires the Redis-backed locking and replay caching built in **[Lab 02: Idempotent Write Endpoint](../M6-L2/README.md)**.

---

### Scenario 3: Query Parameter Limit Out of Bounds (Expected 400 Bad Request)

**What are we testing?** The contract allows a page size from 1 to 50. We deliberately ask for 100, so the server should return a clear 400 Bad Request instead of accepting the invalid value.

Query `/api/v1/orders` with `limit=100`, exceeding the contract maximum of 50:

```bash
curl -i "http://localhost:5000/api/v1/orders?limit=100"
```

<p align="center">
  <img src="images/idempotent-test.png" alt="Test 3 Query Limit Violation" width="700">
</p>

**Expected output:**

```text
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json; charset=utf-8

{
  "type": "https://api.example.com/errors/invalid-parameter",
  "title": "Invalid Query Parameter",
  "status": 400,
  "detail": "The limit parameter must be between 1 and 50. Received: 100",
  "instance": "/api/v1/orders"
}
```

> **What to Observe:** The contract boundary of `maximum: 50` is enforced, rejecting out-of-range `limit` queries before database retrieval.

---

### Scenario 4: Cursor-Based Pagination

**What are we testing?** We ask for the first three orders, read the `next_cursor` value, and use it to request the next page.

If page one ends with order ID 103, page two uses `cursor=103`. The server filters for IDs greater than 103, so the next page starts at 104 in this sample.

Retrieve the first page with `limit=3`:

```bash
curl -s "http://localhost:5000/api/v1/orders?limit=3" | jq .
```

<p align="center">
  <img src="images/pagination part-1.png" alt="Test 4 Pagination Page 1" width="700">
</p>

**Expected output:**

```json
{
  "data": [
    { "id": 101, "item_id": 401, "quantity": 2, "amount": 25.5, "status": "CONFIRMED" },
    { "id": 102, "item_id": 402, "quantity": 3, "amount": 51, "status": "CONFIRMED" },
    { "id": 103, "item_id": 403, "quantity": 1, "amount": 76.5, "status": "CONFIRMED" }
  ],
  "pagination": {
    "next_cursor": 103,
    "has_more": true,
    "count": 3
  }
}
```

Retrieve the second page using `cursor=103`:

```bash
curl -s "http://localhost:5000/api/v1/orders?limit=3&cursor=103" | jq .
```

<p align="center">
  <img src="images/pagination part-2.png" alt="Test 4 Pagination Page 2" width="700">
</p>

**Expected output:**

```json
{
  "data": [
    { "id": 104, "item_id": 404, "quantity": 2, "amount": 102, "status": "CONFIRMED" },
    { "id": 105, "item_id": 400, "quantity": 3, "amount": 127.5, "status": "CONFIRMED" },
    { "id": 106, "item_id": 401, "quantity": 1, "amount": 153, "status": "CONFIRMED" }
  ],
  "pagination": {
    "next_cursor": 106,
    "has_more": true,
    "count": 3
  }
}
```

> **What to Observe:** The second page starts strictly at ID 104 (`WHERE id > 103`), with zero item overlap between page requests.

---

### Scenario 5: The Generated Stub Enforces the Same Contract (Expected 201 Created)

**What are we testing?** The stub in Step 4 was written by a generator, not by hand. It should behave identically to the hand-written server on the rules the contract defines.

Start the generated stub on a different port so it does not collide with the running server:

```bash
PORT=5001 node server.generated.js &
```

Send the same valid order to the stub:

```bash
curl -i -X POST http://localhost:5001/api/v1/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d" \
  -d '{"item_id": 402, "quantity": 2, "amount": 199.00}'
```

**Expected output:**

```text
HTTP/1.1 201 Created
Location: /api/v1/orders/1
Content-Type: application/json; charset=utf-8

{
  "id": 101,
  "item_id": 402,
  "quantity": 2,
  "amount": 150.5,
  "status": "CONFIRMED",
  "created_at": "2026-10-02T10:00:00Z"
}
```

Check that the stub also rejects an out-of-range `limit`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://localhost:5001/api/v1/orders?limit=100"
```

**Expected output:**

```text
400
```

Stop the stub when you are done:

```bash
kill %2 2>/dev/null || pkill -f "node server.generated.js"
```

> **What to Observe:** The stub returns `201 Created` with a `Location` header and rejects `limit=100` with `400`. The validation rules came from `openapi.yaml`, not from a human typing them into a route.

---

### Scenario 6: Automated Contract Validation Test Suite

**What are we testing?** Instead of manually running each request, `test.js` sends several requests automatically and checks their status codes and response data.

The script starts the Express app on a temporary port, runs the checks, and closes the test server when finished.

Create a file named `test.js` with the following contents:

```javascript
const http = require('http');
const app = require('./server');

const server = app.listen(0, async () => {
  const port = server.address().port;
  const baseUrl = `http://localhost:${port}`;

  console.log(`[TEST RUNNER] Running contract verification against port ${port}...\n`);

  function request(method, path, headers = {}, body = null) {
    return new Promise((resolve, reject) => {
      const url = new URL(path, baseUrl);
      const req = http.request(
        url,
        {
          method,
          headers: { 'Content-Type': 'application/json', ...headers }
        },
        (res) => {
          let data = '';
          res.on('data', (chunk) => (data += chunk));
          res.on('end', () => {
            let parsed = null;
            try {
              parsed = JSON.parse(data);
            } catch (e) {
              parsed = data;
            }
            resolve({ status: res.statusCode, headers: res.headers, body: parsed });
          });
        }
      );

      req.on('error', reject);
      if (body) req.write(JSON.stringify(body));
      req.end();
    });
  }

  try {
    // Test 1: Missing Idempotency-Key on POST -> Expect 400 Problem Details
    console.log('Test 1: Asserting 400 Bad Request when Idempotency-Key header is omitted...');
    const res1 = await request('POST', '/api/v1/orders', {}, { item_id: 1, quantity: 2, amount: 50.0 });

    if (res1.status === 400 && res1.body.title === 'Missing Required Header') {
      console.log('  PASS: 400 RFC 7807 returned properly for missing header.\n');
    } else {
      throw new Error(`Test 1 Failed: Received status ${res1.status}`);
    }

    // Test 2: Valid Order Creation with Idempotency-Key -> Expect 201 Created
    console.log('Test 2: Creating an order with valid schema and Idempotency-Key...');
    const res2 = await request(
      'POST',
      '/api/v1/orders',
      { 'Idempotency-Key': '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d' },
      { item_id: 402, quantity: 3, amount: 199.99 }
    );

    if (res2.status === 201 && res2.body.id && res2.headers.location) {
      console.log(`  PASS: 201 Created returned. Order ID: ${res2.body.id}, Location: ${res2.headers.location}\n`);
    } else {
      throw new Error(`Test 2 Failed: Received status ${res2.status}`);
    }

    // Test 3: Pagination Limit Out of Bounds (limit > 50) -> Expect 400 Problem Details
    console.log('Test 3: Requesting pagination limit=100 (exceeding max 50 contract rule)...');
    const res3 = await request('GET', '/api/v1/orders?limit=100');

    if (res3.status === 400 && res3.body.title === 'Invalid Query Parameter') {
      console.log('  PASS: 400 RFC 7807 returned for limit out of bounds.\n');
    } else {
      throw new Error(`Test 3 Failed: Received status ${res3.status}`);
    }

    // Test 4: Cursor-based Pagination Retrieval -> Expect 200 OK with next_cursor
    console.log('Test 4: Querying first page of orders (limit=5)...');
    const res4 = await request('GET', '/api/v1/orders?limit=5');

    if (res4.status === 200 && res4.body.data.length === 5 && res4.body.pagination.next_cursor) {
      const nextCursor = res4.body.pagination.next_cursor;
      console.log(`  PASS: Page 1 returned 5 items. Next cursor token: ${nextCursor}`);
      console.log(`  Querying Page 2 using cursor=${nextCursor}...`);

      const res5 = await request('GET', `/api/v1/orders?limit=5&cursor=${nextCursor}`);

      if (res5.status === 200 && res5.body.data.length === 5 && res5.body.data[0].id > nextCursor) {
        console.log(`  PASS: Page 2 returned next 5 items without overlap. First item ID: ${res5.body.data[0].id}\n`);
      } else {
        throw new Error('Test 4 Page 2 Failed');
      }
    } else {
      throw new Error(`Test 4 Failed: Received status ${res4.status}`);
    }

    // Test 5: Single Order Lookup via the Location header -> Expect 200 OK
    console.log('Test 5: Resolving the created order through its Location header...');
    const res6 = await request('GET', `/api/v1/orders/${res2.body.id}`);

    if (res6.status === 200 && res6.body.id === res2.body.id) {
      console.log(`  PASS: Order ${res6.body.id} resolved successfully.\n`);
    } else {
      throw new Error(`Test 5 Failed: Received status ${res6.status}`);
    }

    console.log('  ALL 5 CONTRACT-FIRST API TESTS PASSED SUCCESSFULLY!');
  } catch (err) {
    console.error('Test suite failed:', err);
    process.exitCode = 1;
  } finally {
    server.close();
  }
});
```

**Explanation:**

- `app.listen(0)` — spawns the Express server on an ephemeral random port to isolate automated tests from the running server on port 5000.
- `request()` — wraps Node.js `http.request` into a promise-based client to test status codes, headers, and parsed JSON responses.
- **Test 1** — asserts that omitting `Idempotency-Key` triggers an HTTP 400 status with RFC 7807 payload title `Missing Required Header`.
- **Test 2** — asserts that supplying a valid body and key produces HTTP 201 Created with a populated `Location` header.
- **Test 3** — asserts that requesting `limit=100` triggers HTTP 400 Bad Request with `Invalid Query Parameter`.
- **Test 4** — asserts that sequential queries across pages yield distinct records without ID overlap using `next_cursor`.
- **Test 5** — asserts that the `Location` header returned by Test 2 resolves to a retrievable resource.

The test script uses `if` statements to compare actual results with expected results. If a check fails, the script throws an error. If every check passes, it prints the success message.

Execute `npm test` to run the test suite:

```bash
npm test
```

<p align="center">
  <img src="images/test.png" alt="Automated Test Suite Output" width="700">
</p>

**Expected output:**

```text
> lab01-contract-first-api@1.0.0 test
> node test.js

[TEST RUNNER] Running contract verification against port 39707...

Test 1: Asserting 400 Bad Request when Idempotency-Key header is omitted...
  PASS: 400 RFC 7807 returned properly for missing header.

Test 2: Creating an order with valid schema and Idempotency-Key...
  PASS: 201 Created returned. Order ID: 126, Location: /api/v1/orders/126

Test 3: Requesting pagination limit=100 (exceeding max 50 contract rule)...
  PASS: 400 RFC 7807 returned for limit out of bounds.

Test 4: Querying first page of orders (limit=5)...
  PASS: Page 1 returned 5 items. Next cursor token: 105
  Querying Page 2 using cursor=105...
  PASS: Page 2 returned next 5 items without overlap. First item ID: 106

Test 5: Resolving the created order through its Location header...
  PASS: Order 126 resolved successfully.

  ALL 5 CONTRACT-FIRST API TESTS PASSED SUCCESSFULLY!
```

---

## Troubleshooting

If a command fails, do not immediately change the code. First check the error message and confirm that you completed the previous step.

| Issue | Cause | Solution |
| :--- | :--- | :--- |
| `Error: listen EADDRINUSE :::5000` | Another process is already running on port 5000 (for example from an earlier run or Lab 02). | Run `lsof -i :5000` to find the process ID, then `kill -9 <PID>`, or run `pkill -f "node server.js"`. |
| `curl: (7) Failed to connect to localhost port 5000` | The Express server process is not active or crashed on boot. | Start the server with `node server.js` in a dedicated terminal and inspect the console error output. |
| `jq: command not found` | The `jq` JSON formatting utility is not installed on the system. | Install it using `sudo apt-get update && sudo apt-get install -y jq`. |
| `swagger-cli: not found` | Dependencies were not installed, or `npm install` was run before `package.json` was saved. | Run `npm install` in the project directory and confirm `node_modules/.bin/swagger-cli` exists. |
| `Error parsing openapi.yaml: bad indentation` | A YAML block has inconsistent indentation, usually a tab character or a misaligned nested key. | The error output shows the offending line number and a caret. Fix the indentation to match the surrounding keys, then run `npm run validate` again. |
| `Token "SomeSchema" does not exist.` | A `$ref` points to a schema name that does not exist in `components`. | Check the spelling of the schema name against `components.schemas`, and confirm the pointer starts with `#/components/`. |
| `Stub generation failed: ...` | The generator could not read or dereference the contract. | Run `npm run validate` first. Fix the reported specification error, then regenerate. |
| `Cannot find module 'express'` | `npm install` was skipped, or the server was started from a different directory. | Run `npm install` inside `~/lab01-contract-first-api` and start the server from that same directory. |

---

## Teardown & Cleanup

When you have finished testing this lab, stop the background Node.js server before starting Lab 02:

```bash
pkill -f "node server.js" || true
pkill -f "node server.generated.js" || true
```

Confirm port 5000 is released:

```bash
lsof -i :5000
```

---

## Conclusion

You designed an OpenAPI 3.0.3 specification enforcing URI versioning, write idempotency headers, cursor pagination bounds, resource lookup, and RFC 7807 Problem Details. You validated the document with Swagger CLI, then proved it was machine-usable by generating an Express stub whose routes and validation rules were read directly from the contract. You implemented a contract-compliant server with real cursor pagination, exposed it through the Poridhi Load Balancer, and verified boundary enforcement, single-resource retrieval, and error responses through cURL and an automated test suite.
