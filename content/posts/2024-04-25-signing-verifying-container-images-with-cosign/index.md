---
title:  "Signing and verifying container images with Cosign and your own PKI"
author: "Mario"
tags: [ "cosing", "sigstore", "byok", "pki", "security" ]
url: "/signing-verifying-container-images-with-cosign-own-pki"
draft: false
date: 2024-04-25
lastmod: 2024-05-03
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Signing and verifying container images with Cosign and your own PKI

In this post we are going to cover how we can sign and verify container images using [Cosign](https://github.com/sigstore/cosign) and our own PKI. You can learn more on how to build your own PKI with CFSSL in [this post](https://linuxera.org/pki-with-cfssl/).

{{<warning>}}
The way we will see to sign and verify images in this post is not the recommended approach. For production usage, you should use ephemeral keys as described [here](https://docs.sigstore.dev/signing/signing_with_containers/). Validity of certificates generated during this post it's not recommended for production usage, make sure in production you have proper validity periods and rotation capabilities in place.
{{</warning>}}

## Solution Overview

We have our own Root CA that issued an Intermediate CA. This Intermediate CA issued two certificates, one for Team A to sign their images and another one for Team B to sign their images.

Then, verification of image signatures will be done by using the Root CA public certificate.

![Solution Overview](./overview.png)

## Generating PKI with CFSSL

1. Install required CFSSL binaries:

    ~~~sh
    sudo curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64 -o /usr/local/bin/cfssl
    sudo curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64 -o /usr/local/bin/cfssljson
    sudo chmod +x /usr/local/bin/{cfssl,cfssljson}
    ~~~

2. Define workdir for the files that will be generated during the post:

    ~~~sh
    export WORKDIR=/var/tmp/cosign/
    ~~~

3. Generate Root and Intermediate CAs:

    ~~~sh
    mkdir -p $WORKDIR/cafiles/{root,intermediate,config,certificates}
    # Generate Root CA csr
    cat << "EOF" > $WORKDIR/cafiles/root/root-csr.json
    {
      "CN": "Linuxera Root Certificate Authority",
      "key": {
        "algo": "ecdsa",
        "size": 256
      },
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "IT",
          "OU": "Security",
          "ST": "Valencia"
        }
      ],
      "ca": {
        "expiry": "87600h"
      }
    }
    EOF

    # Generate Intermediate CA csr
    cat << "EOF" > $WORKDIR/cafiles/intermediate/intermediate-csr.json
    {
      "CN": "Linuxera Intermediate Certificate Authority",
      "key": {
        "algo": "ecdsa",
        "size": 256
      },
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "IT",
          "OU": "Security",
          "ST": "Valencia"
        }
      ]
    }
    EOF

    # Issue RootCA keypair
    cfssl gencert -initca $WORKDIR/cafiles/root/root-csr.json | cfssljson -bare $WORKDIR/cafiles/root/root-ca

    # Generate Intermediate CA key
    cfssl genkey $WORKDIR/cafiles/intermediate/intermediate-csr.json | cfssljson -bare $WORKDIR/cafiles/intermediate/intermediate-ca

    # Define cfssl profile for the intermediate CA
    cat << "EOF" > $WORKDIR/cafiles/config/config.json
    {
      "signing": {
        "default": {
          "expiry": "8760h"
        },
        "profiles": {
          "intermediate": {
            "usages": ["cert sign", "crl sign"],
            "expiry": "70080h",
            "ca_constraint": {
              "is_ca": true,
              "max_path_len": 1
            }
          }
        }
      }
    }
    EOF

    # Issue Intermediate CA cert using RootCA
    cfssl sign -ca $WORKDIR/cafiles/root/root-ca.pem -ca-key $WORKDIR/cafiles/root/root-ca-key.pem -config $WORKDIR/cafiles/config/config.json -profile intermediate $WORKDIR/cafiles/intermediate/intermediate-ca.csr | cfssljson -bare $WORKDIR/cafiles/intermediate/intermediate-ca

    # Put RootCA key to sleep 
    rm -f $WORKDIR/cafiles/root/root-ca-key.pem
    ~~~

