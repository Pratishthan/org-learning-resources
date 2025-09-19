# UT Generation Prompt — Mocha + Chai + Sinon
---

I am writing Mocha, Chai, and Sinon unit tests for a Node.js function.  
Follow this exact UT style:

- Frameworks: **Mocha + Chai (expect)**, **Sinon (sandbox)**, **Chalk**.
- Use `UT-TestUtil` for `pathUtil`, `expect`, `fs`, `sinon`, `chalk`.
- Load test data from `test/data/<functionName>.test.json`.
- Use `loopback.findModel` stubs for mocking models, and return mock objects with `create` / `send` / `publishEvent` methods as required.
- Keep UTs **data-driven and descriptive**, like the existing `<exmaple_file with path>.test.js` example.

Paste the function below between the code fences: ( name and path of your file)

```javascript
// myFunction.js
function yourFunction() {
  // implementation
}
```

---
## version

"chai": "4.2.0",
"loopback": "3.28.0",
"mocha": "7.1.1",
"mock-require": "3.0.3",
"nyc": "15.0.1",
"sinon": "10.0.0"

## Questions you must answer before UT generation ( input from user )

1. Which **LoopBack models** does this function call (list exact model names)?
2. What are the important **valid scenarios** (happy paths)?
3. What are the important **invalid/failure scenarios** (edge/error cases)?
4. Is there an **ErrorUtil constant** to map errors to? If yes, provide the constant name.

---

## Rules / UT Style (follow exactly)

- Use **Mocha** `describe` / `it`. Use `done` callbacks for async code. Keep timeouts reasonable.
- Use `UT-TestUtil` for `pathUtil`, `expect`, `fs`, `sinon`, `chalk`. Never import chai directly.
- Use a **Sinon sandbox** and call `sandbox.restore()` in `after`.
- Stub `loopback.findModel` once in `before`, restore in `after`.
- Store test vectors in `test/data/<functionName>.test.json` with keys like `valid1`, `valid2`, `invalid1`.
- Test descriptions must come from `data.<case>.description`. Assertions must compare against `data.<case>.output`.
- If a model returns an object, attach a `toJSON` method that returns the raw object.

---

## Callback Safety Rules (important!)

To avoid test timeouts, **every stub must always call its callback** (whether success or error):

- In `ErrorUtil` mocks, `getError` must **always call the callback**, never skip it.
- If a stub simulates an error (e.g. `fetchSchmSrvcParams`, `fetchSchemeParams`), it must still call the callback with:

  ```javascript
  cb(err, { errors: [err] });
  ```

  so that the test callback is always triggered.

- Ensure **both error and success paths always call the callback**. No dangling promises, no missing callbacks.

---

## Sample `test/data` structure

```json
{
  "valid1": {
    "description": "should create order and return success",
    "input": { "msg": { "paysysId": "OK1" } },
    "output": { "status": "SUCCESS", "mockData": { "id": "OK1" } }
  },
  "invalid1": {
    "description": "should return error when model create fails",
    "input": { "msg": { "paysysId": "INVALID1" } },
    "output": { "errors": ["Error from RejectEvent.create"] }
  }
}
```

---

## Final Output Format (AI must always generate both)

1. **`test/<functionName>.test.js`**
   - Mocha test file with sandbox, hooks, and loopback stubs.
   - All error/success paths trigger callbacks.

2. **`test/data/<functionName>.test.json`**
   - Valid and invalid scenarios with `description`, `input`, and `output`.

