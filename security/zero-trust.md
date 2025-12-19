# Zero Trust Architecture

Never trust, always verify - security for modern distributed systems.

## Core Principles

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZERO TRUST PILLARS                           │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Identity      │   Device        │   Network                   │
│   - MFA         │   - Health      │   - Micro-segmentation      │
│   - SSO         │   - Compliance  │   - Encryption             │
│   - Conditional │   - MDM         │   - mTLS                   │
├─────────────────┼─────────────────┼─────────────────────────────┤
│   Application   │   Data          │   Visibility               │
│   - API Gateway │   - Classification│ - Logging                │
│   - WAF         │   - Encryption  │   - SIEM                   │
│   - Runtime     │   - DLP         │   - Anomaly Detection     │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

## Service-to-Service Authentication (mTLS)

```typescript
// mTLS configuration with certificate validation
import https from 'https';
import fs from 'fs';

// Server with mTLS
const serverOptions = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),
  ca: fs.readFileSync('ca-cert.pem'),
  requestCert: true,
  rejectUnauthorized: true,
};

const server = https.createServer(serverOptions, (req, res) => {
  const cert = req.socket.getPeerCertificate();

  // Verify client identity
  const clientIdentity = extractServiceIdentity(cert);
  if (!isAuthorized(clientIdentity, req.url, req.method)) {
    res.writeHead(403);
    res.end('Forbidden');
    return;
  }

  // Process request with verified identity
  req.serviceIdentity = clientIdentity;
  handleRequest(req, res);
});

// SPIFFE/SPIRE identity integration
interface SPIFFEIdentity {
  spiffeId: string;  // spiffe://example.com/service/api
  trustDomain: string;
  workload: string;
}

function extractServiceIdentity(cert: Certificate): SPIFFEIdentity {
  // Extract SPIFFE ID from SAN extension
  const san = cert.subjectaltname;
  const spiffeMatch = san?.match(/URI:spiffe:\/\/([^/]+)\/(.+)/);

  if (!spiffeMatch) {
    throw new Error('Invalid SPIFFE identity');
  }

  return {
    spiffeId: `spiffe://${spiffeMatch[1]}/${spiffeMatch[2]}`,
    trustDomain: spiffeMatch[1],
    workload: spiffeMatch[2],
  };
}

// Service authorization matrix
const authorizationMatrix: AuthMatrix = {
  'service/api': {
    'service/database': ['read', 'write'],
    'service/cache': ['read', 'write', 'delete'],
    'service/payment': ['process'],
  },
  'service/worker': {
    'service/database': ['read'],
    'service/queue': ['consume', 'publish'],
  },
};

function isAuthorized(caller: SPIFFEIdentity, path: string, method: string): boolean {
  const permissions = authorizationMatrix[caller.workload];
  // Check if caller can access the requested resource
  return checkPermission(permissions, path, method);
}
```

## Policy-Based Access Control

```typescript
// Open Policy Agent (OPA) integration
import { Rego } from '@open-policy-agent/opa-wasm';

class PolicyEngine {
  private policy: Rego;

  async loadPolicy(wasmPath: string): Promise<void> {
    const policyWasm = fs.readFileSync(wasmPath);
    this.policy = await loadPolicy(policyWasm);
  }

  async evaluate(input: PolicyInput): Promise<PolicyDecision> {
    const result = this.policy.evaluate(input);
    return result[0]?.result as PolicyDecision;
  }
}

// OPA Rego policy
/*
package authz

import future.keywords.if
import future.keywords.in

default allow := false

# Allow if user has required role
allow if {
  required_roles := data.role_permissions[input.resource][input.action]
  some role in input.user.roles
  role in required_roles
}

# Allow if user owns the resource
allow if {
  input.resource_owner == input.user.id
}

# Deny if accessing from untrusted network
deny if {
  not input.network in data.trusted_networks
}

# Deny if device is not compliant
deny if {
  not input.device.compliant
}

# Deny outside business hours for sensitive resources
deny if {
  input.resource in data.sensitive_resources
  not within_business_hours(input.time)
}

within_business_hours(t) if {
  hour := time.clock(t)[0]
  hour >= 9
  hour < 18
}
*/

