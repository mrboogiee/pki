# PKI

> **_WORK IN PROGRESS_** ! This documentation is work in progress !

## Introduction

I set out to create a simple PKI system that would allow me to create a CA, sign certificates, and revoke certificates. Through several blogs, comments and other resources, I was able to piece together a working system. This is a simple implementation and should not be used in a production environment. This is a learning exercise and should be treated as such. My environment ended up running in a kubernetes cluster as I have one in my home lab. This is not necessary and can be run on any system that has the necessary tools installed.

## Configuration

### Database

cfssl can save it’s state in a database. Three database provider are supported: mysql, pgsql and sqlite. Because I want to run this in a kubernetes environment where I will need to have multiple pods accessing the same database, I will not use sqlite but Postgres. Using a database lets us use OCSP related commands and everything related to certificate signature will also store the certificate.

Setting up the database server requires a few steps within kubernetes:

#### Namespace

First of all we need to have a namespace that we will use to deploy our PKI in. This is not necessary but it is a good practice to keep everything organized. The below yaml file will create a namespace called `pki`.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: pki
```

by executing the below against your kubernetes cluster, you will create the namespace.

```bash
kubectl apply -f namespace.yaml
```

#### Postgres

I did not really feel like I would be the right person to redesign how one should deploy a Postgres database in kubernetes. I chose to go with the work found on cnpg.io by using their helm chart:

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg \
  --namespace cnpg-system \
  cnpg/cloudnative-pg
```

This installs the cnpg operators that will manage the Postgres database.

Now I need a username and password for use with the database. I will create a secret in the `pki` namespace that will hold the username and password.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: cfssl-secret
  namespace: pki
type: kubernetes.io/basic-auth
stringData:
  username: cfssluser
  password: some@strongPassw0rd
```

by executing the below against your kubernetes cluster, you will create the secret.

```bash
kubectl apply -f secret.yaml
```

Now it is time to actually deploy my Postgres database cluster. I will use the below yaml file to deploy the database with 10 GB storage, monitoring enabled and references to the secret I defined previously.

```yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
  labels:
    app: pki
    role: db
  namespace: pki
spec:
  instances: 3

  storage:
    size: 10Gi

  monitoring:
    enablePodMonitor: true
  bootstrap:
    initdb:
      database: cfssl
      owner: cfssl
      secret:
        name: cfssl-secret
```

by executing the below against your kubernetes cluster, you will create the Postgres database.

```bash
kubectl apply -f postgres.yaml
```

I now have a Postgres database running in my kubernetes cluster. I can make it accessible by using the below command:

```bash
kubectl port-forward -n pki svc/postgres-rw 5432:5432
```

At this point in time the database does exist, is accessible, but is still empty. In line with the cfssl instructions I will need to create the tables using a tool called `goose`. After cloning the cfssl repository I can find the database schema in the cfssl/certdb/pg folder. Before we can insert the schema we first have to modify the `dbconf.yaml` file in `cfssl/certdb/pg` to look like this:

```bash
git clone https://github.com/cloudflare/cfssl.git
```

```yaml
production:
  driver: postgres
  open: host='localhost' port=5432 user='cfssluser' password='some@strongPassw0rd' dbname='cfssl'
```

This creates an environment for the goose tool named `production` that will connect to the Postgres database I just created. Now I will use the below command to insert the schema:

```bash
cd cfssl
goose -env production -path certdb/pg up
```

The database is now ready to be used by cfssl.

> **_NOTE:_** I only create a database for the intermediate CA. The root CA will only sign intermediate CA certificates and will not be used to sign server certificates. This is a security measure to ensure that the root CA key can always be kept offline. I am still working out how I want the root CA ocsp responder to work. I will update this post when I have that figured out.

---

### cfssl

#### Database access

cfssl needs a bit of configuration to work.

First I have to create the configuration file giving access to the database I just created:

```bash
mkdir -p db
cat <<EOF > db/db.json
{
    "driver": "postgres",
    "data_source": "postgres://cfssluser:some@strongPassw0rd@localhost:5432/cfssl?sslmode=disable"
}
EOF
```

### Signing profiles

Then I have to define the profiles my CAs will use. There are a few things to consider now even if it won’t be used until I can host the PKI:

where my CA will be hosted;
where the CRL will be hosted;
what uri will serve the OCSP responder.
I chose to serve the PKI under the domain pki.valhall.local. Here is a list of the uri I’ll be using (URIs are from the original blog post):

- <http://pki.valhall.local/root/ca>
- <http://pki.valhall.local/root/crl>
- <http://pki.valhall.local/root/ocsp>
- <http://pki.valhall.local/intermediate/ca>
- <http://pki.valhall.local/intermediate/crl>
- <http://pki.valhall.local/intermediate/ocsp>

Those information will later be available as extensions inside the signed certificate. Let’s take a look at the certificate behind google.com:

<!-- ```bash
$ echo | \
  openssl s_client \
    -showcerts \
    -servername google.com \
    -connect google.com:443 2>/dev/null | \
  openssl x509 \
    -inform pem \
    -noout -text

