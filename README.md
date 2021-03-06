# Update all Cloud Foundry buildpacks

As a Cloud Foundry platform operator, I want to ensure my users/developers always have the latest upstream buildpacks, with the latest CVEs, for their applications.

```plain
# 1. cf login as an admin
# 2. run this script
curl https://raw.githubusercontent.com/starkandwayne/update-all-cf-buildpacks/master/update-only.sh | bash
```

## Kubernetes Update Forever

For Cloud Foundry/Eirini/Quarks you can immediately and continously update your buildpacks with a job and subsequent cron job.

Install with:

* `kubectl apply` (hard coded to update buildpacks nightly at 1am)

    ```plain
    kubectl apply -n kubecf -f https://raw.githubusercontent.com/starkandwayne/update-all-cf-buildpacks/master/k8s-update-forever.yaml
    ```

* A local Helm chart (allows some settings overrides):

    ```plain
    helm upgrade --install --namespace kubecf \
        update-all-cf-buildpacks \
        helm/update-all-cf-buildpacks/
    ```

* A remote Helm chart:

    ```plain
    helm repo add starkandwayne https://helm.starkandwayne.com
    helm repo update

    helm upgrade --install --namespace kubecf \
        update-all-cf-buildpacks \
        starkandwayne/update-all-cf-buildpacks
    ```

You can install either of these immediately after installing `cf-operator` and `kubecf` and it will patiently wait until Cloud Foundry is up and running. **This job does not require external DNS to be setup yet.**

Whilst `cf-operator` and `kubecf` deployments are coming up (which can take 20-40 minutes) the job has an init container that waits until the `kubecf-router-0` service is created:

```plain
$ kubectl get pods -n kubecf
NAME                             READY   STATUS       RESTARTS   AGE
cf-operator-77767c7847-rl2xn     1/1     Running      0          5m23s
update-all-cf-buildpacks-xclr5   0/1     Init:0/1     0          4m50s
```

Once the `kubecf-router-0` service exists, the job will start running and will wait until the full Cloud Foundry is running, and then upgrade all the buildpacks.

That is why the `update-all-cf-buildpacks-xxxxx` pod looks like it is running, even though CF itself is not operational yet:

```plain
$ kubectl get pods -n kubecf
NAME                             READY   STATUS       RESTARTS   AGE
cf-operator-77767c7847-rl2xn     1/1     Running      0          9m31s
kubecf-adapter-v1-0                 5/5     Running      0          116s
kubecf-api-v1-0                     0/17    Init:18/45   0          109s
kubecf-bits-v1-0                    0/7     Init:11/15   0          111s
kubecf-cc-worker-v1-0               5/5     Running      1          109s
kubecf-database-v1-0                0/5     Init:5/9     0          115s
kubecf-diego-api-v1-0               0/6     Init:5/13    0          113s
kubecf-doppler-v1-0                 0/11    Init:7/17    0          106s
kubecf-eirini-v1-0                  6/6     Running      0          105s
kubecf-log-api-v1-0                 8/8     Running      0          105s
kubecf-nats-v1-0                    5/5     Running      0          116s
kubecf-router-v1-0                  0/6     Init:Error   3          107s
kubecf-scheduler-v1-0               0/10    Init:7/19    0          108s
kubecf-singleton-blobstore-v1-0     7/7     Running      0          112s
kubecf-uaa-v1-0                     0/7     Init:7/13    0          113s
update-all-cf-buildpacks-xclr5   1/1     Running      0          8m58s
```

Once Cloud Foundry is running, the job will update the buildpacks and complete:

```plain
NAME                                 READY   STATUS              RESTARTS   AGE
pod/cf-operator-77767c7847-rl2xn     1/1     Running             0          29m
pod/kubecf-adapter-v1-0                 5/5     Running             0          22m
pod/kubecf-api-v1-0                     17/17   Running             6          22m
pod/kubecf-bits-v1-0                    7/7     Running             0          22m
pod/kubecf-cc-worker-v1-0               5/5     Running             6          22m
pod/kubecf-database-v1-0                5/5     Running             0          22m
pod/kubecf-diego-api-v1-0               6/6     Running             5          22m
pod/kubecf-doppler-v1-0                 11/11   Running             0          22m
pod/kubecf-eirini-v1-0                  6/6     Running             0          22m
pod/kubecf-log-api-v1-0                 8/8     Running             0          22m
pod/kubecf-nats-v1-0                    5/5     Running             0          22m
pod/kubecf-router-v1-0                  6/6     Running             5          22m
pod/kubecf-scheduler-v1-0               10/10   Running             12         22m
pod/kubecf-singleton-blobstore-v1-0     7/7     Running             0          22m
pod/kubecf-uaa-v1-0                     7/7     Running             0          22m
pod/update-all-cf-buildpacks-xclr5   0/1     Completed           0          29m
```

## Data only

This project also curates a [`buildpacks.json`](https://github.com/starkandwayne/update-all-cf-buildpacks/blob/master/buildpacks.json) file that contains the URLs for the latest buildpacks for each project:

```plain
curl https://raw.githubusercontent.com/starkandwayne/update-all-cf-buildpacks/master/buildpacks.json
```

## CI pipeline

The `update-only.sh` and `buildpacks.json` files are automatically updated via a CI pipeline:

* https://ci2.starkandwayne.com/teams/cfcommunity/pipelines/update-all-cf-buildpacks

The Concourse CI pipeline definition is in `ci/pipeline.yml`.

## Develop Helm

To iterate on the scripts as a Helm chart, you will need to publish a Docker image, and set two `image.` values:

```plain
org=drnic
tag=$(git branch | grep '^*' | awk '{print $2}')
docker build -t "$org/update-all-cf-buildpacks:$tag" .
docker push "$org/update-all-cf-buildpacks:$tag"

helm delete update-all-cf-buildpacks --namespace kubecf
helm upgrade --install --namespace kubecf \
    update-all-cf-buildpacks \
    helm/update-all-cf-buildpacks/ \
    --set "image.repository=$org/update-all-cf-buildpacks" \
    --set "image.tag=$tag" \
    --set "job.cron.schedule=* * * * *"
```

Note, `job.cron.schedule` set to "every minute" to help testing `cronjob.yaml`.

In another terminal, watch logs of the init and run containers:

```plain
stern -n kubecf -l app=update-all-cf-buildpacks
```

To update and test `k8s-update-forever.yaml`:

```plain
helm template helm/update-all-cf-buildpacks -n "" > k8s-update-forever.yaml
```
