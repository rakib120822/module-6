---
name: new-lab
description: Scaffold a new lab directory in this repo — creates the lab folder, a README skeleton, and an images/ folder. Use when the user asks to start a new lab, add Lab 05, or set up the next module section.
---

# New Lab Scaffolder

Creates a new top-level lab directory in the repo root, matching the conventions used by `M6-L1` through `M6-L4`.

## Inputs

Collect from the user (ask if not already stated):

- **Lab ID / directory name** — e.g. `M6-L5`. If the user gives a title but no ID, propose one following the existing `M6-L<n>` pattern and confirm.
- **Lab title** — a human-readable title for the `# ` heading.
- **One-line summary** — what the lab teaches/does.

If the user only says "create a new lab" with no details, ask for the ID and title first. Do not guess content for the lab.

## Steps

1. Create the directory and its `images/` subfolder:
   ```bash
   mkdir -p "<LAB-ID>/images"
   ```

2. Write `<LAB-ID>/README.md` with this skeleton. Fill in the parts the user gave you; leave the remaining sections with a short italic placeholder line like `*TODO*` rather than inventing content.

   ```markdown
   # <Lab Title>

   <One-line summary>

   ## Objectives
   By the end of this lab, you should be able to:
   - <objective>
   - <objective>

   ## Concepts
   **<Term>** <definition>

   ## Prerequisites
   - <tool and why it is needed>
   - <port or resource that must be free>

   ## What You Will Build
   - <file or artifact>

   ## Step 1: <Title>
   **What you are doing:** <one or two sentences>
   **Why it matters:** <one or two sentences>

   ```bash
   <command>
   ```
   **Expected output:**
   ```text
   <output>
   ```

   ## Verification
   ### Scenario 1: <Name> (Expected <status>)
   **What are we testing?** <one sentence>
   ```bash
   <curl or test command>
   ```
   **What to Observe**
   - <observation>

   ## Troubleshooting
   | Issue | Cause | Solution |
   |---|---|---|
   | <error> | <cause> | <fix> |

   ## Teardown & Cleanup
   ```bash
   <cleanup command>
   ```
   ```

3. Tell the user where screenshots belong (`<LAB-ID>/images/`) and how to reference them: `![label](images/name.png)`.

## Conventions to apply

- Labs run inside a Poridhi VM; write Linux shell commands (`apt-get`, `pkill -f "node server.js"`, `lsof -i :5000`).
- The app under test is exposed via the Poridhi Load Balancer on port 5000; the lab's server listens on the VM private IP.
- Keep prose beginner-facing: what the step does, why it matters, the exact command, the expected output, and what to observe.
- Mermaid diagrams are welcome where they clarify architecture or flow.
- Do not add a `package.json` or build tooling unless the lab instructions call for it.

## After scaffolding

Report the files created and the TODO placeholders the user still needs to fill in.