# Certificate
# Data
#
# X509v3 extensions
#
# Authority Information Access
# OCSP - URI:<http://ocsp.pki.goog/gts1c3>
# CA Issuers - URI:<http://pki.goog/repo/certs/gts1c3.der>
# X509v3 CRL Distribution Points
# Full Name
# URI:<http://crls.pki.goog/gts1c3/moVDfISia2k.crl>
```

These extensions, embedded in the certificate, are part of the verification process.

cfssl uses a json file defining the signing profiles, among other things. Signing profiles are predefined sets of parameters used to sign a kind of certificate. I wrote earlier about the root CA that will only sign intermediate CAs. Its profiles file would be:

```json
{
  "signing": {
    "default": {
      "crl_url": "<http://pki.valhall.local/root/crl>",
      "ocsp_url": "<http://pki.valhall.local/root/ocsp>",
      "issuer_urls": [
        "http://pki.valhall.local/root/ca"
      ],
      "expiry": "8760h"
    },
    "profiles": {
      "intermediate": {
        "usages": [
          "signing",
          "digital signature",
          "key encipherment",
          "cert sign",
          "crl sign",
          "server auth",
          "client auth"
        ],
        "ca_constraint": {
          "is_ca": true,
          "max_path_len": 0,
          "max_path_len_zero": true
        },
        "expiry": "87600h"
      },
      "ocsp": {
        "usages": [
          "digital signature",
          "ocsp signing"
        ],
        "expiry": "26280h"
      }
    }
  }
}
```

In the above json configuration I defined two profiles, intermediate that will be used to sign other CA certificates and ocsp that will be used to sign the certificate used by the OCSP responder. The .signing.default object is used to set parameters shared between the profiles.

The intermediate CA will mainly be used to sign certificates for servers and for client authentications. Since I’ll later use this intermediate CA to sign certificates within an automatic renewal process, I chose to make the certificate signed with the server profile short-lived:

```json
{
  "signing": {
    "default": {
      "crl_url": "<http://pki.valhall.local/intermediate/crl>",
      "ocsp_url": "<http://pki.valhall.local/intermediate/ocsp>",
      "issuer_urls": [
        "http://pki.valhall.local/intermediate/ca"
      ],
      "expiry": "8760h"
    },
    "profiles": {
      "client": {
        "usages": [
          "signing",
          "digital signing",
          "key encipherment",
          "client auth"
        ],
        "expiry": "8760h"
      },
      "server": {
        "usages": [
          "signing",
          "digital signing",
          "key encipherment",
          "server auth"
        ],
        "expiry": "2190h"
      },
      "ocsp": {
        "usages": [
          "digital signature",
          "ocsp signing"
        ],
        "expiry": "26280h"
      }
    }
  }
}
```

These profile files will be saved in root/config/profiles.json and intermediate/config/profiles.json.

### CA certificate definition

The structure of a certificate request is defined at cloudflare/cfssl/csr/csr.go#L138.

Here are the two definitions I’ll use, respectfully in root/config/init.json and intermediate/config/init.json:

```json
{
  "CN": "Valhall Root CA Certificate",
  "CA": {
    "expiry": "87600h"
  },
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [{
    "C":  "FR",
    "ST": "Pays de la Loire",
    "L":  "Nantes",
    "O":  "Valhall"
  }]
}
{
  "CN": "Valhall Intermediate CA Certificate",
  "CA": {
    "expiry": "87600h"
  },
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [{
    "C":  "FR",
    "ST": "Pays de la Loire",
    "L":  "Nantes",
    "O":  "Valhall"
  }]
}
```

The creation of a private key and a certificate is quite easy.

The genkey command of cfssl toolkit will create a private key, a signing request and will self-sign it.

```bash
cfssl genkey -initca root/config/init.json | cfssljson -bare root/ca
```

cfssl genkey returns a JSON with three keys: cert, csr and key. cfssljson will create the three files.

Creating the intermediate CA follows the same process. I’ll just have to discard the certificate and sign the CSR with the root CA instead:

```bash
cfssl genkey -initca intermediate/config/init.json | cfssljson -bare intermediate/ca
rm intermediate/ca.pem
cfssl sign \
    -ca root/ca.pem \
    -ca-key root/ca-key.pem \
    -config root/config/profiles.json \
    -profile intermediate \
    -db-config root/config/db.json \
    intermediate/ca.csr | cfssljson -bare intermediate/ca
