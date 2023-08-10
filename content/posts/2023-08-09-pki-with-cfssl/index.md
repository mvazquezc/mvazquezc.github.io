---
title:  "PKI with CFSSL"
author: "Mario"
tags: [ "pki", "private ca", "TLS", "cfssl" ]
url: "/pki-with-cfssl"
draft: false
date: 2023-08-09
#lastmod: 2023-08-09
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# PKI with CFSSL

In this post we will learn how to deploy our own Public Key Infrastructure (PKI) by using the [CFSSL](https://github.com/cloudflare/cfssl) tooling. This may be useful if you want to run your own Certificate Authority (CA) in order to issue certificates for your systems and/or users.

## Introduction to CFSSL

CFSSL is a tool set created by [Cloudflare](https://www.cloudflare.com/) and released as Open Source software. Before you continue reading this post I'd suggest reading this [introductory post to PKI and CFSSL by Cloudflare](https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/).

This post assumes you already have basic knowledge on PKI and in how the CFSSL tooling works, if you don't have it, go read the post linked above.

## Installing the CFSSL tooling

In order to install the CFSSL tooling you can go to the [GitHub Releases](https://github.com/cloudflare/cfssl/releases) and download the binaries from there.

{{<warning>}}
Below commands will only work for Linux x86_64 machines.
{{</warning>}}

~~~sh
sudo curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64 -o /usr/local/bin/cfssl
sudo curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64 -o /usr/local/bin/cfssljson
sudo curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.4/multirootca_1.6.4_linux_amd64 -o /usr/local/bin/multirootca
sudo chmod +x /usr/local/bin/{cfssl,cfssljson,multirootca}
~~~

## PKI Organization

For this example, the following organization will be used.

![CFSSL PKI Organization](./cfssl-pki-org.png)

## Creating the Root CA

1. Let's create a folder to store the PKI files:

    ~~~sh
    mkdir -p ~/cafiles/{root,intermediate,config,certificates}
    ~~~

2. Before issuing the Root CA, we need to define its config:

    {{<tip>}}
The expiration is 10 years. You want to have a long expiration time for your Root CA to avoid having to re-roll the PKI too often.
    {{</tip>}}

    ~~~sh
    cat << "EOF" > ~/cafiles/root/root-csr.json
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
          "O": "Linuxera Internal",
          "OU": "CA Services",
          "ST": "Valencia"
        }
      ],
      "ca": {
        "expiry": "87600h"
      }
    }
    EOF
    ~~~

3. Issue the Root CA with `cfssl`:

    ~~~sh
    cfssl gencert -initca ~/cafiles/root/root-csr.json | cfssljson -bare ~/cafiles/root/root-ca
    ~~~

4. At this point, we have our Root CA ready.

## Creating the Intermediate CA

Issuing certificates directly with the Root CA is not advised. You should be issuing intermediary CAs with the Root CA instead. This allows for better organization of your PKI, and in case of a security incident you won't have to re-roll the whole PKI, instead you will only re-roll the affected Intermediate CA.

For this test, we will be issuing only an Intermediate CA. In real scenarios, is pretty common having multiple intermediates, and sometimes these intermediate CAs will be used to issue other intermediate CAs.

1. Define the Intermediate CA config:

    {{<tip>}}
The expiration is 8 years. For Intermediate CAs you also want to have quite a long expiration time.
    {{</tip>}}

    ~~~sh
    cat << "EOF" > ~/cafiles/intermediate/intermediate-csr.json
    {
      "CN": "Linuxera Intermediate CA",
      "key": {
        "algo": "ecdsa",
        "size": 256
      },
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "Linuxera Internal",
          "OU": "Linuxera Internal Intermediate CA",
          "ST": "Valencia"
        }
      ]
    }
    EOF
    ~~~

2. Generate the key for the Intermediate CA:

    ~~~sh
    cfssl genkey ~/cafiles/intermediate/intermediate-csr.json | cfssljson -bare ~/cafiles/intermediate/intermediate-ca
    ~~~

