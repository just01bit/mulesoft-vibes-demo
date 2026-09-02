---
name: generate-munit-tests
description: Use when the user asks to create, write, add, scaffold, fix, or improve/increase coverage of MUnit tests (a test suite, unit tests) for a Mule 4 app, flow, or sub-flow — including a single "test the 404 path" request. Covers any connector (Salesforce, Database, HTTP, JMS, SFTP…). Trigger words - munit, unit test, test suite, coverage, mock, assert, verify-call, "test the flow", "test the error path".
license: Apache-2.0
compatibility: Mule 4 Maven project. MUnit 3.x for Mule 4.6+/Java 17, MUnit 2.x for older Mule 4. No live endpoints — all I/O is mocked.
metadata:
  author: mulesoft-tim-dai
  version: "4.0.0"
  theme: customer-success
allowed-tools: Read Write Edit Bash AskUserQuestion
---

# MUnit Test Generator

Generate runnable, fully-isolated MUnit test suites for a Mule 4 app: read the real flows, mock
every external call, cover each branch, assert concrete values, and confirm with `mvn clean test`.

## The 4 rules that make tests pass (read first)

1. **Mock every external/side-effecting op.** Any `salesforce:*`, `db:*`, `http:request`, `jms:*`,
   `sftp:*`, `email:*` op must have a `mock-when` matched on `processor="ns:op"` **and** its
   `config-ref`. One unmocked op = the test hits the real system and fails on credentials. Mock even
   ops you expect *not* to run, then prove it with `verify-call times="0"`.

2. **To make a mock THROW, use `then-return` → `<munit-tools:error typeId="NS:TYPE"/>`.** This is the
   only way to inject an extension-namespace error (`APIKIT:*`, `SALESFORCE:*`, `HTTP:*`, `DB:*`).
   Do **not** use `raise-error` for these (fails: "namespace already exists"); there is no
   `then-raise`.

3. **Never fabricate `HttpRequestAttributes` for a flow whose first processor is `apikit:router`.**
   Invoke that flow and **mock the router** instead (Template C). Setting a `munit:attributes` map on
   a flow that *runs* the router throws `ClassCastException: LinkedHashMap cannot be cast to
   HttpRequestAttributes`. (Setting an attributes *map* is fine for a plain resource flow that only
   reads `attributes.uriParams.x` — see Template A — because no router casts it.)

4. **Assert concrete values, and verify writes.** Use `assert-that` with `MunitTools::equalTo(...)`
   (not just `notNullValue`), and `verify-call times="1|0"` to prove a side effect happened or was
   skipped.

## Workflow

### 1. Inventory (do this before writing anything)
Read `src/main/mule/*.xml` and `pom.xml`. In your reply, list briefly:
- Runtime + Java version; pick the matching MUnit version (3.x for 4.6+/Java 17, else 2.x).
- Every `<flow>`/`<sub-flow>` and its processors in order.
- Every external op to mock, with its `config-ref`.
- Every branch (`choice`, `try`, etc.) and every `error-handler` (types + mapped body/status).
- The **routing/source flow** (APIkit main flow) — its handler branches are only reachable by making
  the router throw, so they need their own tests (Template C), not just the resource-flow tests.

### 2. Set up
- One suite per config file: `src/test/munit/<config-file-name>-test-suite.xml`, with
  `<munit:config minMuleVersion="<runtime>"/>`. A distinct concern (e.g. the error handler) may get
  its own suite.
- Add MUnit deps to `pom.xml` if missing (`munit-maven-plugin`, `munit-runner`, `munit-tools`, test
  scope, chosen version) and enable the coverage report. Do **not** touch `src/main`.

### 3. Write ONE test, then run the build — then expand
Do **not** write the whole suite before running anything. Write one happy-path test (Template A),
run `mvn clean test`, get it green, **then** add the rest. This keeps you converging instead of
debugging ten tests at once.

