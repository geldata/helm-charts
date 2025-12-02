# Gel Server Helm Chart

A Helm chart for deploying gel-server on kubernetes with an external PostgreSQL server

  - [Quick Start](#quick-start)
  - [Get Helm Repository Info](#get-helm-repository-info)
  - [Install Helm Chart](#install-helm-chart)
  - [Uninstall Helm Chart](#uninstall-helm-chart)
  - [Upgrading Chart](#upgrading-chart)
  - [Configuration](#configuration)
    - [Providing the PostgreSQL Connection String](#providing-the-postgresql-connection-string)
    - [Providing the Gel Server Password](#providing-the-gel-server-password)
    - [Providing Extra Environment Variables](#providing-extra-environment-variables)
    - [Providing a Custom TLS Certificate](#providing-a-custom-tls-certificate)
  - [Connecting via `gel` CLI](#connecting-via-gel-cli)
    - [Automatically Generated TLS Certificates](#automatically-generated-tls-certificates)
    - [Self-Provided TLS Certificates](#self-provided-tls-certificates)

## Quick Start

1. **Create the database connection secret:**

```console
  kubectl create secret generic gel-db-creds \
    --from-literal=GEL_SERVER_BACKEND_DSN='postgresql://user:pass@postgres.example.com:5432/geldb' \
    -n gel
```

2. **Create the server password secret:**

```console
  kubectl create secret generic gel-server-password \
    --from-literal=GEL_SERVER_PASSWORD='your-secure-password' \
    -n gel
```

3. **Create a values.yaml file:**

```yaml
  extraEnvFromSecrets:
    - name: gel-db-creds
    - name: gel-server-password
   
  config:
    logLevel: "info"
   
  service:
    type: LoadBalancer
```

4. **Install the chart:**

```console
  helm install my-gel-server gel/gel-server \
    -f values.yaml \
    -n gel \
    --create-namespace
```

5. **Get the service endpoint:**

```console
  kubectl get svc -n gel
```

## Get Helm Repository Info

```console
helm repo add gel https://charts.geldata.com
helm repo update
```

_See [`helm repo`](https://helm.sh/docs/helm/helm_repo/) for command documentation._

## Install Helm Chart

```console
helm install [RELEASE_NAME] gel/gel-server
```

_See [configuration](#configuration) below._

_See [helm install](https://helm.sh/docs/helm/helm_install/) for command documentation._


## Uninstall Helm Chart

```console
helm uninstall [RELEASE_NAME]
```

## Upgrading Chart

```console
helm upgrade [RELEASE_NAME] gel/gel-server
```

## Configuration

To see all configurable options with detailed comments:

```console
helm show values gel/gel-server
```

| Parameter | Description | Default |
| ----- | ----------- | ------ |
| `config.logLevel` | Sets the GEL_SERVER_LOG_LEVEL |`"info"`|
| `config.security.tls.existingSecret` | Mounts an existing Kubernetes secret at `/tls` inside the gel-server pod  | `""` |

### Providing the PostgreSQL Connection String

It is recommended that the connection string to an external PostgreSQL server is set using a Kubernetes secret or an external secret manager.

To use a Kubernetes secret, first create a secret:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: gel-db-creds
data:
  GEL_SERVER_BACKEND_DSN: "<base64 encoded string>"
```

Then provide the secret name in `values.yaml`:

```yaml
extraEnvFromSecrets:
  - name: gel-db-creds
```

### Providing the Gel Server Password

It is recommended that the gel server password is set using a Kubernetes secret or an external secret manager.

To use a Kubernetes secret, first create a secret:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: gel-server-password
data:
  GEL_SERVER_PASSWORD: "<base64 encoded string>"
```

Then provide the secret name in `values.yaml`:

```yaml
extraEnvFromSecrets:
  - name: gel-server-password
```

### Providing Extra Environment Variables

You can provide any of the other gel server environment variables via:

```yaml
extraEnv:
    GEL_DOCKER_LOG_LEVEL: "info"
```

_See [gel-server configuration](https://docs.geldata.com/reference/running/configuration) for full list of environment variables._

### Providing a Custom TLS Certificate

To use a custom tls certificate, you must us an existing Kubernetes secret which contains the cert file and key file data.

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: gel-tls-cert
data:
  tls.crt: "<base64 encoded string>"
  tls.key: "<base64 encoded string>"
```

Then update the `values.yaml` to tell gel-server to use it:

```yaml
config:
  security:
    tls:
      existingSecret: "gel-tls-cert"
```

## Connecting via `gel` CLI

To connect to the deployed gel-server, you must provide the certificate file to establish a trusted tls connection.

### Automatically Generated TLS Certificates

After installing `gel-server`, you can run a command to exfiltrate the certificate and key data. Make sure to replace the namespace with your namespace:

```bash
kubectl exec -n gel \
  $(kubectl get pod -n gel -l app.kubernetes.io/name=gel-server \
  -o jsonpath='{.items[0].metadata.name}') -- \
  gel-show-secrets.sh --format=toml --all
```

Write the certificate data to a file:

```bash
cat << CERT > my.crt
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
CERT
```

and specify it when connecting:

```console
gel --dsn 'gel://admin:password@gel.example.com' --tls-ca-file my.crt
```

### Self-Provided TLS Certificates

Write the certificate data to a file:

```bash
cat << CERT > my.crt
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
CERT
```

and specify it when connecting:

```console
gel --dsn 'gel://admin:password@gel.example.com' --tls-ca-file my.crt
```