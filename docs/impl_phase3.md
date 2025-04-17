# Phase 3: Security Implementation

## Overview

This phase focuses on implementing comprehensive security measures for the Endpoint Statistics application. We'll set up network policies, RBAC, and security contexts to ensure the application is properly secured against common threats and follows Kubernetes security best practices.

## Security Principles

The implementation follows key security principles:

- **Defense in depth**: Multiple layers of security controls
- **Least privilege**: Granting only the access necessary for each component
- **Secure by default**: Starting with restricted access and adding permissions as needed
- **Isolation**: Separating application components to limit the impact of compromises

## Implementation Steps

### 1. Network Policies

Network policies act as a firewall within Kubernetes, controlling which pods can communicate with each other. They help prevent lateral movement if an attacker gains access to one pod.

```yaml
# network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: endpoint-stats  # Match namespace from Phase 1
spec:
  podSelector: {}  # Empty selector applies to all pods in namespace
  policyTypes:
  - Ingress
  - Egress  # Blocks both incoming and outgoing traffic by default
```

This policy denies all traffic (ingress and egress) to all pods in the namespace by default, creating a zero-trust starting point.

```yaml
# api-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-access
  namespace: endpoint-stats
spec:
  podSelector:
    matchLabels:
      app: flask-api  # Match labels defined in Phase 1 deployment
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:  # Allow traffic from the same namespace
        matchLabels:
          name: endpoint-stats
    - podSelector:  # Allow ingress controller access for external traffic
        matchLabels:
          app: ingress-nginx
    ports:
    - protocol: TCP
      port: 9999  # Match the port exposed by Flask API in Phase 1
  egress:  # Allow outgoing traffic to PostgreSQL and Redis
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:  # Allow egress to Prometheus for metrics exposure
    - podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090
```

```yaml
# postgres-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-policy
  namespace: endpoint-stats
spec:
  podSelector:
    matchLabels:
      app: postgres  # Match labels from Phase 1 PostgreSQL deployment
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:  # Only allow Flask API to access PostgreSQL
        matchLabels:
          app: flask-api
    ports:
    - protocol: TCP
      port: 5432  # Standard PostgreSQL port
  - from:
    - podSelector:  # Allow Prometheus to scrape PostgreSQL metrics
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9187  # PostgreSQL metrics exporter port
  egress: []  # No outbound connections needed for PostgreSQL
```

```yaml
# redis-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-policy
  namespace: endpoint-stats
spec:
  podSelector:
    matchLabels:
      app: redis  # Match labels from Phase 1 Redis deployment
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:  # Only allow Flask API to access Redis
        matchLabels:
          app: flask-api
    ports:
    - protocol: TCP
      port: 6379  # Standard Redis port
  - from:
    - podSelector:  # Allow Prometheus to scrape Redis metrics
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9121  # Redis metrics exporter port
  egress: []  # No outbound connections needed for Redis
```

```yaml
# monitoring-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: monitoring-policy
  namespace: endpoint-stats
spec:
  podSelector:
    matchLabels:
      app: prometheus  # Match Prometheus deployment from Phase 2
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:  # Allow Grafana to access Prometheus
        matchLabels:
          app: grafana
    ports:
    - protocol: TCP
      port: 9090  # Prometheus server port
  egress:
  - to: []  # Allow Prometheus to scrape all pods
    ports:
    - protocol: TCP
      port: 9999  # Flask API metrics port
    - protocol: TCP
      port: 9187  # PostgreSQL metrics port
    - protocol: TCP
      port: 9121  # Redis metrics port
```

### 2. RBAC Configuration

Role-Based Access Control (RBAC) limits what actions different users and service accounts can perform within the cluster.

