### How to Create a User in k8s using certificates.k8s.io/v1 api

Required: You need to be a cluster admin to perform these operations.

### - Getting Access to the K8s Cluster.

**1. Create a cnf file as follows**


```
[ req ]
        default_bits = 2048
        prompt = no
        encrypt_key = no
        default_md = sha256
        distinguished_name = dn
        req_extensions = req_ext

[ dn ]
        CN = <USERNAME>
        emailAddress = <EMAILID>
        O = <ORG>
        OU = <TEAM>
        L = <CITY>
        ST = <STATE>
        C = <COUNTRY>

[ req_ext ]
        subjectAltName = @alt_names

[alt_names]
        DNS.1   = <ALT_NAME>
```


2. Run openssl command to generate a csr file and private key.



`openssl req -new -newkey rsa:2048 -nodes -out certificate.csr -keyout certificate.key -config config.cnf`



4. Now the cluster admins need to pick this csr file from the user and create an CertificateSigningRequest object in k8s.

  
```
            cat <<EOF | kubectl apply -f -
            apiVersion: certificates.k8s.io/v1
            kind: CertificateSigningRequest
            metadata:
                name: <USER_NAME> # As in CN name of the cnf file
            spec:
                request: $(cat certificate.csr | base64 | tr -d '\n')
                signerName: kubernetes.io/kube-apiserver-client
                expirationSeconds: 86400  # one day
                usages:
                    - client auth #Scopes only client auth with 0 perms to do stuff 
            EOF
```

5. After creating this object the cluster admins can either approve it or deny it.



`kubectl certificate approve/deny <USER_NAME>`



6. If this CSR is approved then the cluster admins can provide the final client cert to the user.



`kubectl get csr <USER_NAME> -o jsonpath='{.status.certificate}' | base64 --decode > USER_NAME.crt`



7. Now the User has his signed cert from the k8s CA and his private key with these now he can authenticate to the k8s cluster.




### - Authenticating to the K8s cluster.

1. Now that the user has his certs signed, he need to request k8s server's ca.cert which is public cert. He can get it from this command



`openssl s_client -showcerts -connect <API-server-URL>`

    

2. After having all the certs required that is USER_NAME.crt, USER_NAME.key , ca.crt. Now the user can create his KubeConfig to access the cluster.


### - Generate KubeConfig. 

There are two ways to add this info in ~/.kube/config.

a.) By using kubectl commands as follows:


```
kubectl config set-cluster <CLUSTER_NAME> --server=https://<API-server-URL>
kubectl config set-cluster <CLUSTER_NAME> --embed-certs --certificate-authority=ca.cert
kubectl config set-context <CLUSTER_NAME> --user=<USER_NAME>
kubectl config set-credentials <USER_NAME> --client-certificate=USER_NAME.cert --client-key=USER_NAME.key
kubectl config set contexts.<CLUSTER_NAME>.cluster <CLUSTER_NAME>
kubectl config view
```

b.) Or by using Yaml.

```
                    apiVersion: v1
                    clusters:
                    - cluster:
                        certificate-authority-data: /path/to/ca.crt
                        server: <API-server-URL>
                    name: <CLUSTER_NAME>
                    contexts:
                    - context:
                        cluster: <CLUSTER_NAME>
                        user: <USER_NAME>
                    name: <CLUSTER_NAME>
                    current-context: <CLUSTER_NAME>
                    kind: Config
                    preferences: {}
                    users:
                    - name: <USER_NAME>
                    user:
                        client-certificate: /path/to/<USER_NAME>.crt
                        client-key: /path/to/<USER_NAME>.key
```


### - Setting up RBAC for the user.

- Now the User has no perms to view the resources inside the cluster. He can check this by using this command

       
`kubectl auth can-i --list`

    
- Inorder to provide access to specific resources the cluster admin has to assign RBAC rules for the user.