```

You should now have the following files in your current directory:

```bash
tree
#
# ├── db
# │   ├── definition.sql
# │   ├── intermediate-certstore.db
# │   └── root-certstore.db
# ├── intermediate
# │   ├── ca.csr
# │   ├── ca-key.pem
# │   ├── ca.pem
# │   └── config
# │       ├── db.json
# │       ├── init.json
# │       └── profiles.json
# └── root
# ├── ca.csr
# ├── ca-key.pem
# ├── ca.pem
# └── config
# ├── db.json
# ├── init.json
# └── profiles.json
#
# 6 directories, 15 files
```

A SQL request in the root database shows that the intermediate CA certificate was also stored.

```bash
$ sqlite3 db/root-certstore.db "SELECT common_name FROM certificates;"
Valhall Intermediate CA Certificate
```

### OCSP key & certificate

Almost done, the only remaining tasks are to generate the certificates and keys for both CA’s ocsp responder and to generate the CRLs.

```bash
cat <<EOF > root/config/ocsp.json
{
  "CN": "Valhall Root OCSP Certificate",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C":  "FR",
    "ST": "Pays de la Loire",
    "L":  "Nantes",
    "O":  "Valhall"
  }]
}
EOF
cfssl gencert \
    -ca root/ca.pem \
    -ca-key root/ca-key.pem \
    -config root/config/profiles.json \
    -profile ocsp \
    -db-config root/config/db.json \
    root/config/ocsp.json | cfssljson -bare root/ocsp
cat <<EOF > intermediate/config/ocsp.json
{
  "CN": "Valhall Intermediate OCSP Certificate",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C":  "FR",
    "ST": "Pays de la Loire",
    "L":  "Nantes",
    "O":  "Valhall"
  }]
}
EOF
cfssl gencert \
    -ca intermediate/ca.pem \
    -ca-key intermediate/ca-key.pem \
    -config intermediate/config/profiles.json \
    -profile ocsp \
    -db-config intermediate/config/db.json \
    intermediate/config/ocsp.json | cfssljson -bare intermediate/ocsp
```

## CRL

The Certificate Revocation Lists are files that should (probably?) be regularly updated. I’m not sure how clients implement the cache mechanism for this feature as the CRL advertises two dates: last update and next update. Meaning if they cache it until next update date, then they could miss a certificate being revoked until the cache ttl is reached. CRL are also signed with their CA’s key, meaning if you want to keep the root CA’s private key offline, this could be quite tricky. I’ve seen different approaches: some create CRL advertising a next update date when the CA will expire and only refresh it manually when needed and others create weekly CRL, which is cfssl’s default:

```bash
$ cfssl crl -h
#
# -expiry=168h0m0s: time from now after which the CRL will expire (default: one week)
```

I chose to do the latter and since I can’t imagine creating weekly CRL and not having the task automated, I’ll later store the private keys in a Hashicorp Vault instance I manage, which is an acceptable risk for my home-lab.

cfssl crl outputs a PEM CRL without header/footer and without line feeds, so I’ll have to handle that:

```bash
echo "-----BEGIN X509 CRL-----" > root/crl.pem
cfssl crl \
    -ca root/ca.pem \
    -ca-key root/ca-key.pem \
    -db-config root/config/db.json | fold -w 64 >> root/crl.pem
