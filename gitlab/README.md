# Deploy GitLab to Google Container with Kubernetes

## Overview

In order to deploy GitLab as an application, we need to deploy 3 components:

1.  [PostgreSQL](https://www.postgresql.org/). Gitlab uses a database
    backend to store its data. GitLab HQ recommends PostgreSQL over
    MySQL.
2.  [Redis](http://redis.io/). GitLab uses redis as its key-value
    store.
3.  [GitLab](https://about.gitlab.com/). The last piece is GitLab
    itself.

Each of them will be in their own Pod and will have their own service
for exposing them. All of the pods will be put in a namespace called
"gitlab".

## Steps

1.  Create the namespace `gitlab`.

    ```bash
    kubectl create namespace gitlab
    ```
    
    This can be verified by running
    
    ```bash
    $ kubectl get namespaces
    NAME          STATUS    AGE
    default       Active    10h
    gitlab        Active    10m
    kube-system   Active    10h

    ```

2.  **PostgreSQL**
    
    