For each flow cover: happy path · empty/boundary result · business error (e.g. 404) · connector
failure (mock the op to throw, rule 2). For the routing flow, add one test per mapped error type
(Template C). Put non-trivial mock data in `src/test/resources/*.json` and `readUrl` it.

### 4. Verify
`mvn clean test`. The terminal is the only source of truth. On failure, match the `[ERROR]` against
the table below, fix, and re-run. Re-run after **every** edit before declaring done.

**Completeness check before finishing:** every flow from Step 1 must have ≥1 happy-path **and** ≥1
failure test. A flow at 0% coverage is a gap to fill, not skip — the only fair exception is a flow
with no business logic (e.g. an APIkit `console` flow). State any deliberate gap and why.

### 5. Summarize
Suites/tests created and where · which flows/branches are covered (and any deliberate gap) · any
spec-vs-implementation discrepancy found (assert the *actual* behavior so tests pass, and flag it).

## Templates (copy and adapt)

> These are filled in with this demo's **Salesforce** names as a worked example — they are **not
> literals to copy verbatim**. Replace every app-specific value with what you found in Step 1:
> the connector `processor="ns:op"` (e.g. `db:select`, `http:request`, `jms:publish`), the
> `config-ref` name, the flow names, the error `typeId`, and the resource file names. Template A′
> shows the same pattern with a Database connector so you can see it generalizes past Salesforce.

**A — Resource flow: mock a query, drive input, assert output.**
```xml
<munit:test name="get-accounts-happy-path-should-return-list"
            description="salesforce:query returns 2 accounts; flow transforms to a JSON array of 2">
    <munit:behavior>
        <munit-tools:mock-when processor="salesforce:query">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="config-ref" whereValue="#['Salesforce_Config']"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload mediaType="application/java"
                    value="#[output application/java --- readUrl('classpath://accounts-list.json','application/json')]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <munit:set-event>
            <munit:payload value="#[null]" mediaType="application/java"/>
            <!-- attributes map is OK here: this flow only reads attributes.uriParams, no router runs -->
            <munit:attributes value="#[{uriParams: {AccountNumber: 'ACC-001'}, queryParams: {}, headers: {}}]"
                              mediaType="application/java"/>
        </munit:set-event>
        <flow-ref name="get:\accounts:demo-salesforce-accounts-api-config"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[sizeOf(payload)]" is="#[MunitTools::equalTo(2)]"/>
        <munit-tools:assert-that expression="#[payload[0].AccountNumber]" is="#[MunitTools::equalTo('ACC-001')]"/>
    </munit:validation>
</munit:test>
```

**A′ — Same pattern, different connector (Database).** Only the namespace, op, `config-ref`, mock
shape, and flow name change — the structure is identical to A. This is what "adapt" means.
```xml
<munit:test name="get-customer-happy-path-should-return-record"
            description="db:select returns 1 customer row; flow maps it to the response object">
    <munit:behavior>
        <munit-tools:mock-when processor="db:select">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="config-ref" whereValue="#['Database_Config']"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload mediaType="application/java"
                    value="#[[{id: 1, name: 'Ada Lovelace', email: 'ada@example.com'}]]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <munit:set-event>
            <munit:payload value="#[null]" mediaType="application/java"/>
            <munit:attributes value="#[{uriParams: {id: '1'}, queryParams: {}, headers: {}}]" mediaType="application/java"/>
        </munit:set-event>
        <flow-ref name="get:\customers\(id):customers-api-config"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[payload.name]" is="#[MunitTools::equalTo('Ada Lovelace')]"/>
    </munit:validation>
</munit:test>
```
To drive its error path, mock `db:select` to throw with `then-return`→`<munit-tools:error typeId="DB:CONNECTIVITY"/>` (rule 2).