echo "-----END X509 CRL-----" >> root/crl.pem
echo "-----BEGIN X509 CRL-----" > intermediate/crl.pem
cfssl crl \
    -ca intermediate/ca.pem \
    -ca-key intermediate/ca-key.pem \
    -db-config intermediate/config/db.json | fold -w 64 >> intermediate/crl.pem
echo "-----END X509 CRL-----" >> intermediate/crl.pem
```

You can check the CRL with the openssl command:

```bash
openssl crl -inform PEM -text -noout -in root/crl.pem
# Certificate Revocation List (CRL)
# Version 2 (0x1)
# Signature Algorithm: sha256WithRSAEncryption
# Issuer: C = FR, ST = Ile de France, L = Nantes, O = Valhall, CN = Valhall Root CA Certificate
# Last Update: Jun 17 08:39:31 2023 GMT
# Next Update: Jun 24 08:39:31 2023 GMT
# CRL extensions
# X509v3 Authority Key Identifier
# D0:33:EF:44:95:BD:B2:0B:61:6D:B8:E0:19:95:6D:80:90:AA:3F:A6
# No Revoked Certificates
# Signature Algorithm: sha256WithRSAEncryption
# Signature Value
# 7b:91:89:00:41:d4:80:72:0b:af:db:7d:e5:19:cd:d0:29:3b
#
```

## Testing

I should now have everything needed to try and sign certificates, revoke them, etc. let’s start the API:

```bash
cfssl serve \
      -ca=intermediate/ca.pem \
      -ca-key=intermediate/ca-key.pem \
      -responder=intermediate/ocsp.pem \
      -responder-key=intermediate/ocsp-key.pem \
      -db-config=intermediate/config/db.json \
      -config=intermediate/config/profiles.json
# 2023/06/17 10:55:12 [INFO] Initializing signer
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/newcert' is enabled
# 2023/06/17 10:55:12 [INFO] setting up key / CSR generator
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/newkey' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/ocspsign' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/info' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/gencrl' is enabled
# 2023/06/17 10:55:12 [INFO] bundler API ready
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/bundle' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/scan' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/revoke' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/certadd' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/sign' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/init_ca' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/scaninfo' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/certinfo' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/' is enabled
# 2023/06/17 10:55:12 [WARNING] endpoint 'authsign' is disabled: {"code":5200,"message":"Invalid or unknown policy"}
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/crl' is enabled
# 2023/06/17 10:55:12 [INFO] endpoint '/api/v1/cfssl/health' is enabled
# 2023/06/17 10:55:12 [INFO] Handler set up complete
# 2023/06/17 10:55:12 [INFO] Now listening on 127.0.0.1:8888
```

You will notice a warning about authsign API endpoint being disabled: I’ll cover this in the article about serving the API in kubernetes. It’s only disabled since I did not set any authentication method in the configuration file. If you want to serve the API as is, you should consider carefully who will be able to access it. Other endpoints won’t all be used and there is also a method to disable those you won’t want to use nor expose.

In another terminal, I’ll use cfssl to create a key, CSR and ask the CA behind the API to sign the CSR.

```bash
cfssl gencert \
      -remote="localhost:8888" \
      -config=intermediate/config/profiles.json \
      -profile server \
      <(echo '
{
  "CN": "Test",
  "hosts": [
    "test.valhall.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C":  "FR",
    "ST": "Pays de la Loire",
    "L":  "Nantes",
    "O":  "Valhall"
  }]
}') | \
    cfssljson -bare test
