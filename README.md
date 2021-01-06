# cloudify-helm
Cloudify Helm Charts

# cloudify manager AIO

## How to use?

git clone git@github.com:cloudify-cosmo/cloudify-helm.git

cd cloudify-helm

kubectl create ns cloudify-aio

helm install cloudify-manager-aio ./cloudify-manager-aio -n cloudify-aio

## Running AIO with custom values

helm install cloudify-manager-aio -f ./cloudify-manager-aio/values.yaml ./cloudify-manager-aio -n cloudify-aio

In values.yaml you can change: image.repository / service.type ...

```
image:
  repository: "cloudifyplatform/community-cloudify-manager-aio"
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP

```

## Development / testing / validation of AIO chart

helm install --dry-run cloudify-manager-aio ./cloudify-manager-aio

helm template ./cloudify-manager-aio


# cloudify manager worker

## Cloudify manager solution for 3 layers includes:

  * One cloudify manager worker: 1 Pod +  1 Deployment + 1 Service

  * One external DB (Postgres)

  * One external QUEUE (RabbitMQ)

## SSL certificates must be provided - at least 3 certs:

* ca.pem, to sign other certificates in case we need auto-generation of certificates

* one-key.pem, this key used for internal / external / db / queue ssl_inputs*

* one-cert.pem, crt used for internal/external/db/queue ssl_inputs

## Create certificates using cloudify manager:

```
cfy_manager generate-test-cert -s 'cloudify-manager-worker.cfy-helm.svc.cluster.local,rabbitmq.cfy-helm.svc.cluster.local,postgres-postgresql.cfy-helm.svc.cluster.local'
```

## Install PostgreSQL to Kubernetes cluster with helm

Create secret with certificates to be used by helm chart:

```
kubectl create secret generic postgresql-certs --from-file=./tls.crt --from-file=./tls.key --from-file=./ca.crt 
```

Use created certificates in postgress-values.yaml

```
volumePermissions.enabled=true
tls:
  enabled: true
  preferServerCiphers: true
  certificatesSecret: 'postgres-tls-secret'
  certFilename: 'tls.crt'
  certKeyFilename: 'tls.key'
```

```
helm install postgres bitnami/postgresql -f ./cloudify-manager-worker/external/postgres-values.yaml -n cfy-helm
```

## Install RabbitMQ to Kubernetes cluster with helm

Create secret with certificates to be used by helm chart:

```
kubectl create secret generic rabbitmq-certs --from-file=./tls.crt --from-file=./tls.key --from-file=./ca.crt 
  certKeyFilename: 'tls.key'
```

Use created certificates in rabbitmq-values.yaml

```
tls:
    enabled: true
    existingSecret: rabbit-tls-secret
    failIfNoPeerCert: false
    sslOptionsVerify: verify_peer
    caCertificate: |-    
    serverCertificate: |-
    serverKey: |-
  certKeyFilename: 'tls.key'
```

Run management console on 15671 port with SSL:

add to rabbitmq-values.yaml

```
configuration: |-
  management.ssl.port       = 15671
  management.ssl.cacertfile = /opt/bitnami/rabbitmq/certs/ca_certificate.pem
  management.ssl.certfile   = /opt/bitnami/rabbitmq/certs/server_certificate.pem
  management.ssl.keyfile    = /opt/bitnami/rabbitmq/certs/server_key.pem

extraPorts:
  - name: manager-ssl
    port: 15671
    targetPort: 15671
```

```
helm install rabbitmq bitnami/rabbitmq -f ./cloudify-manager-worker/external/rabbitmq-values.yaml -n cfy-helm
```

## Install cloudify manager worker

```
git clone git@github.com:cloudify-cosmo/cloudify-helm.git

cd cloudify-helm

kubectl create ns cfy-manager-worker
```

First certificated must be added to certificates.yaml, you need at least 3 certs( I explained how to generate them above):

ca.pem

one-key.pem

one-cert.pem

```
helm install cloudify-manager-worker ./cloudify-manager-worker -n cfy-manager-worker
```
