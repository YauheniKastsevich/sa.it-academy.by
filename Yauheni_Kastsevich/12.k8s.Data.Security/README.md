#
# 12. Kubernetes. Data. Security
#

## Homework Assignment 1. Config maps and secrets

### Add index.php page as config map, which should display hostname of pod as first level header

``` yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  defaul.conf: |
    server {
      listen 80;
      listen [::]:80;
      access_log off;
      root /var/www/html;
      index index.php;
      server_name example.com;
      server_tokens off;
      location / {
        # First attempt to serve request as file, then
        # as directory, then fall back to displaying a 404.
        try_files $uri $uri/ /index.php?$args;
      }
      # pass the PHP scripts to FastCGI server listening on wordpress:9000
      location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;

        # Change The Service Name
        fastcgi_pass phpfpm:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param SCRIPT_NAME $fastcgi_script_name;
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: index-php
data:
  index.php: |
    <!DOCTYPE html>
    <html>
      <head>
          <title>PHP Test</title>
      </head>
      <body>
          <?php echo 'My podname is ' .$_ENV["POD_NAME"] . '!'; ?>
      </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-nginx
  labels:
    app: nginx
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: nginx-config-configmap
          mountPath: /etc/nginx/conf.d/
        - name: index-php-configmap
          mountPath: /var/www/html/
        - name: ssh-secret
          mountPath: /root/.ssh/
          readOnly: true
      volumes:
      - name: nginx-config-configmap
        configMap:
          name: nginx-config
      - name: index-php-configmap
        configMap:
          name: index-php
      - name: ssh-secret
        secret:
          secretName: sshsecret
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpfpm
  labels:
    app: phpfpm
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 2
  selector:
    matchLabels:
      app: phpfpm
  template:
    metadata:
      labels:
        app: phpfpm
    spec:
      containers:
        - name: phpfpm
          image: php:fpm-alpine
          env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          ports:
            - containerPort: 9000
          volumeMounts:
            - name: index-php-configmap
              mountPath: /var/www/html/
      volumes:
      - name: index-php-configmap
        configMap:
          name: index-php
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - nodePort: 30000
    port: 80
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: phpfpm
  labels:
    app: phpfpm
spec:
  type: ClusterIP
  selector:
    app: phpfpm
  ports:
    - port: 9000
      targetPort: 9000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-sa-nginx
  annotations:
    nginx.ingress.kubernetes.io/server-alias: "nginx-test.k8s-26.sa"
spec:
  ingressClassName: nginx
  rules:
    - host: nginx-test.k8s-25.sa
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

### use existing pair public and private keys
```bash
kubectl create secret generic sshsecret --from-file=ssh-privatkey=.ssh/id_rsa --from-file=ssh-publickey=.ssh/id_rsa.pub
```

### screens

![pod1](https://github.com/YauheniKastsevich/my_pictures/blob/master/s1.png)
![pod2](https://github.com/YauheniKastsevich/my_pictures/blob/master/s2.png)