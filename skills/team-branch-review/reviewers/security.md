# Security Review Brief

You are hunting for vulnerabilities, not style issues. Think like an attacker.

## What to Look For

**Input handling**
- User-controlled data flowing into dangerous sinks (SQL, shell, file paths, HTML)
- Missing or incomplete input validation at trust boundaries
- Type coercion or deserialization of untrusted data

**Authentication and authorization**
- Endpoints or operations missing auth checks
- Authorization logic that checks roles but not resource ownership
- Token handling: hardcoded secrets, weak generation, missing expiry, insecure storage
- Session management gaps

**Data exposure**
- Sensitive data in logs, error messages, or API responses
- Credentials, keys, or tokens in code or config files
- PII leaking through debug endpoints or verbose errors
- Missing redaction in serialization

**Cryptography**
- Weak algorithms (MD5, SHA1 for security purposes, ECB mode)
- Hardcoded keys or IVs
- Missing integrity checks on data that needs them
- Rolling your own crypto when a standard library exists

**Dependency and configuration**
- New dependencies with known vulnerabilities
- Overly permissive CORS, CSP, or network policies
- Debug flags or development settings that could leak to production
- Default credentials or configurations

## How to Think

For each changed file, ask:
1. What data enters here? Can an attacker control it?
2. What operations happen with that data? Could they be abused?
3. What assumptions does this code make about its callers? Are those assumptions enforced?
4. If this code fails, what gets exposed?

Trace data flow across file boundaries. A sanitization gap in one file may only be exploitable through a path in another changed file.

## Severity Guidance

- **Critical**: Exploitable without authentication, leads to data breach or RCE
- **High**: Requires some access but enables privilege escalation or significant data exposure
- **Medium**: Requires specific conditions to exploit, limited impact
- **Low**: Defense-in-depth improvements, hardening suggestions
