![GitHub Banner](https://github.com/langfuse/langfuse-k8s/assets/2834609/2982b65d-d0bc-4954-82ff-af8da3a4fac8)

# langfuse-k8s

This is a community-maintained repository that contains resources for deploying Langfuse on Kubernetes.

## Helm Chart

We provide a Helm chart that helps you deploy Langfuse on Kubernetes.

### 1.0.0 Release Candidate

This Chart is a release candidate for the 1.0.0 version of the Langfuse Helm Chart.
Please provide all thoughts and feedbacks on the interface and the upgrade path via our [GitHub Discussion](https://github.com/orgs/langfuse/discussions/5734).

For details on how to migrate from 0.13.x to 1.0.0, refer to our [migration guide](./UPGRADE.md).

### Installation

Configure the required secrets and parameters as defined below in a new `values.yaml` file.
Then install the helm chart using the commands below:

```bash
helm repo add langfuse https://skyu-io.github.io/langfuse-helm-chart
helm repo update
helm install langfuse langfuse/langfuse -f values.yaml
```

### Upgrading

```bash
helm repo update
helm upgrade langfuse langfuse/langfuse
```

Please validate whether the helm sub-charts in the Chart.yaml were updated between versions.
If yes, follow the guide for the respective sub-chart to upgrade it.

### Configuration

The required configuration options to set are:

```yaml
# Optional, but highly recommended. Generate via `openssl rand -hex 32`.
#  langfuse:
#    encryptionKey:
#      value: ""
langfuse: 
  salt:
    value: secureSalt
  nextauth:
    secret:
      value: ""

postgresql:
  auth:
    password: ""

clickhouse:
  auth:
    password: ""

redis:
  auth:
    password: ""
```

They can alternatively set via secret references (the secrets must exist):

```yaml
# Optional, but highly recommended. Generate via `openssl rand -hex 32`.
#  langfuse:
#    encryptionKey:
#      secretKeyRef:
#        name: langfuse-encryption-key-secret
#        key: encryptionKey
langfuse: 
  salt:
    secretKeyRef:
      name: langfuse-general
      key: salt
  nextauth:
    secret:
      secretKeyRef:
        name: langfuse-nextauth-secret
        key: nextauth-secret

postgresql:
  auth:
    existingSecret: langfuse-postgresql-auth
    secretKeys:
      userPasswordKey: password

clickhouse:
  auth:
    existingSecret: langfuse-clickhouse-auth
    secretKeys:
      userPasswordKey: password

redis:
  auth:
    existingSecret: langfuse-redis-auth
    secretKeys:
      userPasswordKey: password
```
      
See the [Helm README](./charts/langfuse/README.md) for a full list of all configuration options.

#### Examples:

##### With an external Postgres server

```yaml
[...]
postgresql:
  deploy: false
  auth:
    username: my-username
    password: my-password
    database: my-database
  host: my-external-postgres-server.com
  directUrl: postgres://my-username:my-password@my-external-postgres-server.com
  shadowDatabaseUrl: postgres://my-username:my-password@my-external-postgres-server.com
```

#### With an external S3 bucket

```yaml
[...]
s3:
  deploy: false
  bucket: "langfuse-bucket"
  region: "eu-west-1"
  endpoint: "https://s3.eu-west-1.amazonaws.com"
  forcePathStyle: false
  accessKeyId:
    value: "mykey"
  secretAccessKey:
    value: "mysecret"
  eventUpload:
    prefix: "events/"
  batchExport:
    prefix: "exports/"
  mediaUpload:
    prefix: "media/"
```

#### Use custom deployment strategy

```yaml
[...]
langfuse:
  deployment:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 50%
        maxUnavailable: 50%
```

##### Enable ingress

```yaml
[...]
langfuse:
  ingress:
    enabled: true
    hosts:
    - host: langfuse.your-host.com
      paths:
      - path: /
        pathType: Prefix
    annotations: []
```

#### Custom Storage Class Definition

The Langfuse chart supports configuring storage classes for all persistent volumes in the deployment. You can configure storage classes in two ways:

1. **Global Storage Class**: Set a global storage class that will be used for all persistent volumes unless overridden.
```yaml
global:
  defaultStorageClass: "your-storage-class"
```

2. **Component-specific Storage Classes**: Override the storage class for specific components.
```yaml
postgresql:
  primary:
    persistence:
      storageClass: "postgres-storage-class"
   
redis:
  primary:
    persistence:
      storageClass: "redis-storage-class"

clickhouse:
  persistence:
    storageClass: "clickhouse-storage-class"

s3:
  persistence:
    storageClass: "minio-storage-class"
```

If no storage class is specified, the cluster's default storage class will be used.

##### With an external Postgres server with client certificates using own secrets and additionalEnv for mappings

```yaml
langfuse:
  salt: null
  nextauth: 
    secret: null
  extraVolumes:
    - name: db-keystore   # referencing an existing secret to mount server/client certs for postgres
      secret:
        secretName: langfuse-postgres  # contain the following files (server-ca.pem, sslidentity.pk12)
  extraVolumeMounts:
    - name: db-keystore
      mountPath: /secrets/db-keystore  # mounting the db-keystore store certs in the pod under the given path
      readOnly: true
  additionalEnv:
    - name: DATABASE_URL  # Using the certs in the url eg. postgresql://the-db-user:the-password@postgres-host:5432/langfuse?ssl=true&sslmode=require&sslcert=/secrets/db-keystore/server-ca.pem&sslidentity=/secrets/db-keystore/sslidentity.pk12&sslpassword=the-ssl-identity-pw
      valueFrom:
        secretKeyRef:
          name: langfuse-postgres  # referencing an existing secret
          key: database-url
    - name: NEXTAUTH_SECRET
      valueFrom:
        secretKeyRef:
          name: langfuse-general # referencing an existing secret
          key: nextauth-secret
    - name: SALT
      valueFrom:
        secretKeyRef:
          name: langfuse-general
          key: salt
service:
  [...]
ingress:
  [...]
postgresql:
  deploy: false
  auth:
    password: null
    username: null
```

##### With overrides for hostAliases

This is going to add a record to the /etc/hosts file of all containers
under the langfuse-web pod in such a way that every traffic towards "oauth.id.jumpcloud.com" is going to be forwarded to the localhost network.

```yaml
langfuse:
  web:
    hostAliases:
      - ip: 127.0.0.1
        hostnames:
          - "oauth.id.jumpcloud.com"
```

##### With topology spread constraints

Distribute pods evenly across different zones to improve high availability:

```yaml
langfuse:
  # Global topology spread constraints applied to all langfuse pods
  pod:
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app.kubernetes.io/instance: langfuse
  
  # Component-specific topology spread constraints
  web:
    pod:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
  
  worker:
    pod:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: worker
```

## Repository Structure

- `examples` directory contains example `yaml` configurations
- `charts/langfuse` directory contains Helm chart for deploying Langfuse with an associated database

Please feel free to contribute any improvements or suggestions.

Langfuse deployment docs: https://langfuse.com/docs/deployment/self-host

## Source Code

* <https://github.com/langfuse/langfuse>
* <https://github.com/langfuse/langfuse-k8s>

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| oci://registry-1.docker.io/bitnamicharts | clickhouse | 8.0.5 |
| oci://registry-1.docker.io/bitnamicharts | common | 2.30.0 |
| oci://registry-1.docker.io/bitnamicharts | s3(minio) | 14.10.5 |
| oci://registry-1.docker.io/bitnamicharts | postgresql | 16.4.9 |
| oci://registry-1.docker.io/bitnamicharts | redis(valkey) | 2.2.4 |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| clickhouse.auth.existingSecret | string | `""` | If you want to use an existing secret for the ClickHouse password, set the name of the secret here. (`clickhouse.auth.username` and `clickhouse.auth.password` will be ignored and picked up from this secret). |
| clickhouse.auth.existingSecretKey | string | `""` | The key in the existing secret that contains the password. |
| clickhouse.auth.password | string | `""` | Password for the ClickHouse user. |
| clickhouse.auth.username | string | `"default"` | Username for the ClickHouse user. |
| clickhouse.clusterEnabled | bool | `true` | Whether to run ClickHouse commands ON CLUSTER |
| clickhouse.deploy | bool | `true` | Enable ClickHouse deployment (via Bitnami Helm Chart). If you want to use an external Clickhouse server (or a managed one), set this to false |
| clickhouse.host | string | `""` | ClickHouse host to connect to. If clickhouse.deploy is true, this will be set automatically based on the release name. |
| clickhouse.httpPort | int | `8123` | ClickHouse HTTP port to connect to. |
| clickhouse.migration.autoMigrate | bool | `true` | Whether to run automatic ClickHouse migrations on startup |
| clickhouse.migration.ssl | bool | `false` | Set to true to establish SSL connection for migration |
| clickhouse.migration.url | string | `""` | Migration URL (TCP protocol) for clickhouse |
| clickhouse.nativePort | int | `9000` | ClickHouse native port to connect to. |
| clickhouse.replicaCount | int | `3` | Number of replicas to use for the ClickHouse cluster. 1 corresponds to a single, non-HA deployment. |
| clickhouse.resourcesPreset | string | `"2xlarge"` | The resources preset to use for the ClickHouse cluster. |
| clickhouse.shards | int | `1` | Subchart specific settings |
| extraManifests | list | `[]` |  |
| fullnameOverride | string | `""` | Override the full name of the deployed resources, defaults to a combination of the release name and the name for the selector labels |
| langfuse.additionalEnv | list | `[]` | List of additional environment variables to be added to all langfuse deployments. See [documentation](https://langfuse.com/docs/deployment/self-host#configuring-environment-variables) for details. |
| langfuse.affinity | object | `{}` | Affinity for all langfuse deployments |
| langfuse.deployment.annotations | object | `{}` | Annotations for all langfuse deployments |
| langfuse.deployment.strategy | object | `{}` | Deployment strategy for all langfuse deployments (can be overridden by individual deployments) |
| langfuse.encryptionKey | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | Used to encrypt sensitive data. Must be 256 bits (64 string characters in hex format). Generate via `openssl rand -hex 32`. |
| langfuse.extraContainers | list | `[]` | Allows additional containers to be added to all langfuse deployments |
| langfuse.extraInitContainers | list | `[]` | Allows additional init containers to be added to all langfuse deployments |
| langfuse.extraVolumeMounts | list | `[]` | Allows additional volume mounts to be added to all langfuse deployments |
| langfuse.extraVolumes | list | `[]` | Allows additional volumes to be added to all langfuse deployments |
| langfuse.features.experimentalFeaturesEnabled | bool | `false` | Enable experimental features |
| langfuse.features.signUpDisabled | bool | `false` | Disable public sign up |
| langfuse.features.telemetryEnabled | bool | `true` | Whether or not to report basic usage statistics to a centralized server. |
| langfuse.image.pullPolicy | string | `"Always"` | The pull policy to use for all langfuse deployments. Can be overridden by the individual deployments. |
| langfuse.image.pullSecrets | list | `[]` | The pull secrets to use for all langfuse deployments. Can be overridden by the individual deployments. |
| langfuse.image.tag | string | `nil` | The image tag to use for all langfuse deployments. Can be overridden by the individual deployments. Falls back to appVersion if not set. |
| langfuse.ingress.additionalLabels | object | `{}` | Additional labels for the ingress resource |
| langfuse.ingress.annotations | object | `{}` | Annotations for the ingress resource |
| langfuse.ingress.className | string | `""` | The class name for the ingress resource |
| langfuse.ingress.enabled | bool | `false` | Set to `true` to enable the ingress resource |
| langfuse.ingress.hosts | list | `[]` | The hosts for the ingress resource |
| langfuse.ingress.tls.enabled | bool | `false` | Set to `true` to enable use HTTPS on the ingress |
| langfuse.ingress.tls.secretName | string | `""` | The name of the secret to use for TLS Key |
| langfuse.licenseKey | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | Langfuse EE license key. |
| langfuse.logging.format | string | `"text"` | Set the log format for the application (text or json) |
| langfuse.logging.level | string | `"info"` | Set the log level for the application (trace, debug, info, warn, error, fatal) |
| langfuse.nextauth.secret | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | Used to encrypt the NextAuth.js JWT, and to hash email verification tokens. Can be configured by value or existing secret reference. |
| langfuse.nextauth.url | string | `"http://localhost:3000"` | When deploying to production, set the `nextauth.url` value to the canonical URL of your site. |
| langfuse.nodeEnv | string | `"production"` | Node.js environment to use for all langfuse deployments |
| langfuse.nodeSelector | object | `{}` | Node selector for all langfuse deployments |
| langfuse.pod.annotations | object | `{}` | Annotations for all langfuse pods |
| langfuse.pod.labels | object | `{}` | Labels for all langfuse pods |
| langfuse.pod.topologySpreadConstraints | list | `[]` | Topology spread constraints for all langfuse pods |
| langfuse.podSecurityContext | object | `{}` | Pod security context for all langfuse deployments |
| langfuse.replicas | int | `1` | Number of replicas to use for all langfuse deployments. Can be overridden by the individual deployments |
| langfuse.resources | object | `{}` | Resources for all langfuse deployments. Can be overridden by the individual deployments |
| langfuse.salt | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | Used to hash API keys. Can be configured by value or existing secret reference. To generate a new salt, run `openssl rand -base64 32`. |
| langfuse.securityContext | object | `{}` | Security context for all langfuse deployments |
| langfuse.serviceAccount.annotations | object | `{}` | Annotations for the service account |
| langfuse.serviceAccount.create | bool | `true` | Whether to create a service account for all langfuse deployments |
| langfuse.serviceAccount.name | string | `""` | Override the name of the service account to use, discovered automatically if not set |
| langfuse.tolerations | list | `[]` | Tolerations for all langfuse deployments |
| langfuse.web.deployment.annotations | object | `{}` | Annotations for the web deployment |
| langfuse.web.deployment.strategy | object | `{}` | Deployment strategy for the web deployment. Overrides the global deployment strategy |
| langfuse.web.hostAliases | list | `[]` | Adding records to /etc/hosts in the pod's network. |
| langfuse.web.hpa.enabled | bool | `false` | Set to `true` to enable HPA for the langfuse web pods Note: When both KEDA and HPA are enabled, the deployment will fail. |
| langfuse.web.hpa.maxReplicas | int | `2` | The maximum number of replicas to use for the langfuse web pods |
| langfuse.web.hpa.minReplicas | int | `1` | The minimum number of replicas to use for the langfuse web pods |
| langfuse.web.hpa.targetCPUUtilizationPercentage | int | `50` | The target CPU utilization percentage for the langfuse web pods |
| langfuse.web.image.pullPolicy | string | `nil` | The pull policy to use for the langfuse web pods. Using `langfuse.image.pullPolicy` if not set. |
| langfuse.web.image.pullSecrets | string | `nil` | The pull secrets to use for the langfuse web pods. Using `langfuse.image.pullSecrets` if not set. |
| langfuse.web.image.repository | string | `"langfuse/langfuse"` | The image repository to use for the langfuse web pods. |
| langfuse.web.image.tag | string | `nil` | The tag to use for the langfuse web pods. Using `langfuse.image.tag` if not set. |
| langfuse.web.keda.containerName | string | `""` | Optional container name to target for metrics (leave empty to target all containers) |
| langfuse.web.keda.enabled | bool | `false` | Set to `true` to enable KEDA for the langfuse web pods Note: When both KEDA and HPA are enabled, the deployment will fail. |
| langfuse.web.keda.maxReplicas | int | `2` | The maximum number of replicas to use for the langfuse web pods |
| langfuse.web.keda.metricType | string | `"Utilization"` | The metric type for scaling (Utilization or AverageValue) |
| langfuse.web.keda.minReplicas | int | `1` | The minimum number of replicas to use for the langfuse web pods |
| langfuse.web.keda.pollingInterval | int | `30` | The polling interval in seconds for checking metrics |
| langfuse.web.keda.triggerType | string | `"cpu"` | The trigger type for scaling (cpu or memory) |
| langfuse.web.keda.value | string | `"50"` | The target utilization percentage for the langfuse web pods |
| langfuse.web.livenessProbe.failureThreshold | int | `3` | Failure threshold for livenessProbe. |
| langfuse.web.livenessProbe.initialDelaySeconds | int | `20` | Initial delay seconds for livenessProbe. |
| langfuse.web.livenessProbe.path | string | `"/api/public/health"` | Path to check for liveness. |
| langfuse.web.livenessProbe.periodSeconds | int | `10` | Period seconds for livenessProbe. |
| langfuse.web.livenessProbe.successThreshold | int | `1` | Success threshold for livenessProbe. |
| langfuse.web.livenessProbe.timeoutSeconds | int | `5` | Timeout seconds for livenessProbe. |
| langfuse.web.pod.annotations | object | `{}` | Annotations for the web pods |
| langfuse.web.pod.labels | object | `{}` | Labels for the web pods |
| langfuse.web.pod.topologySpreadConstraints | string | `nil` | Topology spread constraints for the web pods. Overrides the global topologySpreadConstraints |
| langfuse.web.readinessProbe.failureThreshold | int | `3` | Failure threshold for readinessProbe. |
| langfuse.web.readinessProbe.initialDelaySeconds | int | `20` | Initial delay seconds for readinessProbe. |
| langfuse.web.readinessProbe.path | string | `"/api/public/ready"` | Path to check for readiness. |
| langfuse.web.readinessProbe.periodSeconds | int | `10` | Period seconds for readinessProbe. |
| langfuse.web.readinessProbe.successThreshold | int | `1` | Success threshold for readinessProbe. |
| langfuse.web.readinessProbe.timeoutSeconds | int | `5` | Timeout seconds for readinessProbe. |
| langfuse.web.replicas | string | `nil` | Number of replicas to use if HPA is not enabled. Defaults to the global replicas |
| langfuse.web.resources | object | `{}` | Resources for the langfuse web pods. Defaults to the global resources |
| langfuse.web.service.additionalLabels | object | `{}` | Additional labels for the langfuse web application service |
| langfuse.web.service.annotations | object | `{}` | Annotations for the langfuse web application service |
| langfuse.web.service.nodePort | string | `nil` | The node port to use for the langfuse web application |
| langfuse.web.service.port | int | `3000` | The port to use for the langfuse web application |
| langfuse.web.service.type | string | `"ClusterIP"` | The type of service to use for the langfuse web application |
| langfuse.web.vpa.controlledResources | list | `[]` | The resources to control for the langfuse web pods |
| langfuse.web.vpa.enabled | bool | `false` | Set to `true` to enable VPA for the langfuse web pods |
| langfuse.web.vpa.maxAllowed | object | `{}` | The maximum allowed resources for the langfuse web pods |
| langfuse.web.vpa.minAllowed | object | `{}` | The minimum allowed resources for the langfuse web pods |
| langfuse.web.vpa.updatePolicy.updateMode | string | `"Auto"` | The update policy mode for the langfuse web pods |
| langfuse.worker.deployment.annotations | object | `{}` | Annotations for the worker deployment |
| langfuse.worker.deployment.strategy | object | `{}` | Deployment strategy for the worker deployment. Overrides the global deployment strategy |
| langfuse.worker.hpa.enabled | bool | `false` | Set to `true` to enable HPA for the langfuse worker pods Note: When both KEDA and HPA are enabled, the deployment will fail. |
| langfuse.worker.hpa.maxReplicas | int | `2` | The maximum number of replicas to use for the langfuse worker pods |
| langfuse.worker.hpa.minReplicas | int | `1` | The minimum number of replicas to use for the langfuse worker pods |
| langfuse.worker.hpa.targetCPUUtilizationPercentage | int | `50` | The target CPU utilization percentage for the langfuse worker pods |
| langfuse.worker.image.pullPolicy | string | `nil` | The pull policy to use for the langfuse worker pods. Using `langfuse.image.pullPolicy` if not set. |
| langfuse.worker.image.pullSecrets | string | `nil` | The pull secrets to use for the langfuse worker pods. Using `langfuse.image.pullSecrets` if not set. |
| langfuse.worker.image.repository | string | `"langfuse/langfuse-worker"` | The image repository to use for the langfuse worker pods |
| langfuse.worker.image.tag | string | `nil` | The tag to use for the langfuse worker pods. Using `langfuse.image.tag` if not set. |
| langfuse.worker.keda.containerName | string | `""` | Optional container name to target for metrics (leave empty to target all containers) |
| langfuse.worker.keda.enabled | bool | `false` | Set to `true` to enable KEDA for the langfuse worker pods Note: When both KEDA and HPA are enabled, the deployment will fail. |
| langfuse.worker.keda.maxReplicas | int | `2` | The maximum number of replicas to use for the langfuse worker pods |
| langfuse.worker.keda.metricType | string | `"Utilization"` | The metric type for scaling (Utilization or AverageValue) |
| langfuse.worker.keda.minReplicas | int | `1` | The minimum number of replicas to use for the langfuse worker pods |
| langfuse.worker.keda.pollingInterval | int | `30` | The polling interval in seconds for checking metrics |
| langfuse.worker.keda.triggerType | string | `"cpu"` | The trigger type for scaling (cpu or memory) |
| langfuse.worker.keda.value | string | `"50"` | The target utilization percentage for the langfuse worker pods |
| langfuse.worker.livenessProbe.failureThreshold | int | `3` | Failure threshold for livenessProbe. |
| langfuse.worker.livenessProbe.initialDelaySeconds | int | `20` | Initial delay seconds for livenessProbe. |
| langfuse.worker.livenessProbe.periodSeconds | int | `10` | Period seconds for livenessProbe. |
| langfuse.worker.livenessProbe.successThreshold | int | `1` | Success threshold for livenessProbe. |
| langfuse.worker.livenessProbe.timeoutSeconds | int | `5` | Timeout seconds for livenessProbe. |
| langfuse.worker.pod.annotations | object | `{}` | Annotations for the worker pods |
| langfuse.worker.pod.labels | object | `{}` | Labels for the worker pods |
| langfuse.worker.pod.topologySpreadConstraints | string | `nil` | Topology spread constraints for the worker pods. Overrides the global topologySpreadConstraints |
| langfuse.worker.replicas | string | `nil` | Number of replicas to use if HPA is not enabled. Defaults to the global replicas |
| langfuse.worker.resources | object | `{}` | Resources for the langfuse worker pods. Defaults to the global resources |
| langfuse.worker.vpa.controlledResources | list | `[]` | The resources to control for the langfuse worker pods |
| langfuse.worker.vpa.enabled | bool | `false` | Set to `true` to enable VPA for the langfuse worker pods |
| langfuse.worker.vpa.maxAllowed | object | `{}` | The maximum allowed resources for the langfuse worker pods |
| langfuse.worker.vpa.minAllowed | object | `{}` | The minimum allowed resources for the langfuse worker pods |
| langfuse.worker.vpa.updatePolicy.updateMode | string | `"Auto"` | The update policy mode for the langfuse worker pods |
| nameOverride | string | `""` | Override the name for the selector labels, defaults to the chart name |
| postgresql.architecture | string | `"standalone"` |  |
| postgresql.args | string | `""` | Additional database connection arguments |
| postgresql.auth.args | string | `""` | Additional database connection arguments |
| postgresql.auth.database | string | `"postgres_langfuse"` | Database name to use for Langfuse. |
| postgresql.auth.existingSecret | string | `""` | If you want to use an existing secret for the postgres password, set the name of the secret here. (`postgresql.auth.username` and `postgresql.auth.password` will be ignored and picked up from this secret). |
| postgresql.auth.password | string | `""` | Password to use to connect to the postgres database deployed with Langfuse. In case `postgresql.deploy` is set to `true`, the password will be set automatically. |
| postgresql.auth.secretKeys | object | `{"userPasswordKey":"password"}` | The key in the existing secret that contains the password. |
| postgresql.auth.username | string | `"postgres"` | Username to use to connect to the postgres database deployed with Langfuse. In case `postgresql.deploy` is set to `true`, the user will be created automatically. |
| postgresql.deploy | bool | `true` | Enable PostgreSQL deployment (via Bitnami Helm Chart). If you want to use an external Postgres server (or a managed one), set this to false |
| postgresql.directUrl | string | `""` | If `postgresql.deploy` is set to false, Connection string of your Postgres database used for database migrations. Use this if you want to use a different user for migrations or use connection pooling on DATABASE_URL. For large deployments, configure the database user with long timeouts as migrations might need a while to complete. |
| postgresql.host | string | `""` | PostgreSQL host to connect to. If postgresql.deploy is true, this will be set automatically based on the release name. |
| postgresql.migration.autoMigrate | bool | `true` | Whether to run automatic migrations on startup |
| postgresql.port | string | `nil` | Port of the postgres server to use. Defaults to 5432. |
| postgresql.primary.service.ports.postgresql | int | `5432` |  |
| postgresql.shadowDatabaseUrl | string | `""` | If your database user lacks the CREATE DATABASE permission, you must create a shadow database and configure the "SHADOW_DATABASE_URL". This is often the case if you use a Cloud database. Refer to the Prisma docs for detailed instructions. |
| redis.architecture | string | `"standalone"` |  |
| redis.auth.database | int | `0` |  |
| redis.auth.existingSecret | string | `""` | If you want to use an existing secret for the redis password, set the name of the secret here. (`redis.auth.password` will be ignored and picked up from this secret). |
| redis.auth.existingSecretPasswordKey | string | `""` | The key in the existing secret that contains the password. |
| redis.auth.password | string | `""` | Configure the password by value or existing secret reference |
| redis.deploy | bool | `true` | Enable valkey deployment (via Bitnami Helm Chart). If you want to use a Redis or Valkey server already deployed, set to false. |
| redis.host | string | `""` | Redis host to connect to. If redis.deploy is true, this will be set automatically based on the release name. |
| redis.port | int | `6379` | Redis port to connect to. |
| redis.primary.extraFlags | list | `["--maxmemory-policy noeviction"]` | Extra flags for the valkey deployment. Must include `--maxmemory-policy noeviction`. |
| redis.tls.caPath | string | `""` | Path to the CA certificate file for TLS verification |
| redis.tls.certPath | string | `""` | Path to the client certificate file for mutual TLS authentication |
| redis.tls.enabled | bool | `false` | Set to `true` to enable TLS/SSL encrypted connection to the Redis server |
| redis.tls.keyPath | string | `""` | Path to the client private key file for mutual TLS authentication |
| s3.accessKeyId | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 accessKeyId to use for all uploads. Can be overridden per upload type. |
| s3.auth.existingSecret | string | `""` | If you want to use an existing secret for the root user password, set the name of the secret here. (`s3.auth.rootUser` and `s3.auth.rootPassword` will be ignored and picked up from this secret). |
| s3.auth.rootPassword | string | `""` | Password for MinIO root user |
| s3.auth.rootPasswordSecretKey | string | `""` | Key where the Minio root user password is being stored inside the existing secret `s3.auth.existingSecret` |
| s3.auth.rootUser | string | `"minio"` | root username |
| s3.batchExport.accessKeyId | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 accessKeyId to use for batch exports. |
| s3.batchExport.bucket | string | `""` | S3 bucket to use for batch exports. |
| s3.batchExport.enabled | bool | `true` | Enable batch export. |
| s3.batchExport.endpoint | string | `""` | S3 endpoint to use for batch exports. |
| s3.batchExport.forcePathStyle | string | `nil` | Whether to force path style on requests. Required for MinIO. |
| s3.batchExport.prefix | string | `""` | Prefix to use for batch exports within the bucket. |
| s3.batchExport.region | string | `""` | S3 region to use for batch exports. |
| s3.batchExport.secretAccessKey | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 secretAccessKey to use for batch exports. |
| s3.bucket | string | `""` | S3 bucket to use for all uploads. Can be overridden per upload type. |
| s3.concurrency.reads | int | `50` | Maximum number of concurrent read operations to S3. Defaults to 50. |
| s3.concurrency.writes | int | `50` | Maximum number of concurrent write operations to S3. Defaults to 50. |
| s3.defaultBuckets | string | `"langfuse"` |  |
| s3.deploy | bool | `true` | Enable MinIO deployment (via Bitnami Helm Chart). If you want to use a custom BlobStorage, e.g. S3, set to false. |
| s3.endpoint | string | `""` | S3 endpoint to use for all uploads. Can be overridden per upload type. |
| s3.eventUpload.accessKeyId | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 accessKeyId to use for event uploads. |
| s3.eventUpload.bucket | string | `""` | S3 bucket to use for event uploads. |
| s3.eventUpload.endpoint | string | `""` | S3 endpoint to use for event uploads. |
| s3.eventUpload.forcePathStyle | string | `nil` | Whether to force path style on requests. Required for MinIO. |
| s3.eventUpload.prefix | string | `""` | Prefix to use for event uploads within the bucket. |
| s3.eventUpload.region | string | `""` | S3 region to use for event uploads. |
| s3.eventUpload.secretAccessKey | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 secretAccessKey to use for event uploads. |
| s3.forcePathStyle | bool | `true` | Whether to force path style on requests. Required for MinIO. Can be overridden per upload type. |
| s3.mediaUpload.accessKeyId | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 accessKeyId to use for media uploads. |
| s3.mediaUpload.bucket | string | `""` | S3 bucket to use for media uploads. |
| s3.mediaUpload.downloadUrlExpirySeconds | int | `3600` | Expiry time for download URLs. Defaults to 1 hour. |
| s3.mediaUpload.enabled | bool | `true` | Enable media uploads. |
| s3.mediaUpload.endpoint | string | `""` | S3 endpoint to use for media uploads. |
| s3.mediaUpload.forcePathStyle | string | `nil` | Whether to force path style on requests. Required for MinIO. |
| s3.mediaUpload.maxContentLength | int | `1000000000` | Maximum content length for media uploads. Defaults to 1GB. |
| s3.mediaUpload.prefix | string | `""` | Prefix to use for media uploads within the bucket. |
| s3.mediaUpload.region | string | `""` | S3 region to use for media uploads. |
| s3.mediaUpload.secretAccessKey | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 secretAccessKey to use for media uploads. |
| s3.region | string | `"auto"` | S3 region to use for all uploads. Can be overridden per upload type. |
| s3.secretAccessKey | object | `{"secretKeyRef":{"key":"","name":""},"value":""}` | S3 secretAccessKey to use for all uploads. Can be overridden per upload type. |