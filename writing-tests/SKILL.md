---
name: writing-tests
description:
  Write or update Vitest tests for Cloud Functions and helpers in
  referush_firebase/functions, in the format of
  tests/dashboard/referrals/program-settings/set-program-settings.test.js, with
  it blocks ordered top to bottom in the same order as the source code, and
  full line and statement coverage against the Firestore emulator. Use whenever
  the user asks to write, add, extend, review or fix tests, or runs
  /writing-tests.
---

# OBJECTIVE

Write one test file per source file under `functions_1/`. The test file must:

- read the source like a compiler does, top to bottom, with one `it` block per
  path through each line, in the order those lines appear
- cover every statement and line
- run only against the Firestore emulator
- never send a real request to a third-party service

# REQUIRED RULES (NON-NEGOTIABLE)

1. Tests run only against the Firestore emulator. `vitest.config.js` sets
   `FIRESTORE_EMULATOR_HOST` and the `demo-referrush` project. Never change that.
2. The first thing `beforeAll` does is call `checkIfEmulatorRunning()`. Import
   firebase-admin and the source file only after the guard passes, with
   dynamic `await import(...)`. Never use static imports for either.
3. Mock every third-party call: Shopify, ClickHouse, Cloud Tasks, Resend,
   Razorpay, WhatsApp, SMS and so on. Firestore reads and writes stay real and
   go to the emulator.
4. Never `vi.mock` `config/firebase-admin.js`. That would also replace
   `dbAdmin`, and Firestore would stop reaching the emulator.
5. Line and statement coverage: every line and statement must be reached. The
   only accepted gaps are catch blocks that only log, and the `onCall` wrapper.
6. Put `it` blocks in source order (see TEST ORDER). This is not optional and
   comes before any grouping by test type.
7. Do not run anything. Don't run tests, coverage, the emulator, `npm` or `npx`
   unless the user explicitly asks in this conversation. Write the file, then
   give the user the command. Never run the suite without a file filter,
   because `tests/` still holds legacy `.test.js` scripts that hit real
   services.

# TEST ORDER: TOP TO BOTTOM, LIKE A COMPILER READS THE CODE

`it` blocks follow the order of execution, one step at a time. For a function
shaped like this:

1. param validation check
2. db read
3. db read response check
4. third-party call
5. third-party call response check
6. db write
7. response return

the `it` blocks go in exactly this order:

1. **Param validation:** `it.each(requiredParams)("fails when %s is missing", ...)`,
   then one `it` per remaining validation rule, in the order the source checks
   them.
2. **DB read:** reaching it is enough when nothing branches on it. If the read
   can throw or return nothing, the test for that sits here.
3. **DB read response check:** one `it` per branch on the read result: doc
   missing, empty snapshot, wrong status, and so on.
4. **Third-party call:** one `it` that asserts the exact arguments sent, for
   example `toHaveBeenCalledWith`.
5. **Third-party response check:** one `it` per branch on the response:
   `success: false`, empty data, no match.
6. **DB write:** one `it` per branch that changes what is written, such as
   create versus update, or optional fields. Read each doc back from the
   emulator.
7. **Response return:** the full success path that reaches the final
   `return`.
8. **Catch block:** a catch sits below its `try` in the source, so its tests
   come after every test of lines inside that `try`. Force the error with
   `mockRejectedValueOnce` on the call inside the `try`.

Ordering rules:

- **Position:** each `it` sits at the source line it exists to reach, which is
  the branch, early return or write it targets.
- **Repeated steps:** a function that does read, then call, then read, then
  write has `it` blocks in that same interleaved order. Never group all reads
  or all third-party calls together.
- **Internal helpers:** when the function calls an internal helper (same file
  or another file), the helper's tests go at the call site, in the helper's own
  top-to-bottom order. After them, the caller's tests continue from the next
  line.
- **Inline transforms and defaults** (trim, uppercase, `|| 90`, `Number(...)`):
  test them at the line where they happen.
- **Every `it` reaches at least one line or branch** that no earlier `it`
  reached. If it doesn't, delete it.
