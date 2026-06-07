# Security Best Practices for Real Estate Operating System

## Overview

This document provides security best practices for the real estate operating system, covering API design, authentication, data handling, and operational security.

---

## 1. Authentication & Authorization

### 1.1 Local CLI Authentication

**Current Model:**
- No credentials stored in the repository
- User's local CLI handles authentication
- API returns only booleans (`installed`, `loggedIn`, `authenticated`)
- Dashboard never exposes tokens, cookies, API keys, or credential file contents

**Best Practices:**

```typescript
// ✅ CORRECT: Status reporting without credential exposure
async function readAuthStatus(): Promise<AuthStatus> {
  const response = await fetch(`/api/auth/status`);
  // Returns: { installed: true, loggedIn: true, authenticated: true }
  // NEVER: { token: "...", refreshToken: "...", cookieValue: "..." }
  return response.json();
}

// ✅ CORRECT: Redact secrets from logs and artifacts
function redactSecrets(content: string): string {
  return content
    .replace(/Authorization:\s*Bearer\s+[^\s]*/g, 'Authorization: [REDACTED]')
    .replace(/token=[^\s&]*/g, 'token=[REDACTED]')
    .replace(/api[_-]?key[=:]\s*[^\s&]*/gi, 'api_key=[REDACTED]')
    .replace(/password[=:]\s*[^\s&]*/gi, 'password=[REDACTED]');
}

// ❌ WRONG: Storing or passing credentials through repo
config.apiToken = process.env.API_TOKEN; // Never do this
```

### 1.2 Environment Variable Handling

**Pattern:**
- Sensitive values loaded from local environment only
- Never commit `.env` files or credential files
- Use `.env.example` to document required variables without values

```bash
# ✅ .gitignore
.env
.env.local
.env.*.local
*.pem
*.key
credentials/
secrets/
```

```bash
# ✅ .env.example (document structure, NO values)
# Application configuration (local machine only)
API_CLI_PATH=~/.local/bin/api-cli
AUTH_ORG_ID=

# Application
NODE_ENV=development
LOG_LEVEL=info
PORT=3000
DASHBOARD_PORT=5173
```

### 1.3 API Key Management

**For any future API integrations:**

```typescript
// ✅ CORRECT: Keys loaded from secure environment
interface ApiConfig {
  apiKey: string;     // Never logged or exposed
  apiSecret?: string; // Only loaded, never returned
  endpoint: string;   // Can be logged
}

function loadApiConfig(): ApiConfig {
  const apiKey = process.env.EXTERNAL_API_KEY;
  if (!apiKey) {
    throw new Error('EXTERNAL_API_KEY not set in environment');
  }
  return {
    apiKey,
    endpoint: process.env.EXTERNAL_API_ENDPOINT || 'https://api.example.com',
  };
}

// ✅ CORRECT: Sanitize before logging
function sanitizeForLog(config: ApiConfig): Partial<ApiConfig> {
  return {
    endpoint: config.endpoint,
    // apiKey is omitted
  };
}
```

---

## 2. API Design Principles

### 2.1 Scoped Permissions (Least Privilege)

**Core Principle:** Every API consumer (user, service, workflow) has **minimum required permissions** only.

```typescript
// ✅ CORRECT: Fine-grained permission scopes
interface ApiScope {
  tenant: 'read' | 'write' | 'admin';
  property: 'read' | 'write' | 'admin';
  lease: 'read' | 'write' | 'admin';
  document: 'read' | 'write' | 'delete';
  export: 'json' | 'pdf' | 'all';
}

type UserPermissions = Partial<ApiScope>;

// Examples:
const viewerPerms: UserPermissions = {
  tenant: 'read',
  property: 'read',
  lease: 'read',
};

const operatorPerms: UserPermissions = {
  tenant: 'write',
  property: 'write',
  lease: 'write',
  document: 'write',
  export: 'all',
};

const apiClientPerms: UserPermissions = {
  tenant: 'read',
  property: 'read',
  export: 'json',
};
```

### 2.2 API Endpoint Security

**Pattern: Enforce Origin and Content-Type Validation**

```typescript
// ✅ CORRECT: Cross-origin document upload protection
app.post('/api/properties/:propertyId/documents', (req, res) => {
  // Validate origin
  const origin = req.headers['origin'];
  const allowedOrigins = [
    'http://localhost:5173',
    'http://localhost:3000',
  ];
  
  if (!allowedOrigins.includes(origin)) {
    return res.status(403).json({ error: 'Forbidden origin' });
  }

  // Validate content structure
  const { fileName, contentBase64 } = req.body;
  
  if (!fileName || typeof fileName !== 'string') {
    return res.status(400).json({ error: 'Invalid fileName' });
  }

  if (!contentBase64 || typeof contentBase64 !== 'string') {
    return res.status(400).json({ error: 'Missing contentBase64' });
  }

  // Validate base64 encoding
  try {
    Buffer.from(contentBase64, 'base64');
  } catch (err) {
    return res.status(400).json({ error: 'Invalid base64 document content' });
  }

  // Process...
});

// ✅ CORRECT: Rate limiting for API endpoints
import rateLimit from 'express-rate-limit';

const uploadLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                   // Max 100 requests per window
  message: 'Too many upload attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

app.post('/api/properties/:propertyId/documents', uploadLimiter, handler);
```

