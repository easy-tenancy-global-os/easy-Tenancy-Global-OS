# Security Setup & Implementation Guide

Complete setup instructions for securing the real estate operating system.

---

## Quick Start (5 minutes)

### 1. Environment Configuration

```bash
# Copy template (no values)
cp .env.example .env.local

# Edit with your values (NOT committed to git)
nano .env.local
```

### 2. Install Security Dependencies

```bash
npm install --save-dev \
  express-rate-limit \
  helmet \
  cors \
  express-validator \
  dotenv
```

### 3. Initialize Security Logging

```bash
# Create audit log directory
mkdir -p data/logs

# Initialize empty audit log
touch data/logs/audit.log
echo "# Audit log initialized at $(date)" >> data/logs/audit.log
```

### 4. Update .gitignore

```bash
cat >> .gitignore << 'EOF'

# Secrets & Configuration
.env
.env.local
.env.*.local
*.key
*.pem
credentials/
secrets/

# Runtime Data & Sensitive Files
data/
data/status/
data/logs/
data/output/
data/runs/
data/properties/*/documents/
data/properties/*/uploads/

EOF

git add .gitignore
git commit -m "Update .gitignore to exclude sensitive files"
```

---

## Environment Variables

### .env.example Template

```bash
# Application Configuration
NODE_ENV=development
LOG_LEVEL=info
PORT=3000
DASHBOARD_PORT=5173

# Security
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
SESSION_SECRET_LENGTH=32
JWT_SECRET=your-secret-here

# Database (if applicable)
DATABASE_URL=
DB_USER=
DB_PASSWORD=

# External APIs
EXTERNAL_API_KEY=
EXTERNAL_API_ENDPOINT=

# Audit & Logging
AUDIT_LOG_PATH=data/logs/audit.log
ENABLE_AUDIT_LOGGING=true
LOG_REDACTION_ENABLED=true
LOG_LEVEL_AUDIT=info

# Rate Limiting
RATE_LIMIT_UPLOAD_WINDOW_MS=900000
RATE_LIMIT_UPLOAD_MAX=100
RATE_LIMIT_TRANSACTION_WINDOW_MS=60000
RATE_LIMIT_TRANSACTION_MAX=10
RATE_LIMIT_REPORT_WINDOW_MS=300000
RATE_LIMIT_REPORT_MAX=20
```

---

## Rate Limiting

### File: `src/middleware/rateLimiter.ts`

```typescript
import rateLimit from 'express-rate-limit';

export const limiters = {
  upload: rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 100,
    message: 'Too many upload requests',
    standardHeaders: true,
    legacyHeaders: false,
  }),

  transaction: rateLimit({
    windowMs: 1 * 60 * 1000,
    max: 10,
    message: 'Too many transaction requests',
    standardHeaders: true,
    legacyHeaders: false,
  }),

  report: rateLimit({
    windowMs: 5 * 60 * 1000,
    max: 20,
    message: 'Too many report requests',
    standardHeaders: true,
    legacyHeaders: false,
  }),
};
```

---

## Permission Middleware

### File: `src/middleware/permissions.ts`

```typescript
import express from 'express';

export type PermissionScope =
  | 'property:read' | 'property:write' | 'property:delete'
  | 'tenant:read' | 'tenant:write' | 'tenant:delete'
  | 'lease:read' | 'lease:write' | 'lease:sign'
  | 'payment:read' | 'payment:write' | 'payment:refund'
  | 'document:read' | 'document:write' | 'document:delete'
  | 'report:export' | 'report:advanced'
  | 'audit:read' | 'user:manage';

export const ROLE_PERMISSIONS: Record<string, PermissionScope[]> = {
  viewer: [
    'property:read', 'tenant:read', 'lease:read',
    'payment:read', 'document:read',
  ],

  manager: [
    'property:read', 'property:write',
    'tenant:read', 'tenant:write',
    'lease:read', 'lease:write', 'lease:sign',
    'payment:read', 'payment:write',
    'document:read', 'document:write',
    'report:export',
  ],

  accountant: [
    'property:read', 'tenant:read', 'lease:read',
    'payment:read', 'payment:write', 'payment:refund',
    'document:read', 'document:write',
    'report:export', 'report:advanced',
  ],

  admin: [
    'property:read', 'property:write', 'property:delete',
    'tenant:read', 'tenant:write', 'tenant:delete',
    'lease:read', 'lease:write', 'lease:sign',
    'payment:read', 'payment:write', 'payment:refund',
    'document:read', 'document:write', 'document:delete',
    'report:export', 'report:advanced',
    'audit:read', 'user:manage',
  ],
};

export function requirePermission(permission: PermissionScope) {
  return async (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const userId = (req as any).user?.id;
    if (!userId) return res.status(401).json({ error: 'Unauthorized' });

    const userRole = (req as any).user?.role || 'viewer';
    const permissions = ROLE_PERMISSIONS[userRole] || [];

    if (!permissions.includes(permission)) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    next();
  };
}
```

---

## Audit Logging

### File: `src/services/auditLog.ts`

