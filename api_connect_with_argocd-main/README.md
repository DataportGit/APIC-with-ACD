# Install API Connect with Argo CD

https://admin.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud/

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=docm-deploying-operators-in-single-namespace-api-connect-cluster

- [Install API Connect with Argo CD](#install-api-connect-with-argo-cd)
  - [Create Kubernetes Cluster in IBM Cloud](#create-kubernetes-cluster-in-ibm-cloud)
  - [Install Argo CD](#install-argo-cd)
  - [Configure Argo CD on Kubernetes IBM Cloud](#configure-argo-cd-on-kubernetes-ibm-cloud)
  - [Change the default `app.kubernetes.io/instance` label for resource tracking](#change-the-default-appkubernetesioinstance-label-for-resource-tracking)
  - [Create Argo CD ApplicationSet](#create-argo-cd-applicationset)
- [Preparation steps](#preparation-steps)
  - [Prepare yaml files](#prepare-yaml-files)
  - [Prepare certificates](#prepare-certificates)
- [Install API Connect](#install-api-connect)
  - [Install Cert Manager](#install-cert-manager)
  - [Generate custom certificates](#generate-custom-certificates)
  - [Create registry pull secret](#create-registry-pull-secret)
  - [Install ibm-apiconnect CRDs](#install-ibm-apiconnect-crds)
  - [Install the ibm-apiconnect](#install-the-ibm-apiconnect)
  - [Install Datapower](#install-datapower)
  - [Install the ingress-ca Issuer to be used by cert-manager](#install-the-ingress-ca-issuer-to-be-used-by-cert-manager)
  - [Installing the API Connect subsystems](#installing-the-api-connect-subsystems)
    - [Installing the Management subsystem cluster](#installing-the-management-subsystem-cluster)
    - [Installing the DataPower Gateway subsystem](#installing-the-datapower-gateway-subsystem)
    - [Installing the Developer Portal subsystem](#installing-the-developer-portal-subsystem)
    - [Installing the Analytics subsystem](#installing-the-analytics-subsystem)
- [Update api-connect version 10.0.7.0](#update-api-connect-version-10070)
  - [Prepare](#prepare)
  - [Upgrading API Connect on Kubernetes](#upgrading-api-connect-on-kubernetes)
- [Using ambasssador Ingress](#using-ambasssador-ingress)
- [Download api-connect online images](#download-api-connect-online-images)
- [Create mirror registry](#create-mirror-registry)
- [Set security context](#set-security-context)
- [Update api-connect to version 10.0.8.0](#update-api-connect-to-version-10080)
- [Install LA fix 10.0.8-ifix.LI83205](#install-la-fix-1008-ifixli83205)
- [Uninstalling API Connect](#uninstalling-api-connect)
  - [Uninstalling API Connect subsystems](#uninstalling-api-connect-subsystems)
  - [Removing API Connect](#removing-api-connect)


## Create Kubernetes Cluster in IBM Cloud

https://cloud.ibm.com/docs/containers?topic=containers-cluster-create-vpc-gen2

```
vpc=r010-78aff23b-852d-4c6e-bcf6-d7c3312dcf01
subnet1=02b7-d9299b76-19c1-4271-ae1e-3700a11f7254
subnet2=02c7-b36f9d68-96de-47f3-8ff3-1a8f1ab0672f
subnet3=02d7-a1c5bbc6-4133-4e49-9d3f-d4525208bff7
```

```
ibmcloud ks cluster create vpc-gen2 \
--name cluster-poc \
--zone eu-de-1 \
--vpc-id $vpc \
--subnet-id $subnet1 \
--flavor bx2.8x32 \
--workers 1
```

Get zones

```
ibmcloud ks zone ls --provider vpc-gen2
```

Get VPC id

```
ibmcloud ks vpcs
```

Get subnets

```
ibmcloud ks subnets --provider vpc-gen2 \
--vpc-id $vpc \
--zone eu-de-1
```

Get flavors

```
ibmcloud ks flavors --zone $vpc --provider vpc-gen2
```

Verify cluster

```
ibmcloud ks cluster ls
```

Check status worker nodes

```
ibmcloud ks worker ls --cluster cluster-poc
```

Optional create worker pool

```
ibmcloud ks worker-pool create vpc-gen2 \
--cluster cluster-poc \
--flavor bx2.8x32 \
--name cluster-poc-workerpool-8x32 \
--size-per-zone 3 \
--vpc-id $vpc
```

```
ibmcloud ks zone add vpc-gen2 \
--zone eu-de-1 \
--subnet-id $subnet1 \
--cluster cluster-poc \
--worker-pool cluster-poc-workerpool-8x32
```

```
ibmcloud ks worker-pool rm -c cluster-poc --worker-pool default -f
```

Add zones to worker pool (optional)

https://cloud.ibm.com/docs/containers?topic=containers-add-workers-vpc

```
ibmcloud ks worker-pool ls --cluster cluster-poc
```

```
workerpool=default
```

```
ibmcloud ks zone add vpc-gen2 \
--zone eu-de-2 \
--subnet-id $subnet2 \
--cluster cluster-poc \
--worker-pool $workerpool
```

```
ibmcloud ks zone add vpc-gen2 \
--zone eu-de-3 \
--subnet-id $subnet3 \
--cluster cluster-poc \
--worker-pool $workerpool
```

```
ibmcloud ks worker ls --cluster cluster-poc --worker-pool $workerpool
```

Get kubectl cluster config

```
ibmcloud ks cluster config -c cluster-poc
```

```
kubectl config set-context --current --namespace apic
```

Optional:

```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

## Install Argo CD

https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd

```
kubectl create namespace argocd
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Configure Ingress

https://cloud.ibm.com/docs/containers?topic=containers-managed-ingress-about

https://cloud.ibm.com/docs/containers?topic=containers-managed-ingress-setup

https://cloud.ibm.com/docs/containers?topic=containers-secrets

```
ibmcloud ks ingress secret ls -c cluster-poc

ibmcloud ks ingress status-report get -c cluster-poc

kubectl get ingressclass
```

https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-ssl-termination-at-ingress-controller

Create domain

```
ibmcloud ks ingress alb ls -c cluster-poc
```

```
ibmcloud ks ingress domain create \
--cluster cluster-poc \
--domain argocd-web \
--hostname ae90ca92-eu-de.lb.appdomain.cloud \
--secret-namespace default
```

```
ibmcloud ks ingress domain create \
--cluster cluster-poc \
--domain argocd \
--hostname ae90ca92-eu-de.lb.appdomain.cloud \
--secret-namespace argocd
```

Create ingress

```
DOMAIN=argocd-web.eu-de.containers.appdomain.cloud
SECRET=argocd-web
```

```
kubectl get secret $SECRET -n default \
-o yaml | sed \
-e "s/namespace: default/namespace: argocd/" \
-e "/resourceVersion/d" \
-e "/uid: /d" | \
kubectl apply -f -
```

Optional if using letsencrypt

```
DOMAIN=argocd.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud
SECRET=argocd-ingress-secret
```

```yaml
cat << EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-issuer
spec:
  acme:
    preferredChain: ""
    privateKeySecretRef:
      name: letsencrypt-ca
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: public-iks-k8s-nginx
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: $SECRET
spec:
    secretName: $SECRET
    dnsNames:
    - $DOMAIN
    issuerRef:
        name: letsencrypt-issuer
    usages:
    - "server auth"
    - "signing"
    - "key encipherment"
    duration: 17520h0m0s # 17520h # 2 years
    renewBefore: 720h0m0s # 720h # 30 days
    privateKey:
      rotationPolicy: Always
EOF
```

```yaml
cat << EOF | kubectl create -f -
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: public-iks-k8s-nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: http
    host: $DOMAIN
  tls:
  - hosts:
    - $DOMAIN
    secretName: $SECRET
EOF
```

Edit ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
data:
  server.insecure: "true"
```

## Configure Argo CD on Kubernetes IBM Cloud

Add to ~/.bashrc

```sh
# User specific aliases and functions
PS1='[\u@\h \w] $(basename $KUBECONFIG) \$ '
PROMPT_DIRTRIM=2

alias kub1='export KUBECONFIG=~/kub1'
alias kub2='export KUBECONFIG=~/kub2'

export KUBECONFIG=~/kub1
```

```

ibmcloud plugin install kubernetes-service

ibmcloud ks cluster ls

ibmcloud ks cluster config -c cluster

kubectl get nodes
```

```
ibmcloud login --sso
ibmcloud ks cluster config -c cluster-poc

kubectl config set-context --current --namespace apic
```

Get admin login password

```
kubectl get secret -n argocd argocd-initial-admin-secret \
-o jsonpath='{ .data.password }' | base64 -d
```

```
oc get ing -n argocd
```

Add clusters in Argo CD

https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters

```yaml
cat << EOF | kubectl create -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-manager
subjects:
- kind: ServiceAccount
  name: argocd-server
  namespace: argocd
- kind: ServiceAccount
  name: argocd-application-controller
  namespace: argocd
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-default-cluster-config
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: in-cluster
  server: https://kubernetes.default.svc
  namespaces: apic,ibm-common-services,cert-manager,kube-system
  clusterResources: "true"
  config: |
    {
      "tlsClientConfig": {
        "insecure":false
      }
    }
EOF
```

```
kubectl create -f argocd.yaml
```

```
kubectl -n argocd get secret \
-l argocd.argoproj.io/secret-type=cluster
```

```
kubectl get secret \
-n argocd \
-l argocd.argoproj.io/secret-type=cluster \
-o custom-columns=\
U0VSVkVSCg==:.data.name,Q0xVU1RFUgo=:.data.server,bmFtZXNwYWNlcw==:.data.namespaces \
| while read server cluster ns
do
  echo "$(echo $server | base64 -d) $(echo $cluster | base64 -d) $(echo $ns | base64 -d)"
done
```

Add Git Repository

https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repositories

Create github token

https://github.ibm.com/settings/tokens

```
token=...
```

```
cat << EOF | kubectl create -f - -n argocd
apiVersion: v1
kind: Secret
metadata:
  name: api-connect-repo
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.ibm.com/Martin-Caesar/install-api-connect-with-argocd.git
  username: token
  password: $token
EOF
```

Get git repositories in Argo CD

```
kubectl -n argocd get secret \
-l argocd.argoproj.io/secret-type=repository
```

## Change the default `app.kubernetes.io/instance` label for resource tracking

https://argo-cd.readthedocs.io/en/stable/user-guide/resource_tracking/

Several resources in api-connect also use the label `app.kubernetes.io/instance` that also is used by Argo CD.

Change the label used by Argo CD to `argocd.argoproj.io/instance` in the ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  application.instanceLabelKey: argocd.argoproj.io/instance
```

## Create Argo CD ApplicationSet

```
kubectl create -f appset.yaml -n argocd
```

# Preparation steps

## Prepare yaml files

Download yaml files from fixcentral

## Prepare certificates

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=kubernetes-custom-certificates-reference

# Install API Connect

## Install Cert Manager

Argo CD sync cert-manager

Optional if using kustomize:

```
kubectl apply -k cert-manager
```

Check if cert-manager has been successfully installed

https://cert-manager.io/docs/installation/kubectl/#2-optional-end-to-end-verify-the-installation

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
  selfSigned: {}
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

```
kubectl apply -f test-resources.yaml
```

Check certificate

```
kubectl get certificate -n cert-manager-test
```

Clean up the test resources

```
kubectl delete -f test-resources.yaml
```

## Generate custom certificates

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=kubernetes-generating-custom-certificates

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=kubernetes-configuring-custom-certificates-before-installation

Argo CD sync ingress-issuer

## Create registry pull secret

```
kubectl create secret docker-registry apic-registry-secret \
--docker-server=cp.icr.io \
--docker-username=cp \
--docker-password=$(cat ibm_entitlement_key.txt) \
-n apic
```

Note: Create ibm-entitlement-key in ibm_entitlement_key.txt file

Note: Get entitlement key from https://myibm.ibm.com/products-services/containerlibrary

```
kubectl create secret docker-registry apic-registry-secret \
--from-file .dockerconfigjson=auth.json \
-n apic
```

```
ibmcloud cr login

kubectl create secret docker-registry icr-registry \
--from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json
```

```
cp ${XDG_RUNTIME_DIR}/containers/auth.json ${XDG_RUNTIME_DIR}/containers/.dockerconfigjson

podman run --rm \
-v ${XDG_RUNTIME_DIR}/containers:/root/.docker \
--user 0 \
localhost/apiconnect-image-tool-10.0.7.0 upload de.icr.io/poc-apiconnect
```

## Install ibm-apiconnect CRDs

Argo CD sync ibm-apiconnect-crds

```
kubectl apply -k ibm-apiconnect-crds
```

## Install the ibm-apiconnect

Argo CD sync ibm-apiconnect

Optional if using kustomize:

```
kubectl apply -k ibm-apiconnect
```

Optional:

```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```
kubectl config set-context --current --namespace apic
```

## Install Datapower

Create secret with admin password

```
kubectl create secret generic datapower-admin-credentials \
--from-literal password=$(openssl rand -hex 16) \
-n apic
```

Get admin password

```
kubectl get secret datapower-admin-credentials \
-o jsonpath={.data.password} | base64 -d
```

Argo CD sync datapower-operator

Optional if using kustomize:

```
kubectl apply -k datapower-operator
```

## Install the ingress-ca Issuer to be used by cert-manager

Argo CD sync

Optional if using kustomize:

```
kubectl apply -k ingress-issuer
```

Verify certificates

```
kubectl get certificates -n apic
```

## Installing the API Connect subsystems

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=kubernetes-installing-api-connect-subsystems

The first time that you access the Cloud Manager user interface, you enter admin for the user name and 7iron-hide for the password.

Login to https://admin.$STACK_HOST/admin

https://admin.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud/

Get admin password

```
kubectl get secret datapower-admin-credentials \
-o jsonpath={.data.password} | base64 -d
```

### Installing the Management subsystem cluster

Argo CD sync cr-management

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=subsystems-installing-management-subsystem-cluster

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=requirements-api-connect-licenses

### Installing the DataPower Gateway subsystem

Argo CD sync cr-apigateway

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=subsystems-installing-datapower-gateway-subsystem

### Installing the Developer Portal subsystem

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=subsystems-installing-developer-portal-subsystem

Argo CD sync cr-portal

### Installing the Analytics subsystem

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=subsystems-installing-analytics-subsystem

# Update api-connect version 10.0.7.0

## Prepare

Download operator custom resource files from fix central

Copy and replace the follwing files:
- ibm-apiconnect-crds/ibm-apiconnect-crds.yaml
- ibm-apiconnect/ibm-apiconnect.yaml
- datapower-operator/ibm-datapower.yaml
- cr-management/management_cr.yaml

Note: The files apigateway_cr.yaml portal_cr.yaml cr-analytics_cr.yaml are unchanged in 10.0.7.0 version.

```
cp version_10.0.7.0/ibm-apiconnect-crds.yaml ibm-apiconnect-crds/
cp version_10.0.7.0/ibm-apiconnect.yaml ibm-apiconnect/
cp version_10.0.7.0/ibm-datapower.yaml datapower-operator/

cp version_10.0.7.0/helper_files/management_cr.yaml cr-management
cp version_10.0.7.0/helper_files/apigateway_cr.yaml cr-apigateway
cp version_10.0.7.0/helper_files/portal_cr.yaml cr-portal
cp version_10.0.7.0/helper_files/analytics_cr.yaml cr-analytics
```

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=requirements-api-connect-licenses

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=kubernetes-installing-api-connect-subsystems

Update the patch-* files with new version and license:
- ibm-apiconnect/patch-ibm-apiconnect.yaml
- datapower-operator/patch-ibm-datapower.yaml
- cr-management/patch-management_cr.yaml
- cr-apigateway/patch-apigateway_cr.yaml
- cr-portal/patch-portal_cr.yaml
- cr-analytics/patch-analytics_cr.yaml

## Upgrading API Connect on Kubernetes

https://www.ibm.com/docs/en/api-connect/10.0.x?topic=kubernetes-upgrading-api-connect

In Argo CD sync the Applications:
1) ibm-apiconnect-crds
2) ibm-datapower
3) cert-manager

Note: Clean up webhook before deploying newer API Connect opertor

```
kubectl delete mutatingwebhookconfiguration ibm-apiconnect-mutating-webhook-configuration
kubectl delete validatingwebhookconfiguration ibm-apiconnect-validating-webhook-configuration
```

4) ibm-apiconnect

Upgrade the API Connect subsystems, in Argo CD sync the Applications:
1) cr-management
2) cr-portal
3) cr-analytics
4) cr-gateway

Note: If you are upgrading from v10.0.6.0 to v10.0.7.0 and were not able to configure an S3 object-store for your management database backups, then remove the databaseBackup section in managementcluster.

https://admin.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud/

Login with user admin

Get admin password

```
kubectl get secret datapower-admin-credentials \
-o jsonpath={.data.password} | base64 -d
```

# Using ambasssador Ingress

https://www.getambassador.io/docs/emissary/latest/topics/running/host-crd

https://www.getambassador.io/docs/emissary/latest/topics/using/intro-mappings

# Download api-connect online images

```
curl -L \
https://github.com/IBM/ibm-pak/releases/download/v1.11.2/oc-ibm_pak-linux-amd64.tar.gz | \
tar zxf - -C ~/bin
mv ~/bin/oc-ibm_pak-linux-amd64 ~/bin/oc-ibm_pak
```

```
export CASE_NAME=ibm-apiconnect
export CASE_VERSION=5.1.0
export ARCH=amd64

oc ibm-pak get $CASE_NAME --version $CASE_VERSION

oc ibm-pak generate online-manifests $CASE_NAME --version $CASE_VERSION

ls ~/.ibm-pak/data/online/$CASE_NAME/$CASE_VERSION
```

```
export EDB_CASE_NAME=ibm-cloud-native-postgresql
export EDB_CASE_VERSION=4.18.0

oc ibm-pak get $EDB_CASE_NAME --version $EDB_CASE_VERSION
oc ibm-pak generate online-manifests $EDB_CASE_NAME --version $EDB_CASE_VERSION

ls ~/.ibm-pak/data/online/$EDB_CASE_NAME/$EDB_CASE_VERSION
```

# Create mirror registry

```yaml
cat << EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: airgap-registry
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: airgap-registry
  name: airgap-registry
  namespace: airgap-registry
spec:
  ports:
  - port: 5000
    protocol: TCP
    targetPort: 5000
  selector:
    app: airgap-registry
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airgap-registry-storage
  namespace: airgap-registry
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  volumeMode: Filesystem
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: airgap-registry
  name: airgap-registry
  namespace: airgap-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: airgap-registry
  template:
    metadata:
      labels:
        app: airgap-registry
    spec:
      containers:
      - env:
        - name: REGISTRY_AUTH
          value: htpasswd
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: Image Private Registry
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: /auth/htpasswd
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: /certs/tls.crt
        - name: REGISTRY_HTTP_TLS_KEY
          value: /certs/tls.key
        image: registry:2
        imagePullPolicy: IfNotPresent
        name: airgap-registry
        ports:
        - containerPort: 5000
        volumeMounts:
        - mountPath: /var/lib/registry
          name: registry-vol
        - mountPath: /certs
          name: tls-vol
          readOnly: true
        - mountPath: /auth
          name: auth-vol
          readOnly: true
      volumes:
      - name: registry-vol
        persistentVolumeClaim:
          claimName: airgap-registry-storage
      - name: tls-vol
        secret:
          secretName: airgap-registry-certificate
      - name: auth-vol
        secret:
          secretName: airgap-registry-auth
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: airgap-registry-ca-issuer
  namespace: airgap-registry
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: airgap-registry-issuer
  namespace: airgap-registry
spec:
  ca:
    secretName: airgap-registry-ca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: airgap-registry-ca-certificate
  namespace: airgap-registry
spec:
  commonName: ca.airgap-registry
  duration: 175200h0m0s
  isCA: true
  issuerRef:
    kind: Issuer
    name: airgap-registry-ca-issuer
  keyAlgorithm: rsa
  keyEncoding: pkcs8
  keySize: 4096
  organization:
  - IBM
  renewBefore: 2160h0m0s
  secretName: airgap-registry-ca
  subject:
    countries:
    - DE
    localities:
    - Berlin
    organizationalUnits:
    - IBM
    streetAddresses:
    - Berlin
  usages:
  - cert sign
  - digital signature
  - key encipherment
  - server auth
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: airgap-registry-certificate
  namespace: airgap-registry
spec:
  commonName: airgap-registry
  dnsNames:
  - airgap-registry.airgap-registry.svc
  duration: 175200h0m0s
  issuerRef:
    kind: Issuer
    name: airgap-registry-issuer
  organization:
  - IBM
  renewBefore: 2160h0m0s
  secretName: airgap-registry-certificate
  subject:
    countries:
    - DE
    localities:
    - Berlin
    organizationalUnits:
    - IBM
    streetAddresses:
    - Berlin
  usages:
  - cert sign
  - digital signature
  - key encipherment
  - server auth
EOF
```

Create secret with Registry password

```
REGISTRY_PASSWORD=$(openssl rand -hex 8)
REGISTRY_HTPASSWD=$(htpasswd -Bbn admin $REGISTRY_PASSWORD)
```

```
kubectl create secret generic airgap-registry-auth -n airgap-registry \
--from-literal password="$REGISTRY_PASSWORD" \
--from-literal htpasswd="$REGISTRY_HTPASSWD"
```

Get Registry password from secret

```
REGISTRY_PASSWORD=$(kubectl get secret -n airgap-registry airgap-registry-auth \
-o jsonpath={.data.password} | base64 -d)
```

Create ingress

```
DOMAIN=image-registry.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud
SECRET=airgap-registry
```

```yaml
cat << EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-issuer
  namespace: airgap-registry
spec:
  acme:
    preferredChain: ""
    privateKeySecretRef:
      name: letsencrypt-ca
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: public-iks-k8s-nginx
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: airgap-registry
  namespace: airgap-registry
spec:
    secretName: $SECRET
    dnsNames:
    - $DOMAIN
    issuerRef:
        name: letsencrypt-issuer
    usages:
    - "server auth"
    - "signing"
    - "key encipherment"
    duration: 17520h0m0s # 17520h # 2 years
    renewBefore: 720h0m0s # 720h # 30 days
    privateKey:
      rotationPolicy: Always
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airgap-registry
  namespace: airgap-registry
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: public-iks-k8s-nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: airgap-registry
            port:
              number: 5000
    host: $DOMAIN
  tls:
  - hosts:
    - $DOMAIN
    secretName: $SECRET
EOF
```

```
REGISTRY_PASSWORD=$(kubectl get secret -n airgap-registry airgap-registry-auth \
-o jsonpath={.data.password} | base64 -d)
```

```
DOMAIN=image-registry.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud
```

```
podman login -u admin -p "$REGISTRY_PASSWORD" $DOMAIN --authfile auth.json
```

Note: Add airgap-registry.airgap-registry.svc to auth.json

```
kubectl create secret generic auth-json -n airgap-registry --from-file auth.json
```

```
kubectl create deploy skopeo -n airgap-registry \
--image registry.access.redhat.com/ubi9/skopeo \
-- tail -f /dev/null
```

```json
kubectl patch deploy skopeo -n airgap-registry \
-p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "skopeo",
            "image": "registry.access.redhat.com/ubi9/skopeo",
            "volumeMounts": [
              {
                "mountPath": "/auth",
                "name": "auth-vol"
              }
            ]
          }
        ],
        "volumes": [
          {
            "name": "auth-vol",
            "secret": {
              "secretName": "auth-json"
            }
          }
        ]
      }
    }
  }
}'
```

```
cat api-img-10.0.7.0 | while read i
do
echo $i
kubectl exec -n airgap-registry deploy/skopeo -- skopeo copy \
--authfile /auth/auth.json \
docker://$i \
docker://airgap-registry.airgap-registry.svc:5000/$(basename $i) \
--dest-tls-verify=false
done
```

```
DOMAIN=image-registry.cluster-poc-7be4149f30480846757a2468f333dd09-0000.eu-de.containers.appdomain.cloud
```

```
i=icr.io/cpopen/ibm-cpd-cloud-native-postgresql-operator:1.18.7-amd64
j=ibm-apiconnect-management-edb-operator:1.18.7-ubi8-20231031

kubectl exec -n airgap-registry deploy/skopeo -- skopeo copy \
--authfile /auth/auth.json \
docker://$i \
docker://airgap-registry.airgap-registry.svc:5000/$j \
--dest-tls-verify=false

skopeo inspect --authfile auth.json \
--format '{{.Name}}@{{.Digest}}' \
docker://$DOMAIN/$j
```

```
i=icr.io/cpopen/ibm-cpd-cloud-native-postgresql-operator:1.18.7-amd64

skopeo inspect \
--format '{{.Name}}@{{.Digest}}' \
docker://$i
```

```
i=icr.io/cpopen/edb/postgresql:15.4-4.18.0v2-amd64

skopeo inspect \
--format '{{.Name}}@{{.Digest}}' \
docker://$i
```

```
i=icr.io/cpopen/edb/postgresql:15.4-4.18.0v2-amd64
j=ibm-apiconnect-management-edb-postgresql:15.4-4.18.0v2-20231105144748

kubectl exec -n airgap-registry deploy/skopeo -- skopeo copy \
--dest-authfile /auth/auth.json \
docker://$i \
docker://airgap-registry.airgap-registry.svc:5000/$j \
--dest-tls-verify=false

skopeo inspect --authfile auth.json \
--format '{{.Name}}@{{.Digest}}' \
docker://$DOMAIN/$j
```

Source

```
icr.io/cpopen/ibm-cpd-cloud-native-postgresql-operator:1.18.7-amd64
icr.io/cpopen/edb/postgresql:15.4-4.18.0v2-amd64
```

Destination

```
ibm-apiconnect-management-edb-operator:1.18.7-ubi8-20231031
ibm-apiconnect-management-edb-postgresql:15.4-4.18.0v2-20231105144748
```

# Set security context

https://kubernetes.io/docs/concepts/security/pod-security-admission/

Privileged namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-privileged-namespace
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
```

Baseline namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest
```

Restricted namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-restricted-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/

```
kubectl label --overwrite ns apic \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/enforce-version=latest
```

```
kubectl label --overwrite ns apic \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

# Update api-connect to version 10.0.8.0

Download images https://www.ibm.com/support/fixcentral/swg/selectFixes?parent=ibm%7EWebSphere&product=ibm/WebSphere/IBM+API+Connect&release=10.0.7.0&platform=Linux&function=all&source=fc

- apiconnect-operator-release-files_10.0.8.0
- apiconnect-image-tool_10.0.8.0

Load images

https://www.ibm.com/docs/en/api-connect/10.0.8?topic=procedures-obtaining-product-files

```
podman load < apiconnect-image-tool_10.0.8.0.tar.gz
```

```
podman run --rm localhost/apiconnect-image-tool-10.0.8.0 version --images
```

```
podman run --rm localhost/apiconnect-image-tool-10.0.8.0 version --images | \
awk '{ print $2 }' | awk -F: '{ print $1 }'> api-img-10.0.8.0
```

```
cat api-img-10.0.8.0 | while read i
do
j=cp.icr.io/cp/apic/$i:10.0.8.0
skopeo inspect --authfile auth.json \
--format '{{.Name}}@{{.Digest}}' \
docker://$j
done
```

```
icr.io/cpopen/datapower-operator
icr.io/cpopen/datapower-operator-conversion-webhook

# ibm-apiconnect-management-edb-operator
icr.io/cpopen/

#ibm-apiconnect-management-edb-postgresq
icr.io/cpopen/edb/postgresql

icr.io/cpopen/ibm-apiconnect-operator
```

https://www.ibm.com/docs/en/api-connect/10.0.8?topic=installation-installing-bastion-host

```
export CASE_NAME=ibm-apiconnect
export CASE_VERSION=5.2.0
export ARCH=amd64

oc ibm-pak get $CASE_NAME --version $CASE_VERSION

oc ibm-pak generate online-manifests $CASE_NAME --version $CASE_VERSION

ls ~/.ibm-pak/data/online/$CASE_NAME/$CASE_VERSION
```

```
export EDB_CASE_NAME=ibm-cloud-native-postgresql
export EDB_CASE_VERSION=5.0.0

oc ibm-pak get $EDB_CASE_NAME --version $EDB_CASE_VERSION
oc ibm-pak generate online-manifests $EDB_CASE_NAME --version $EDB_CASE_VERSION

ls ~/.ibm-pak/data/online/$EDB_CASE_NAME/$EDB_CASE_VERSION
```

```
skopeo list-tags docker://icr.io/cpopen/ibm-cpd-cloud-native-postgresql-operator | \
grep 1.22.3

skopeo list-tags docker://icr.io/cpopen/edb/postgresql | grep 15.6-5
```

Source

```
icr.io/cpopen/ibm-cpd-cloud-native-postgresql-operator:1.22.3-amd64
icr.io/cpopen/edb/postgresql:15.6-5.0.0-amd64
```

Destination

```
ibm-apiconnect-management-edb-operator:1.22.3-ubi9
ibm-apiconnect-management-edb-postgresql:15.6-5.0.0-ubi9
```

```
i=ibm-cpd-cloud-native-postgresql-operator:1.22.3-amd64
j=ibm-apiconnect-management-edb-operator:15.6-5.0.0-ubi9

kubectl exec -n airgap-registry deploy/skopeo -- skopeo copy \
--authfile /auth/auth.json \
docker://airgap-registry.airgap-registry.svc:5000/$i \
docker://airgap-registry.airgap-registry.svc:5000/$j \
--src-tls-verify=false --dest-tls-verify=false
```

```
i=postgresql:15.6-5.0.0-amd64
j=ibm-apiconnect-management-edb-postgresql:15.6-5.0.0-ubi9

kubectl exec -n airgap-registry deploy/skopeo -- skopeo copy \
--authfile /auth/auth.json \
docker://airgap-registry.airgap-registry.svc:5000/$i \
docker://airgap-registry.airgap-registry.svc:5000/$j \
--src-tls-verify=false --dest-tls-verify=false
```

# Install LA fix 10.0.8-ifix.LI83205

Download tar file from fixcentral

https://ibm.biz/BdKmzW

Upload image to repository

```
podman login quay.io -u mcaesar
```

```
skopeo copy \
dir:10.0.8-ifix.83205/laifix-images/ui/\
10.0.8-ifix.LI83205-7-89e551135ae7bd76a1b5f980141688466beb09ae/ \
docker://quay.io/mcaesar/cp.icr.io/cp/apic/\
ibm-apiconnect-management-ui:10.0.8-ifix.83205
```

Edit your APIC subsystem CR

```
spec:
...
  template:
  - name: ui
    containers:
    - name: ui
      image: ibm-apiconnect-management-ui:10.0.8-ifix.83205
```

# Uninstalling API Connect

## Uninstalling API Connect subsystems

https://www.ibm.com/docs/en/api-connect/10.0.8?topic=connect-uninstalling-api-subsystems

Uninstall Management

```
kubectl delete mgmt --all -n <namespace>
```

Uninstall Analytics

```
kubectl delete a7s --all -n <namespace>
kubectl delete a7b --all -n <namespace>
kubectl delete a7r --all -n <namespace> 
```

Uninstall Portal

```
kubectl delete ptl --all -n <namespace>
kubectl delete pb --all -n <namespace>
kubectl delete pr --all -n <namespace>
```

Uninstall Gateway

```
kubectl delete gw --all -n <namespace>
```

## Removing API Connect

https://www.ibm.com/docs/en/api-connect/10.0.8?topic=connect-removing-api

```
kubectl delete apic --all -n <namespace>
```

Delete Certificates and Secrets

```
kubectl delete certificates,secrets --all -n <namespace>
```
