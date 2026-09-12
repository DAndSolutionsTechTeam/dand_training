# Hands-On Lab: Build a CI/CD Pipeline

**Goal:** Write a real GitHub Actions pipeline with lint/test/build stages, see it fail, fix it, and add path filtering so it scales across multiple services in one repo.
**Time:** 15 min

---

## 0. Facilitator Setup

Reuse or create a repo with two tiny "services" so path filtering has something real to filter:

```
repo/
├── services/
│   ├── api/
│   │   ├── index.js
│   │   └── index.test.js
│   └── worker/
│       ├── worker.js
│       └── worker.test.js
└── .github/workflows/
```

**`services/api/index.js`**
```javascript
function add(a, b) { return a + b; }
module.exports = { add };
```

**`services/api/index.test.js`**
```javascript
const { add } = require("./index");
test("adds two numbers", () => {
  expect(add(2, 3)).toBe(5);
});
```

Duplicate the same pattern for `services/worker/` with a different trivial function (e.g. `multiply`).

Each participant works in their own fork or their own `services/<name>/` folder to avoid stepping on each other.

---

## 1. Exercise A — Your First Workflow (15 min)

Create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: ["**"]
  pull_request:

jobs:
  test-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm init -y && npm install --save-dev jest
      - run: npx jest services/api
```

Commit and push on a branch, open a PR. Watch the **Actions** tab — the job should run and pass.

**Break it on purpose:** change `expect(add(2, 3)).toBe(5)` to `.toBe(6)`, push again, and watch the PR show a red ❌ check. This is the point — a required check should block a bad merge. Fix it back to `5` and watch it go green.

**Debrief:** point out the required-status-check setting in branch protection (Settings → Branches) that would actually prevent merging while red.

---

## 2. Exercise B — Add More Stages (10 min)

Extend the workflow with lint and a build step (even a trivial one):

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "pretend lint step — replace with eslint in a real repo"

  test-api:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm install --save-dev jest
      - run: npx jest services/api

  build:
    needs: test-api
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "pretend build/package step"
```

Push and confirm in the Actions tab that jobs run in the order implied by `needs:` — lint → test → build — and that a failure anywhere upstream stops the rest.

---

## 3. Exercise C — Path Filtering for Multiple Services (15 min)

Now make the pipeline scale: only test the service that actually changed.

```yaml
name: CI
on:
  push:
    branches: ["**"]
  pull_request:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      worker: ${{ steps.filter.outputs.worker }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'services/api/**'
            worker:
              - 'services/worker/**'

  test-api:
    needs: changes
    if: needs.changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm install --save-dev jest
      - run: npx jest services/api

  test-worker:
    needs: changes
    if: needs.changes.outputs.worker == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm install --save-dev jest
      - run: npx jest services/worker
```

