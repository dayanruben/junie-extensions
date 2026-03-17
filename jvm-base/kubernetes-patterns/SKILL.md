---
name: "kubernetes-patterns"
description: "Kubernetes manifests best practices: Deployments, health checks, resource limits, ConfigMaps. Use when working with k8s YAML files."
---

# Kubernetes Patterns Skill

Write production-ready Kubernetes manifests.

## Scope and Boundaries

- Use this skill for Kubernetes manifest patterns: Deployments, Services, Ingress, ConfigMaps, Secrets.
- Use `docker-patterns` for Dockerfile optimization and Docker Compose.
- Use `spring-actuator-patterns` for configuring the health endpoints that k8s probes call.
- Treat examples as starting points; adapt resource limits, replica counts, and probe paths to the current workload.

## When to Use
- Creating or reviewing Kubernetes manifests
- Setting up Deployments, Services, Ingress
- Configuring health checks and resource limits

---

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0   # ✅ specific tag, not latest
          ports:
            - containerPort: 8080
          resources:            # ✅ always set resource limits
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:        # ✅ health checks
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          env:
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:   # ✅ use Secrets for sensitive data
                  name: db-secret
                  key: password
```

---

## ConfigMap & Secret

```yaml
# ✅ ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"

---
# ✅ Secret for sensitive data (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=  # base64
```

---

## Service & Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# ✅ Ingress — route external traffic to the service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
  tls:
    - hosts:
        - my-app.example.com
      secretName: my-app-tls   # TLS certificate stored as a Secret
```

---

## Security Context

```yaml
# ✅ Good — non-root, read-only filesystem
spec:
  containers:
    - name: my-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
      volumeMounts:
        - name: tmp
          mountPath: /tmp         # writable tmp for JVM
  volumes:
    - name: tmp
      emptyDir: {}
```

---

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Anti-Patterns

- Missing resource `requests` and `limits` — pods get evicted unpredictably under memory pressure.
- Using `latest` image tag — no rollback guarantee, cache confusion between nodes.
- Missing liveness/readiness probes — Kubernetes cannot detect unhealthy pods.
- Storing secrets in ConfigMaps — use Secret resources (base64) or external secret managers.
- Setting `initialDelaySeconds` too low for JVM apps — Spring Boot startup can take 10-30s; pods get killed before they're ready.
- Running as root without `securityContext` — violates least-privilege principle.