// Policy decision point middleware
async function policyEnforcement(req: Request, res: Response, next: NextFunction) {
  const input: PolicyInput = {
    user: {
      id: req.user.id,
      roles: req.user.roles,
      department: req.user.department,
    },
    resource: extractResource(req),
    action: mapMethodToAction(req.method),
    device: {
      id: req.headers['x-device-id'],
      compliant: await checkDeviceCompliance(req.headers['x-device-id']),
      platform: req.headers['x-device-platform'],
    },
    network: {
      ip: req.ip,
      type: classifyNetwork(req.ip),
    },
    time: new Date().toISOString(),
    context: {
      mfaVerified: req.session?.mfaVerified,
      sessionAge: Date.now() - req.session?.createdAt,
    },
  };

  const decision = await policyEngine.evaluate(input);

  if (!decision.allow || decision.deny) {
    auditLog('access_denied', input, decision.reasons);
    return res.status(403).json({ error: 'Access denied', reasons: decision.reasons });
  }

  // Apply any required actions (step-up auth, etc.)
  if (decision.require_mfa && !input.context.mfaVerified) {
    return res.status(401).json({ error: 'MFA required', redirect: '/mfa' });
  }

  next();
}
```

## Network Micro-Segmentation

```yaml
# Kubernetes Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only allow traffic from ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
          podSelector:
            matchLabels:
              app: nginx-ingress
      ports:
        - protocol: TCP
          port: 8080
    # Allow health checks from kubelet
    - from:
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 8081
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Allow database access
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow external APIs (via egress gateway)
    - to:
        - namespaceSelector:
            matchLabels:
              name: egress-gateway
      ports:
        - protocol: TCP
          port: 443
---
# Istio Authorization Policy (Layer 7)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-server-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-server
  rules:
    # Allow authenticated requests from web frontend
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/web-frontend
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
            paths: ["/api/*"]
      when:
        - key: request.auth.claims[iss]
          values: ["https://auth.example.com"]
    # Allow internal health checks
    - from:
        - source:
            namespaces: ["kube-system"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/health", "/ready"]
```

## Continuous Verification

```typescript
// Real-time risk assessment
class RiskEngine {
  async assessRisk(context: AccessContext): Promise<RiskScore> {
    const signals = await Promise.all([
      this.checkDeviceRisk(context.device),
      this.checkBehaviorRisk(context.user, context.action),
      this.checkNetworkRisk(context.network),
      this.checkTimeRisk(context.timestamp),
      this.checkGeoRisk(context.location),
    ]);

    const weights = [0.25, 0.30, 0.15, 0.15, 0.15];
    const score = signals.reduce((acc, signal, i) => acc + signal * weights[i], 0);

    return {
      score,
      level: this.scoreToLevel(score),
      signals: signals.map((s, i) => ({ name: this.signalNames[i], value: s })),
      recommendations: this.getRecommendations(score, signals),
    };
  }

  private async checkBehaviorRisk(user: User, action: Action): Promise<number> {
    const baseline = await this.getBaseline(user.id);

    // Check for anomalies
    const anomalies = [
      this.checkAccessTimeAnomaly(action.timestamp, baseline.typicalHours),
      this.checkVolumeAnomaly(user.id, baseline.typicalVolume),
      this.checkPatternAnomaly(action, baseline.typicalPatterns),
    ];

    return Math.max(...anomalies);
  }

  private async checkDeviceRisk(device: Device): Promise<number> {
    let risk = 0;

    // New device
    if (!await this.isKnownDevice(device.id)) {
      risk += 0.3;
    }

    // Device not compliant
    if (!device.compliant) {
      risk += 0.4;
    }

    // Outdated OS
    if (device.osVersion < this.minimumOsVersion[device.platform]) {
      risk += 0.2;
    }

    // No security software
    if (!device.securitySoftware) {
      risk += 0.1;
    }

    return Math.min(1, risk);
  }
}

// Adaptive authentication based on risk
class AdaptiveAuth {
  async authenticate(context: AuthContext): Promise<AuthResult> {
    const basicAuth = await this.verifyCredentials(context.credentials);
    if (!basicAuth.success) {
      return { success: false, reason: 'invalid_credentials' };
    }

    const risk = await this.riskEngine.assessRisk(context);

    // Low risk - basic auth sufficient
    if (risk.level === 'low') {
      return { success: true, user: basicAuth.user };
    }

    // Medium risk - require MFA
    if (risk.level === 'medium') {
      if (!context.mfaToken) {
        return { success: false, reason: 'mfa_required', riskLevel: risk.level };
      }
      const mfaValid = await this.verifyMFA(basicAuth.user.id, context.mfaToken);
      if (!mfaValid) {
        return { success: false, reason: 'invalid_mfa' };
      }
      return { success: true, user: basicAuth.user };
    }

    // High risk - require step-up + additional verification
    if (risk.level === 'high') {
      // Require biometric or hardware key
      if (!context.biometric && !context.hardwareKey) {
        return { success: false, reason: 'strong_auth_required', riskLevel: risk.level };
      }

      // Notify user of suspicious access
      await this.notifyUser(basicAuth.user, 'suspicious_access', context);

      // Require admin approval for very high risk
      if (risk.score > 0.9) {
        const approved = await this.requestApproval(basicAuth.user.id, context);
        if (!approved) {
          return { success: false, reason: 'access_denied_high_risk' };
        }
      }

      return { success: true, user: basicAuth.user, elevated: true };
    }

    return { success: false, reason: 'unknown_risk_level' };
  }
}
```

---

*Learned: 2024*
*Tags: Security, Zero Trust, mTLS, SPIFFE, Policy, Network Security*
