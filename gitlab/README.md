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
    $ kubectl create namespace gitlab
    namespace "gitlab" created
    ```
    
    This can be verified by running
    
    ```bash
    $ kubectl get namespaces
    NAME          STATUS    AGE
    default       Active    10h
    gitlab        Active    10m
    kube-system   Active    10h
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
    
2.  Persistent Disk for PostgreSQL

    We absolutely do not want to lose our data if postgresql is down
    or being restarted for upgrade. Therefore, we would like it to
    store the data on a
    [persistent disk](https://cloud.google.com/compute/docs/disks/).

    The following command creates a persistent disk in Google Compute
    Engine with capacity of 200GB.
    
    ```bash
    $ cloud compute disks create --size 200GB gitlab-postgresql-disk
    ```
    
    Note that I chose `gitlab-postgresql-disk` as the name for the
    persistent disk.
    
    To make it available for Pods to attach, we need to create the
    persistent volume and the persistent volume claim.
    
    This can be done with the following commands
    
    ```bash
    $ kubectl create -f gitlab-postgresql-disk.yml
    persistentvolume "gitlab-postgresql-disk" created
    $ kubectl create -f gitlab-postgresql-disk-claim.yml 
    persistentvolumeclaim "gitlab-postgresql-disk-claim" created
    ```
    
    This can now be verified:
    
    ```bash
    $ kubectl get pv
    NAME                     CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                                 REASON    AGE
    gitlab-postgresql-disk   200Gi      RWO           Retain          Bound     gitlab/gitlab-postgresql-disk-claim             9m
    $ kubectl get pvc
    NAME                           STATUS    VOLUME                   CAPACITY   ACCESSMODES   AGE
    gitlab-postgresql-disk-claim   Bound     gitlab-postgresql-disk   200Gi      RWO           10s
    ```

    The instructions are mainly from Eric Oestrich's excellent
    [blog post](https://blog.oestrich.org/2015/08/running-postgres-inside-kubernetes/).
    
3.  **PostgreSQL**
    
    *   Create the PostgreSQL service under namespace `gitlab`.
    
        ```bash
        $ kubectl create -f postgresql-svc.yml
        service "gitlab-postgresql" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get services
        NAME                CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
        gitlab-postgresql   10.3.252.137   <none>        5432/TCP   21m
        ```
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
        
4.  **Redis**

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
        
5.  **GitLab**

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
    
        The deployment of GitLab is a bit more complicated than the
        other ones because there are a lot of options to be set before
        you can safely deploy it. Something notable are:
        
        *   TZ: Set to your timezone, such as `America/Los_Angeles`.
        *   GITLAB_TIMEZONE: See
            [all valid values](http://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html).
        *   GITLAB_ROOT_PASSWORD: If not set, this can lead to the bug
            described
            [here](https://github.com/sameersbn/docker-gitlab/issues/657).
        *   GITLAB\_SECRETS\_DB\_KEY\_BASE,
            GITLAB\_SECRETS\_SECRET\_KEY\_BASE,
            GITLAB\_SECRETS\_OTP\_KEY\_BASE: Generate one for each of
            them with `pswgen -Bsv1 64`.
            
        The full description of all those options can be found at
        [here](https://github.com/sameersbn/docker-gitlab#available-configuration-parameters).
    
        ```bash
        $ kubectl create -f gitlab-deployment.yml
        deployment "gitlab" created
        ```












