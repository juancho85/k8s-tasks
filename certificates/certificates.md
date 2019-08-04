# Certificates

References:
* [Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md)
* [Kubernetes documentation](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
* [Cloudflase cfssl nd cfssljson](https://github.com/cloudflare/cfssl)
* [CoreOS documentation](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)

## Prerequisites

Install cfssl and cfssljson:

```
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64

sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

```

Verification:

```
cfssl version

```

Output:

```
Version: 1.2.0
Revision: dev
Runtime: go1.6
```

## Provision a Certificate Authority
This CA can be used to generate additional TLS certificates.

The commands below generate a self-signed root CA certificate.

The request json (ca-csr.json) follows the same format as 'genkey'.

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Generates CA certificate and private key (plus a csr):
```
ca.pem
ca-key.pem
```

An easy way of starting the json templates is to type:

```
cfssl print-defaults config > ca-config.json
cfssl print-defaults csr > ca-csr.json
```
It gives:
```

{
    "signing": {
        "default": {
            "expiry": "168h"
        },
        "profiles": {
            "www": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}

{
    "CN": "example.net",
    "hosts": [
        "example.net",
        "www.example.net"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```

## Create a configuration file with a profile

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
            ],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

It can be used to generate further certificates like so:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

## Certificates needed

![Architecture diagram!](architecture.png)

We need a certificate for all the places where there are arrows. Potentially the components can be in different machines

Image from [Kubernetes.io](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)

### Control plane
* ca.pem
* ca-key.pem
* service-account.pem (Service account tokens generation)
* service-account-key.pem (Service account tokens generation)
* kubernetes.pem (API server)
* kubernetes-key.pem (API server)
* etcd.pem (ETCD)
* etcd.pem (ETCD)
* kube-controller-manager.pem / kube-controller-manager.pem (via kubeconfig files)
* kube-scheduler.pem / kube-scheduler.pem (via kubeconfig files)

### Workers
* ca.pem
* instance.pem (Kubelet and kubeconfig file as well)
* instance-key.pem (Kubelet and kubeconfig file as well)
* kube-proxy.pem / kube-proxy-key.pem (via kubeconfig files)

### kubectl client
* admin.pem
* admin-key.pem

### Certificates used to generate kubeconfig files
* kube-proxy
* kube-controller-manager
* kube-scheduler
* kubelet
* admin

## Certificates generation and particularities

### admin
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
 ```
 
### Workers / Kubelets
CN has to be like this `system:node:<nodeName>`
 
### Controller manager
CN has to be like this `system:kube-controller-manager`

## Kube-proxy
CN has to be like this `system:kube-proxy`

## Kube-scheduler
CN has to be like this `system:kube-scheduler`

## Service accounts
CN has to be like this `service-accounts` (is it mandatory?)

## API server
CN has to be like this `kubernetes` (is it mandatory?)

Additionally, specify the hosts
* kubernetes.default
* 127.0.0.1
* KUBERNETES_PUBLIC_ADDRESS (load balancer public address if accessed like this)
* 10.32.0.1 (first valid address of kubernetes service IP range, --service-cluster-ip-range)
* 10.240.0.1X (master node X)

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

## Creating kubeconfig files for clients

```
kubectl config set-cluster ${CLUSTER_NAME} \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
--kubeconfig=${instance}.kubeconfig

kubectl config set-credentials system:node:${instance} \
--client-certificate=${instance}.pem \
--client-key=${instance}-key.pem \
--embed-certs=true \
--kubeconfig=${instance}.kubeconfig

kubectl config set-context default \
--cluster=kubernetes-the-hard-way \
--user=system:node:${instance} \
--kubeconfig=${instance}.kubeconfig

kubectl config use-context default --kubeconfig=${instance}.kubeconfig
```

The last step ensures the context is correct when the file is copied to destination