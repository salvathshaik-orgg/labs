# Rancher Installation Guide

## Step 1: Install Cert-Manager
Cert-Manager is required for Rancher to manage TLS certificates.

```sh
wget https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.yaml
wget https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.crds.yaml
kubectl apply -f cert-manager.yaml
kubectl apply -f cert-manager.crds.yaml
```

## Step 2: Install NGINX Ingress Controller
Ensure that you have the NGINX Ingress Controller installed before proceeding with Rancher installation.

## Step 3: Install Rancher Using Helm
Add the Rancher Helm repository:

```sh
helm repo add rancher https://releases.rancher.com/server-charts/stable
helm repo update
```

Install Rancher with Helm:

```sh
helm upgrade --install rancher rancher/rancher \
  --namespace cattle-system --create-namespace \
  --set hostname=pocrancher.corp.equinix.com \
  --set ingress.tls.source=cert-manager \
  --set letsEncrypt.email=sbashashaik@equinix.com \
  --wait
```

## Step 4: Configure Rancher Ingress
Delete any existing Rancher ingress configuration and apply the following:

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

## Step 5: Access Rancher
Now access the Rancher cluster URL:

```
Rancher URL: https://hostname.example.com:31577
Rancher Password: xcfpehp5aHRR
```

## Step 6: Create a Rancher Cluster from UI
1. Go to **Clusters** → Click on **CREATE** (Not the Import option).
2. Click on **RKE1** → Select **Custom**.
3. Select **Kubernetes Version (e.g., 1.30 from RKE)**.
4. Select **Calico** as the Container Network.
5. Copy the generated Docker command and run it on the server to create and connect the RKE cluster.

### Sample Docker Command:
```sh
sudo docker run -d --privileged --restart=unless-stopped --net=host \
  -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run \
  rancher/rancher-agent:v2.10.2 --server https://hostname.example.com:31577 \
  --token xrxmbj877tkvqt7kqzp4gw7nz5pjzzfx9n5hph9gsv8 --etcd --controlplane --worker
```

Now, your Rancher cluster is successfully set up and can be managed through the Rancher UI.

