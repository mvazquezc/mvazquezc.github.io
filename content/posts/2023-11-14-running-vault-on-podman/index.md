---
title:  "Running Vault on Podman"
author: "Mario"
tags: [ "vault", "secrets", "hashicorp", "security", "podman" ]
url: "/running-vault-on-podman"
draft: false
date: 2023-11-14
#lastmod: 2023-08-10
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Running Vault on Podman

This post explains how to run a local Vault deployment on Podman for **non-production** use. I typically use this setup for my lab environments.

This setup was tested with:

- Podman v4.7.2
- Podman-compose v1.0.6
- Vault v1.15.2

## Prerequisites

1. Install the vault client, you can get the binary for your O.S [here](https://developer.hashicorp.com/vault/install).

    ~~~sh
    curl -L https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip -o /tmp/vault.zip
    unzip /tmp/vault.zip && rm -f /tmp/vault.zip
    sudo mv vault /usr/local/bin/
    ~~~

2. Generate folder for storing the configs, data, and certs.

    ~~~sh
    mkdir -p ${HOME}/vault-server/data/{certs,storage}
    ~~~

3. Generate self-signed cert.

    {{<attention>}}
Make sure to edit certificate details to match your environment.
    {{</attention>}}

    ~~~sh
    openssl req -new -newkey rsa:2048 -sha256 -days 3650 -nodes -x509 -extensions v3_ca -keyout ${HOME}/vault-server/data/certs/private.key -out ${HOME}/vault-server/data/certs/public.crt -subj "/C=ES/ST=Valencia/L=Valencia/O=Linuxera/OU=Blog/CN=vault.linuxera.org" -addext "subjectAltName = DNS:vault.linuxera.org,DNS:192.168.122.1"
    ~~~

4. Configure privileges.

    ~~~sh
    sudo chmod 777 data/storage
    sudo chmod 744 data/certs/{private.key,public.crt}
    ~~~

5. At this point you can go for [Vault in-memory](#vault-storage-in-memory) or for [Vault in-disk](#vault-storage-in-disk) depending on your data persistency preference.

## Vault storage in-memory

1. Generate Vault server config.

    ~~~sh
    cat <<EOF > ${HOME}/vault-server/data/in-memory-config.hcl
    ui                = true
    default_lease_ttl = "168h"
    max_lease_ttl     = "720h"
    api_addr          = "https://127.0.0.1:8201"
    disable_mlock     = true

    storage "inmem" {}

    listener "tcp" {
      address         = "0.0.0.0:8201"
      tls_disable     = "0"
      tls_cert_file   = "/data/certs/public.crt"
      tls_key_file    = "/data/certs/private.key"
    }
    EOF
    ~~~

2. Generate podman-compose config.

    ~~~sh
    cat <<EOF > ${HOME}/vault-server/vault-compose-in-memory.yaml
    version: '3.6'
    services:
      vault:
        image: docker.io/hashicorp/vault:1.15.2
        container_name: vault
        restart: on-failure:10
        ports:
          - "8201:8201"
        environment:
          VAULT_ADDR: 'https://0.0.0.0:8201'
        cap_add:
          - IPC_LOCK
        volumes:
          - $HOME/vault-server/data:/data:rw,Z
        healthcheck:
          retries: 5
        command: server -config /data/in-memory-config.hcl
    EOF
    ~~~

3. Run. Once the server is up you can continue reading the section [Initialize Vault Server](#initialize-vault-server).

    {{<attention>}}
The secrets stored in this Vault instance will be lost once the server is stopped.
    {{</attention>}}

    ~~~sh
    podman-compose -f $HOME/vault-server/vault-compose-in-memory.yaml up -d
    ~~~

4. Stop.

    ~~~sh
    podman-compose -f $HOME/vault-server/vault-compose-in-memory.yaml down
    ~~~

## Vault storage in-disk

1. Generate Vault server config.

    ~~~sh
    cat <<EOF > ${HOME}/vault-server/data/persistent-config.hcl
    ui                = true
    default_lease_ttl = "168h"
    max_lease_ttl     = "720h"
    api_addr          = "https://127.0.0.1:8201"
    disable_mlock     = true

    storage "file" { 
      path            = "/data/storage"
    }

    listener "tcp" {
      address         = "0.0.0.0:8201"
      tls_disable     = "0"
      tls_cert_file   = "/data/certs/public.crt"
      tls_key_file    = "/data/certs/private.key"
    }
    EOF
    ~~~

2. Generate podman-compose config.

    ~~~sh
    cat <<EOF > ${HOME}/vault-server/vault-compose-file-storage.yaml
    version: '3.6'
    services:
      vault:
        image: docker.io/hashicorp/vault:1.15.2
        container_name: vault
        restart: on-failure:10
        ports:
          - "8201:8201"
        environment:
          VAULT_ADDR: 'https://0.0.0.0:8201'
        cap_add:
          - IPC_LOCK
        volumes:
          - $HOME/vault-server/data:/data:rw,Z
        healthcheck:
          retries: 5
        command: server -config /data/persistent-config.hcl
    EOF
    ~~~

3. Run. Once the server is up you can continue reading the section [Initialize Vault Server](#initialize-vault-server).

    ~~~sh
    podman-compose -f $HOME/vault-server/vault-compose-file-storage.yaml up -d
    ~~~

4. Stop.

    ~~~sh
    podman-compose -f $HOME/vault-server/vault-compose-file-storage.yaml down
    ~~~

## Initialize Vault Server

1. Initialize the Vault.

    {{<tip>}}
You can export the `VAULT_SKIP_VERIFY` env var with its value set to `true` to ignore self-signed certs.
    {{</tip>}}

    ~~~sh
    export VAULT_ADDR='https://192.168.122.1:8201'
    vault operator init | grep -E "Unseal Key|Initial Root" > $HOME/vault-server/init-keys.txt
    ~~~

2. Unseal the Vault and login.

    ~~~sh
    UNSEAL_KEY1=$(grep "Key 1" $HOME/vault-server/init-keys.txt | awk -F ": " '{print $2}')
    UNSEAL_KEY2=$(grep "Key 2" $HOME/vault-server/init-keys.txt | awk -F ": " '{print $2}')
    UNSEAL_KEY3=$(grep "Key 3" $HOME/vault-server/init-keys.txt | awk -F ": " '{print $2}')
    VAULT_TOKEN=$(grep "Root Token" $HOME/vault-server/init-keys.txt | awk -F ": " '{print $2}')
    vault operator unseal $UNSEAL_KEY1
    vault operator unseal $UNSEAL_KEY2
    vault operator unseal $UNSEAL_KEY3
    vault login $VAULT_TOKEN   
    ~~~

3. Enable the kv [secrets engine v2](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2).

    ~~~sh
    vault secrets enable -version=2 kv
    ~~~

4. Configure the ACL for our user.

    ~~~sh
    cat <<EOF > $HOME/vault-server/team1.hcl
    path "kv/data/team1/*" {
      capabilities = ["create", "update", "read", "delete", "list"]
    }
    EOF
    
    vault policy write team1-policy $HOME/vault-server/team1.hcl
    ~~~

5. Enable userpass auth and add a user.

    ~~~sh
    vault auth enable userpass
    vault write auth/userpass/users/mario password=str0ngp4ss policies=team1-policy
    ~~~

6. Login with the user.

    ~~~sh
    vault login -method=userpass username=mario password=str0ngp4ss
    ~~~

7. Put a key/value into the Vault.

    ~~~sh
    vault kv put -mount=kv team1/mysecret foo=a bar=b
    ~~~

8. Get a key/value from the Vault.

    ~~~sh
    vault kv get -mount=kv team1/mysecret
    ~~~

9. Access the WebUI by pointing your browser to the IP where podman is exposing port 8201. For example [https://192.168.122.1:8201/ui](https://192.168.122.1:8201/ui).

## Useful Resources

- [Vault Docs](https://developer.hashicorp.com/vault/docs)
- [Vault Introduction](https://developer.hashicorp.com/vault/docs/what-is-vault)