# 2023/06/17 11:17:28 [INFO] generate received request
# 2023/06/17 11:17:28 [INFO] received CSR
# 2023/06/17 11:17:28 [INFO] generating key: rsa-2048
# 2023/06/17 11:17:28 [INFO] encoded CSR
```

You can see new logs on the server side:

```bash
# 2023/06/17 11:17:28 [INFO] signature request received
# 2023/06/17 11:17:28 [INFO] signed certificate with serial number 485014236354530765875959101436276396320072239922
# 2023/06/17 11:17:28 [INFO] wrote response
# 2023/06/17 11:17:28 [INFO] 127.0.0.1:46402 - "POST /api/v1/cfssl/sign" 200
```

You can also use the cfssl sign command to sign an existing CSR created directly with openssl.

Let’s see if all the configuration I made earlier paid of:

```bash
openssl x509 -text -in test.pem
# Certificate
# Data
# Version: 3 (0x2)
# Serial Number
# 54:f4:ca:61:42:01:e9:0d:8a:ae:93:65:42:b1:66:37:5d:91:8b:32
# Signature Algorithm: sha512WithRSAEncryption
# Issuer: C = FR, ST = Ile de France, L = Nantes, O = Valhall, CN = Valhall Intermediate CA Certificate
# Validity
# Not Before: Jun 17 09:12:00 2023 GMT
# Not After : Sep 16 15:12:00 2023 GMT
# Subject: C = FR, ST = Pays de la Loire, L = Nantes, O = Valhall, CN = Test
# Subject Public Key Info
# Public Key Algorithm: rsaEncryption
# Public-Key: (2048 bit)
# Modulus
# 00:c3:6e:f3:0a:21:ff:fa:be:10:11:48:63:60:1a
#
# cf:f1
# Exponent: 65537 (0x10001)
# X509v3 extensions
# X509v3 Key Usage: critical
# Digital Signature, Key Encipherment
# X509v3 Extended Key Usage
# TLS Web Server Authentication
# X509v3 Basic Constraints: critical
# CA:FALSE
# X509v3 Subject Key Identifier
# 33:29:48:E7:3A:B6:4E:90:92:1E:F7:F2:33:E6:1F:99:F2:42:5E:EF
# X509v3 Authority Key Identifier
# C8:80:42:BF:8B:0D:C9:9F:55:78:DD:56:E2:8B:1A:AE:57:49:37:1B
# Authority Information Access
# OCSP - URI:<http://pki.valhall.local/intermediate/ocsp>
# CA Issuers - URI:<http://pki.valhall.local/intermediate/ca>
# X509v3 Subject Alternative Name
# DNS:test.valhall.local
# X509v3 CRL Distribution Points
# Full Name
# URI:<http://pki.valhall.local/intermediate/crl>
# Signature Algorithm: sha512WithRSAEncryption
# Signature Value
# a2:e2:9a:dd:83:57:ff:4e:3c:92:b3:cc:78:1b:4c:0e:f0:da:
#
# 0e:3c:54:81:ef:04:9b:af
# -----BEGIN CERTIFICATE-----
# MIIFsDCCA5igAwIBAgIUVPTKYUIB6Q2KrpNlQrFmN12RizIwDQYJKoZIhvcNAQEN
#
# qVBNU7XEsx7X+n4rDjxUge8Em68=
# -----END CERTIFICATE-----
```

The server profile was correctly used: validity time is three months and OCSP, CA and CRL endpoint are correct.

Let’s start the OCSP responder:

```bash
cfssl ocspserve -db-config intermediate/config/db.json -port 8889
# 2023/06/18 10:23:06 [INFO] Registering OCSP responder handler
# 2023/06/18 10:23:06 [INFO] Now listening on 127.0.0.1:8889
```

The test.pem certificate can be verified with the following openssl command:

```bash
openssl ocsp \
    -issuer <(cat intermediate/ca.pem) \
    -CAfile <(cat root/ca.pem) \
    -cert test.pem \
    -url <http://localhost:8889>
```

If you ran this command right away, you should have received the error unauthorized. It’s because cfssl uses pre-signed ocsp responses, meaning it has to sign a new response each time I sign or revoke a certificate.

```bash
# Responder Error: unauthorized (6)
```

Let’s refresh the OCSP response - it will be stored in the database - then retry the openssl ocsp command:

```bash
cfssl ocsprefresh \
    -ca intermediate/ca.pem \
    -responder intermediate/ocsp.pem \
    -responder-key intermediate/ocsp-key.pem \
    -db-config intermediate/config/db.json