```typescript
import fs from 'fs';
import path from 'path';

export interface AuditEvent {
  timestamp: string;
  userId: string;
  action: string;
  resourceType: string;
  resourceId: string;
  status: 'success' | 'failure';
  details: Record<string, any>;
}

const AUDIT_LOG_PATH = path.join(
  process.cwd(),
  process.env.AUDIT_LOG_PATH || 'data/logs/audit.log'
);

export async function logAuditEvent(event: AuditEvent): Promise<void> {
  try {
    const dir = path.dirname(AUDIT_LOG_PATH);
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }

    fs.appendFileSync(AUDIT_LOG_PATH, JSON.stringify(event) + '\n');

    const sensitiveSeverityActions = [
      'PAYMENT_RECORDED', 'PAYMENT_REFUNDED',
      'REPORT_GENERATED', 'USER_ROLE_CHANGED',
    ];

    if (sensitiveSeverityActions.includes(event.action)) {
      console.log(`[AUDIT] ${event.action} by ${event.userId}`);
    }
  } catch (err) {
    console.error('[AUDIT_LOG_ERROR]', err);
  }
}

export async function readAuditLog(
  filter?: Partial<AuditEvent>,
  limit: number = 100,
): Promise<AuditEvent[]> {
  try {
    if (!fs.existsSync(AUDIT_LOG_PATH)) return [];

    const lines = fs.readFileSync(AUDIT_LOG_PATH, 'utf-8').split('\n');
    let events: AuditEvent[] = lines
      .filter(line => line.trim())
      .map(line => {
        try {
          return JSON.parse(line);
        } catch {
          return null;
        }
      })
      .filter((e): e is AuditEvent => e !== null);

    if (filter) {
      events = events.filter(e => {
        if (filter.action && e.action !== filter.action) return false;
        if (filter.userId && e.userId !== filter.userId) return false;
        if (filter.resourceId && e.resourceId !== filter.resourceId) return false;
        return true;
      });
    }

    return events.slice(-limit).reverse();
  } catch (err) {
    console.error('[AUDIT_LOG_READ_ERROR]', err);
    return [];
  }
}
```

---

## Secret Redaction

### File: `src/utils/redaction.ts`

```typescript
const REDACTION_PATTERNS = [
  /authorization:\s*bearer\s+[^\s]*/gi,
  /token=[^\s&]*/gi,
  /api[_-]?key[=:]\s*[^\s&]*/gi,
  /password[=:]\s*[^\s&]*/gi,
  /cookie:\s*[^\n]*/gi,
];

export function redactSecrets(content: string): string {
  let redacted = content;
  REDACTION_PATTERNS.forEach(pattern => {
    redacted = redacted.replace(pattern, '[REDACTED]');
  });
  return redacted;
}

export function sanitizeForLog(obj: any): any {
  if (typeof obj !== 'object' || obj === null) return obj;

  if (Array.isArray(obj)) return obj.map(item => sanitizeForLog(item));

  const sanitized: any = {};
  for (const [key, value] of Object.entries(obj)) {
    if (/password|token|secret|key|credential/i.test(key)) {
      sanitized[key] = '[REDACTED]';
    } else if (typeof value === 'object') {
      sanitized[key] = sanitizeForLog(value);
    } else {
      sanitized[key] = value;
    }
  }

  return sanitized;
}
```

---

## API Endpoint Examples

### File: `src/routes/api.ts`

```typescript
import express from 'express';
import { limiters } from '../middleware/rateLimiter';
import { requirePermission } from '../middleware/permissions';
import { logAuditEvent } from '../services/auditLog';

const router = express.Router();

// Upload document
router.post(
  '/api/properties/:propertyId/documents',
  limiters.upload,
  requirePermission('document:write'),
  async (req, res) => {
    try {
      const { propertyId } = req.params;
      const { fileName, contentBase64 } = req.body;
      const userId = (req as any).user?.id;

      if (!fileName || !contentBase64) {
        return res.status(400).json({ error: 'Missing required fields' });
      }

      try {
        Buffer.from(contentBase64, 'base64');
      } catch {
        return res.status(400).json({ error: 'Invalid base64 content' });
      }

      const docId = `doc-${Date.now()}`;

      await logAuditEvent({
        timestamp: new Date().toISOString(),
        userId,
        action: 'DOCUMENT_UPLOADED',
        resourceType: 'document',
        resourceId: docId,
        status: 'success',
        details: { propertyId, fileName },
      });

      res.json({ success: true, docId });
    } catch (err) {
      console.error(err);
      res.status(500).json({ error: 'Internal server error' });
    }
  },
);

// Record payment
router.post(
  '/api/payments',
  limiters.transaction,
  requirePermission('payment:write'),
  async (req, res) => {
    try {
      const { propertyId, amount, method, date } = req.body;
      const userId = (req as any).user?.id;

      const paymentId = `pay-${Date.now()}`;

      await logAuditEvent({
        timestamp: new Date().toISOString(),
        userId,
        action: 'PAYMENT_RECORDED',
        resourceType: 'payment',
        resourceId: paymentId,
        status: 'success',
        details: { propertyId, amount, method, date },
      });

      res.json({ success: true, paymentId });
    } catch (err) {
      console.error(err);
      res.status(500).json({ error: 'Internal server error' });
    }
  },
);

export default router;
```

---

## Monitoring & Maintenance

### Daily
- Review audit logs for suspicious activity
- Check error logs for security issues
- Monitor rate limit violations

### Weekly
- Review permission changes
- Check for failed login attempts

### Monthly
- Run `npm audit`
- Update dependencies
- Review security incidents

### Quarterly
- Full security audit
- Penetration testing
- Policy review