```yaml
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: endpoint-stats-sa
  namespace: endpoint-stats
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: endpoint-stats-reader
  namespace: endpoint-stats
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]  # Read-only access to pod and service info
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]  # Read-only access to ConfigMaps
  resourceNames: ["api-config"]  # Limit to specific ConfigMap
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch"]  # Needed for service discovery
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: endpoint-stats-reader-binding
  namespace: endpoint-stats
subjects:
- kind: ServiceAccount
  name: endpoint-stats-sa
  namespace: endpoint-stats
roleRef:
  kind: Role
  name: endpoint-stats-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: endpoint-stats
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-role
  namespace: endpoint-stats
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]  # Required for service discovery and target scraping
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["prometheus-config", "prometheus-rules"]  # Only access relevant ConfigMaps
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-role-binding
  namespace: endpoint-stats
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: endpoint-stats
roleRef:
  kind: Role
  name: prometheus-role
  apiGroup: rbac.authorization.k8s.io
```

### 3. Security Contexts

Security contexts define privilege and access control settings for pods and containers, implementing the principle of least privilege.

```yaml
# flask-api-security.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-api
  namespace: endpoint-stats
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"  # Enable Prometheus metrics scraping
        prometheus.io/path: "/metrics"  # Path for metrics
        prometheus.io/port: "9999"  # Port for metrics
    spec:
      serviceAccountName: endpoint-stats-sa  # Use the service account defined above
      securityContext:
        runAsNonRoot: true  # Don't run as root user
        runAsUser: 1000     # Run as non-privileged user
        fsGroup: 2000       # Set file system group for volume access
      containers:
      - name: flask-api
        securityContext:
          allowPrivilegeEscalation: false  # Prevent privilege escalation
          readOnlyRootFilesystem: true  # Make root filesystem read-only
          capabilities:
            drop:
            - ALL  # Drop all Linux capabilities
          seccompProfile:
            type: RuntimeDefault  # Use default seccomp profile
        volumeMounts:
        - name: tmp-volume  # Mount a writable volume for temporary files
          mountPath: /tmp
        - name: api-config  # Mount ConfigMap as read-only volume
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: tmp-volume  # Define writable volume for temp files
        emptyDir: {}
      - name: api-config  # Define ConfigMap volume
        configMap:
          name: api-config
```

```yaml
# postgres-security.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: endpoint-stats
spec:
  template:
    spec:
      securityContext:
        fsGroup: 999  # Default PostgreSQL group
      containers:
      - name: postgres
        securityContext:
          allowPrivilegeEscalation: false
          runAsUser: 999  # Default PostgreSQL user
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
            add:
            - CHOWN
            - FOWNER
            - SETGID
            - SETUID  # Minimum capabilities needed for PostgreSQL
```

```yaml
# redis-security.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: endpoint-stats
spec:
  template:
    spec:
      securityContext:
        fsGroup: 999  # Common non-root group
      containers:
      - name: redis
        securityContext:
          allowPrivilegeEscalation: false
          runAsUser: 999  # Common non-root user
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL  # Redis doesn't require special capabilities
```

### 4. ConfigMap for Application Configuration

Separate sensitive configuration from code to enhance security:

```yaml
# api-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: endpoint-stats
data:
  config.json: |
    {
      "LOG_LEVEL": "INFO",
      "ALLOWED_ORIGINS": "https://api.endpoint-stats.com",
      "METRICS_ENABLED": "true",
      "RATE_LIMIT": "100"
    }
```

### 5. Secret Management

Secrets management involves securely storing and accessing sensitive information like API keys and passwords.

```yaml
# secret-management.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-credentials
  namespace: endpoint-stats
type: Opaque
data:
  # Base64 encoded credentials - in production these should be managed by a secure vault service
  DB_USER: cG9zdGdyZXM=        # "postgres" in base64
  DB_PASSWORD: cG9zdGdyZXM=    # "postgres" in base64
  REDIS_PASSWORD: ""           # Empty password for development
  API_KEY: ZW5kcG9pbnQtc3RhdHMta2V5  # "endpoint-stats-key" in base64
```

