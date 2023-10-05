# Buildpacks + Maven: how to use a custom `settings.xml` in Google Cloud Build

This repository shows how to use a custom `settings.xml` during a [remote build of a Maven app using buildpacks on Google Cloud Build](https://cloud.google.com/docs/buildpacks/build-application). Using a custom `settings.xml` can be necessary when, for example, a Maven application pulls dependencies from a protected Maven repository.

The example below uses a [build configuration file](https://cloud.google.com/build/docs/configuring-builds/create-basic-configuration). We'll start with the following `cloudbuild.yaml`:

```yaml
steps:
  - name: gcr.io/k8s-skaffold/pack
    entrypoint: pack
    args:
      - build
      - '$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA'
      - --builder
      - gcr.io/buildpacks/builder:latest
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - '$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA'

images: ['$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA']

substitutions:
  _DEPLOY_REGION: europe-west3
  _REPOSITORY: your-repository
  _SERVICE_NAME: your-service
```

This file uses the [`pack` CLI to build the application](https://cloud.google.com/docs/buildpacks/build-application#build_with_configuration_files) and to create a container image that is then pushed to Artifact Registry. See the [configuration file specification](https://cloud.google.com/build/docs/build-config-file-schema) for a complete documentation on the structure of this file.

If your application depends on Maven artifacts stored in a private repository, the build will fail with an error similar to the following:

```console
[builder] [ERROR] Failed to execute goal on project jobs: Could not resolve dependencies for project com.example:your-service:jar:0.0.1-SNAPSHOT: Failed to collect dependencies at com.example:your-dependency:jar:0.0.1-SNAPSHOT: Failed to read artifact descriptor for com.example:your-dependency:jar:0.0.1-SNAPSHOT: The following artifacts could not be resolved: com.example:your-dependency:pom:0.0.1-SNAPSHOT (absent): Could not transfer artifact com.example:your-dependency:pom:0.0.1-SNAPSHOT from/to your-private-repository (https://maven.pkg.github.com/...): status code: 401, reason phrase: Unauthorized (401) -> [Help 1]
```

## Add the content of `settings.xml` in Secret Manager

Because the content of `settings.xml` can contain sensitive data like passwords, we'll store it in Secret Manager.

[Create a new secret in Secret Manager](https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets) with the content of `settings.xml`.

## Give Cloud Build the permission to retrieve secrets

To use the newly created secret in Cloud Build, [grant the Secret Manager Secret Accessor (`roles/secretmanager.secretAccessor`) IAM role to the Cloud Build service account](https://cloud.google.com/build/docs/securing-builds/use-secrets).

## Import and use the secret at build time

Append the following to the bottom of `cloudbuild.yaml`:

```yaml
availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_NUMBER/secrets/${name-of-your-secret}/versions/latest
      env: 'MVN_SETTINGS'
```

Substitute `${name-of-your-secret}` by the name of the secret previously created. The page [Use secrets from Secret Manager](https://cloud.google.com/build/docs/securing-builds/use-secrets?hl=fr) documents in more details how to access secrets from a Cloud Build step.

Then, add the following step at the beginning of the file:

```yaml
steps:
  - name: bash
    secretEnv: ['MVN_SETTINGS']
    script: |
      #!/usr/bin/env bash
      mkdir -p /workspace/.m2
      echo "$MVN_SETTINGS" > /workspace/.m2/settings.xml
```

This step executes a script that first creates the directory `/workspace/.m2` if it doesn't exist. It then creates a `settings.xml` file in it with the content of your secret.

The `/workspace` directory retains its content between build steps. By putting `settings.xml` under `/workspace`, we make sure that subsequent steps will be able to access it: see [Passing data between build steps](https://cloud.google.com/build/docs/configuring-builds/pass-data-between-steps).

## Use the custom `settings.xml` with the `pack` CLI

Use the [`GOOGLE_BUILD_ARGS`](https://cloud.google.com/docs/buildpacks/java#hosted_maven_central_mirror) environment variable to specify the path to the custom `settings.xml` file. Edit the `pack` step to add this variable, as shown below:

```yaml
  - name: gcr.io/k8s-skaffold/pack
    entrypoint: pack
    args:
      - build
      - '$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA'
      - --builder
      - gcr.io/buildpacks/builder:latest
      - --env
      - 'GOOGLE_BUILD_ARGS=--settings /workspace/.m2/settings.xml'
```

Below is the final `cloudbuild.yaml`:

```yaml
steps:
  - name: bash
    secretEnv: ['MVN_SETTINGS']
    script: |
      #!/usr/bin/env bash
      mkdir -p /workspace/.m2
      echo "$MVN_SETTINGS" > /workspace/.m2/settings.xml
  - name: gcr.io/k8s-skaffold/pack
    entrypoint: pack
    args:
      - build
      - '$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA'
      - --builder
      - gcr.io/buildpacks/builder:latest
      - --env
      - 'GOOGLE_BUILD_ARGS=--settings /workspace/.m2/settings.xml'
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - '$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA'

images: ['$_DEPLOY_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA']

availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_NUMBER/secrets/${name-of-your-secret}/versions/latest
      env: 'MVN_SETTINGS'

substitutions:
  _DEPLOY_REGION: europe-west3
  _REPOSITORY: your-repository
  _SERVICE_NAME: your-service
```