4. Generate container signing certificates for Team A and Team B:

    {{<attention>}}
For Cosign v2 verification to work, we need our certificates to provide some information in the SAN. We require it to provide a valid email and also to provide a valid oidcIssuer. The oidcIssuer is added via OID, you can find the details [here](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md). In the certs below you can see two OIDs added as extensions, the value for the one ending in `1,1` is a base64 encoded raw string, while the one ending in `1,8` is the base64 encoded ASN.1 string (you can check the code [here](#generate-asn1-string-for-fulcio-extensions) to generate the string).
    {{</attention>}}

    ~~~sh
    # Define a new cosign profile
    cat << "EOF" > $WORKDIR/cafiles/config/config.json
    {
      "signing": {
        "default": {
          "expiry": "8760h"
        },
        "profiles": {
          "intermediate": {
            "usages": ["cert sign", "crl sign"],
            "expiry": "70080h",
            "ca_constraint": {
              "is_ca": true,
              "max_path_len": 1,
              "copy_extensions": true
            }
          },
          "cosign": {
            "usages": ["signing", "digital signing"],
            "expiry": "8760h",
            "copy_extensions": true
          }
        }
      }
    }
    EOF

    # Generate cosign keypair for team-a and team-b
    # cosign v2 requires email and uri SAN values which are configured under hosts
    cat << "EOF" > $WORKDIR/cafiles/certificates/team-a-csr.json
    {
      "CN": "Team A Cosign Certificate",
      "key": {
        "algo": "ecdsa",
        "size": 256
      },
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "IT",
          "OU": "Security"
        }
      ],
      "hosts": [
        "team-a@linuxera.org"
      ],
      "extensions": [
        {
            "id": [1,3,6,1,4,1,57264,1,8],
            "value": "ExRodHRwczovL2xpbnV4ZXJhLm9yZw=="
        },
        {
            "id": [1,3,6,1,4,1,57264,1,1],
            "value": "aHR0cHM6Ly9saW51eGVyYS5vcmc="
        }
      ]
    }
    EOF

    cat << "EOF" > $WORKDIR/cafiles/certificates/team-b-csr.json
    {
      "CN": "Team B Cosign Certificate",
      "key": {
        "algo": "ecdsa",
        "size": 256
      },
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "IT",
          "OU": "Security"
        }
      ],
      "hosts": [
        "team-b@linuxera.org"
      ],
      "extensions": [
        {
            "id": [1,3,6,1,4,1,57264,1,8],
            "value": "ExRodHRwczovL2xpbnV4ZXJhLm9yZw=="
        },
        {
            "id": [1,3,6,1,4,1,57264,1,1],
            "value": "aHR0cHM6Ly9saW51eGVyYS5vcmc="
        }
      ]
    }
    EOF

    # Issue certificates via the intermediate CA
    cfssl gencert -ca $WORKDIR/cafiles/intermediate/intermediate-ca.pem -ca-key $WORKDIR/cafiles/intermediate/intermediate-ca-key.pem -config $WORKDIR/cafiles/config/config.json -profile cosign $WORKDIR/cafiles/certificates/team-a-csr.json | cfssljson -bare $WORKDIR/cafiles/certificates/team-a
    cfssl gencert -ca $WORKDIR/cafiles/intermediate/intermediate-ca.pem -ca-key $WORKDIR/cafiles/intermediate/intermediate-ca-key.pem -config $WORKDIR/cafiles/config/config.json -profile cosign $WORKDIR/cafiles/certificates/team-b-csr.json | cfssljson -bare $WORKDIR/cafiles/certificates/team-b
    ~~~

### Alternative OpenSSL commands