3. Define a CFSSL `signing` profile for the Intermediate CAs. This is done via a config file.

    - `cert sign` and `crl sign`
    - Expiration set to 8 years.
    - CA constraints define that the certificates issued will be used by CAs `is_ca: true` and `max_path_len: 1` limits this intermediate CA to only be able to issue sub-intermediate CAs that cannot issue additional CAs. (This could be allowed with `max_path_len: 0` and `max_path_len_zero: true`).

    ~~~sh
    cat << "EOF" > ~/cafiles/config/config.json
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
    ~~~

4. Sign the Intermediate CA with the Root CA:

    ~~~sh
    cfssl sign -ca ~/cafiles/root/root-ca.pem -ca-key ~/cafiles/root/root-ca-key.pem -config ~/cafiles/config/config.json -profile intermediate ~/cafiles/intermediate/intermediate-ca.csr | cfssljson -bare ~/cafiles/intermediate/intermediate-ca
    ~~~

5. At this point, our Intermediate CA is ready to issue certificates, and we can take our Root CA offline. Usually, the private key gets stored in an HSM and after that it's deleted from the file system.

    ~~~sh
    rm -f ~/cafiles/root/root-ca-key.pem
    ~~~

## Issuing certificates with the Intermediate CA

1. Before issuing the certificate, we will add a new signing profile to our config. We will be defining a `host` signing profile that defines different usages as well as an expiration of 1 year for the certificates.

    ~~~sh
    cat << "EOF" > ~/cafiles/config/config.json
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
          },
          "host": {
            "usages": ["signing", "digital signing", "key encipherment", "server auth"],
            "expiry": "8760h"
          }
        }
      }
    }
    EOF
    ~~~

2. With the profile ready, let's create the certificate config:

    ~~~sh
    cat << "EOF" > ~/cafiles/certificates/my-host-csr.json
    {
      "CN": "testhost.linuxera.org",
      "hosts": ["testhost.linuxera.org", "192.168.122.120"],
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "Linuxera Internal",
          "OU": "Linuxera Internal Hosts"
        }
      ]
    }
    EOF
    ~~~

3. Finally, use the `cfssl` tooling to issue this certificate with the Intermediate CA using the `host` profile:

    ~~~sh
    cfssl gencert -ca ~/cafiles/intermediate/intermediate-ca.pem -ca-key ~/cafiles/intermediate/intermediate-ca-key.pem -config ~/cafiles/config/config.json -profile host ~/cafiles/certificates/my-host-csr.json | cfssljson -bare ~/cafiles/certificates/my-host
    ~~~

4. At this point, we can verify the cert we just created:

    ~~~sh
    openssl x509 -in ~/cafiles/certificates/my-host.pem -noout -subject -issuer -startdate -enddate
    ~~~

    {{<tip>}}
We can see the issuer is our Intermediate CA.
    {{</tip>}}

    ~~~console
    subject=C = ES, L = Valencia, O = Linuxera Internal, OU = Linuxera Internal Hosts, CN = testhost.linuxera.org
    issuer=C = ES, ST = Valencia, L = Valencia, O = Linuxera Internal, OU = Linuxera Internal Intermediate CA, CN = Linuxera Intermediate CA
    notBefore=Aug 9 10:09:00 2023 GMT
    notAfter=Aug  8 10:09:00 2024 GMT
    ~~~

5. If we check the certificate file `~/cafiles/certificates/my-host.pem`, we will see that it only contains the certificate for the host and not the full bundle (Intermediate CAs + Cert). We can generate a full chain cert with the command below:

    {{<tip>}}
Bundles are useful when you intend to use the certificate for an app like a web server, that way you will be sending the certificate + all the intermediate CAs certificates up to the Root CA so the client can verify its trust. Including the Root CA cert is not required, your client should already trust the Root CA, if it doesn't trust it that won't change even if you send it as part of the bundle.
    {{</tip>}}

    ~~~sh
    cfssl bundle -ca-bundle ~/cafiles/root/root-ca.pem -int-bundle ~/cafiles/intermediate/intermediate-ca.pem -cert ~/cafiles/certificates/my-host.pem | cfssljson -bare ~/cafiles/certificates/my-host-fullchain
    ~~~