- **Step comments:** above each group of `it` blocks, put a CAPS comment naming
  the source step and the function it belongs to, for example
  `// PARAM VALIDATION`, `// DB READ: merchants/{shop}`,
  `// SHOPIFY: getRecentGiftCardTransactions RESPONSE CHECK`,
  `// HELPER: updateGiftCardTransactionHistory`, `// CATCH`.

# ORDER INSIDE AN it BLOCK

1. **Arrange:** read the before state you need (`versionBefore`,
   `historyCountBefore`), seed test-specific docs, and set `...Once` mock
   overrides.
2. **Act:** call the function once.
3. **Assert, in the order the source runs:**
   - each third-party call, in call order
   - each Firestore doc, in write order, read back from the emulator
   - the returned response last, because the return is the last line run

# WORKFLOW

1. Read the source in full, then read every internal helper it calls.
2. Write out the execution path as a numbered list of lines. Mark each branch,
   early return, db read, db write, third-party call and catch. When the source
   calls a helper, expand the helper's lines at that call. This list is the
   order of the `it` blocks.
3. Read the modules the source imports. Note any that call a real service when
   called or when loaded; those must be mocked. Keep Firestore helpers like
   `mergeOrderData` and `logAllErrorsOnFirestore` real so they write to the
   emulator, unless loading them pulls in a real service.
4. Before writing any assertions, report anything in the source that breaks
   the `{ success, message, data }` response shape. Do not edit the source
   unless asked.
5. Write the test at the mirrored path. Follow FILE LAYOUT for the code
   outside `describe`, and TEST ORDER for the `it` blocks.
6. Check coverage against the step 2 list. For each line, name the `it` that
   reaches it, and confirm the `it` blocks are in list order. Report any line
   not reached, apart from the accepted gaps.
7. Give the user the run command, which is the same as the file's header line.

# FILE LAYOUT (IN THIS ORDER)

1. **Header line:** the exact command to run this file with coverage:
   `// npm test -- tests/<mirrored path>.test.js --coverage --coverage.include=functions_1/<source path>.js`
2. **Imports:** `constants/` first, then `tests/common/` helpers, then the
   multi-line `vitest` import. Use relative paths only.
3. **`vi.hoisted` + `vi.mock` blocks**, only if the source imports a
   third-party call directly (see MOCKING).
4. **`let` bindings** under `// SET IN beforeAll, AFTER THE EMULATOR GUARD PASSES`:
   `dbAdmin`, the function under test, and any spies.
5. **Standalone mocks** such as `const enqueueMock = vi.fn()`, each with a
   CAPS comment saying what it stands in for.
6. **Fixture constants** in UPPER_SNAKE_CASE: `TEST_SHOP`, `NEW_SHOP`,
   `TEST_ORDER_ID`, fixed dates. Use a separate shop or id for each branch that
   needs its own starting state. Derive time-dependent values from one fixed
   date constant.
7. **Fixture objects:** existing docs to seed, and `validPayload` (onCall) or
   `validArgs` (helper). Give each a CAPS comment on why it exists.
8. **`requiredParams` array**, in the same order the source validates the
   params.
9. **Seed and read helpers**, as `async function`: `seed<Thing>Data()`, then
   one `get<Thing>Doc(...)` or `get<Thing>Docs(...)` per path the function
   writes.
10. **`beforeAll`**, in this order:
    1. `checkIfEmulatorRunning()`
    2. import firebase-admin and set `dbAdmin`
    3. `vi.spyOn` shared clients and set default mock responses
    4. import the source and pull the export
    5. `clearEmulatorData({ emulatorHost, projectId })`
    6. seed
11. **`beforeEach`:** `mockClear()` every spy and mock.
12. **`afterAll`:** `mockRestore()` every `vi.spyOn` spy, using `?.`.
13. **One `describe("<plain words name>", function () { ... })`** that holds
    every `it`, in TEST ORDER.

# NAMING AND ASSERTIONS

- Name tests in lowercase present tense, stating the behaviour plus the
  condition: "fails when %s is missing", "returns early when the buyer has no
  gift card", "fails when shopify transactions cannot be read". Never include
  the function name or "should".
