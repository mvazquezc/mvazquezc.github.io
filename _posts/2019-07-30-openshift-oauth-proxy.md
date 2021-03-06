---
layout: post
title:  "Using OpenShift OAuth Proxy to secure your Applications on OpenShift"
author: "Mario"
categories: [ okd, origin, containers, kubernetes, openshift, oauth, proxy ]
featured: false
image: assets/images/2019-07-30-openshift-oauth-secure-apps.jpg
image-author: "Micah Williams"
image-author-link: "https://unsplash.com/@mr_williams_photography"
image-source: "Unsplash"
image-source-link: "https://unsplash.com/photos/lmFJOx7hPc4"
permalink: /oauth-proxy-secure-applications-openshift/
hidden: false
---

# What is OAuth Proxy

A reverse proxy and static file server that provides authentication and authorization to an OpenShift OAuth server or Kubernetes master supporting the 1.6+ remote
authorization endpoints to validate access to content. It is intended for use withing OpenShift clusters to make it easy to run both end-user and infrastructure
services that do not provider their own authentication.

[[Source]](https://github.com/openshift/oauth-proxy)


## Securing an Application with OAuth Proxy

In this blog post we are going to deploy OAuth Proxy in front of a [simple application](https://github.com/mvazquezc/reverse-words).

We will go through the following scenarios:

1. Application deployed without OAuth Proxy
2. Application + OAuth Proxy limiting access to authenticated users
3. Application + OAuth Proxy limiting access to specific users

After following these three scenarios you will be able to secure applications on **OpenShift** and **Kubernetes** using the _OAuth Proxy_.

## Scenario 1 - Deploying the Application without OAuth Proxy

Not a big deal, just a regular deployment.

### Required files

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reverse-words
  labels:
    name: reverse-words
spec:
  replicas: 1
  selector:
    matchLabels:  
      name: reverse-words
  template:
    metadata:
      labels:
        name: reverse-words
    spec:
      containers:
        - name: reverse-words
          image: quay.io/mavazque/reversewords:latest 
          imagePullPolicy: Always
          ports:
            - name: reverse-words
              containerPort: 8080
              protocol: TCP
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: reverse-words
  name: reverse-words
spec:
  ports:
  - name: app
    port: 8080
    protocol: TCP
    targetPort: reverse-words
  selector:
    name: reverse-words
  sessionAffinity: None
  type: ClusterIP
```

### Deploy

```sh
oc create namespace reverse-words
oc -n reverse-words create -f deployment.yaml
oc -n reverse-words create -f service.yaml
oc -n reverse-words create route edge reverse-words --service=reverse-words --port=app --insecure-policy=Redirect
```

Now we should be able to reach our application without providing any authentication details.

```sh
curl https://$(oc -n reverse-words get route reverse-words -o jsonpath='{.status.ingress[*].host}') -X POST -d '{"word": "PALC"}'

{"reverse_word":"CLAP"}
```

Let's go ahead and secure our application to be accessible only to authenticated users.

## Scenario 2 - Limiting Access to Authenticated Users

In order to use OAuth Proxy we need a couple of things:

1. Create a session `Secret` used by OAuth Proxy to encrypt the login cookie
2. A `ServiceAccount` used by our application and annotated to redirect traffic to a given route to the OAuth Proxy
3. TLS Certificates for be used by the proxy (We will leverage OpenShift TLS service serving certificate)
4. Modify our `Deployment` to include OAuth Proxy container
5. Modify our `Service` to include OAuth Proxy port and annotation for certificate creation

### Prerequisites

1. Create the Secret

    ```sh
    oc -n reverse-words create secret generic reversewords-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
    ```
2. Create and annotate the ServiceAccount

    ```sh
    oc -n reverse-words create serviceaccount reversewords
    oc -n reverse-words annotate serviceaccount reversewords serviceaccounts.openshift.io/oauth-redirectreference.reversewords='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"reverse-words-authenticated"}}'
    ```
3. Modify the deployment

    **deployment.yaml**
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: reverse-words
      labels:
        name: reverse-words
    spec:
      replicas: 1
      selector:
        matchLabels:  
          name: reverse-words
      template:
        metadata:
          labels:
            name: reverse-words
        spec:
          containers:
            - name: reverse-words
              image: quay.io/mavazque/reversewords:latest 
              imagePullPolicy: Always
              ports:
                - name: reverse-words
                  containerPort: 8080
                  protocol: TCP
            - name: oauth-proxy 
              args:
                - -provider=openshift
                - -https-address=:8888
                - -http-address=
                - -email-domain=*
                - -upstream=http://localhost:8080
                - -tls-cert=/etc/tls/private/tls.crt
                - -tls-key=/etc/tls/private/tls.key
                - -client-secret-file=/var/run/secrets/kubernetes.io/serviceaccount/token
                - -cookie-secret-file=/etc/proxy/secrets/session_secret
                - -openshift-service-account=reversewords
                - -openshift-ca=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                - -skip-auth-regex=^/metrics
              image: quay.io/openshift/origin-oauth-proxy:4.1
              imagePullPolicy: IfNotPresent
              ports:
                - name: oauth-proxy
                  containerPort: 8888    
                  protocol: TCP
              volumeMounts:
                - mountPath: /etc/tls/private
                  name: secret-reversewords-tls
                - mountPath: /etc/proxy/secrets
                  name: secret-reversewords-proxy
          serviceAccountName: reversewords
          volumes:
            - name: secret-reversewords-tls
              secret:
                defaultMode: 420
                secretName: reversewords-tls
            - name: secret-reversewords-proxy
              secret:
                defaultMode: 420
                secretName: reversewords-proxy
    ```
4. Modify the service

    **service.yaml**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.alpha.openshift.io/serving-cert-secret-name: reversewords-tls
      labels:
        name: reverse-words
      name: reverse-words
    spec:
      ports:
      - name: proxy
        port: 8888
        protocol: TCP
        targetPort: oauth-proxy
      - name: app
        port: 8080
        protocol: TCP
        targetPort: reverse-words
      selector:
        name: reverse-words
      sessionAffinity: None
      type: ClusterIP
    ```

### Deploy

```sh
oc -n reverse-words apply -f service.yaml
oc -n reverse-words apply -f deployment.yaml
oc -n reverse-words create route reencrypt reverse-words-authenticated --service=reverse-words --port=proxy --insecure-policy=Redirect
```

Now we should be able to reach our application, let's see what happens when we try to access without providing any authentication details.

```sh
curl -I https://$(oc -n reverse-words get route reverse-words-authenticated -o jsonpath='{.status.ingress[*].host}')

HTTP/1.1 403 Forbidden
Set-Cookie: _oauth_proxy=; Path=/; Domain=reverse-words-authenticated-reverse-words.apps.okd.linuxlabs.org; Expires=Tue, 30 Jul 2019 15:08:22 GMT; HttpOnly; Secure
Date: Tue, 30 Jul 2019 16:08:22 GMT
Content-Type: text/html; charset=utf-8
Set-Cookie: 24c429aac95893475d1e8c1316adf60f=255a07dc5b1af1d2d01721678f463c09; path=/; HttpOnly; Secure
```

Now we are going to access to our application using our browser and authenticating with a valid user:

![Scenario 2 Login](https://linuxera.org/assets/post_resources/2019-07-30-openshift-oauth-proxy/scenario2-login.gif)

## Scenario 3 - Limiting Access to Specific Authenticated Users

In this scenario we are going to modify the OAuth Proxy configuration so only users with access to the _reverse-words_ `Namespace` can access the application.

### Prerequisites

1. Modify the deployment. Add the line below to the oauth-proxy container arguments

    ```sh
    oc -n reverse-words edit deployment reverse-words
    ```
    ```yaml
    <OMITTED OUTPUT>
    - -openshift-service-account=reversewords
    - -openshift-sar={"resource":"namespaces","resourceName":"reverse-words","namespace":"reverse-words","verb":"get"}
    <OMITTED OUTPUT>
    ```

### Deploy

The deployment should be updated and the OAuth Proxy should be configured to allow access only to users with access to the _reverse-words_ `namespace`.

As we did before, let's try to access with `user1` to our application:

![Scenario 3 Failed Login](https://linuxera.org/assets/post_resources/2019-07-30-openshift-oauth-proxy/scenario3-login-failed.gif)

It failed! That is because `user1` does not have access to the _reverse-words_ namespace, let's grant access to `user2` and try to login again.

```sh
oc -n reverse-words adm policy add-role-to-user view user2
```

Back on the browser:

![Scenario 3 Correct Login](https://linuxera.org/assets/post_resources/2019-07-30-openshift-oauth-proxy/scenario3-login-correct.gif)


# Final Thoughts

This is just an sneak peek of what OAuth Proxy can do, if you want to know more you can check the project's repository [here](https://github.com/openshift/oauth-proxy).

Keep in mind that OAuth Proxy is not intended to replace your application authentication and authorization mechanisms, it is just another security layer on top of your applications.
