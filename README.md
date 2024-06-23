These helm charts are intended to work with AWS, but can be modified to work in other cloud platforms to support various features (like secret management, managed Kafka).

All images used in these charts are under the Confluent Community License or Apache 2 License. See https://docs.confluent.io/platform/current/installation/docker/image-reference.html for more information.

#### Kafka Brokers

##### Kubernetes

This helm chart deploys Kafka brokers and Zookeeper alongside Schema Registry by default. StatefulSets are used with persistent volumes for both components.

If using this chart for the brokers, it can be combined with AWS Route 53 to expose the brokers outside the kubernetes cluster.
For example:

1. Create a private hosted zone in Route 53 with a domain (ex. `test.kafka.com`)
2. In the helm values for the kafka chart :
  - Set the `loadbalancer.enabled` to true
  - Set the `loadbalancer.externalDNS` to the domain from above
  - Choose a port other than 9092 and set it to `loadbalancer.externalPort` (9092 is used inside the cluster and separate advertised listeners cannot use the same port)
3. Once the load balancers for the brokers have been created, create CNAME records in the hosted zone for the brokers in the format `kafka-#.externalDNS` and point it to the respective load balancer internal DNS:
  - `kafka-0.test.kafka.com`
  - `kafka-1.test.kafka.com`
  - `kafka-2.test.kafka.com`

##### AWS MSK

However the recommended approach is to use AWS MSK for the Kafka brokers and just use this chart for Schema Registry, KSQL Server, Kafka Connect, Kafka Rest proxy, and Kafka proxy as needed.
There are 2 options to specify the MSK brokers for the different helm charts:

1. In `kafka.bootstrapServers`
2. In `kafka.secret`

The Schema Registry, KSQL, Connect, and Rest proxy charts include these 2 options (For Rest Proxy its `cp-kafka`). To use a secret:
- set `kafka.secret.enabled` to true
- set `kafka.secret.name` to the name of the kubernetes secret
- set `kafka.secret.key` to the key containing the bootstrap servers

The Kafka [proxy](misc/proxy.yaml) currently requires a secret to pass the brokers. This will be changed to support passing directly in the future.
The secret name must be `kafka-brokers` and the keys must be `broker_1', `broker_2`, and `broker_3` but this can be changed in the file.

To apply, run:
```bash
kubectl apply -f misc/proxy.yaml --namespace kafka
```

#### Conduktor

To setup Conduktor Console:
1. Install Docker Desktop and make sure its running
2. Run docker compose up -d

The console will be available at localhost:9092 in your browser

Use `host.docker.internal` instead of localhost when configuring the cluster brokers, schema registry, connect, etc.

#### Prometheus

There is a [helmfile](misc/kube-prometheus-stack.yaml) to deploy the [kube-prometheus-stack](kube-prometheus-stack) for monitoring.
Assuming the namespace in kubernetes with the Kafka components is named `kafka` the helmfile, the helmfile will deploy a service monitor in that namespace to sync JMX metrics for Kafka Connect in Prometheus.
There is also an option to add an additional scrape config for MSK brokers in `prometheus.prometheusSpec.additionalScrapeConfigs[Secret]`. See the [template](misc/scrape-config.yaml) for an example. MSK uses ports 11001 and 11002 to [expose JMX metrics](https://docs.aws.amazon.com/msk/latest/developerguide/open-monitoring.html#:~:text=Amazon%20MSK%20uses%20port%2011001%20for%20the%20JMX%20Exporter%20and%20port%2011002%20for%20the%20Node%20Exporter).

The helm chart also includes Grafana. The default admin password is available in `grafana.adminPassword`.

To add the helm chart, run:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

To install the helm chart, run:
```bash
helm install prometheus prometheus-community/kube-prometheus-stack --values misc/kube-prometheus-stack.yaml --namespace prometheus
```

The chart should be installed in a namespace called `prometheus`.

#### Grafana

Grafana dashboards are available in the grafana-dashboard folder. The main ones are
- [kafka-connect.json](grafana-dashboard/kafka-connect.json)
- [msk-monitoring.json](grafana-dashboard/msk-monitoring.json)

The open source version is still there but it uses `cp_` metrics so its incompatible with standard naming.

There is a [template](misc/grafana-contact-point.yaml) for creating a contact point in Grafana.
Change the default notification policy to use it instead of the default contact point.

The logic for a kafka connect monitoring alert rule is also in a [template](misc/grafana-alert-rule.yaml).

#### Secret Management

To make managing kubernetes secrets easier, the [External Secrets Operator](https://external-secrets.io/latest/) (ESO) is recommended. It supports syncing secrets from AWS Secrets Manager to kubernetes.

To add the helm chart, run:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```

To install the helm chart, run:
```bash
helm install external-secrets external-secrets/external-secrets --namespace external-secrets
```

This installs the operator in the `external-secrets` namespace.

The CRD templates can be modified based on region and required components. The existing templates include secrets for MSK, Kafka proxy, and additional prometheus scrape config.

To install the cluster secret store, run:
```bash
kubectl apply -f misc/secret-store.yaml
```

To install the cluster external secret, run:
```bash
kubectl apply -f misc/external-secret.yaml
```

#### To do:
- Move the Kafka proxy to a sidecar container for the kafka stateful set/schema registry deployment so it doesn't have to be deployed separately and support passing brokers directly