### 2.3 Workflow Execution Authorization

```typescript
// ✅ CORRECT: Verify resource ownership and workflow access
async function verifyWorkflowAccess(
  userId: string,
  resourceId: string,
  workflowId: string,
): Promise<boolean> {
  // 1. User must own or have read access to resource
  const resourceAccess = await checkResourceAccess(userId, resourceId);
  if (!resourceAccess) return false;

  // 2. User must be allowed to execute this workflow
  const userPerms = await getUserPermissions(userId);
  if (userPerms.workflow !== 'execute' && userPerms.workflow !== 'manage') {
    return false;
  }

  // 3. Workflow must not require sensitive operations
  const workflow = await loadWorkflow(workflowId);
  if (workflow.requiresLiveAgents && userPerms.workflow !== 'manage') {
    return false; // Only admins can launch live workflows
  }

  return true;
}
```

---

## 3. Data Handling & Storage

### 3.1 Local Storage Isolation

**Principle:** Sensitive data stored locally, never committed to git.

```bash
# ✅ .gitignore: Exclude runtime outputs
data/
data/status/
data/output/
data/reports/
data/runs/
data/properties/*/documents/
data/properties/*/uploads/
data/logs/

# ✅ Structure: Local-only outputs
data/
├── properties/
│   └── {propertyId}/
│       ├── property.json (not committed)
│       ├── documents/ (not committed)
│       ├── uploads/ (not committed)
│       └── packages/ (not committed)
├── status/ (not committed)
├── output/ (not committed)
├── reports/ (not committed)
├── logs/ (not committed)
└── runs/ (not committed)
```

### 3.2 Sensitive Data in Logs

**Pattern: Aggressive redaction before persistence**

```typescript
interface SecretRedactionConfig {
  patterns: RegExp[];
  replacements: string;
}

const REDACTION_CONFIG: SecretRedactionConfig = {
  patterns: [
    /authorization:\s*bearer\s+[^\s]*/gi,
    /token=[^\s&]*/gi,
    /api[_-]?key[=:]\s*[^\s&]*/gi,
    /password[=:]\s*[^\s&]*/gi,
    /cookie:\s*[^\n]*/gi,
    /x-api-key:\s*[^\s]*/gi,
  ],
  replacements: '[REDACTED]',
};

function redactSecretsFromLog(logContent: string): string {
  let redacted = logContent;
  REDACTION_CONFIG.patterns.forEach((pattern) => {
    redacted = redacted.replace(pattern, REDACTION_CONFIG.replacements);
  });
  return redacted;
}

// Applied before writing to file
const sanitizedLog = redactSecretsFromLog(rawOutput);
fs.writeFileSync(logPath, sanitizedLog);
```

### 3.3 Document Upload Validation

```typescript
// ✅ CORRECT: Validate file type and size
interface DocumentUploadConfig {
  maxFileSize: number;
  allowedExtensions: string[];
  allowedMimeTypes: string[];
}

const UPLOAD_CONFIG: DocumentUploadConfig = {
  maxFileSize: 50 * 1024 * 1024, // 50MB
  allowedExtensions: [
    '.csv', '.txt', '.md',
    '.xlsx', '.xls',
    '.pdf', '.doc', '.docx',
  ],
  allowedMimeTypes: [
    'text/csv',
    'text/plain',
    'text/markdown',
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    'application/vnd.ms-excel',
    'application/pdf',
    'application/msword',
  ],
};

async function validateUpload(
  fileName: string,
  fileSize: number,
  mimeType: string,
): Promise<{ valid: boolean; error?: string }> {
  // Check file size
  if (fileSize > UPLOAD_CONFIG.maxFileSize) {
    return {
      valid: false,
      error: `File exceeds maximum size of ${UPLOAD_CONFIG.maxFileSize / 1024 / 1024}MB`,
    };
  }

  // Check extension
  const ext = path.extname(fileName).toLowerCase();
  if (!UPLOAD_CONFIG.allowedExtensions.includes(ext)) {
    return {
      valid: false,
      error: `File type ${ext} not allowed`,
    };
  }

  // Check MIME type
  if (!UPLOAD_CONFIG.allowedMimeTypes.includes(mimeType)) {
    return {
      valid: false,
      error: `MIME type ${mimeType} not allowed`,
    };
  }

  return { valid: true };
}
```

---

## 4. Audit & Compliance

### 4.1 Audit Logging

