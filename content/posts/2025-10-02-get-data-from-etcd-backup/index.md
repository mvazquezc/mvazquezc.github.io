---
title:  "Get data from an etcd backup (encrypted or not!)"
author: "Mario"
tags: [ "etcd", "kubernetes", "etcd", "encrypted etcd", "backup", "restore" ]
url: "/rag-beginners-guide"
draft: false
date: 2025-10-02
lastmod: 2025-10-02
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Get data from an etcd backup (encrypted or not!)

The following post aims to provide a step by step procedure to recover data from an etcd snapshot (even if it’s encrypted) from a Kubernetes cluster.

The post targets the following use cases:

- We have a non-encrypted etcd snapshot file, and we want to get some data from it.
- We have an encrypted etcd snapshot file, and we want to get some data from it.

For non-encrypted etcd snapshots, you can read the data with the etcdctl tool directly. In this post, since I have to cover encrypted as well, I'll describe how to consume the restored data from a temporary local kube-apiserver.

## Prereqs

1. Download etcd tooling and Kubernetes binaries (you want to use the same versions as the ones used in the cluster where the backup was taken, it may work using different ones thought).

    ~~~sh
    mkdir -p /var/tmp/etcd-restore-tests/bin
    cd /var/tmp/etcd-restore-tests
    curl -LO https://github.com/etcd-io/etcd/releases/download/v3.5.18/etcd-v3.5.18-linux-amd64.tar.gz
    curl -LO https://dl.k8s.io/v1.31.11/kubernetes-server-linux-amd64.tar.gz
    tar xvf etcd-v3.5.18-linux-amd64.tar.gz -C ./bin/ etcd-v3.5.18-linux-amd64/etcd etcd-v3.5.18-linux-amd64/etcdutl --transform='s|.*/||'
    tar xvf kubernetes-server-linux-amd64.tar.gz -C ./bin/ kubernetes/server/bin/kube-apiserver --transform='s|.*/||'
    ~~~

2. Generate the required certificates for the temporary kube-apiserver.

    ~~~sh
    openssl req -x509 -newkey rsa:2048 -nodes -keyout ca.key -out ca.crt -subj "/CN=etcd-ca" -days 3650
    openssl req -newkey rsa:2048 -nodes -keyout apiserver.key -out apiserver.csr -subj "/CN=127.0.0.1"
    openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out apiserver.crt -days 3650 -extensions v3_req -extfile <(printf "[v3_req]\nsubjectAltName=IP:127.0.0.1,DNS:localhost")
    openssl req -newkey rsa:2048 -nodes -keyout admin.key -out admin.csr -subj "/CN=admin/O=system:masters"
    openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 3650
    ~~~

## Non-Encrypted Scenario

1. Check etcd snapshot consistency.

    ~~~sh
    ./bin/etcdutl snapshot status snapshot_2025-10-02_092618.db -w table

    +----------+----------+------------+------------+
    |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
    +----------+----------+------------+------------+
    | 7804fc7a |  9078653 |      11199 |      94 MB |
    +----------+----------+------------+------------+
    ~~~

