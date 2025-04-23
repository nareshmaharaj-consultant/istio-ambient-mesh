# Setting Up Kubernetes with Istio 1.25 on Windows Using Vagrant

This guide walks through setting up a Kubernetes cluster on Windows using Vagrant, VirtualBox, and Cygwin, with Istio 1.25 deployed in Ambient Mesh mode.
It will then use Istio's bookinfo example application and an Ingress Gateway to route
external client traffic into the Kubernetes cluster and backend pods.
The connection will take advantage of securing https traffic using Cert-Manager and self created 
certificates for the demo.

## Table of Contents
1. [Prerequisites](#10-preregs)
2. [Build Vagrant Hosts](#2-build-vagrant-hosts)
3. [Install Kubernetes](#3-install-kubernetes)
4. [Install Istio](#4-install-istio)
5. [Istio Sample Application](#5-istio-sample-application)
6. [Visualize with Kiali](#6-visualise-with-kiali)
7. [Cert Manager](#7-cert-manager)
8. [Enable mTLS Secure Mesh](#5-enable-mtlssecure-mesh)
9. [Appendix](#appendix-1)

---

## 1.0 Preregs

Before you begin, ensure you have the following installed and configured.

### 1.1 Install Vagrant
To begin, install Vagrant, a tool that helps you build and manage virtual machine environments in a simple and efficient way.
```bash
# Install from official site
https://www.vagrantup.com/downloads
```

### 1.2 Install VirtualBox
Next, you’ll need VirtualBox, which Vagrant uses to manage your virtual machines.
```bash
# Download from VirtualBox official site
https://www.virtualbox.org/wiki/Downloads
```

### 1.3 Install Cygwin
Cygwin provides a Linux-like environment on Windows. It's needed for running bash scripts that may not be natively supported in Windows.
```bash
# Follow installation instructions from Cygwin official site
https://www.cygwin.com/   
```

### 1.4 dos2unix
Converts Windows line endings to Unix format:
```bash
choco install dos2unix
dos2unix _start.sh
```

**Important**: Ensure that everything is working as expected before moving on to the next steps. 
Check that your installation is running without errors.

---

## 2. Build Vagrant Hosts

Now that you’ve installed all necessary software, it's time to set up the virtual machines for your Kubernetes cluster.

### 2.1 Checkout Vagrant Scripts
```bash
git clone https://github.com/nareshmaharaj-consultant/vagrant_for_kubernetes.git
```

### 2.2 Build VMs
Configure the allowed address space for Virtual Box. Only required once during set up.
```bash
mkdir /etc/box
# Add to /etc/vbox/networks.conf:
10.0.0.0/8 192.168.0.0/16  
2001::/64
```

Now you’re ready to build the VMs. Run the following script to start things off:
```bash
./_start.sh
```

Verify all the VMs are running:
```bash
vagrant status
```
You should see something like 
```text
Current machine states:
master                    running (virtualbox)
node01                    running (virtualbox)
node02                    running (virtualbox)
node03                    running (virtualbox)
```

---

## 3. Install Kubernetes

**Setting Up Kubernetes on Your Virtual Machines**

Now that your VMs are up and running, the next step is to deploy Kubernetes on them.

**Important Note:** We cannot use container-based Kubernetes distributions like Kind, K3s, or similar solutions. The Istio ambient mesh requires low-level networking modifications that are not supported in these environments.

Let’s proceed with a standard Kubernetes installation for your VMs ( _the hard way_ ).

### 3.1 Log into Each VM

In a new Cygwin shell window open an ssh connection to each Virtual Machine as follows.

```bash
vagrant ssh master
vagrant ssh node01
vagrant ssh node02
vagrant ssh node03
```

### 3.2 Run Setup Script
Inside each VM, run the setup script that installs and configures Kubernetes.
Use dos2unix to convert the file if you encounter any interpretation errors.
```bash
./setup-k8s.sh
```

Once the master node setup completes, it will provide you with the necessary commands to join the Kubernetes cluster.
You’ll need to run these join commands on each worker node to connect them to the cluster. 
The command will look something like this:
```text
sudo kubeadm join 10.0.0.10:6443 --token 14wpdz.sxivz9sp49t56a56 --discovery-token-ca-cert-hash sha256:249a10a62d32696090a6e8a0cb237423dfdff3a0fc35341eb69e0aa298321014
````

```bash
kubectl get nodes -w
```

---

## 4 Install Istio
If you want to use a Secure `https` connection for the external client install the cert-manager first in section 7 below.
But we recommend doing this mich later once you know you have a fully working application.

Install Istio using Helm and in Ambient Mesh mode.

**_Note_**: All the following commands are run from the master node, i.e. the kubernetes control plane.

### 4.1 Install Helm
```bash
curl -fssL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### 4.2 Add Repo to Helm
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### 4.3 Base components
```bash
helm install istio-base istio/base -n istio-system --create-namespace --wait
```

### 4.4 Kubernetes Gateway API

Using Kubernetes Gateway API Instead of Istio Ingress Gateway

We are opting for the Kubernetes Gateway API rather than the traditional Istio Ingress Gateway.

Why?
The Gateway API is the modern, standardized approach for managing traffic in Kubernetes and is designed to eventually replace the need for Istio's built-in Ingress and Egress Gateways.
```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

### 4.5 Istiod Control Plane
```bash
helm install istiod istio/istiod --namespace istio-system --set profile=ambient --wait
```

### 4.6 CNI Node Agent
```bash
helm install istio-cni istio/cni -n istio-system --set profile=ambient --wait
```

### 4.7 Ztunnel
```bash
helm install ztunnel istio/ztunnel -n istio-system --wait
```

### 4.8 Verify Pods Running
```bash
kubectl get pods -n istio-system

NAME                      READY   STATUS    RESTARTS   AGE
istio-cni-node-2sf68      1/1     Running   0          2m55s
istio-cni-node-484wj      1/1     Running   0          2m55s
istio-cni-node-czbsf      1/1     Running   0          2m55s
istio-cni-node-mzxd7      1/1     Running   0          2m55s
istiod-79f49d85c5-sqgzf   1/1     Running   0          3m19s
ztunnel-gw8mf             1/1     Running   0          73s
ztunnel-h79pj             1/1     Running   0          73s
ztunnel-kv4px             1/1     Running   0          73s
ztunnel-whv9g             1/1     Running   0          73s
```

---

## 5 Istio Sample Application

### 5.1 Download the Sample App

Use the following to download the sample application and istio tools.

```bash
curl -L https://istio.io/downloadIstio | sh -cd istio-1.25.2
export PATH=$PWD/bin:$PATH
```

### 5.2 Install the App

![img.png](img.png)

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo-versions.yaml
```

### 5.3 Verify

Make sure all the pods are running:
```bash
kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-649d7678b5-4sxd6       1/1     Running   0          45s
productpage-v1-5c5fb9b4b4-vnzzk   1/1     Running   0          45s
ratings-v1-794db9df8f-vl6p5       1/1     Running   0          45s
reviews-v1-7f9f5df695-jphnc       1/1     Running   0          45s
reviews-v2-65c9797659-mwzlk       1/1     Running   0          45s
reviews-v3-84b8cc6647-cmstq       1/1     Running   0          45s
````

### 5.4 Kubernetes Gateway API ( _not_ Istio )
```bash
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

### 5.5 Remove Default Load Balancer
For non cloud environments remove the default load balancer 
that gets automatically created.

```bash
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default
```

### 5.6 Verify running

```bash
kubectl get gateway
NAME               CLASS   ADDRESS                                            PROGRAMMED   AGE
bookinfo-gateway   istio   bookinfo-gateway-istio.default.svc.cluster.local   True         79s
```

### 5.7 Access the App
From a new shell window run the following on the master node.

```bash
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
```
**_Note:_**  you cannot view the running app from the Vagrant host at http://localhost:8080/productpage without first configuring kubernetes port forwarding and Vagrant VM host traffic forwarding.

Here's a clearer and more structured rewrite of your instructions:

---

### 5.8 Setting Up Port Forwarding for bookinfo-gateway-istio from Vagrant Host

1. **Get the Master Node IP Address**  
   On the master node, run this command to find the 192.x.y.z address:
   ```bash
   ip a | grep 192 | grep eth1 | awk '{print $2}' | cut -f1 -d'/'
   ```
   Example output:  
   `inet 192.168.0.131/24 metric 100 brd 192.168.0.255 scope global dynamic eth1`


2. **Set Up SSH Port Forwarding**  
   From a new Windows terminal, establish an SSH tunnel using the IP address found above (password: `vagrant`):
   ```bash
   ssh -L 8080:localhost:8080 vagrant@192.168.0.131
   ```

3. **Access the Application**  
   After setting up the tunnel, open your web browser and visit:  
   [http://localhost:8080/productpage](http://localhost:8080/productpage)

![img_1.png](img_1.png)

---


## 6 Visualise with Kiali

### 6.1 Install Kiali
```bash
kubectl apply -f samples/addons
```

### 6.2 Start Kiali
```bash
istioctl dashboard kiali
```

### 6.3 Port Forwarding for Kiali
```bash
ssh -L 20001:localhost:20001 vagrant@192.168.0.131
```
Access via: http://localhost:20001/kiali

### 6.4 Client Web Requests
```bash
for i in $(seq 1 100); do curl -sSI -o /dev/null http://localhost:8080/productpage; sleep 1; done
```

### 6.5.2 With Ambient Mesh Enabled
```bash
kubectl label namespace default istio.io/dataplane-mode=ambient
```

---

## 7 Cert Manager

### 7.1 Install Cert Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.yaml
kubectl get pods --namespace cert-manager
```

### 7.2 Test Cert Manager
```yaml
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfsigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

### 7.3 Secure https

#### 7.3.1 Create Issuer and Certificate
```yaml
cat <<EOF> cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfsigned: {}
EOF

cat <<EOF> tls-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: bookinfo-gateway-cert
spec:
  commonName: bookinfo.example.com
  dnsNames:
    - bookinfo.example.com
  secretName: bookinfo-gateway-tls
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
EOF
```

#### 7.3.4 Edit the Gateway
Modified `bookinfo-gateway.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  gatewayClassName: istio
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: bookinfo-gateway-tls
  allowedRoutes:
    namespaces:
      from: Same
```

#### 7.3.8 Test Connection
```bash
curl -v --cacert my-cert.crt --resolve bookinfo.example.com:8443:127.0.0.1 https://bookinfo.example.com:8443
```

---

## 5 Enable mtls secure Mesh

### Appendix 1: API Gateway Example

#### GatewayClass
```yaml
apiVersion: gateway.networking.k8s.io/v1  
kind: GatewayClass  
metadata:  
  name: istio  
spec:  
  controllerName: istio.io/gateway-controller
```

#### Gateway
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "example.com"
```

#### HTTPRoute
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: my-service
      port: 80
```

---

## Key Differences: Old vs Gateway API

| Feature          | Old Istio CRDs                     | Gateway API                     |
|------------------|------------------------------------|---------------------------------|
| Ingress Gateway  | Created via Helm/istioctl          | Configured via Gateway resource |
| Gateway Config   | networking.istio.io/v1beta1        | gateway.networking.k8s.io/v1    |
| Route Definition | VirtualService                     | HTTPRoute/TCPRoute              |

The Gateway API provides a more standardized approach to configuring ingress in Kubernetes, while Istio implements the control plane that enacts these configurations.

---

## Appendix 2: Alternative Kubernetes Installations

### Kind Configuration
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/vialpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

### k3d Installation
```bash
k3d cluster create --api-port 6550 -p '9080:80@loadbalancer' -p '9443:443@loadbalancer' --agents 2 --k3s-arg '--disable=traefik@server:*'
```

---

## Appendix 3: Network Tools

Debug with netshoot:
```bash
kubectl debug -it details-v1-7c5d957895-7wb6d --image=nicolaka/netshoot --image-pull-policy=Always --profile=general
```

Packet capture:
```bash
POD=$(kubectl get pods -l app=details -o jsonpath="(.items[0].metadata.name)")
kubectl debug $POD -i --image=nicolaka/netshoot -- tcpdump -nAi eth0 port 9080 or port 15008
```

---

## Links
- [K3d Documentation](https://k3d.io)
- [Istio Official Site](https://istio.io)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io)
