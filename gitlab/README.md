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

The instructions involves persistent disk are mainly from Eric
Oestrich's excellent
[blog post](https://blog.oestrich.org/2015/08/running-postgres-inside-kubernetes/).

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
    
2.  Create Persistent Disk

    We absolutely do not want to lose our data if gitlab is down or
    being reconfigured and restarted. Redis operates mostly in memory
    and does not requires any persistent data storage, but the other
    two services do:
    
    *   **Postgresql**, for storing the metadata such as the user
        information, repository structures. The mount point is
        `/var/lib/postgresql`.
    *   **GitLab**, for storing the actual git repositories. The mount
        point is `/home/git/data`.
        
    Therefoer, we need to create
    [persistent disks](https://cloud.google.com/compute/docs/disks/),
    and mount one for them respectively.
    
    The plan is to create a 20 GB persistent disk for Postgresql, and
    a 50 GB persistent disk for GitLab. Having separate disks is great
    for modularizing the deployment (another reason is that I do not
    know how to have two pods reading the same EXT4 formatted
    persistent disk).

    The following commands create:
    
    *   One 20 GB persistent disk for Postgresql:
    
        ```bash
        $ gcloud compute disks create --size 20GB gitlab-postgresql-data-disk
        ```
    *   One 50 GB persistent disk for GitLab
    
        ```bash
        $ gcloud compute disks create --size 50GB gitlab-data-disk
        ```
        
    The disks are named `gitlab-postgresql-data-disk` and
    `gitlab-data-disk` respectively. Both newly created persisten disk
    needs to be formatted before actually used. See the appendix on
    how to format it with filesystem `ext-4`.
    
3.  Disk Claims

    After creating the disks above, we need to let kubernetes know
    about their existence, as well as create **claims** for them. The
    **claims** act as the handles of the disks and the deployment can
    use the claims to attach to their desiring disks.
    
    To notify kubernetes about the disks, run
    
    ```bash
    $ kubectl create -f gitlab/gitlab-data-disk.yml
    $ kubectl create -f postgresql/gitlab-postgresql-data-disk.yml
    ```
    
    Those commands connects the persistent disks to our kubernetes
    manager. Now we need to expose the handles of them by creating the claims.
    Run
    
    ```bash
    $ kubectl create -f gitlab/disk-claim.yml
    $ kubectl create -f postgresql/disk-claim.yml
    ```
    
    We can now verify by running:
    
    ```bash
    $ kubectl get pv
    NAME                          CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                                      REASON    AGE
    gitlab-data-disk              50Gi       RWO           Retain          Bound     gitlab/gitlab-data-disk-claim                        1h
    gitlab-postgresql-data-disk   20Gi       RWO           Retain          Bound     gitlab/gitlab-postgresql-data-disk-claim             1h
    
    $ kubectl get pvc
    NAME                                STATUS    VOLUME                        CAPACITY   ACCESSMODES   AGE
    gitlab-data-disk-claim              Bound     gitlab-data-disk              50Gi       RWO           1h
    gitlab-postgresql-data-disk-claim   Bound     gitlab-postgresql-data-disk   20Gi       RWO           1h
    ```
    
    Note how persitent volumes are claimed by those claims.
    
4.  **Redis**

    *   Create the Redis service (under namespace `gitlab`).
    
        ```bash
        $ kubectl create -f redis/svc.yml
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
        $ kubectl create -f redis/deployment.yml 
        deployment "gitlab-redis" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get pods
        NAME                                 READY     STATUS    RESTARTS   AGE
        gitlab-postgresql-2981213592-cezcd   1/1       Running   0          2h
        gitlab-redis-3473415130-563lx        1/1       Running   0          29s
        ```
    
5.  **PostgreSQL**

    *   Create the PostgreSQL service (under namespace `gitlab`).
    
        ```bash
        $ kubectl create -f postgresql/svc.yml
        service "gitlab-postgresql" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get svc
        NAME                CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
        gitlab-postgresql   10.3.252.137   <none>        5432/TCP   21m
        ```
    *   Deploy PostgreSQL Pod.
    
        ```bash
        $ kubectl create -f postgresql/deployment.yml
        deployment "gitlab-ostgresql" created
        ```
        
        This can be verified by
        
        ```bash
        $ kubectl get deployments
        NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
        gitlab-postgresql   1         1         1            1           11m
        ```
        
6.  **GitLab**

    *   Create the GitLab service (under the namespace `gitlab`).
    
        ```bash
        $ kubectl create -f gitlab/svc.yml 
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
        
        Warning: When restarting the gitlab pod, do not restart the
        service, or you will lose the extern IP and are assigned
        another one. This is usually bad when you have this IP
        registered with your domain name.
        
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
        $ kubectl create -f gitlab/deployment.yml
        deployment "gitlab" created
        ```
        
7.  **Gitlab** root password issue.

    Now you should be able to connect to your GitLab by typing the
    external IP into your browswer. However your might run into one of
    the following problems:
    
    *   Cannot log in as `root` with the specified root password. This
        has puzzled me for quite some time and you can find the
        details in this
        [issue](https://github.com/gitlabhq/gitlabhq/issues/7226). In
        the end it is solved by:
        
        1.  SSH into your compute engine instance with `gcloud`.
        2.  Find your gitlab docker container by `docker ps`.
        3.  Start a separate bash on the container by
        
            ```bash
            $ docker exec -i -t <container ID> /bin/bash
            ```
        4.  Run
            
            ```bash
            sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
            ```
            
        This will reset your gitlab instance, so that you will be
        prompted to set your root password the next time you open the
        gitlab web app.
        
    *   One first-run/root-password-reset, running into this
        [issue](https://github.com/sameersbn/docker-gitlab/issues/657).
        GitLab would complaining about password change no matter what
        you have put in.
        
        I do not know whether this will be a legitimate solution but
        you can try to use another Browser, be it chrome, firefox or
        Safari.
        
        In my case switching from Chrome to Firefox (only for changing
        root password) works.

## Appendix

### Format your newly created persistent disk.

The official instruction can be found
[here](https://cloud.google.com/compute/docs/disks/add-persistent-disk).

I am going to describe the process with some extra details, but in
general we are just borrowing a compute instance in the same
project to perform the format.

1.  Find your compute engine instance, select it and click `EDIT`
    from the top bar.
2.  Add the persistent disk we just created to `additional disk`
    section.
3.  The, ssh to the instance via the `gcloud` command and follow
    the instruction to format the disk with the `mkfs` command.

    ```bash
    $ sudo mkfs.ext4 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/disk/by-id/google-gitlab-data-disk
    mke2fs 1.43 (17-May-2016)
    Discarding device blocks: done                            
    Creating filesystem with 52428800 4k blocks and 13107200 inodes
    Filesystem UUID: f68c7941-585a-4133-b46b-fa7fee963879
    Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872

    Allocating group tables: done                            
    Writing inode tables: done                            
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done
    ```
4.  You can now detach the persistent disk from your instance via
    the `EDIT` options.

### Update/Restart the Pods.

Sometimes we want to restart the jobs, for example because of gitlab
has a new version. To upgrade to a newer version of gitlab is
extremely simple. You first stop the pod by

```bash
$ kubectl delete -f gitlab/deployment.yml
```

And change the deployment configuration's image to your desired
version. And run

```bash
$ kubectl create -f gitlab/deployment.yml
```

You will not lose anything because all your data are stored in
persistent disks. Hooary!