6. We should have the bundled cert available:

    {{<warning>}}
In some Linux distributions, the previous `cfssl bundle` command may not generate the bundled cert. If that's the case you can get the same result by running `cat ~/cafiles/certificates/my-host.pem ~/cafiles/intermediate/intermediate-ca.pem  > ~/cafiles/certificates/my-host-fullchain.pem`
    {{</warning>}}

    ~~~sh
    cat ~/cafiles/certificates/my-host-fullchain.pem
    ~~~

    ~~~console
    -----BEGIN CERTIFICATE-----
    MII...
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MII...
    -----END CERTIFICATE-----
    ~~~

7. Finally, we could verify the cert:

    ~~~sh
    openssl verify -CAfile <(cat ~/cafiles/root/root-ca.pem  ~/cafiles/intermediate/intermediate-ca.pem) ~/cafiles/certificates/my-host.pem
    ~~~

    ~~~console
    /home/mario/cafiles/certificates/my-host.pem: OK
    ~~~

## Exposing our PKI to remote systems with MultiRootCA

So far, we have been using `cfssl` tooling to issue certificates while connected to a system where our PKI is stored. In real environments, you may need to issue certificates for different people/systems in a more convenient way.

The MultiRootCA program is an authenticated-signer-only server that is used as a remote server for `cfssl` instances. It is intended for:

- Running `cfssl` as a service on servers to generate keys.
- Act as a remote signer to manage the CA keys for issuing certificates.

1. Let's start by issuing a certificate for the multirooca server:

    ~~~sh
    cat << "EOF" > ~/cafiles/certificates/multirootca-server-csr.json
    {
      "CN": "multirootca-server.linuxera.org",
      "hosts": ["multirootca-server.linuxera.org", "192.168.122.153"],
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "Linuxera Internal",
          "OU": "Linuxera Internal Hosts"
        }
      ]
    }
    EOF
    ~~~

    ~~~sh
    cfssl gencert -ca ~/cafiles/intermediate/intermediate-ca.pem -ca-key ~/cafiles/intermediate/intermediate-ca-key.pem -config ~/cafiles/config/config.json -profile host ~/cafiles/certificates/multirootca-server-csr.json  | cfssljson -bare ~/cafiles/certificates/multirootca-server
    ~~~

2. We will secure the signing profiles in our config. We will be defining an `auth_key` that clients requesting a signed certificate must provide in order to get it signed.

    {{<tip>}}
The Auth Key is a 16 byte hexadecimal string. You can generate one by running `openssl rand -hex 16`
    {{</tip>}}

    ~~~sh
    cat << "EOF" > ~/cafiles/config/config.json
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
          },
          "host": {
            "usages": ["signing", "digital signing", "key encipherment", "server auth", "client auth"],
            "expiry": "8760h",
            "auth_key": "default"
          }
        }
      },
      "auth_keys": {
        "default": {
          "key": "b50ed348c4643d34706470f36a646fd4",
          "type": "standard"
        }
      }
    }
    EOF
    ~~~

3. We need to tell multirootca where to find the different certificates for our Intermediate CA:

    ~~~sh
    cat <<EOF > ~/cafiles/config/multiroot-profile.ini
    [linuxeraintermediate]
    private = file://${HOME}/cafiles/intermediate/intermediate-ca-key.pem
    certificate = ${HOME}/cafiles/intermediate/intermediate-ca.pem
    config = ${HOME}/cafiles/config/config.json
    EOF
    ~~~

4. Finally, we can run the multirootca server:

    ~~~sh
    multirootca -a 0.0.0.0:8000 -l default -roots ~/cafiles/config/multiroot-profile.ini -tls-cert ~/cafiles/certificates/multirootca-server.pem -tls-key ~/cafiles/certificates/multirootca-server-key.pem
    ~~~