```typescript
interface AuditLog {
  timestamp: string;        // ISO 8601
  userId: string;
  action: string;          // 'PROPERTY_CREATED', 'TENANT_UPDATED', etc.
  resourceType: string;    // 'property', 'tenant', 'lease', 'document'
  resourceId: string;
  status: 'SUCCESS' | 'FAILURE';
  details: {
    changes?: Record<string, unknown>;
    error?: string;
    ip?: string;
    userAgent?: string;
  };
}

async function logAuditEvent(log: AuditLog): Promise<void> {
  // 1. Write to audit log file (never deleted)
  fs.appendFileSync(
    `data/logs/audit.log`,
    JSON.stringify(log) + '\n',
  );

  // 2. For sensitive actions, also alert
  if (['WORKFLOW_LAUNCHED', 'EXPORT_GENERATED', 'BULK_UPDATE'].includes(log.action)) {
    console.log(`[AUDIT] ${log.action} by ${log.userId}`);
  }
}

// Usage:
await logAuditEvent({
  timestamp: new Date().toISOString(),
  userId: currentUser.id,
  action: 'PROPERTY_CREATED',
  resourceType: 'property',
  resourceId: propertyId,
  status: 'SUCCESS',
  details: {
    changes: { propertyName, address },
  },
});
```

---

## 5. External Integration Security

### 5.1 Third-Party Runtime Boundary

**Key Rules:**

1. **Credentials stay external:** External services handle authentication locally
2. **No token passing:** Dashboard never receives or sends credentials
3. **Data approval before send:** Only send context after user approval
4. **Selective data:** Only send required data, not entire file contents

```typescript
// ✅ CORRECT: Safe execution with data boundaries
interface WorkflowExecutionContext {
  resourceId: string;           // Identifier only
  workflowId: string;
  phase: string;
  selectedAgents: string[];     // Explicit agent whitelist
  resourceContext: {
    propertyAddress?: string;
    propertyType?: string;
    unitCount?: number;
    monthlyRevenue?: number;
    // NOT included: full tenant list, bank details, sensitive docs
  };
}

async function launchWorkflow(context: WorkflowExecutionContext): Promise<string> {
  // 1. Verify user approval for this execution
  const approved = await checkWorkflowApproval(context.resourceId);
  if (!approved) {
    throw new Error('Workflow not approved for execution');
  }

  // 2. Log the execution
  await logAuditEvent({
    timestamp: new Date().toISOString(),
    userId: currentUser.id,
    action: 'WORKFLOW_LAUNCHED',
    resourceType: 'workflow',
    resourceId: context.workflowId,
    status: 'SUCCESS',
    details: {
      resourceId: context.resourceId,
      agents: context.selectedAgents,
    },
  });

  // 3. Execute workflow
  const runId = await executeWorkflow(context);
  return runId;
}
```

### 5.2 Third-Party Integration Checklist

When adding external API integrations:

- [ ] API key sourced from environment only
- [ ] Credentials never logged or exported
- [ ] API calls rate-limited and timeout-configured
- [ ] Failures don't expose credential details
- [ ] Integration documented in SECURITY.md
- [ ] Audit log includes who/when/what, never credentials
- [ ] User consent collected before sending sensitive data
- [ ] Data retention policy documented

---

## 6. Operational Security

### 6.1 Version Management & Updates

```bash
# ✅ CORRECT: Dependency security
npm audit
npm audit fix
npm outdated

# ✅ Regular checks
npm run security:check  # Custom script
npm run lint:security   # ESLint security plugin
```

### 6.2 Error Handling

```typescript
// ✅ CORRECT: User-safe error messages without internal details
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public userMessage: string,
    public internalMessage?: string,
  ) {
    super(userMessage);
  }
}

app.use((err: Error, req: express.Request, res: express.Response) => {
  const apiErr = err instanceof ApiError ? err : 
    new ApiError(500, 'An error occurred', err.message);

  // Log internal details
  if (process.env.NODE_ENV !== 'production') {
    console.error('[INTERNAL]', apiErr.internalMessage || apiErr.message);
  }

  // Send user-safe response
  res.status(apiErr.statusCode).json({
    error: apiErr.userMessage,
    // NEVER: internalMessage, stack trace, SQL queries, etc.
  });
});
```

### 6.3 Local Development Security

```bash
# ✅ CORRECT: Development setup
.env.local               # Local secrets (gitignored)
.env.example            # Structure only

# ✅ CORRECT: Pre-commit hooks
# .husky/pre-commit
npm run lint:security
npm audit
npm run test:security
```

---

## 7. Incident Response

### 7.1 Credential Compromise

**If a credential is exposed:**

1. **Immediately:**
   - Revoke the credential in the external service
   - Search git history for any commits containing it
   - Notify users if sensitive data was accessible

2. **Clean up:**
   - Use `git filter-branch` or `bfg-repo-cleaner` if in history
   - Force push (with team coordination)
   - Add to `.gitignore` and commit cleanup

3. **Review:**
   - Check logs for unauthorized access
   - Rotate all related credentials
   - Document in security incident log

### 7.2 Reporting Process

Email **security concerns** to: `security@easy-tenancy.dev`

Include:
- Description of vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if you have one)

Response time: **48 hours acknowledgment**

---

## 8. References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CWE Top 25](https://cwe.mitre.org/top25/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/nodejs-security/)
- [Express.js Security](https://expressjs.com/en/advanced/best-practice-security.html)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