Update the Flask API deployment to use these secrets:

```yaml
# api-deployment-with-secrets.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-api
  namespace: endpoint-stats
spec:
  # ... other settings remain the same
  template:
    spec:
      containers:
      - name: flask-api
        # ... other settings remain the same
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: api-credentials
              key: DB_USER
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: api-credentials
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-credentials
              key: API_KEY
        envFrom:
        - configMapRef:
            name: api-config
```

### 6. TLS Configuration

Secure communication requires TLS certificates for encryption in transit.

```yaml
# tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: endpoint-stats
type: kubernetes.io/tls
data:
  # In production, these should be actual certificates
  # For development, you can generate self-signed certificates
  tls.crt: BASE64_ENCODED_CERT
  tls.key: BASE64_ENCODED_KEY
```

Update the Ingress to use TLS:

```yaml
# ingress-with-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-api-ingress
  namespace: endpoint-stats
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"  # Force HTTPS
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"  # Increase allowed request size
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"  # Increase timeout values
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  tls:
  - hosts:
    - api.endpoint-stats.com
    secretName: tls-secret
  rules:
  - host: api.endpoint-stats.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: flask-api
            port:
              number: 9999
```

### 7. Image Security

Container image security helps prevent malicious code from being deployed.

```yaml
# deployment-with-image-security.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-api
  namespace: endpoint-stats
spec:
  template:
    spec:
      containers:
      - name: flask-api
        image: endpoint-stats:v2@sha256:abc123...  # Use digest instead of tag
        imagePullPolicy: Always  # Always pull to get latest security updates
```

## Security Best Practices

1. **Regular Updates**: Keep all components patched and updated regularly
2. **Image Scanning**: Use tools like Trivy, Clair, or Anchore to scan for vulnerabilities
3. **Audit Logging**: Enable audit logging for all cluster operations
4. **Secret Rotation**: Regularly rotate credentials and secrets
5. **Network Segmentation**: Implement strict network policies
6. **Penetration Testing**: Regularly test security controls
7. **Cluster Hardening**: Follow CIS Kubernetes Benchmarks

## Implementation Checklist

- [x] Apply network policies
- [x] Configure RBAC roles and bindings
- [x] Set up security contexts
- [x] Configure secret management
- [x] Set up TLS for ingress
- [x] Implement image security controls
- [x] Test security configurations
- [x] Verify access controls
- [x] Document security procedures

## Common Vulnerabilities Prevented

| Vulnerability | Prevention Mechanism |
|---------------|----------------------|
| Container escape | Security contexts, drop capabilities |
| Privilege escalation | `allowPrivilegeEscalation: false`, non-root users |
| Credential exposure | Kubernetes Secrets, proper env var usage |
| Network attacks | Network Policies, TLS encryption |
| Data exfiltration | Egress network policies |
| Supply chain attacks | Image digest pinning, scanning |
| Unauthorized access | RBAC, network policies |

## Troubleshooting Security Issues

1. **Permission Issues**:

   ```bash
   # Check if service account has proper RBAC
   kubectl auth can-i --as=system:serviceaccount:endpoint-stats:endpoint-stats-sa get pods -n endpoint-stats
   ```

2. **Network Policy Issues**:

   ```bash
   # Install a temporary debug pod
   kubectl run -n endpoint-stats debug --image=busybox --rm -it -- /bin/sh

   # Test connectivity from inside the pod
   wget -O- --timeout=2 http://flask-api:9999/health
   ```

3. **Security Context Issues**:

   ```bash
   # Check container processes and users
   kubectl exec -it -n endpoint-stats <pod-name> -- ps aux
   ```

4. **TLS Issues**:

   ```bash
   # Test TLS configuration
   curl -v https://api.endpoint-stats.com
   ```

## Next Steps

After completing Phase 3, proceed to [Phase 4: Deployment Strategy](impl_phase4.md) to implement deployment strategies and rolling updates.