In case you want to rather use OpenSSL, these are the commands you need to run. Thanks to [Frédéric Herrmann](https://www.linkedin.com/in/fredericherrmann/) for sharing the information.

{{<attention>}}
Commands below use RSA as encryption algorithm.
{{</attention>}}

1. Generate Root CA

    ~~~sh
    openssl req -x509 -newkey rsa:4096 -keyout root-ca-key.pem -sha256 -noenc -days 9999 -subj "/C=ES/L=Valencia/O=IT/OU=Security/CN=Linuxera Root Certificate Authority" -out root-ca.pem
    ~~~

2. Generate Intermediate CA

    ~~~sh
    # Create request
    openssl req -noenc -newkey rsa:4096 -keyout intermediate-ca-key.pem -addext "subjectKeyIdentifier = hash" -addext "keyUsage = critical,keyCertSign" -addext "basicConstraints = critical,CA:TRUE,pathlen:2" -subj "/C=ES/L=Valencia/O=IT/OU=Security/CN=Linuxera Intermediate Certificate Authority" -out intermediate-ca.csr
    
    # Issue cert with Root CA
    openssl x509 -req -days 9999 -sha256 -in intermediate-ca.csr -CA root-ca.pem -CAkey root-ca-key.pem -copy_extensions copy -out intermediate-ca.pem
    ~~~

3. Issue Team A cert

    ~~~sh
    # OID_1_1 is the hexadecimal representation of the oidcissuer url
    OID_1_1=$(echo -n "https://linuxera.org" | xxd -p -u)
    # Create request
    openssl req -noenc -newkey rsa:4096 -keyout team-a-key.pem -addext "subjectKeyIdentifier = hash" -addext "basicConstraints = critical,CA:FALSE" -addext "keyUsage = critical,digitalSignature" -addext "subjectAltName = email:team-a@linuxera.org" -addext "1.3.6.1.4.1.57264.1.1 = DER:${OID_1_1}" -addext "1.3.6.1.4.1.57264.1.8 = ASN1:UTF8String:https://linuxera.org" -subj "/C=ES/L=Valencia/O=IT/OU=Security/CN=Team A Cosign Certificate" -out team-a.csr
    
    # Issue cert with Intermediate CA
    openssl x509 -req -in team-a.csr -CA intermediate-ca.pem -CAkey intermediate-ca-key.pem -copy_extensions copy -days 9999 -sha256 -out team-a.pem
    ~~~

4. Issue Team B cert

    ~~~sh
    # OID_1_1 is the hexadecimal representation of the oidcissuer url
    OID_1_1=$(echo -n "https://linuxera.org" | xxd -p -u)
    # Create request
    openssl req -noenc -newkey rsa:4096 -keyout team-b-key.pem -addext "subjectKeyIdentifier = hash" -addext "basicConstraints = critical,CA:FALSE" -addext "keyUsage = critical,digitalSignature" -addext "subjectAltName = email:team-b@linuxera.org" -addext "1.3.6.1.4.1.57264.1.1 = DER:${OID_1_1}" -addext "1.3.6.1.4.1.57264.1.8 = ASN1:UTF8String:https://linuxera.org" -subj "/C=ES/L=Valencia/O=IT/OU=Security/CN=Team B Cosign Certificate" -out team-b.csr
    
    # Issue cert with Intermediate CA
    openssl x509 -req -in team-b.csr -CA intermediate-ca.pem -CAkey intermediate-ca-key.pem -copy_extensions copy -days 9999 -sha256 -out team-b.pem
    ~~~

## Generate our container image

We need a container image we will sign, on top of that we require our registry to support the upload of Cosign signatures. For this post I'll be using [quay.io](https://quay.io).

1. Define our Quay image repository:

    ~~~sh
    IMAGE_ENDPOINT=quay.io/example/signingtest
    ~~~

2. Generate a Dockerfile:

    ~~~sh
    cat <<'EOF' > $WORKDIR/Dockerfile
    FROM registry.fedoraproject.org/fedora-minimal:39
    ARG GREETING
    RUN echo $GREETING > /tmp/HelloWorld
    USER 1024
    CMD ["sleep", "infinity"]
    EOF
    ~~~

3. Build the images for Team A and Team B:

    ~~~sh
    podman build -f $WORKDIR/Dockerfile --build-arg GREETING="Hello from Team A" -t $IMAGE_ENDPOINT:team-a
    podman build -f $WORKDIR/Dockerfile --build-arg GREETING="Hello from Team B" -t $IMAGE_ENDPOINT:team-b
    ~~~

4. Push the images to the registry and store the images digest locally, this is required since we sign image digests, not tags:

    ~~~sh
    podman push --digestfile $WORKDIR/image-digest-team-a $IMAGE_ENDPOINT:team-a
    podman push --digestfile $WORKDIR/image-digest-team-b $IMAGE_ENDPOINT:team-b
    ~~~

## Signing container images with Cosign

{{<attention>}}
During the exploratory work I started testing Cosign v1, that's why I will document here how you can sign container images using v1. The steps that differ between both versions will have both options described. Nevertheless, you should be using v2.
{{</attention>}}

1. Download Cosign CLI

    ~~~sh
    sudo curl -L https://github.com/sigstore/cosign/releases/download/v1.13.6/cosign-linux-amd64 -o /usr/local/bin/cosign-v1
    sudo curl -L https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64 -o /usr/local/bin/cosign-v2
    sudo chmod +x /usr/local/bin/cosign-v1
    sudo chmod +x /usr/local/bin/cosign-v2
    ~~~

2. Build trust chain bundle (must start with the parent intermediate CA certificate of the signing certificate and end with the root certificate):

    ~~~sh
    cat $WORKDIR/cafiles/intermediate/intermediate-ca.pem $WORKDIR/cafiles/root/root-ca.pem > $WORKDIR/ca-bundle.pem
    ~~~

### Team-A Image Signing

1. Store container image digest:

    ~~~sh
    TEAM_A_IMAGE_DIGEST=$(cat $WORKDIR/image-digest-team-a)
    ~~~

2. Import key into Cosign:

    1. With v1:

        ~~~sh
        cosign-v1 import-key-pair --key $WORKDIR/cafiles/certificates/team-a-key.pem
        mv import-cosign.key $WORKDIR/import-team-a.key
        mv import-cosign.pub $WORKDIR/import-team-a.pub
        ~~~

    2. With v2:

        ~~~sh
        cosign-v2 import-key-pair --key $WORKDIR/cafiles/certificates/team-a-key.pem --output-key-prefix=$WORKDIR/import-team-a
        ~~~

3. Sign with Cosign and our own key without uploading to transparency log:

    1. With v1:

        ~~~sh
        cosign-v1 sign --key $WORKDIR/import-team-a.key --no-tlog-upload --cert $WORKDIR/cafiles/certificates/team-a.pem --cert-chain $WORKDIR/ca-bundle.pem $IMAGE_ENDPOINT@$TEAM_A_IMAGE_DIGEST
        ~~~

    2. With v2:

        ~~~sh
        cosign-v2 sign --key $WORKDIR/import-team-a.key --tlog-upload=false --cert $WORKDIR/cafiles/certificates/team-a.pem --cert-chain $WORKDIR/ca-bundle.pem $IMAGE_ENDPOINT@$TEAM_A_IMAGE_DIGEST
        ~~~

### Team-B Image Signing

1. Store container image digest:

    ~~~sh
    TEAM_B_IMAGE_DIGEST=$(cat $WORKDIR/image-digest-team-b)
    ~~~

2. Import key into Cosign:

    1. With v1:

        ~~~sh
        cosign-v1 import-key-pair --key $WORKDIR/cafiles/certificates/team-b-key.pem
        mv import-cosign.key $WORKDIR/import-team-b.key
        mv import-cosign.pub $WORKDIR/import-team-b.pub
        ~~~

    2. With v2:

        ~~~sh
        cosign-v2 import-key-pair --key $WORKDIR/cafiles/certificates/team-b-key.pem --output-key-prefix=$WORKDIR/import-team-b
        ~~~

3. Sign with Cosign and our own key without uploading to transparency log:

    1. With v1:

        ~~~sh
        cosign-v1 sign --key $WORKDIR/import-team-b.key --no-tlog-upload --cert $WORKDIR/cafiles/certificates/team-b.pem --cert-chain $WORKDIR/ca-bundle.pem $IMAGE_ENDPOINT@$TEAM_B_IMAGE_DIGEST
        ~~~

    2. With v2:

        ~~~sh
        cosign-v2 sign --key $WORKDIR/import-team-b.key --tlog-upload=false --cert $WORKDIR/cafiles/certificates/team-b.pem --cert-chain $WORKDIR/ca-bundle.pem $IMAGE_ENDPOINT@$TEAM_B_IMAGE_DIGEST
        ~~~

## Verifying container images with Cosign and our Root CA

We can verify the image signature in multiple ways, in this case we want to use only the Root CA public certificate to run the verification. Keep in mind that Cosign v2 doesn't work with BYOK like v1 used to do. [More info here](https://github.com/sigstore/cosign/issues/2632).

### Cosign v1

We need to disable rekor-url since we haven't uploaded anything to the transparency log. [More here](https://github.com/sigstore/cosign/issues/2858).

~~~sh
SIGSTORE_ROOT_FILE=$WORKDIR/cafiles/root/root-ca.pem COSIGN_EXPERIMENTAL=1 cosign-v1 verify --rekor-url="" $IMAGE_ENDPOINT@$TEAM_A_IMAGE_DIGEST $IMAGE_ENDPOINT@$TEAM_B_IMAGE_DIGEST
~~~

~~~output
Verification for quay.io/example/signingtest@sha256:81a274fb4ed001132969dcf03de244d6f9b36d42d2b688b09a1c2d67691e6e41 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Any certificates were verified against the Fulcio roots.

[{"critical":{"identity":{"docker-reference":"quay.io/example/signingtest"},"image":{"docker-manifest-digest":"sha256:81a274fb4ed001132969dcf03de244d6f9b36d42d2b688b09a1c2d67691e6e41"},"type":"cosign container image signature"},"optional":{"1.3.6.1.4.1.57264.1.1":"https://linuxera.org","Issuer":"https://linuxera.org","Subject":"team-a@linuxera.org"}}]

Verification for quay.io/example/signingtest@sha256:3bf90da5cd72d0057e4f06faf5669004262885005e662a93706fa85627d62cc5 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Any certificates were verified against the Fulcio roots.

[{"critical":{"identity":{"docker-reference":"quay.io/example/signingtest"},"image":{"docker-manifest-digest":"sha256:3bf90da5cd72d0057e4f06faf5669004262885005e662a93706fa85627d62cc5"},"type":"cosign container image signature"},"optional":{"1.3.6.1.4.1.57264.1.1":"https://linuxera.org","Issuer":"https://linuxera.org","Subject":"team-b@linuxera.org"}}]
~~~

### Cosign v2

For Cosign v2 to work we need to have an email and uri defined in the SAN fields of the signing certificate. On top of that, we need to pass a certificate identity and a certificate oidc issuer to the verify command. We can pass specific identities and issuers, or use a regexp. We will show both.

1. Use specific issuers and identities:

    1. Team A:

        ~~~sh
        cosign-v2 verify --certificate-identity='team-a@linuxera.org' --certificate-oidc-issuer='https://linuxera.org' --insecure-ignore-sct --insecure-ignore-tlog --cert-chain=$WORKDIR/cafiles/root/root-ca.pem  $IMAGE_ENDPOINT@$TEAM_A_IMAGE_DIGEST
        ~~~

    2. Team B:

        ~~~sh
        cosign-v2 verify --certificate-identity='team-b@linuxera.org' --certificate-oidc-issuer='https://linuxera.org' --insecure-ignore-sct --insecure-ignore-tlog --cert-chain=$WORKDIR/cafiles/root/root-ca.pem  $IMAGE_ENDPOINT@$TEAM_B_IMAGE_DIGEST
        ~~~

2. Use a regexp:

    ~~~sh
    cosign-v2 verify --certificate-identity-regexp='team-.*@linuxera.org' --certificate-oidc-issuer-regexp='https://linuxer.*.org' --insecure-ignore-sct --insecure-ignore-tlog --cert-chain=$WORKDIR/cafiles/root/root-ca.pem $IMAGE_ENDPOINT@$TEAM_A_IMAGE_DIGEST $IMAGE_ENDPOINT@$TEAM_B_IMAGE_DIGEST
    ~~~

3. Do not verify identities or issuers:

    ~~~sh
    cosign-v2 verify --certificate-identity-regexp='.*' --certificate-oidc-issuer-regexp='.*' --insecure-ignore-sct --insecure-ignore-tlog --cert-chain=$WORKDIR/cafiles/root/root-ca.pem  $IMAGE_ENDPOINT@$TEAM_A_IMAGE_DIGEST $IMAGE_ENDPOINT@$TEAM_B_IMAGE_DIGEST
    ~~~

The output for the different validations will be the same as seen in v1.

## Get signature details from our image with Skopeo

We can check the containers signature details by using Skopeo. Let's see an example for Team A image.

1. Get the signature tag:

    ~~~sh
    SIGNATURE_TAG=$(echo $TEAM_A_IMAGE_DIGEST | sed "s/:/-/" | sed "s/$/.sig/")
    ~~~

2. Get the certificate that signed the image:

    ~~~sh
    skopeo inspect docker://$IMAGE_ENDPOINT:$SIGNATURE_TAG | jq -r '.LayersData[].Annotations."dev.sigstore.cosign/certificate"' | sed 's/\\n/\n/g' | grep -v null | openssl x509 -text
    ~~~

3. Get the trust chain for the certificate:

    ~~~sh
    skopeo inspect docker://$IMAGE_ENDPOINT:$SIGNATURE_TAG | jq -r '.LayersData[].Annotations."dev.sigstore.cosign/chain"' | sed 's/\\n/\n/g' | grep -v null
    ~~~

4. Get the signature:

    ~~~sh
    skopeo inspect docker://$IMAGE_ENDPOINT:$SIGNATURE_TAG | jq -r '.LayersData[].Annotations."dev.cosignproject.cosign/signature"'
    ~~~

## References

- [https://docs.sigstore.dev/key_management/signing_with_self-managed_keys/](https://docs.sigstore.dev/key_management/signing_with_self-managed_keys/)
- [https://github.com/sigstore/cosign/issues/2632](https://github.com/sigstore/cosign/issues/2632)
- [https://github.com/sigstore/cosign/issues/2858](https://github.com/sigstore/cosign/issues/2858)
- [https://github.com/sigstore/cosign/issues/3616](https://github.com/sigstore/cosign/issues/3616)

## Generate ASN.1 string for Fulcio Extensions

{{<tip>}}
Thanks to [Hayden Blauzvern](https://github.com/haydentherapper) for providing the information below and helping with the certificate extensions correctness in the certificate generation section.
{{</tip>}}

In Fulcio's cert profile, extensions `1,1` through `1,6` are raw strings. In `1,8` and future Fulcio extensions this changes to ASN.1 encoded string. You can encode a string using the code below.

~~~go
package main

import (
	"encoding/asn1"
	"encoding/base64"
	"fmt"
)

func main() {
	s := "https://linuxera.org"
	a, _ := asn1.Marshal(s)
	fmt.Println(a)
	fmt.Println(base64.StdEncoding.EncodeToString(a))
}
~~~