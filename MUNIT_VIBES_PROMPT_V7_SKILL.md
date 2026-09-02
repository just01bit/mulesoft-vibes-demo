---
name: generate-munit-tests
description: Use when the user asks to create, write, add, scaffold, fix, or improve/increase coverage of MUnit tests (a test suite, unit tests) for a Mule 4 app, flow, or sub-flow — including a single "test the 404 path" request. Covers any connector (Salesforce, Database, HTTP, JMS, SFTP…). This skill authors the MUnit XML with the `generate_or_modify_munit` MCP tool and verifies it with `mvn clean test`. Trigger words - munit, unit test, test suite, coverage, mock, assert, verify-call, "test the flow", "test the error path".
license: Apache-2.0
compatibility: Mule 4 Maven project. MUnit 3.x for Mule 4.6+/Java 17, MUnit 2.x for older Mule 4. Requires the `mulesoft-dx` MCP server (tool `generate_or_modify_munit`). No live endpoints — all I/O is mocked.
metadata:
  author: mulesoft-tim-dai
  version: "7.2.0"
  theme: customer-success
allowed-tools: read_file replace_in_file write_to_file execute_command use_mcp_tool generate_or_modify_munit
---

# MUnit Test Generator

Generate runnable, fully-isolated MUnit test suites for a Mule 4 app. **You orchestrate and verify —
you do not hand-write the test XML.** The authoritative generator is the `generate_or_modify_munit`
MCP tool (server `mulesoft-dx`, invoked via `use_mcp_tool`): it understands the flow XML, connector namespaces, and mock shapes,
and returns the pom dependencies (versions, groupIds, classifier, and plugin bindings derived
server-side) plus correct MUnit XML. Your job is to inventory the app, feed the tool one test case at
a time, apply the pom blocks it returns, assemble the suite, and confirm with `mvn clean test`.

## The 4 rules (use these to judge the tool's output, and to write good prompts)

1. **Every external/side-effecting op must be mocked.** Any `salesforce:*`, `db:*`, `http:request`,
   `jms:*`, `sftp:*`, `email:*` op needs a `mock-when` matched on `processor="ns:op"` **and** its
   `config-ref`. One unmocked op = the test hits the real system. Mock even ops that should *not*
   run, then prove it with `verify-call times="0"`. If the tool omits a mock, ask it to add one.

2. **Errors are injected with `then-return` → `<munit-tools:error typeId="NS:TYPE"/>`** — the only
   way for extension namespaces (`APIKIT:*`, `SALESFORCE:*`, `HTTP:*`, `DB:*`). Never `raise-error`
   for these ("namespace already exists"); there is no `then-raise`.

3. **A flow whose first processor is `apikit:router` must not be fed fabricated
   `HttpRequestAttributes`.** Mock the router instead (see the APIkit routing-flow recipe). A `munit:attributes` map on a
   flow that *runs* the router throws `ClassCastException … cannot be cast to HttpRequestAttributes`.
   (A map is fine for a plain resource flow that only reads `attributes.uriParams.x`.)

4. **Assertions must be concrete, and writes must be verified.** `assert-that` with
   `MunitTools::equalTo(...)` (not just `notNullValue`), and `verify-call times="1|0"`.

## Workflow

### 1. Inventory (you do this — the tool needs it as input)
**Precondition:** confirm the `mulesoft-dx` MCP server is connected and `generate_or_modify_munit` is
available. If not, stop and tell the user this skill requires the `mulesoft-dx` MCP server.

`read_file` on `src/main/mule/*.xml` and `pom.xml`. List briefly:
- Runtime + Java version → this is `minMuleVersion` for the tool. The tool derives the correct MUnit
  versions from it — you do not pick or maintain them.
- Every `<flow>`/`<sub-flow>` and its processors in order. The flow XML you pass as `projectSource`;
  any flows it references become `dependencies.flowDependencies`.
- Every external op to mock, with its `config-ref`. The referenced connector configs usually live in
  a global config file (e.g. `global.xml`), not the flow file — note each config's **name**, since
  that string is what your mocks match on.
- Every branch (`choice`, `try`, …) and every `error-handler` (types + mapped body/status).
- The **routing/source flow** (APIkit main flow) — its handler branches are only reachable by making
  the router throw, so they need their own test cases (APIkit routing-flow recipe), not just
  resource-flow tests.

### 2. Generate the first test — this also sets up the pom
Call `generate_or_modify_munit` for the first (happy-path) test. **Tool inputs (same for every call):**

- `projectSource` — the flow XML under test (from Step 1). Just the flow(s) under test; global
  connector configs aren't needed, since external ops are mocked.
- `munitFileName` — the target suite file name (`src/test/munit/<config-file-name>-test-suite.xml`).
- `prompt` — plain English describing exactly what this test validates (mock X to return Y; assert
  payload/vars/status; verify-call count). Write it using the 4 rules so the tool produces a
  correctly-mocked, concretely-asserting test. Include the test name (`<mule-flow-name>-<test-name>`)
  in the prompt, or the tool may pick a different convention.
- `minMuleVersion` — the runtime from Step 1.
- `dependencies.flowDependencies` — every flow the `projectSource` flow references.
- `existingMunit` — the full XML of the specific `<munit:test>` being fixed (`read_file` the suite to
  get it); send **only** when modifying or fixing, not for new tests.

Follow the tool's own **CRITICAL EXECUTION ORDER** — apply every returned pom dependency and plugin
block to `pom.xml` (`replace_in_file`) **before** writing any MUnit XML. Versions, groupIds,
classifier, and plugin bindings are derived from `minMuleVersion` server-side, so don't hand-maintain
a version table. Add the Anypoint Exchange / MuleSoft `<repositories>` if missing, or resolution
fails. Then write the returned MUnit XML to the suite file. Don't touch `src/main`.

