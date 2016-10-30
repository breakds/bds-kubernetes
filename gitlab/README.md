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
    
    *   Create the PostgreSQL service under namespace `gitlab`.
    
        ```bash
        $ kubectl create -f postgresql-svc.yml
        service "gitlab-postgresql" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get services --namespace=gitlab
        NAME                CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
        gitlab-postgresql   10.3.252.137   <none>        5432/TCP   21m
        ```
        
        Note that `kubectl` by default will only look at the namespace
        `default`. Therefore, every command we issued such as `kubectl
        get services` will only look at the namespace `deafult`. One
        way to deal with this is to specify the namespace for each
        command, as above. Another way is to set the context of
        kubectl so that it will look at the namespace `gitlab` by
        default. This can be done as following:
        
        ```bash
        $ kubectl config current-context
        gke_personal-servers-147616_us-west1-b_gitlab-cluster
        ```
        
        This gives you the name of the current context. Now we can
        attach the namespace `gitlab` to this context:
        
        ```bash
        $ kubectl config set-context gke_personal-servers-147616_us-west1-b_gitlab-cluster --namespace=gitlab
        ```
        
        This is recommended as typing `--namespace=gitlab` every time
        we issue an `kubectl` command is cumbersome.
    
    *   Deploy PostgreSQL Pod.
    
        ```bash
        $ kubectl create -f postgresql-deployment.yml
        deployment "gitlab-ostgresql" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get deployments
        NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
        gitlab-postgresql   1         1         1            1           11m
        ```
        
3.  **Redis**

    *   Create the Redis service under namespace `gitlab`.
    
        ```bash
        $ kubectl create -f redis-svc.yml
        service "gitlab-redis" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get services
        NAME                CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
        gitlab-postgresql   10.3.252.137   <none>        5432/TCP   2h
        gitlab-redis        10.3.241.15    <none>        6379/TCP   22s
        ```

    *   Deploy Redis Pod.
    
        ```bash
        $ kubectl create -f redis-deployment.yml 
        deployment "gitlab-redis" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get pods
        NAME                                 READY     STATUS    RESTARTS   AGE
        gitlab-postgresql-2981213592-cezcd   1/1       Running   0          2h
        gitlab-redis-3473415130-563lx        1/1       Running   0          29s
        ```
        
4.  **GitLab**

    *   Create the GitLab service under the namespace `gitlab`.
    
        ```bash
        $ kubectl create -f gitlab-svc.yml 
        service "gitlab" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get services
        NAME                CLUSTER-IP     EXTERNAL-IP      PORT(S)         AGE
        gitlab              10.3.242.247   104.198.13.109   80/TCP,22/TCP   55s
        gitlab-postgresql   10.3.252.137   <none>           5432/TCP        3h
        gitlab-redis        10.3.241.15    <none>           6379/TCP        1h
        ```
        
        Note that since we specified the type as `LoadBalancer`, this
        service is assigned an external IP.
        
    *   Deploy GitLab Pod.
    
        Note that, you might want to change the environment variables
        
        *   TZ: Set to your timezone, such as `America/Los_Angeles`.
        *   GITLAB_TIMEZONE: See
            [all valid values](http://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html).
    
        ```bash
        $ kubectl create -f gitlab-deployment.yml
        deployment "gitlab" created
        ```

    










