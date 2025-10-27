---
apiVersion: v1
kind: Namespace
metadata:
  name: webdav
---
apiVersion: v1
kind: Secret
metadata:
  name: webdav-auth
  namespace: webdav
type: Opaque
stringData:
  WEBDAV_USER: admin
  WEBDAV_PASS: supersecret

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webdav-data
  namespace: webdav
spec:
  accessModes: ["ReadWriteOnce"]        # use ReadWriteMany if your storage supports it
  resources:
    requests:
      storage: 20Gi

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: webdav-config
  namespace: webdav
data:
  config.yml: |
    address: 0.0.0.0
    port: 6065
    prefix: /
    behindProxy: true
    directory: /data
    permissions: CRUD
    users:
      - username: "{env}WEBDAV_USER"
        password: "{env}WEBDAV_PASS"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webdav
  namespace: webdav
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webdav
  template:
    metadata:
      labels:
        app: webdav
    spec:
      containers:
        - name: webdav
          image: ghcr.io/hacdias/webdav:latest
          args: ["-c", "/etc/webdav/config.yml"]
          ports:
            - containerPort: 6065
          envFrom:
            - secretRef:
                name: webdav-auth
          volumeMounts:
            - name: config
              mountPath: /etc/webdav/config.yml
              subPath: config.yml
              readOnly: true
            - name: data
              mountPath: /data
          readinessProbe:
            httpGet:
              path: / # PROPFINDs still work; liveness/readiness can be a simple GET
              port: 6065
            initialDelaySeconds: 3
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 6065
            initialDelaySeconds: 10
            periodSeconds: 20
      volumes:
        - name: config
          configMap:
            name: webdav-config
        - name: data
          persistentVolumeClaim:
            claimName: webdav-data

---
apiVersion: v1
kind: Service
metadata:
  name: webdav
  namespace: webdav
spec:
  selector:
    app: webdav
  ports:
    - name: http
      port: 80
      targetPort: 6065

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webdav
  namespace: webdav
  annotations:
    kubernetes.io/ingress.class: nginx
    # Rewrite the WebDAV Destination header so COPY/MOVE work behind the proxy
    nginx.ingress.kubernetes.io/configuration-snippet: |
      set $dest $http_destination;
      if ($http_destination ~ "^https?://[^/]+(?<path>/.*)$") {
        set $dest $path;
      }
      proxy_set_header Destination $dest;
spec:
  rules:
    - host: webdav.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webdav
                port:
                  number: 80
  tls:
    - hosts: [webdav.example.com]
      secretName: webdav-tls