2. Restore the snapshot data to a local folder.

    ~~~sh
    ./bin/etcdutl snapshot restore snapshot_2025-10-02_092618.db --data-dir etcd-data

    2025-10-02T11:26:59Z    info    snapshot/v3_snapshot.go:265     restoring snapshot      {"path": "snapshot_2025-10-02_092618.db", "wal-dir": "etcd-data/member/wal", "data-dir": "etcd-data", "snap-dir": "etcd-data/member/snap", "initial-memory-map-size": 10737418240}
    2025-10-02T11:26:59Z    info    membership/store.go:141 Trimming membership information from the backend...
    2025-10-02T11:26:59Z    info    membership/cluster.go:421       added member    {"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["http://localhost:2380"]}
    2025-10-02T11:26:59Z    info    snapshot/v3_snapshot.go:293     restored snapshot       {"path": "snapshot_2025-10-02_092618.db", "wal-dir": "etcd-data/member/wal", "data-dir": "etcd-data", "snap-dir": "etcd-data/member/snap", "initial-memory-map-size": 10737418240}
    ~~~

3. Start a single unauthenticated etcd instance with this data.

    ~~~sh
    ./bin/etcd --data-dir etcd-data \
      --listen-client-urls http://127.0.0.1:2379 \
      --advertise-client-urls http://127.0.0.1:2379
    ~~~

4. Start a local kube-apiserver and connect it to the etcd instance.

    > **NOTE**: The `--etcd-prefix` parameter may be different for your cluster. Some Kubernetes distributions change this.

    ~~~sh
    ./bin/kube-apiserver \
      --etcd-servers=http://127.0.0.1:2379 \
      --authorization-mode=AlwaysAllow \
      --secure-port=6443 \
      --tls-cert-file=./apiserver.crt \
      --tls-private-key-file=./apiserver.key \
      --client-ca-file=./ca.crt \
      --service-account-issuer=https://kubernetes.default.svc \
      --service-account-key-file=./apiserver.key \
      --service-account-signing-key-file=./apiserver.key \
      --etcd-prefix /kubernetes.io
    ~~~

5. Get the data you want.

    ~~~sh
    kubectl --kubeconfig /dev/null \
      --server=https://127.0.0.1:6443 \
      --certificate-authority=./ca.crt \
      --client-certificate=./admin.crt \
      --client-key=./admin.key \
      -n kube-system get secret \
      pull-secret -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

    {"auths":...<redacted>
    ~~~

## Encrypted Scenario

1. Check snapshot consistency.

    ~~~sh
    ./bin/etcdutl snapshot status snapshot_2025-10-02_102135.db -w table

    +----------+----------+------------+------------+
    |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
    +----------+----------+------------+------------+
    | 32c89b47 |  9119744 |      16292 |     102 MB |
    +----------+----------+------------+------------+
    ~~~

2. Restore the snapshot data to a local folder.

    ~~~sh
    ./bin/etcdutl snapshot restore snapshot_2025-10-02_102135.db --data-dir etcd-data-encrypted

    2025-10-02T11:36:57Z    info    snapshot/v3_snapshot.go:265     restoring snapshot      {"path": "snapshot_2025-10-02_102135.db", "wal-dir": "etcd-data-encrypted/member/wal", "data-dir": "etcd-data-encrypted", "snap-dir": "etcd-data-encrypted/member/snap", "initial-memory-map-size": 10737418240}
    2025-10-02T11:36:57Z    info    membership/store.go:141 Trimming membership information from the backend...
    2025-10-02T11:36:57Z    info    membership/cluster.go:421       added member    {"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["http://localhost:2380"]}
    2025-10-02T11:36:57Z    info    snapshot/v3_snapshot.go:293     restored snapshot       {"path": "snapshot_2025-10-02_102135.db", "wal-dir": "etcd-data-encrypted/member/wal", "data-dir": "etcd-data-encrypted", "snap-dir": "etcd-data-encrypted/member/snap", "initial-memory-map-size": 10737418240}
    ~~~

3. Start a single unauthenticated etcd instance with this data.

    ~~~sh
    ./bin/etcd --data-dir etcd-data-encrypted \
      --listen-client-urls http://127.0.0.1:2379 \
      --advertise-client-urls http://127.0.0.1:2379
    ~~~

4. Get the encryption provider config.

    > NOTE: This file is the one that contains the encryption config configuration from the cluster where the etcd snapshot was taken.

    ~~~sh
    cat ./encryption-config

    {"kind":"EncryptionConfiguration","apiVersion":"apiserver.config.k8s.io/v1","resources":[{"resources":["configmaps"],"providers":[{"aesgcm":{"keys":[{"name":"1","secret":"sTD5PvlTrSwtq1O5PSHEuQdggYYXmyNhnjTtJbHZsP0="}]}},{"identity":{}}]},{"resources":["secrets"],"providers":[{"aesgcm":{"keys":[{"name":"1","secret":"sTD5PvlTrSwtq1O5PSHEuQdggYYXmyNhnjTtJbHZsP0="}]}},{"identity":{}}]}]}
    ~~~

5. Start a local kube-apiserver wit the encryption config and connect it to the local etcd instance.

    ~~~sh
    ./bin/kube-apiserver \
      --etcd-servers=http://127.0.0.1:2379 \
      --encryption-provider-config=./encryption-config \
      --authorization-mode=AlwaysAllow \
      --secure-port=6443 \
      --tls-cert-file=./apiserver.crt \
      --tls-private-key-file=./apiserver.key \
      --client-ca-file=./ca.crt \
      --service-account-issuer=https://kubernetes.default.svc \
      --service-account-key-file=./apiserver.key \
      --service-account-signing-key-file=./apiserver.key \
      --etcd-prefix /kubernetes.io
    ~~~

6. Get the data you want.

    ~~~sh
    kubectl --kubeconfig /dev/null \
      --server=https://127.0.0.1:6443 \
      --certificate-authority=./ca.crt \
      --client-certificate=./admin.crt \
      --client-key=./admin.key \
      -n kube-system get secret \
      pull-secret -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

    {"auths":...<redacted>
    ~~~
