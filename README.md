# Google Cloud CronJob

This repository defines a Dockerfile used to create a Docker image with [gcloud](https://cloud.google.com/sdk/gcloud) and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

It is meant to be used in Kubernetes [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) objects in order execute commands in a container.

## Usage example

1. Create a service account at the Google Cloud Console by following the
steps described at [Authorizing with a service account](https://cloud.google.com/sdk/docs/authorizing#authorizing_with_a_service_account).
2. Create a Kubernetes secret with the service account's JSON file that resulted from the above step:

```
kubectl create secret generic api --from-file=key=./your-service-account.json
```

Create the following Kubernetes object:

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: drupal-cron
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: drupal-cron
              image: juampynr/gcloud-cronjob:latest
              env:
                - name: GCLOUD_SA_KEY
                  valueFrom:
                    secretKeyRef:
                      name: api
                      key: key
              command: ["/bin/sh","-c"]
              args:
                - echo $GCLOUD_SA > gcloud_sa.json
                - gcloud auth activate-service-account --key-file=./gcloud_sa.json
                - gcloud container clusters get-credentials your-cluster-id --zone your-cluster-zone --project your-project-id
                - POD_NAME=$(kubectl get pods -l tier=frontend -o=jsonpath='{.items[0].metadata.name}')
                - kubectl exec POD_NAME -c drupal -- vendor/bin/drush core:cron
          restartPolicy: OnFailure
```