- Result: `expect(result.success).toEqual(true|false)` and
  `expect(result.data).toEqual(null)`. Assert on `message` only when the exact
  text matters to a caller.
- Primitives use `toBe`, objects use `toEqual`, null uses `toBeNull()`, and
  generated timestamps use `toBeDefined()`.
- Third-party calls:
  - `toHaveBeenCalledTimes(n)` plus `toHaveBeenCalledWith(<exact args>)`
  - `not.toHaveBeenCalled()` for every call after an early return
  - for partial checks, `const [payload] = mock.mock.calls[0]` plus
    `expect.objectContaining({...})`
- Keep tests order-independent: read the before state inside the test, and
  never rely on an earlier test's writes.
- Build ids, URLs and queue names from `PROJECT_ID` and `LOCATION` in
  `constants/`. Never hardcode them.

# CALLING THE FUNCTION

- **onCall function:** call the internal `...Defined` export, never the
  wrapper. Build the request with
  `buildOnCallFunctionRequest({ payload: validPayload, overrides: { ... } })`.
  To make a param missing, set it to `undefined` in `overrides`.
- **Plain helper** (takes an args object): call the export directly with
  `{ ...validArgs, ...overrides }`. Pass nested overrides in full.

# MOCKING

- **Shared client from `config/firebase-admin.js`** (for example
  `functions.taskQueue`): `vi.spyOn(...)` in `beforeAll`, and `mockRestore()`
  it in `afterAll`.
- **Third-party function or client imported by the source:** `vi.mock` that
  module at top level, with mocks created in `vi.hoisted`. The path is
  relative to the test file. If other code still needs the module's real
  exports, keep them with `importOriginal`.
- **Default responses:** set the realistic success response once in `beforeAll`
  with `mockResolvedValue`. Override it per test with `mockResolvedValueOnce`
  or `mockRejectedValueOnce`.
- **Pure in-repo logic** (pricing, date and number helpers): keep it real.
  Mock it only to force a branch you can't reach through input, and say so in
  a CAPS comment.

# TEMPLATE