**B — Business branch + prove a write did NOT happen (not-found path).**
```xml
<munit:test name="patch-account-not-found-should-return-404"
            description="query-all returns []; flow returns 404; salesforce:update is never called">
    <munit:behavior>
        <munit-tools:mock-when processor="salesforce:query-all">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="config-ref" whereValue="#['Salesforce_Config']"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return><munit-tools:payload mediaType="application/java" value="#[[]]"/></munit-tools:then-return>
        </munit-tools:mock-when>
        <!-- mock update too, so even an accidental call can't reach the real org -->
        <munit-tools:mock-when processor="salesforce:update">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="config-ref" whereValue="#['Salesforce_Config']"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return><munit-tools:payload mediaType="application/java" value="#[[]]"/></munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <munit:set-event>
            <munit:payload value="#[output application/java --- {Name: 'X'}]" mediaType="application/java"/>
            <munit:attributes value="#[{uriParams: {AccountNumber: 'ACC-999'}, headers: {}}]" mediaType="application/java"/>
        </munit:set-event>
        <flow-ref name="patch:\accounts\(AccountNumber):application\json:demo-salesforce-accounts-api-config"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[vars.httpStatus]" is="#[MunitTools::equalTo(404)]"/>
        <munit-tools:verify-call processor="salesforce:update" times="0">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="config-ref" whereValue="#['Salesforce_Config']"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
    </munit:validation>
</munit:test>
```

**C — APIkit routing flow: mock the router to throw, assert the mapped response.**
The handler is `on-error-propagate` (it re-throws), so declare `expectedErrorType` to accept the
error; validation still runs against the handler-modified event, so assert the mapped body/status
there. (Repeat once per mapped type: BAD_REQUEST/400, NOT_FOUND/404, METHOD_NOT_ALLOWED/405,
NOT_ACCEPTABLE/406, UNSUPPORTED_MEDIA_TYPE/415, NOT_IMPLEMENTED/501, plus custom types.)
```xml
<munit:test name="main-flow-bad-request-should-return-400"
            description="apikit:router throws APIKIT:BAD_REQUEST; handler maps to {message:'Bad request'} / 400"
            expectedErrorType="APIKIT:BAD_REQUEST">
    <munit:behavior>
        <munit-tools:mock-when processor="apikit:router">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="config-ref" whereValue="#['demo-salesforce-accounts-api-config']"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return><munit-tools:error typeId="APIKIT:BAD_REQUEST"/></munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <munit:set-event><munit:payload value="#[null]" mediaType="application/java"/></munit:set-event>
        <flow-ref name="demo-salesforce-accounts-api-main"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[payload.message]" is="#[MunitTools::equalTo('Bad request')]"/>
        <munit-tools:assert-that expression="#[vars.httpStatus]" is="#[MunitTools::equalTo(400)]"/>
    </munit:validation>
</munit:test>
```
*Alternative if `expectedErrorType` ever swallows your validation:* wrap the `flow-ref` in a `<try>`
with `<on-error-continue type="APIKIT:BAD_REQUEST"/>` and assert in `<munit:validation>` instead.

## Error → fix

| `[ERROR]` you see | Fix |
| --- | --- |
| `ClassCastException … cannot be cast to HttpRequestAttributes` | Flow runs `apikit:router` — mock the router (Template C), don't set an attributes map. |
| `Cannot use error type 'NS:TYPE': namespace already exists` | You used `raise-error` on an extension type — use `then-return` → `<munit-tools:error typeId=…/>`. |
| `only flows can be used, not sub-flows` | `then-call` targeted a `<sub-flow>` — target a `<flow>`, or drop `then-call` and use `then-return`+`error`. |
| Test hits a real endpoint / missing credentials | An external op wasn't mocked, or the `config-ref` in the mock doesn't match the flow's op. Add/fix the mock. |
| Mock payload rejected downstream | Mock's shape/media type doesn't match what the flow reads — mirror the real contract (load a resource file). |
| Coverage lower than expected / routing flow at 0% | Add the missing branch's input/mock; for the main flow, add the Template C error tests. |
