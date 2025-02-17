# Rancher Installation Guide

## Prerequisites
1. Install **cert-manager** (Required for Rancher TLS certificates)
   ```sh
   wget https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.yaml
   wget https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.crds.yaml
   
kubectl apply -f cert-manager.yaml
kubectl apply -f cert-manager.crds.yaml
   ```

2. Ensure that **NGINX Ingress Controller** is installed in your cluster.

---

## Install Rancher using Helm

1. Add the Rancher Helm repository:
   ```sh
   helm repo add rancher https://releases.rancher.com/server-charts/stable
   ```

2. Install Rancher with a specific hostname:
   ```sh
   helm upgrade --install rancher rancher/rancher \
     --namespace cattle-system --create-namespace \
     --set hostname=pocrancher.corp.equinix.com \
     --set ingress.tls.source=cert-manager \
     --set letsEncrypt.email=sbashashaik@equinix.com \
     --wait
   ```
   > Note: You can modify the hostname as needed. Later, you can update the Ingress file to allow traffic from any domain.

---

## Configure Rancher Ingress for Any Host Access

1. **Delete existing Rancher Ingress**
   ```sh
   kubectl delete ingress rancher -n cattle-system
   ```

2. **Apply the updated Ingress configuration:**
   
   Create a file named `rancher-ingress.yml` and add the following content:
   
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: rancher-ingress
     namespace: cattle-system
     annotations:
       kubernetes.io/ingress.class: "nginx"
       nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
       nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
       nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
   spec:
     ingressClassName: nginx
     tls:
     - hosts:
         - amp.corp.equinix.com
         - sv2lxelaappr023.corp.equinix.com
         - sv2lxelaappr023
       secretName: tls-rancher-ingress
     rules:
     - http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: rancher
               port:
                 number: 80
   ```

3. **Apply the new Ingress configuration:**
   ```sh
   kubectl apply -f rancher-ingress.yml
   ```

---

## Access Rancher
Once the installation is complete, access Rancher using any of the specified hostnames with the **HTTPS** port.

```sh
https://<your-hostname>
```

Replace `<your-hostname>` with any of the allowed domains.

---

## Troubleshooting
- Verify if the Rancher pods are running:
  ```sh
  kubectl get pods -n cattle-system
  ```
- Check Ingress resources:
  ```sh
  kubectl get ingress -n cattle-system
  ```
- Describe Ingress for debugging:
  ```sh
  kubectl describe ingress rancher -n cattle-system
  ```

