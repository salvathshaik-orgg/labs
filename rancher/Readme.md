# Rancher Installation Guide

## Step 1: Install Cert-Manager

Cert-Manager is required to manage SSL/TLS certificates for Rancher.

1. Download the Cert-Manager manifests:
   ```sh
   wget https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.yaml
   wget https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.crds.yaml
   ```
2. Apply the manifests to the cluster:
   ```sh
   kubectl apply -f cert-manager.yaml
   kubectl apply -f cert-manager.crds.yaml
   ```

## Step 2: Install the Nginx Ingress Controller

Ensure that the Nginx Ingress Controller is installed before proceeding with Rancher installation.
```sh
wget https://github.com/salwad-basha-shaik/observability-tools-k8-configs/raw/refs/heads/main/grafana/ingress-controller-nginx-new-version-second.yaml
```

## Step 3: Configure SSL Certificates

## Before proceeding for below step please create self-signed certificates for your server/hostname. Follow below document to create certificates.

```sh
https://github.com/salvathshaik-orgg/kubernetes/blob/master/scripts/self-signed-certificate-openssl.md
```

### 1. Get SSL Certificates and Validate
Obtain the certificates with new server names and validate them:
```sh
openssl x509 -in qaamp.crt -text | grep <server-name>
openssl x509 -text -inform DER -in certificate.crt | grep  <server-name>
```

#### Example Certificate Format
**For Certificate (.crt file):**
```
-----BEGIN CERTIFICATE-----
(base64-encoded certificate data)
-----END CERTIFICATE-----
```
**For Key (.key file):**
```
-----BEGIN RSA PRIVATE KEY-----
(base64-encoded certificate data)
-----END RSA PRIVATE KEY-----
```

### 2. Create a Kubernetes Secret for SSL Certificates
```sh
kubectl create secret tls qa-secret-tls --cert=/apps/cert/qa.crt --key=/apps/cert/key-decrypted.key -n ingress-nginx-second
```

### 3. Modify the Ingress Controller Deployment
Update the following file to set the default SSL certificate:

**File Path:** `/apps/observability_tools/main/grafana/ingress-nginx-controller-new-deployment.yml`

**Modify the following field:**
```yaml
spec:
  containers:
    - args:
        - /nginx-ingress-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --default-ssl-certificate=$(POD_NAMESPACE)/qa-secret-tls
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        - --enable-metrics=false
```

## Step 4: Install Rancher using Helm

Add the Rancher Helm repository:
```sh
helm repo add rancher https://releases.rancher.com/server-charts/stable
```

Install Rancher with the following command:
```sh
helm upgrade --install rancher rancher/rancher \
  --namespace cattle-system --create-namespace \
  --set hostname=pocrancher.example.com \
  --set ingress.tls.source=cert-manager \
  --set cert-manager.email=salwad@example.com \
  --wait
```

## Step 5: Configure Rancher Ingress

Replace the existing ingress file with the following configuration to allow Rancher access from any host:

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
      - pocrancher.example.com
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

## Step 6: Access Rancher

Now you can access the Rancher UI using the following details:

- **Rancher URL:** [https://hostname.example.com:31577](https://hostname.example.com:31577)
- **Rancher Password:** `xcfpehp5aHRR`

## Step 7: Create a Rancher Cluster from UI

1. Go to **Clusters** > Click on **CREATE** (not importing option).
2. Select **RKE1**.
3. Choose **Custom**.
4. Select **Kubernetes Version** (e.g., 1.30 from RKE).
5. Choose **Calico** as the container network.
6. Copy the generated Docker command and run it on the server to add it to the Rancher cluster.

### Sample Docker Command:
```sh
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run \
  rancher/rancher-agent:v2.10.2 --server https://hostname.example.com:31577 \
  --token xrxmbj877tkvqt7kqzp4gw7nz5pjzzfx9n5hph9gsv8 --etcd --controlplane --worker
```

Now, Rancher will create an RKE cluster, and you can manage and monitor the cluster from the UI. 🎉