# WARNING: no nonce in response
# Response verify OK
# test.pem: good
# This Update: Jun 18 08:00:00 2023 GMT
# Next Update: Jun 22 08:00:00 2023 GMT
```

You can disable the warning about the nonce not being present with the parameter -no_nonce. As you saw, cfssl uses pre-signed ocsp responses, and therefore they cannot include nonces (this is compatible with RFC 5019, section 4). The cfssl project does not intend to support nonces as written at cloudflare/cfssl/ocsp/responder.go#L336.

Let’s now revoke the certificate. The revoke API takes three parameters:

- `serial`: the certificate serial number in decimal (strangely);
- `authority_key_id`: the authority key identifier, in lowercase hexadecimal without separators;
- `reason`: a reason for the revocation.

Possible reasons for revocation are listed in RFC 5280, section 6.3.2. Their syntax for cfssl api are written in cloudflare/cfssl/ocsp/ocsp.go#L26:

```golang
// revocationReasonCodes is a map between string reason codes
// to integers as defined in RFC 5280
var revocationReasonCodes = map[string]int{
    "unspecified":          ocsp.Unspecified,
    "keycompromise":        ocsp.KeyCompromise,
    "cacompromise":         ocsp.CACompromise,
    "affiliationchanged":   ocsp.AffiliationChanged,
    "superseded":           ocsp.Superseded,
    "cessationofoperation": ocsp.CessationOfOperation,
    "certificatehold":      ocsp.CertificateHold,
    "removefromcrl":        ocsp.RemoveFromCRL,
    "privilegewithdrawn":   ocsp.PrivilegeWithdrawn,
    "aacompromise":         ocsp.AACompromise,
}
```

In the earlier certificate description with `openssl x509` I could see the value for the two other parameters:

The serial number `54:f4:ca:61:42:01:e9:0d:8a:ae:93:65:42:b1:66:37:5d:91:8b:32` becomes `485014236354530765875959101436276396320072239922`;
The authority key identifier `C8:80:42:BF:8B:0D:C9:9F:55:78:DD:56:E2:8B:1A:AE:57:49:37:1B` becomes `c88042bf8b0dc99f5578dd56e28b1aae5749371b`.

```bash
curl -d '{
  "serial": "485014236354530765875959101436276396320072239922",
  "authority_key_id": "c88042bf8b0dc99f5578dd56e28b1aae5749371b",
  "reason": "cessationofoperation"
}' <http://localhost:8888/api/v1/cfssl/revoke>
# {"success":true,"result":{},"errors":[],"messages":[]}
```

As you can see, there is no authentication whatsoever provided for this endpoint, contrary to the sign endpoint also being available with at least HMAC authentication as authsign so it’s not a good idea to expose it as is.

After refreshing the cached OCSP response, the `openssl ocsp` command answers with:

```bash
# Response verify OK
# test.pem: revoked
# This Update: Jun 19 08:00:00 2023 GMT
# Next Update: Jun 23 08:00:00 2023 GMT
# Reason: cessationOfOperation
# Revocation Time: Jun 19 08:25:56 2023 GMT
```

And after regenerating the CRL file, `openssl crl` outputs:

```bash
# Certificate Revocation List (CRL)
# Version 2 (0x1)
# Signature Algorithm: sha256WithRSAEncryption
# Issuer: C = FR, ST = Ile de France, L = Nantes, O = Valhall, CN = Valhall Intermediate CA Certificate
# Last Update: Jun 19 08:46:10 2023 GMT
# Next Update: Jun 26 08:46:10 2023 GMT
# CRL extensions
# X509v3 Authority Key Identifier
# C8:80:42:BF:8B:0D:C9:9F:55:78:DD:56:E2:8B:1A:AE:57:49:37:1B
# Revoked Certificates
# Serial Number: 54F4CA614201E90D8AAE936542B166375D918B32
# Revocation Date: Jun 19 08:25:56 2023 GMT
# Signature Algorithm: sha256WithRSAEncryption
# Signature Value
# d6:3d:18:16:6c:6a:db:07:99:41:02:76:aa:4b:16:b5:da:bd:
#
# 42:df:ef:c2:6f:17:6b:3c
```

We know from both methods that the certificate is indeed revoked. -->

## Sources

- [https://bouchaud.org/blog/en/posts/initializing-root-intermediate-ca-with-cfssl/](https://bouchaud.org/blog/en/posts/initializing-root-intermediate-ca-with-cfssl/) This blog was the most helpful in getting me started. It provided a clear and concise guide to setting up a CA and intermediate CA. The majority of instructions on setting up the root and intermediate CA certificates were taken from this blog. The only reason for me to copy the majority of the instructions to this README was to have a single source of truth for the entire process. Kudos still go out to [@vbouchaud](https://github.com/vbouchaud) for the great blog post. Modifications were made to the instructions to fit my environment and to make the process more clear to me. Also some context was removed as I don't need to document the entire process like the blogpost does.