### 3. Validate the first test
`execute_command`: `mvn clean test`. A green run here proves the pom + harness are correct before you
scale up — catch dependency/plugin problems on one test, not ten.

### 4. Generate the remaining test cases (one call per test)
For each remaining case, call `generate_or_modify_munit` with the same inputs, apply any new returned
pom deps first, then append the returned MUnit XML to the suite (`replace_in_file`: replace the
closing `</mule>` with the new `<munit:test>…</munit:test>` followed by `</mule>`). The tool
generates a single test per call. **Per flow, request:** happy path · empty/boundary result ·
business error (e.g. 404) · connector failure (ask the tool to mock the op to throw its error type,
rule 2). For the routing flow, request **one test per mapped error type** (APIkit routing-flow recipe).

**Completeness check before finishing:** every flow from Step 1 must have ≥1 happy-path **and** ≥1
failure test. A flow at 0% coverage is a gap to fill, not skip — the only fair exception is a flow
with no business logic (e.g. an APIkit `console` flow). State any deliberate gap and why.

### 5. Final validation
`execute_command`: `mvn clean test` — all tests green. The terminal is the only source of truth. On
failure, prefer **re-calling `generate_or_modify_munit` with `existingMunit`** = the failing
`<munit:test>` (not the whole suite) plus a prompt describing the `[ERROR]`; the tool repairs it, then
replace that test in the suite. Use the error→fix table to diagnose.
Re-run after every change before declaring done.

### 6. Summarize
Suites/tests created and where · which flows/branches are covered (and any deliberate gap) · any
spec-vs-implementation discrepancy found (assert the *actual* behavior so tests pass, and flag it).

## Prompt recipes (what to ask the tool for)

Write each `prompt` in plain English — the tool produces the XML. These cover the standard cases;
substitute your own flow / op / `config-ref` names.

- **Happy path:** mock `<ns:op>` at its `config-ref` to return a realistic record/list; assert the
  transformed payload and any vars set along the way.
- **Empty / boundary:** mock `<ns:op>` to return `[]` (or null / single / many); assert the branch
  that handles it.
- **Business error (e.g. 404):** mock the lookup to return `[]`; assert the error body and
  `vars.httpStatus`, and `verify-call times="0"` on any write op that must not run.
- **Connector failure:** mock `<ns:op>` to throw its error type (`SALESFORCE:CONNECTIVITY`,
  `DB:CONNECTIVITY`, `HTTP:TIMEOUT`, …); assert the handler's response.
- **APIkit routing flow (non-obvious — spell this out in the prompt):** mock `apikit:router` to throw
  `APIKIT:<TYPE>` (do **not** fabricate HTTP attributes). The handler is `on-error-propagate`, so the
  test needs `expectedErrorType="APIKIT:<TYPE>"` and asserts the mapped body + `vars.httpStatus` in
  `<munit:validation>`. Request one per mapped type: BAD_REQUEST/400, NOT_FOUND/404,
  METHOD_NOT_ALLOWED/405, NOT_ACCEPTABLE/406, UNSUPPORTED_MEDIA_TYPE/415, NOT_IMPLEMENTED/501, plus
  any custom types. *If `expectedErrorType` ever swallows validation, wrap the invocation in a
  `<try>` with `<on-error-continue type="APIKIT:<TYPE>">` instead.*

## Error → fix (diagnosis; prefer re-calling the tool with `existingMunit` + the error)

| `[ERROR]` you see | Fix |
| --- | --- |
| `MULE:EXPRESSION` raised from a mock `value` | The mock value used `readUrl(...,'application/json')` — replace with an inline DW literal (`#[[{...}]]`) in the mock value. |
| `MULE:EXPRESSION` in an **assertion** (e.g. `read(payload,'application/json')`) | After an `ee:transform output application/json`, `payload` is already an Array/Object — assert on it directly (`payload.field`, `sizeOf(payload)`), don't re-parse it. |
| `Could not resolve` / `Missing artifact … munit-runner`/`munit-tools` | Add `<classifier>mule-plugin</classifier>` + `test` scope; pin a real `munit.version` (3.7.0 for 4.11); ensure Anypoint/MuleSoft `<repositories>` present. |
| `mvn test` runs but **no MUnit tests execute** | `munit-maven-plugin` (groupId `com.mulesoft.munit.tools`) missing, or `test`/`coverage-report` goals not bound to the `test` phase. Add the Step 2 plugin block. |
| `ClassCastException … cannot be cast to HttpRequestAttributes` | Flow runs `apikit:router` — mock the router (APIkit routing-flow recipe), don't set an attributes map. |
| `Cannot use error type 'NS:TYPE': namespace already exists` | `raise-error` on an extension type — use `then-return` → `<munit-tools:error typeId=…/>`. |
| `only flows can be used, not sub-flows` | `then-call` targeted a `<sub-flow>` — target a `<flow>`, or use `then-return`+`error`. |
| Test hits a real endpoint / missing credentials | An external op wasn't mocked, or the mock `config-ref` doesn't match the flow's op. Add/fix the mock. |
| MUnit **fails to start** / property-placeholder or `secure-properties` error | Config properties must resolve at test time even though connectors are mocked — provide a test `secure-config.yaml` under `src/test/resources` (+ the decryption key) or a test properties file. |
| Mock payload rejected downstream | Mock shape/media type doesn't match what the flow reads — mirror the real contract. |
| Coverage lower than expected / routing flow at 0% | Add the missing branch's test; for the main flow, add the APIkit routing-flow error tests (recipe above). |
