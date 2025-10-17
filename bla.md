apiVersion: v1
kind: ConfigMap
metadata:
  name: insecure-proxy-haproxy
data:
  haproxy.cfg: |
    global
      log stdout format raw local0

    defaults
      mode http
      timeout connect 5s
      timeout client  30s
      timeout server  30s

    frontend fe
      bind :8080
      default_backend be

    backend be
      # CHANGE example.com:443
      server s1 example.com:443 ssl verify none sni str(example.com)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insecure-proxy-haproxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: insecure-proxy-haproxy
  template:
    metadata:
      labels:
        app: insecure-proxy-haproxy
    spec:
      containers:
        - name: haproxy
          image: haproxy:3.2.6-alpine
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: cfg
              mountPath: /usr/local/etc/haproxy/haproxy.cfg
              subPath: haproxy.cfg
      volumes:
        - name: cfg
          configMap:
            name: insecure-proxy-haproxy
---
apiVersion: v1
kind: Service
metadata:
  name: insecure-proxy-haproxy
spec:
  selector:
    app: insecure-proxy-haproxy
  ports:
    - port: 80
      targetPort: 8080
      name: http

# patch
---
spec:
  template:
    spec:
      containers:
        - name: haproxy-sidecar
          image: haproxy:3.2.6-alpine
          ports:
            - name: proxy
              containerPort: 8080
          volumeMounts:
            - name: haproxy-config
              mountPath: /usr/local/etc/haproxy/haproxy.cfg
              subPath: haproxy.cfg
      volumes:
        - name: haproxy-config
          configMap:
            name: haproxy-sidecar-config

