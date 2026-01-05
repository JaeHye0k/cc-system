---
description: Run tests with optional file pattern
argument-hint: [file-pattern]
allowed-tools: Bash
---

Run tests for: $ARGUMENTS

Execute appropriate test suite based on the pattern:

**No arguments**: Run all tests
```bash
npm test
```

**With file pattern**: Run matching tests
```bash
npm test -- $ARGUMENTS
```

**E2E tests**: Use specific command
```bash
npm run test:e2e $ARGUMENTS
```

After running tests:
1. Analyze any failures
2. Identify root causes (code bug vs test bug)
3. Suggest fixes
4. Detect missing edge cases:
   - Empty states ([], null, undefined)
   - Error states (network failures, 404, 500)
   - Loading states
   - Boundary values

If tests are failing, I'll help fix them while preserving the original test intent.
