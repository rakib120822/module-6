---
name: write-module-lab
description: Write a full Module 06 lab README in the established house style — title block, architecture diagram, prerequisites, concepts table, numbered build steps with inline code and explanations, verification scenarios, troubleshooting, teardown, and conclusion. Use when the user asks to write a lab, author a lab README, expand a lab outline into a full walkthrough, or match the existing M6-L1..M6-L4 style.
---

# Module Lab Writer

Produces a complete, self-contained lab README in the style used by `M6-L1` through `M6-L4` in this repo.

The output is a single `README.md`. Every code block is complete and copy-pasteable. Every command is stated exactly. Every step explains *what it does* and *why it matters*.

## Before writing

Read the source material the user gives you (an outline, a spec, a design doc, a reference implementation) and extract:

- Lab number and title
- The technology stack and the one problem the lab solves
- Every artifact to be created (files, directories, configs)
- Every endpoint, command, or config directive the lab teaches
- The verification surface: which success paths, failure paths, and boundaries to prove

Then write. Do not ask for permission to begin unless the lab number is ambiguous.

## Structure

Use these headings in this order. Skip a section only when it genuinely does not apply.

```
# Lab NN — <Title>

> **Module 06 — API Design & Networking Essentials** · Lab N
> **Difficulty:** <Beginner|Intermediate|Advanced> · **Estimated Time:** <N–N minutes>

<architecture diagram>

<intro paragraph: what you will build, the stack, the outcome>

## Prerequisites

## Concepts

## Objectives

## What You Will Build

## Step N: <Title>

## Verification

## Troubleshooting

## Teardown & Cleanup

## Conclusion
```

Separate major sections with `---` when the document is long (over roughly 500 lines).

## Title block

```markdown
# Lab 01 — Contract-First API Design with OpenAPI 3

> **Module 06 — API Design & Networking Essentials** · Lab 1
> **Difficulty:** Intermediate · **Estimated Time:** 25–30 minutes
```

Use an em dash in the title. The module line names the module and the lab ordinal. The difficulty line uses a bold label, an em dash separator, and a time range.

## Intro paragraph

Two to five sentences, present tense, no bullet points. Cover: what the learner builds, the technologies involved, the concrete capability it adds, and where it is exposed. Name the endpoints and protocols explicitly.

Follow with a short second paragraph when it helps — the "why these tools" sentence:

> Why these tools? Node.js runs the API, Docker Compose starts Redis, and `curl`/`jq` help you test and inspect the API. Port 5000 is used by the application and port 6379 by Redis.

## Architecture diagram

Use a centered image pointing at the shared asset host:

```html
<p align="center">
  <img src="https://raw.githubusercontent.com/poridhioss/lab-asset/main/System-Design-Labs/Milestones/MileStone-02/Module-06/lab-01/images/lab01-architecture.svg" alt="Contract-First API Architecture Overview" width="750">
</p>
```

Rules for the path:

- Base: `https://raw.githubusercontent.com/poridhioss/lab-asset/main/System-Design-Labs/Milestones/MileStone-02/Module-06/`
- Lab folder: `lab-01`, `lab-02`, `lab-03`, `lab-04` (zero-padded)
- Image names: `lab01-*.svg` / `lab01-*.png` for Lab 01, `lab02-*` for Lab 02, and so on. Prefix matches the lab number with no dash.

Place the architecture diagram right after the title block. Place per-step screenshots inside their step.

For step screenshots use the same pattern with the step number:

```html
<p align="center">
  <img src="https://raw.githubusercontent.com/poridhioss/lab-asset/main/System-Design-Labs/Milestones/MileStone-02/Module-06/lab-01/images/step1-create-dir.png" alt="Create Project Directory" width="700">
</p>
```

Use the `step<N>-<kebab-description>.png` convention. Width `700` for screenshots, `750` for the architecture SVG.

## Prerequisites

Bulleted list, bold tool name, colon, then the requirement and the verification command in backticks.

```markdown
Before starting this lab, ensure you have:

- **Node.js:** v20.x or v22.x LTS installed (`node -v`)
- **CLI Utilities:** `curl` and `jq` installed (`curl --version`, `jq --version`)
- **Port Availability:** Host port `5000` must be free. Check with `lsof -i :5000` or `netstat -tuln | grep 5000`.
```

Use a table instead when there are more than four tools with distinct purposes, and when each tool needs a "why you need it" note:

```markdown
| Tool | Required version | Verification |
| --- | --- | --- |
| Node.js | v20.x or v22.x LTS | `node -v` |
| Docker | v24+ | `docker --version` |
```

Follow the list with callouts for cross-lab constraints and scope boundaries:

```markdown
> [!NOTE]
> **Architectural Callout: Header Requirement vs. Distributed De-duplication (H-02)**
> In this lab, the server enforces the *presence* and *UUID format* of the `Idempotency-Key` header as part of the OpenAPI interface contract. However, true de-duplication, atomic mutual exclusion locks (`SETNX`), and replay caching require an external distributed state store (Redis), which is implemented in **[Lab 02: Idempotent Write Endpoint](../lab-02/README.md)**.
```

Callout types: `[!NOTE]`, `[!WARNING]`, `[!IMPORTANT]`. Bold the label, then write the body. Use these for:
- Scope boundaries between labs
- Port conflicts with a previously run lab
- Behavior that looks like a bug but is intentional

## Concepts

A two-column table, formal definitions.

```markdown
| Term | Definition |
| :--- | :--- |
| **Contract-First Design** | An API architecture approach where the interface specification is defined and reviewed before implementation code is written. |
```

Left column can be plain or bold. Definitions are one or two sentences, technically precise, and name the relevant standard or protocol where one exists.

Follow the table with a prose paragraph that explains how the concepts connect into the flow the learner is about to build. This paragraph is the conceptual bridge between the reference table and the implementation.

## Objectives

Bulleted list, imperative mood, one line each, action verbs.

```markdown
- Configure the project workspace with Express and YAML parser dependencies.
- Author an OpenAPI 3.0.3 specification defining `/orders` endpoints, write idempotency headers, query limits, single-order lookups, and RFC 7807 error models.
- Implement an Express server stub that enforces the OpenAPI contract rules and manages in-memory order data.
- Expose the application using the Poridhi Load Balancer on port 5000.
- Verify contract enforcement, boundary restrictions, cursor pagination, and automated unit tests.
```

## What You Will Build

A fenced `text` tree with a `#` comment per file.

```text
lab01-contract-first-api/
├── openapi.yaml       # OpenAPI 3.0.3 contract specification
├── package.json       # Node.js dependencies and script definitions
├── server.js          # Express server enforcing contract rules
└── test.js            # Automated contract validation test suite
```

Follow the tree with a short paragraph tying the files to the contract concept. Keep it to one or two sentences.

## Steps

Each step uses this shape:

```markdown
## Step N: <Title>

<One or two sentences on what this step accomplishes and why it matters.>

```bash
<the command>
```

<p align="center">
  <img src="..." alt="..." width="700">
</p>

**Explanation:**

- `key`: what this element does and why it matters.
- `another_key`: what this element does and why it matters.
```

Rules:

- The step heading title is a short noun phrase, not a command. "Initialize the Project Workspace", "Define the OpenAPI 3.0.3 Contract", "Implement the Contract-Compliant Server Stub", "Expose the Server via Poridhi Load Balancer".
- Every code block is preceded by a sentence naming the file and the action: "Create a file named `openapi.yaml` with the following contents:".
- File contents are shown in full. Never use `# ... rest of the file ...` elisions in a block the learner is meant to type.
- The `**Explanation:**` list uses the pattern `- \`identifier\` — what it does and why it matters.` Each bullet names one element of the code, not a summary of the whole file.
- Use the em dash separator consistently in these bullets.
- Put the screenshot after the command or code block, not before.

### Code block languages

Use the correct fence tag so highlighting works: `bash`, `json`, `yaml`, `javascript`, `text`, `http`, `nginx`, `mermaid`.

### Exposing through the Load Balancer

This step is standard across labs and keeps the same shape:

```markdown
## Step N: Expose the Server via Poridhi Load Balancer

Start the server process in the background on port 5000:

```bash
node server.js &
```

Retrieve the private IP address of your virtual machine:

```bash
hostname -I | awk '{print $1}'
```

Configure the Poridhi Load Balancer:

1. Open the **Load Balancer** modal in the Poridhi interface.
2. Enter the IP address obtained from `hostname -I`.
3. Set the target port to `5000`.

Click **Expose** to generate an active public URL:

<p align="center">
  <img src="..." alt="Load Balancer Exposed" width="700">
</p>

Open the generated Load Balancer URL in your web browser to confirm connectivity:
```

The lab's own server listens on the VM's private IP. The Load Balancer exposes it publicly. Do not have the learner start the server on `localhost` for the exposed URL.

## Verification

Lead with a sentence describing what the verification covers and which tools are used.

```markdown
You will verify contract validation, mutation handling, query parameter limits, single-item retrieval, and cursor pagination using `curl` and the automated test runner.
```

Then numbered scenarios:

```markdown
### Scenario 1: Missing Idempotency-Key Header (Expected 400 Bad Request)

Execute a `POST` request without the `Idempotency-Key` header:

```bash
curl -i -X POST http://localhost:5000/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"item_id": 402, "quantity": 1, "amount": 99.50}'
```

<p align="center">
  <img src="..." alt="Test 1 Missing Idempotency Key" width="700">
</p>

Expected output:

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
```

Scenario rules:

- Heading format: `### Scenario N: <Name> (Expected <Status>)`
- Name states the condition being tested.
- One command block per scenario, with a one-line lead-in sentence.
- Output block shows the real HTTP status line, the relevant headers, and the body.
- End every scenario with a `> **What to Observe:**` blockquote naming the specific signal to look for — a header, a status, a field value.
- Use `curl -i` when headers matter, plain `curl -s ... | jq .` when the body is the point.
- Expected output for JSON bodies uses a `json` fence; raw HTTP responses use `text`.

### Automated test suite

When the lab includes a test file, present it as the final scenario. Introduce it with what the suite covers and why it is automated, then give the file, the explanation bullets, the command, and the expected console output.

```markdown
### Scenario 5: Automated Contract Validation Test Suite

Create a file named `test.js` with the following contents:

```javascript
<full file>
```

**Explanation:**

- `app.listen(0)`: Spawns the Express server on an ephemeral random port to isolate automated tests from active server instances.
- `request()`: Wraps Node.js `http.request` into a promise-based client to test status codes, headers, and parsed JSON responses.
- `Test 1`: Asserts that omitting `Idempotency-Key` triggers an `HTTP 400` status with RFC 7807 payload title `Missing Required Header`.

Execute `npm test` to run the test suite:

```bash
npm test
```

Expected output:

```text
> lab01-contract-first-api@1.0.0 test
> node test.js

Test 1: Asserting 400 Bad Request when Idempotency-Key header is omitted...
 PASS: 400 RFC 7807 returned properly for missing header.

 ALL 4 CONTRACT-FIRST API TESTS PASSED SUCCESSFULLY!
```
```

The expected console output must match the real code's output, including the PASS markers and the final summary line.

## Troubleshooting

Three-column table: Issue, Cause, Solution.

```markdown
| Issue | Cause | Solution |
| :--- | :--- | :--- |
| `Error: listen EADDRINUSE :::5000` | Another process is already running on port 5000 (e.g. from an earlier run or Lab 02). | Run `lsof -i :5000` to find the process ID and run `kill -9 <PID>`, or run `pkill -f "node server.js"`. |
| `curl: (7) Failed to connect to localhost port 5000` | The Express server process is not active or crashed on boot. | Start the server with `node server.js` in a dedicated terminal and inspect console error outputs. |
| `jq: command not found` | The `jq` JSON parsing utility is not installed in the system. | Install `jq` using `sudo apt-get update && sudo apt-get install -y jq`. |
```

Cover the real failure modes: port already in use, server not started, missing CLI tool, missing dependency, service container not running, and auth or validation rejections that are expected behavior rather than bugs.

## Teardown & Cleanup

One sentence on why, then the commands.

```markdown
When you have finished testing this lab, stop the background Node.js server before starting Lab 02:

```bash
pkill -f "node server.js" || true
```

Confirm port 5000 is released:

```bash
lsof -i :5000
```
```

Include container teardown when the lab uses Docker:

```bash
# Stop background Node.js server
pkill -f "node server.js" || true

# Stop and remove Redis container and volumes
docker compose down -v
```

## Conclusion

One paragraph, past tense, recapping what was built and what was verified. Name the mechanisms, not just the topic.

```markdown
You designed an OpenAPI 3.0.3 specification enforcing URI versioning, write idempotency headers, query boundaries, resource lookup, and RFC 7807 Problem Details. You implemented an Express server stub directly compliant with this contract and verified boundary enforcement, cursor pagination, and error responses through cURL and automated test suites.
```

## Environment rules

These are hard constraints from the repo's lab environment:

- Shell examples are Linux. Write `apt-get`, `pkill -f "node server.js"`, `lsof -i :5000`, `hostname -I`. Never `brew`, `killall`, `ps aux | grep` with BSD flags, or macOS-specific paths.
- The application under test is exposed through the Poridhi Load Balancer on port 5000. The lab's own server listens on the VM's private IP, obtained with `hostname -I | awk '{print $1}'`.
- `curl` against `http://localhost:5000` is correct for local verification, since the server runs on the same VM.
- No `package.json` at the repo root for lab purposes. Each lab creates its own inside `~/labNN-<slug>/`.

## Style rules

- No emoji anywhere. Not in headings, tables, bullets, callouts, or commit-style prefixes.
- No icons of any kind.
- Plain, direct prose. Short sentences. Active voice in instructions, passive or present in explanations.
- Second person for the learner ("you will build", "execute the request"). Avoid "we" and "one".
- Backticks for every file name, command, endpoint, header name, field name, and value.
- Em dash `—` for parenthetical asides and list separators. En dash `–` in time ranges like `25–30 minutes`. Hyphen in hyphenated words.
- Keep lines under roughly 110 characters in prose paragraphs. Do not wrap code.
- Use `application/problem+json`, `application/json`, `text/yaml` and other content types exactly.

## After writing

Report the file path, the section list, and anything the learner must supply (screenshot images that do not exist yet, exact package versions, real expected output values that could not be derived from the code).