5. A more appropriate way of running the server would be using a systemd service:

    ~~~sh
    cat <<EOF | sudo tee /etc/systemd/system/multirootca.service
    [Unit]
    Description=CFSSL PKI Certificate Authority
    After=network.target

    [Service]
    User=${USER}
    ExecStart=/usr/local/bin/multirootca -a 0.0.0.0:8000 -l linuxeraintermediate -roots ${HOME}/cafiles/config/multiroot-profile.ini -tls-cert ${HOME}/cafiles/certificates/multirootca-server.pem -tls-key ${HOME}/cafiles/certificates/multirootca-server-key.pem
    Restart=on-failure
    Type=simple

    [Install]
    WantedBy=multi-user.target
    EOF
    ~~~

    ~~~sh
    sudo systemctl daemon-reload
    sudo systemctl enable multirootca --now
    ~~~

## Requesting certificates to the multirootca

Now that the Intermediate CA has been exposed with the multirootca program, we can go ahead and request it to sign some certificates. We can do this from a remote location, or from the same server where multirootca is running.

1. Generate a certificate config:

    ~~~sh
    cat << "EOF" > my-cert-request-csr.json
    {
      "CN": "myserver.linuxera.org",
      "hosts": ["myserver.linuxera.org", "192.168.122.222"],
      "names": [
        {
          "C": "ES",
          "L": "Valencia",
          "O": "Linuxera Internal",
          "OU": "Linuxera Internal Hosts"
        }
      ]
    }
    EOF
    ~~~

2. Generate a request profile. This is required for `cfssl` to know how to request the certificate to the multirootca:

    {{<warning>}}
We need to define the Auth key, otherwise multirootca will not sign our certificate. And the location of the multirootca server, we can use IP:Port or DNS:Port.
    {{</warning>}}

    ~~~sh
    cat <<EOF > request-profile.json
    {
      "signing": {
        "default": {
          "auth_remote": {
            "remote": "ca_server",
            "auth_key": "default"
          }
        }
      },
      "auth_keys": {
        "default": {
          "key": "b50ed348c4643d34706470f36a646fd4",
          "type": "standard"
        }
      },
      "remotes": {
        "ca_server": "https://multirootca-server.linuxera.org:8000"
      }
    }
    EOF
    ~~~

3. Finally, we send the request by specifying the `host` profile, which is the one we will be using for signing host certificates:

    {{<warning>}}
We need to specify the Intermediate CA certificate via the `-tls-remote-ca` flag.
    {{</warning>}}

    ~~~sh
    cfssl gencert -config ./request-profile.json -tls-remote-ca ./intermediate-ca.pem -profile host ./my-cert-request-csr.json | cfssljson -bare my-cert
    ~~~

    ~~~console
    2023/08/09 11:43:15 [INFO] generate received request
    2023/08/09 11:43:15 [INFO] received CSR
    2023/08/09 11:43:15 [INFO] generating key: ecdsa-256
    2023/08/09 11:43:15 [INFO] encoded CSR
    2023/08/09 11:43:15 [INFO] Using trusted CA from tls-remote-ca: ./intermediate-ca.pem
    ~~~

4. We should have a valid certificate now:

    ~~~sh
    openssl x509 -in ./my-cert.pem -noout -subject -issuer -startdate -enddate
    ~~~

    ~~~console
    subject=C = ES, L = Valencia, O = Linuxera Internal, OU = Linuxera Internal Hosts, CN = myserver.linuxera.org
    issuer=C = ES, ST = Valencia, L = Valencia, O = Linuxera Internal, OU = Linuxera Internal Intermediate CA, CN = Linuxera Intermediate CA
    notBefore=Aug 9 11:38:00 2023 GMT
    notAfter=Aug  8 11:38:00 2024 GMT
    ~~~

## Closing Thoughts

We have seen how to run our own PKI with the CFSSL tooling, in the [next post](https://linuxera.org/integrating-cert-manager-with-cfssl-multirootca) we will see how to leverage this PKI from Kubernetes by using [cert-manager](https://cert-manager.io/).

## Useful Resources

- [https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/](https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/)
- [https://www.ekervhen.xyz/posts/private-ca-with-cfssl/](https://www.ekervhen.xyz/posts/private-ca-with-cfssl/)
