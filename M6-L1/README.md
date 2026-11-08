
⠿Lab 01 — Contract-First API
⠿
 · 



⠿Introduction and Learning Objectives
⠿In this lab, you will build a small Orders REST API using Node.js, Express, and OpenAPI 3.0.3.

⠿You will first describe how the API should work in an openapi.yaml file. That file is the API contract: it documents the endpoints, required headers, request data, response data, and error formats. Then you will write a small Express server that follows the important rules in the contract.

⠿By the end of the lab, you will be able to:

⠿
Create an order with POST /api/v1/orders.

Retrieve one order with GET /api/v1/orders/{id}.

List orders in pages with GET /api/v1/orders.

Return errors in the RFC 7807 Problem Details format.

Run the service locally and expose it through the Poridhi Load Balancer on port 5000.

⠿
Important scope note: This lab checks that an Idempotency-Key header is present and has UUID format. It does not prevent duplicate orders when the same key is reused. Real duplicate prevention using Redis is covered in Lab 02. Also, you will write server.js yourself in this lab; it is a contract-following server skeleton, not code generated automatically by OpenAPI Generator.

⠿Figure 01: Request-Response Workflow


⠿Objectives
⠿By the end of the lab, you should be able to explain what each of these pieces does — not just copy the commands:

⠿
Configure the project workspace with Express and YAML parser dependencies.

Author an OpenAPI 3.0.3 specification defining /orders endpoints, write idempotency headers, query limits, single-order lookups, and RFC 7807 error models.

Implement an Express server stub that enforces the OpenAPI contract rules and manages in-memory order data.

Expose the application using the Poridhi Load Balancer on port 5000.

Verify contract enforcement, boundary restrictions, cursor pagination, and automated unit tests.


⠿Concepts and Prerequisites

⠿Concepts
Term

Definition

Contract-First Design

An API architecture approach where the interface specification is defined and reviewed before implementation code is written.

OpenAPI 3.0.3

A standardized specification format used to describe endpoints, headers, payload schemas, and response codes.

Cursor Pagination

A pagination method that retrieves records sequentially after a unique pointer (WHERE id > cursor LIMIT limit), preventing duplicate items during concurrent writes.

Idempotency Key

A unique client-provided token supplied via the Idempotency-Key HTTP header that identifies a write mutation across retries.

RFC 7807 Problem Details

An IETF standard (application/problem+json) defining machine-readable HTTP error payloads containing type, title, status, detail, and instance.

URI Versioning

Explicit versioning within the URL path (/api/v1/...) that preserves backward compatibility for existing consumers.


⠿Prerequisites
⠿Why do you need these tools?

Tool

Why you need it

Node.js

Runs the Express server and the test script.

npm

Installs the libraries used by the project.

curl

Sends HTTP requests from your terminal.

jq

Formats JSON output so it is easier to read.

Port 5000

Where the local API server will listen.

⠿Before starting this lab, ensure you have:

⠿
Node.js: v20.x or v22.x LTS installed (node -v)

CLI utilities: curl and jq installed (curl --version, jq --version)

Port availability: Host port 5000 must be free. Check with lsof -i :5000 or netstat -tuln | grep 5000


⠿Project Structure and Step 1: Initialize the Workspace
⠿You will create four files:

⠿
openapi.yaml — the written API contract.

package.json — project information, dependencies, and shortcuts for running commands.

server.js — the Express application that handles requests.

test.js — a script that sends requests to your server and checks the results.

text

⠿lab01-contract-first-api/
├── openapi.yaml   # OpenAPI 3.0.3 contract specification
├── package.json   # Node.js dependencies and script definitions
├── server.js      # Express server enforcing contract rules
└── test.js        # Automated contract validation test suite

⠿Step 1: Initialize the Project Workspace
⠿What you are doing: creating an empty folder for the lab and telling Node.js which packages the project needs.

⠿Why it matters: keeping the files together makes the project easier to run, test, and share.


⠿Step 1: Initialize the Project Workspace
⠿Start the Poridhi VM and go to VS Code.

⠿Create the project directory:

Bash

⠿mkdir -p ~/lab01-contract-first-api
cd ~/lab01-contract-first-api
⠿Create a file named package.json with the following contents:

JSON

⠿{
  "name": "lab01-contract-first-api",
  "version": "1.0.0",
  "description": "Contract-First API with OpenAPI 3",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "test": "node test.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "yaml": "^2.4.2"
  }
}

⠿Explanation
Field

Purpose

main

Points to server.js as the entrypoint file of the application.

scripts.start

Defines the command node server.js to run the Express server.

scripts.test

Defines the command node test.js to execute the automated verification test suite.

dependencies.express

Installs the Express web framework for routing and HTTP handling.

dependencies.yaml

Installs the YAML parser library to load and inspect openapi.yaml at runtime.

⠿Install the project dependencies:

Bash

⠿npm install
⠿Expected output:

text

⠿added 69 packages in 2s
found 0 vulnerabilities

⠿Step 2: Define the OpenAPI 3.0.3 Contract

⠿Step 2: Define the OpenAPI 3.0.3 Contract
⠿What you are doing: writing the API's agreement in YAML before implementing the routes.

⠿Why it matters: the contract gives frontend developers, backend developers, and testers one shared description of the API.

⠿A useful way to read the file:

⠿
openapi identifies the OpenAPI specification version.

info describes the API.

servers gives the base URL.

paths lists the endpoints and their methods.

components contains reusable schemas and response definitions.

⠿Take your time with indentation: YAML uses spaces to show which fields belong inside other fields.


⠿Step 2: Define the OpenAPI 3.0.3 Contract
⠿Create a file named openapi.yaml with the following contents:

YAML

⠿openapi: 3.0.3
info:
  title: Orders Service API
  version: 1.0.0
  description: Contract-first API specification supporting cursor pagination, idempotency keys, and RFC 7807 error formats.
servers:
  - url: http://localhost:5000/api/v1
    description: Local Development Server

paths:
  /orders:
    post:
      summary: Create an order (Idempotent write)
      description: Submits a new order mutation. The client MUST include an Idempotency-Key header to prevent duplicate orders upon retries.
      operationId: createOrder
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          description: Unique client-generated token (UUID v4) guaranteeing idempotent execution.
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
      description: Retrieves a chronological stream of orders using high-performance cursor pagination.
      operationId: listOrders
      parameters:
        - name: cursor
          in: query
          required: false
          description: Opaque cursor identifier representing the last seen order ID.
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
          format: float
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

⠿Explanation
Contract area

Purpose

servers.url

Specifies http://localhost:5000/api/v1 as the base URI path to enforce explicit API versioning.

paths./orders.post.parameters

Declares the Idempotency-Key header with required: true and UUID formatting.

paths./orders.get.parameters

Defines cursor and limit query parameters with explicit validation boundaries (minimum: 1, maximum: 50, default: 10).

paths./orders/{id}.get

Specifies the resource retrieval endpoint matching the Location header emitted on successful creation.

components.schemas.CreateOrderRequest

Enforces mandatory fields (item_id, quantity, amount) and minimum values (quantity >= 1, amount >= 0.01).

components.schemas.ProblemDetails

Declares the RFC 7807 error format with type, title, status, detail, and instance properties.

components.responses.ProblemDetails400 and ProblemDetails404

Map standard HTTP client errors to application/problem+json.

