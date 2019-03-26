# Self Signed Certificate and Key to try out TLS Ingress

Self-signed certificate and key has been provided for someone wanting to quickly try out TLS also

### One-Liner to create a Self-Signed certificate and key
```
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=IN/ST=Karnataka/L=Bangalore/O=Citrix/CN=citrix-ingress-gke.com" -keyout citrix-ingress-gke.key -out citrix-ingress-gke.crt
```

### Create a Kubernetes secret using the cert and key
```
kubectl  create secret tls apache --cert citrix-ingress-gke.crt --key citrix-ingress-gke.key
```

### TLS Ingress Example

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cpx-ingress-tls
  annotations:
   kubernetes.io/ingress.class: "citrix-adc-cpx"
spec:
  tls:
  - secretName: apache
  rules:
  - host: citrix-ingress-tls-gke.com
    http:
      paths:
      - path: /
        backend:
          serviceName: apache
          servicePort: 80
```
