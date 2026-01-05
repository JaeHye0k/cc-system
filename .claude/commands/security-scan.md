---
description: Run security audit on codebase
allowed-tools: Bash
---

Perform comprehensive security audit:

## 1. Dependency Vulnerabilities
```bash
npm audit --production
```

## 2. Scan for Exposed Secrets
```bash
# Check for hardcoded API keys, passwords, tokens
grep -ri "API_KEY\|SECRET\|PASSWORD\|TOKEN" --include="*.ts" --include="*.tsx" src/ | grep -v "NEXT_PUBLIC"

# Check for common secret patterns
grep -ri "sk-\|pk_\|Bearer \|ghp_\|aws_" --include="*.ts" --include="*.tsx" src/
```

## 3. Check .env Exposure
```bash
# Verify .env files are not tracked
git ls-files | grep "^\.env$"

# Check if .env is in .gitignore
cat .gitignore | grep ".env"
```

## 4. XSS Vulnerabilities
```bash
# Check for dangerous HTML rendering
grep -r "dangerouslySetInnerHTML\|innerHTML\|eval(" --include="*.tsx" --include="*.ts" src/
```

## 5. Insecure Token Storage
```bash
# Check for localStorage token storage
grep -r "localStorage.setItem.*token\|sessionStorage.setItem.*token" --include="*.ts" --include="*.tsx" src/
```

Generate security report with findings categorized by severity:
- **CRITICAL**: Fix immediately (exposed secrets, XSS)
- **HIGH**: Fix soon (insecure token storage, missing validation)
- **MEDIUM**: Consider fixing (missing CSRF protection)
- **LOW**: Monitor (outdated dependencies)

Each finding will include:
- File location and line number
- CWE/OWASP reference
- Impact explanation
- Concrete fix with code example
