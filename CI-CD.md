# CI/CD

## Σκοπός

Σκοπός αυτού του εγγράφου είναι να τεκμηριώσει το CI/CD Pipeline του ΠΣ ΕΡΜΗΣ.

## Γενικές Πληροφορίες

Ως automation tool χρησιμοποιείται το [Google Cloud Build](https://cloud.google.com/build), λόγω του ότι:

- Είναι serverless λύση και δεν χρειάζεται να διαχειριζόμαστε κάποιο VM όπως θα χρειαζόταν με τον Jenkins
- Προσφέρει μεγάλο integration με άλλες λύσεις του GCP όπως το [Artifact Registry](https://cloud.google.com/artifact-registry) 
και το [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine).
- Είναι γενικότερα πολύ εύκολο στη χρήση.

## Περιγραφή των pipelines

Τα pipelines των διάφορων repo του πρότζεκτ παρουσιάζουν μεγάλες ομοιότητεες μεταξύ τους, οπότε θα αρκεστούμε στην περιγράφη ενός από αυτών και συγκεκριμένα του [hermes-backend](https://github.com/NickDelta/hermes-backend).

### Βήματα

1. Χτίζεται το `.jar` αρχείο (κανονικά θα έτρεχαν και τεστ αλλά εμείς τα έχουμε απενεργοποίησει με το `-DskipTests=true`).
2. Χτίζεται η `docker` εικόνα βάσει του `.Dockerfile` του απωθετηρίου με 2 tags : το `:latest` και το `:$SHORT_SHA` που αντιστοιχεί στα 7 πρώτα ψηφία του SHA του commit.
3. Η εικόνα γίνεται push στο Artifact Registry.
4. Γίνεται ένα [rolling update](https://cloud.google.com/kubernetes-engine/docs/how-to/updating-apps#overview) στο `hermes-cluster` που είναι ο GKE Cluster του πρότζεκτ.

![](diagrams/devops-diagram.png?raw=true)

Ενδεικτικά, μπορείτε παρακάτω. να δείτε το `cloudbuild.yaml` του hermes-backend:

```yaml
steps:
  # Use docker image because mvn cloud builder doesn't support jdk11
  - name: maven:3.6.3-jdk-11-slim
    entrypoint: 'mvn'
    # -Dhttp.keepAlive=false fixes a bug that causes cloud build to fail downloading dependencies
    args: ['package', '-DskipTests','-Dhttp.keepAlive=false']
  - name: 'gcr.io/cloud-builders/docker'
    args: [ 'build', '-t', 'europe-west2-docker.pkg.dev/$PROJECT_ID/back-end/prod:latest', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: [ 'build', '-t', 'europe-west2-docker.pkg.dev/$PROJECT_ID/back-end/prod:$SHORT_SHA', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: [ 'push', 'europe-west2-docker.pkg.dev/$PROJECT_ID/back-end/prod:latest' ]
  - name: 'gcr.io/cloud-builders/docker'
    args: [ 'push', 'europe-west2-docker.pkg.dev/$PROJECT_ID/back-end/prod:$SHORT_SHA' ]
  - name: "gcr.io/cloud-builders/gke-deploy"
    args:
    - run
    - --filename=k8s
    - --image=europe-west2-docker.pkg.dev/$PROJECT_ID/back-end/prod:$SHORT_SHA
    - --location=europe-west2-c
    - --cluster=hermes-cluster
```