```js
// npm test -- tests/<dir>/<file>.test.js --coverage --coverage.include=functions_1/<dir>/<file>.js

import { LOCATION, PROJECT_ID } from "../../constants/firebase/index.js"
import { buildOnCallFunctionRequest } from "../common/build-function-request.js"
import {
  clearEmulatorData,
  checkIfEmulatorRunning,
} from "../common/firebase-emulator.js"
import {
  vi,
  it,
  expect,
  afterAll,
  beforeAll,
  describe,
  beforeEach,
} from "vitest"

// CREATED WITH vi.hoisted SO THE vi.mock FACTORY BELOW CAN USE IT
const { fetchSomethingMock } = vi.hoisted(function () {
  return { fetchSomethingMock: vi.fn() }
})

// STANDS IN FOR THE SHOPIFY QUERY SO NO REAL REQUEST IS EVER SENT
vi.mock("../../fetches/shopify/queries/<path>.js", function () {
  return { fetchSomething: fetchSomethingMock }
})

// SET IN beforeAll, AFTER THE EMULATOR GUARD PASSES
let dbAdmin
let myFunctionDefined

const TEST_SHOP = "test-shop.myshopify.com"
const MISSING_SHOP = "missing-shop.myshopify.com"

// VALID PAYLOAD THAT REACHES THE FINAL RETURN
const validPayload = { shop: TEST_SHOP /* , ... */ }

// SAME ORDER AS THE SOURCE VALIDATES THEM
const requiredParams = ["shop" /* , ... */]

async function seedMyFunctionData() {
  await dbAdmin
    .doc(`merchants/${TEST_SHOP}`)
    .set({ shop: TEST_SHOP }, { merge: true })
}

async function getMerchantDoc(shop) {
  return (await dbAdmin.doc(`merchants/${shop}`).get()).data()
}

beforeAll(async function () {
  // THIS FUNCTION WILL THROW AND WONT RUN TESTS IF EMULATOR IS NOT RUNNING
  const emulatorConfig = await checkIfEmulatorRunning()

  // IMPORT AFTER THE GUARD SO FIREBASE-ADMIN ONLY INITIALISES AGAINST THE EMULATOR
  const firebaseAdmin = await import("../../config/firebase-admin.js")
  dbAdmin = firebaseAdmin.dbAdmin

  // DEFAULT SUCCESS RESPONSE, TESTS OVERRIDE WITH ...Once
  fetchSomethingMock.mockResolvedValue({ success: true, data: {} })

  const myFunctionModule = await import("../../functions_1/<dir>/<file>.js")
  myFunctionDefined = myFunctionModule.myFunctionDefined

  await clearEmulatorData({
    emulatorHost: emulatorConfig.emulatorHost,
    projectId: emulatorConfig.projectId,
  })

  await seedMyFunctionData()
})

// RESET CALL HISTORY SO EACH TEST ONLY SEES ITS OWN CALLS
beforeEach(function () {
  fetchSomethingMock.mockClear()
})

describe("my function", function () {
  // PARAM VALIDATION
  it.each(requiredParams)("fails when %s is missing", async function (param) {
    const result = await myFunctionDefined(
      buildOnCallFunctionRequest({
        payload: validPayload,
        overrides: { [param]: undefined },
      }),
    )

    expect(fetchSomethingMock).not.toHaveBeenCalled()
    expect(result.success).toEqual(false)
    expect(result.data).toEqual(null)
  })

  // DB READ RESPONSE CHECK: merchants/{shop}
  it("fails when the merchant does not exist", async function () {
    const result = await myFunctionDefined(
      buildOnCallFunctionRequest({
        payload: validPayload,
        overrides: { shop: MISSING_SHOP },
      }),
    )

    expect(fetchSomethingMock).not.toHaveBeenCalled()
    expect(result.success).toEqual(false)
    expect(result.data).toEqual(null)
  })

  // SHOPIFY: fetchSomething RESPONSE CHECK
  it("fails when shopify returns an error", async function () {
    fetchSomethingMock.mockResolvedValueOnce({
      success: false,
      message: "boom",
    })

    const result = await myFunctionDefined(
      buildOnCallFunctionRequest({ payload: validPayload }),
    )

    expect(fetchSomethingMock).toHaveBeenCalledTimes(1)
    expect(result.success).toEqual(false)
    expect(result.data).toEqual(null)
  })

  // DB WRITE + RESPONSE RETURN
  it("writes the merchant doc and returns success", async function () {
    const result = await myFunctionDefined(
      buildOnCallFunctionRequest({ payload: validPayload }),
    )

    // SHOPIFY
    expect(fetchSomethingMock).toHaveBeenCalledWith({ shop: TEST_SHOP })

    // MERCHANT DOC
    const merchantDoc = await getMerchantDoc(TEST_SHOP)
    expect(merchantDoc.shop).toBe(TEST_SHOP)

    // RESPONSE
    expect(result.success).toEqual(true)
    expect(result.data).toEqual(null)
  })

  // CATCH
  it("returns failure when shopify throws", async function () {
    fetchSomethingMock.mockRejectedValueOnce(new Error("network down"))

    const result = await myFunctionDefined(
      buildOnCallFunctionRequest({ payload: validPayload }),
    )

    expect(result.success).toEqual(false)
    expect(result.data).toEqual(null)
  })
})
```

# CODE STYLE (FROM CLAUDE.md)

- Use function declarations and `function ()` callbacks. No arrow functions.
- Use camelCase. Booleans start with `is`.
- Write comments in CAPITAL LETTERS, one line, saying why.
- No semicolons, double quotes, trailing commas: match Prettier as in the
  reference file.

# RUNNING (GIVE TO THE USER, DO NOT RUN)

From `referush_firebase/`:
`firebase emulators:start --only firestore --project demo-referrush`

From `referush_firebase/functions/`: the command on line 1 of the test file.

# BEFORE OPENING A PR

- The file passes when run on its own against the emulator.
- Coverage shows every line and statement hit, apart from the accepted gaps.
- `it` blocks are in source order, and each one reaches a line no earlier `it`
  reached.
- Nothing reaches the network except the emulator.
