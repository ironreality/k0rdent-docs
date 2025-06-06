# Installing {{{ docsVersionInfo.k0rdentName }}} Observability and FinOps

## Prerequisites

Before beginning KOF installation, you should have the following components in place:

* A {{{ docsVersionInfo.k0rdentName }}} management cluster - You can get instructions to create one in the [quickstart guide](https://docs.k0rdent.io/v{{{ extra.docsVersionInfo.k0rdentDotVersion }}}/quickstart-1-mgmt-node-and-cluster/)
    * To test on [macOS](https://docs.k0sproject.io/stable/system-requirements/#host-operating-system) you can install using:
      `brew install kind && kind create cluster -n {{{ docsVersionInfo.k0rdentName }}}`
* You will also need your infrastructure provider credentials, such as those shown in the [guide for AWS](https://docs.k0rdent.io/v{{{ extra.docsVersionInfo.k0rdentDotVersion }}}/quickstarts/quickstart-2-aws/)
    * Note that you should skip the "Create your ClusterDeployment" and later sections.
* Finally, you either need access to create DNS records for service endpoints such as `kof.example.com`,
  or you may configure [Istio](https://github.com/k0rdent/kof/blob/main/docs/istio.md) instead.

### DNS auto-config

To avoid [manual configuration of DNS records](./kof-verification.md#manual-dns-config) for service endpoints later,
you can automate the process now using [external-dns](https://kubernetes-sigs.github.io/external-dns/latest/).

#### AWS

For AWS, use the [Node IAM Role](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#node-iam-role)
or [IRSA](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-roles-for-service-accounts) methods in production.

For now, however, just for the sake of this demo based on the `aws-standalone` template,
you can use the most straightforward (though less secure) [static credentials](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#static-credentials) method:

1. Create an `external-dns` IAM user with [this policy](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-policy).
2. Create an access key and `external-dns-aws-credentials` file, as in:
    ```
    [default]
    aws_access_key_id = <EXAMPLE_ACCESS_KEY_ID>
    aws_secret_access_key = <EXAMPLE_SECRET_ACCESS_KEY>
    ```
3. Create the `external-dns-aws-credentials` secret in the `kof` namespace:
    ```shell
    kubectl create namespace kof
    kubectl create secret generic \
      -n kof external-dns-aws-credentials \
      --from-file external-dns-aws-credentials
    ```
#### Azure

To enable DNS auto-config on Azure, use DNS Zone Contributor.

1. Create an Azure service principal with the DNS Zone Contributor permissions. You can find an example [here](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md#creating-a-service-principal).

2. Create the `azure.json` text file containing [the service principal configuration data](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md#configuration-file):
    ```
    {
      "tenantId": "SP_TENANT_SP_TENANT",
      "subscriptionId": "SUBSCRIPTION_ID_SUBSCRIPTION_ID",
      "resourceGroup": "MyDnsResourceGroup",
      "aadClientId": "SP_APP_ID_SP_APP_ID",
      "aadClientSecret": "SP_PASSWORD_SP_PASSWORD"
    }
    ```

3. Create the `external-dns-azure-credentials` secret in the `kof` namespace:
    ```shell
    kubectl create namespace kof
    kubectl create secret generic \
      -n kof external-dns-azure-credentials \
      --from-file azure.json
    ```
See [external-dns Azure documentation](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md) for more details.

## Management Cluster

To install KOF on the management cluster,
look through the default values of the [kof-mothership](https://github.com/k0rdent/kof/blob/main/charts/kof-mothership/README.md)
and [kof-operators](https://github.com/k0rdent/kof/blob/main/charts/kof-operators/values.yaml) charts,
and apply this example, or use it as a reference:

1. Install `kof-operators` required by `kof-mothership`:
    ```shell
    helm install --wait --create-namespace -n kof kof-operators \
      oci://ghcr.io/k0rdent/kof/charts/kof-operators --version {{{ extra.docsVersionInfo.kofVersions.kofDotVersion }}}
    ```

2. Create the `mothership-values.yaml` file:
    ```yaml
    kcm:
      installTemplates: true
    ```
    This enables installation of `ServiceTemplates` such as `cert-manager` and `kof-storage`,
    to make it possible to reference them from the Regional and Child `ClusterDeployments`.

3. If you want to use a [default storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/#default-storageclass),
    but `kubectl get sc` shows no `(default)`, create it.
    Otherwise you can use a non-default storage class in the `mothership-values.yaml` file:
    ```yaml
    global:
      storageClass: <EXAMPLE_STORAGE_CLASS>
    ```

4. If you've applied the [DNS auto-config](#dns-auto-config) section,
    add to the `kcm:` object in the `mothership-values.yaml` file.

    For AWS, add:

    ```yaml
      kof:
        clusterProfiles:
          kof-aws-dns-secrets:
            matchLabels:
              k0rdent.mirantis.com/kof-aws-dns-secrets: "true"
            secrets:
              - external-dns-aws-credentials
    ```

    For Azure, add:

    ```yaml
      kof:
        clusterProfiles:
          kof-azure-dns-secrets:
            matchLabels:
              k0rdent.mirantis.com/kof-azure-dns-secrets: "true"
            secrets:
              - external-dns-azure-credentials
    ```

    This enables Sveltos to auto-distribute DNS secret to regional clusters.

5. Two secrets are auto-created by default:
    * `storage-vmuser-credentials` is a secret used by VictoriaMetrics.
        You don't need to use it directly.
        It is auto-distributed to other clusters by the Sveltos `ClusterProfile` [here](https://github.com/k0rdent/kof/blob/121b61f5f6de6ddfdf3525b98f3ad4cb8ce57eaa/charts/kof-mothership/values.yaml#L25-L31).
    * `grafana-admin-credentials` is a secret that we will use in the [Grafana](./kof-using.md#access-to-grafana) section.
        It is auto-created [here](https://github.com/k0rdent/kof/blob/121b61f5f6de6ddfdf3525b98f3ad4cb8ce57eaa/charts/kof-mothership/values.yaml#L64-L65).

6. Install `kof-mothership`:
    ```shell
    helm install --wait -f mothership-values.yaml -n kof kof-mothership \
      oci://ghcr.io/k0rdent/kof/charts/kof-mothership --version {{{ extra.docsVersionInfo.kofVersions.kofDotVersion }}}
    ```

7. Install [kof-regional](./kof-architecture.md/#kof-regional) and [kof-child](./kof-architecture.md/#kof-child) charts into the management cluster:
    ```shell
    helm install --wait -n kof kof-regional \
      oci://ghcr.io/k0rdent/kof/charts/kof-regional --version {{{ extra.docsVersionInfo.kofVersions.kofDotVersion }}}
    helm install --wait -n kof kof-child \
      oci://ghcr.io/k0rdent/kof/charts/kof-child --version {{{ extra.docsVersionInfo.kofVersions.kofDotVersion }}}
    ```

8. Wait for all pods to show that they're `Running`:
    ```shell
    kubectl get pod -n kof
    ```

## Regional Cluster

To install KOF on the regional cluster,
look through the default values of the [kof-storage](https://github.com/k0rdent/kof/blob/main/charts/kof-storage/values.yaml) chart,
and apply this example for AWS, or use it as a reference:

1. Set your KOF variables using your own values:
    ```shell
    REGIONAL_CLUSTER_NAME=cloud1-region1
    REGIONAL_DOMAIN=$REGIONAL_CLUSTER_NAME.kof.example.com
    ADMIN_EMAIL=$(git config user.email)
    echo "$REGIONAL_CLUSTER_NAME, $REGIONAL_DOMAIN, $ADMIN_EMAIL"
    ```

2. Use the up-to-date `ClusterTemplate`, as in:
    ```shell
    kubectl get clustertemplate -n kcm-system | grep aws
    TEMPLATE=aws-standalone-cp-{{{ extra.docsVersionInfo.providerVersions.dashVersions.awsStandaloneCpCluster }}}
    ```

3. Compose the regional `ClusterDeployment`:

    For AWS:

    ```shell
    cat >regional-cluster.yaml <<EOF
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: $REGIONAL_CLUSTER_NAME
      namespace: kcm-system
      labels:
        k0rdent.mirantis.com/kof-storage-secrets: "true"
        k0rdent.mirantis.com/kof-aws-dns-secrets: "true"
        k0rdent.mirantis.com/kof-cluster-role: regional
    spec:
      template: $TEMPLATE
      credential: aws-cluster-identity-cred
      config:
        clusterAnnotations:
          k0rdent.mirantis.com/kof-regional-domain: $REGIONAL_DOMAIN
          k0rdent.mirantis.com/kof-cert-email: $ADMIN_EMAIL
        clusterIdentity:
          name: aws-cluster-identity
          namespace: kcm-system
        controlPlane:
          instanceType: t3.large
        controlPlaneNumber: 1
        publicIP: false
        region: us-east-2
        worker:
          instanceType: t3.medium
        workersNumber: 3
    EOF
    ```

    For Azure:

    ```shell
    REGION=AZURE_LOCATION
    AZURE_SUBSCRIPTION_ID=SUBSCRIPTION_ID_SUBSCRIPTION_ID
    TEMPLATE=azure-standalone-cp-{{{ docsVersionInfo.providerVersions.dashVersions.azureStandaloneCpCluster }}}
    cat >regional-cluster.yaml <<EOF
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: $REGIONAL_CLUSTER_NAME
      namespace: kcm-system
      labels:
        kof: storage
    spec:
      template: $TEMPLATE
      credential: azure-cluster-identity-cred
      config:
        clusterIdentity:
          name: azure-cluster-identity
          namespace: kcm-system
        subscriptionID: $AZURE_SUBSCRIPTION_ID
        controlPlane:
          vmSize: Standard_A4_v2
        controlPlaneNumber: 1
        location: $REGION
        worker:
          vmSize: Standard_A4_v2
        workersNumber: 3
        clusterLabels:
          k0rdent.mirantis.com/kof-storage-secrets: "true"
          k0rdent.mirantis.com/kof-azure-dns-secrets: "true"
      serviceSpec:
        priority: 100
        services:
          - name: ingress-nginx
            namespace: ingress-nginx
            template: ingress-nginx-{{{ docsVersionInfo.servicesVersions.dashVersions.ingressNginx }}}
            values: |
              ingress-nginx:
                controller:
                  service:
                    annotations:
                      service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: /healthz
          - name: cert-manager
            namespace: cert-manager
            template: cert-manager-{{{ docsVersionInfo.servicesVersions.dashVersions.certManager }}}
            values: |
              cert-manager:
                crds:
                  enabled: true
          - name: kof-storage
            namespace: kof
            template: kof-storage-{{{ docsVersionInfo.kofVersions.kofStorageVersion }}}
            values: |
              external-dns:
                enabled: true
                provider:
                  name: azure
                extraVolumeMounts:
                  - name: azure-config-file
                    mountPath: /etc/kubernetes
                    readOnly: true
                extraVolumes:
                  - name: azure-config-file
                    secret:
                      secretName: external-dns-azure-credentials
              victoriametrics:
                vmauth:
                  ingress:
                    host: vmauth.$REGIONAL_DOMAIN
                security:
                  username_key: username
                  password_key: password
                  credentials_secret_name: storage-vmuser-credentials
              grafana:
                ingress:
                  host: grafana.$REGIONAL_DOMAIN
                security:
                  credentials_secret_name: grafana-admin-credentials
              cert-manager:
                email: sample@example.com
    ---
    apiVersion: kof.k0rdent.mirantis.com/v1alpha1
    kind: PromxyServerGroup
    metadata:
      labels:
        app.kubernetes.io/name: promxy-operator
        k0rdent.mirantis.com/promxy-secret-name: kof-mothership-promxy-config
      name: $REGIONAL_CLUSTER_NAME-metrics
      namespace: kof
    spec:
      cluster_name: $REGIONAL_CLUSTER_NAME
      targets:
        - "vmauth.$REGIONAL_DOMAIN:443"
      path_prefix: /vm/select/0/prometheus/
      scheme: https
      http_client:
        dial_timeout: "5s"
        tls_config:
          insecure_skip_verify: true
        basic_auth:
          credentials_secret_name: storage-vmuser-credentials
          username_key: username
          password_key: password
    ---
    apiVersion: grafana.integreatly.org/v1beta1
    kind: GrafanaDatasource
    metadata:
      labels:
        app.kubernetes.io/managed-by: Helm
      name: $REGIONAL_CLUSTER_NAME-logs
      namespace: kof
    spec:
      valuesFrom:
        - targetPath: "basicAuthUser"
          valueFrom:
            secretKeyRef:
              key: username
              name: storage-vmuser-credentials
        - targetPath: "secureJsonData.basicAuthPassword"
          valueFrom:
            secretKeyRef:
              key: password
              name: storage-vmuser-credentials
      datasource:
        name: $REGIONAL_CLUSTER_NAME
        url: https://vmauth.$REGIONAL_DOMAIN/vls
        access: proxy
        isDefault: false
        type: "victoriametrics-logs-datasource"
        basicAuth: true
        basicAuthUser: \${username}
        secureJsonData:
          basicAuthPassword: \${password}
      instanceSelector:
        matchLabels:
          dashboards: grafana
      resyncPeriod: 5m
    EOF
    ```

4. This `ClusterDeployment` uses propagation of its `.metadata.labels`
    to the resulting `Cluster` because there are no `.spec.config.clusterLabels` here.
    Only if you add them, please copy `.metadata.labels` there too.

5. The `ClusterTemplate` above provides the [default storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/#default-storageclass)
    (`ebs-csi-default-sc` for AWS). If you want to use a non-default storage class,
    add it to the `regional-cluster.yaml` file in the `.spec.config.clusterAnnotations`:
    ```yaml
    k0rdent.mirantis.com/kof-storage-class: <EXAMPLE_STORAGE_CLASS>
    ```

6. The `kof-operator` creates and configures `PromxyServerGroup` and `GrafanaDatasource` [automatically](https://github.com/k0rdent/kof/blob/a71b0524bd86215a37efeb1e478a97279fc90846/kof-operator/internal/controller/clusterdeployment_kof_cluster_role.go#L501-L507).
    It uses the [endpoints](https://github.com/k0rdent/kof/blob/a71b0524bd86215a37efeb1e478a97279fc90846/kof-operator/internal/controller/clusterdeployment_kof_cluster_role.go#L42-L48) listed below by default.
    Only if you want to disable the built-in metrics, logs, and traces to use your own existing instances instead,
    add custom endpoints to the `regional-cluster.yaml` file in the `.spec.config.clusterAnnotations`:
    ```yaml
    k0rdent.mirantis.com/kof-write-metrics-endpoint: https://vmauth.$REGIONAL_DOMAIN/vm/insert/0/prometheus/api/v1/write
    k0rdent.mirantis.com/kof-read-metrics-endpoint: https://vmauth.$REGIONAL_DOMAIN/vm/select/0/prometheus
    k0rdent.mirantis.com/kof-write-logs-endpoint: https://vmauth.$REGIONAL_DOMAIN/vls/insert/opentelemetry/v1/logs
    k0rdent.mirantis.com/kof-read-logs-endpoint: https://vmauth.$REGIONAL_DOMAIN/vls
    k0rdent.mirantis.com/kof-write-traces-endpoint: https://jaeger.$REGIONAL_DOMAIN/collector
    ```

7. The `MultiClusterService` named [kof-regional-cluster](https://github.com/k0rdent/kof/blob/d0baccd068f08f0f1d95ae0a26173176d106d284/charts/kof-regional/templates/regional-multi-cluster-service.yaml)
    configures and installs `cert-manager`, `ingress-nginx`, and `kof-storage` charts automatically.
    To pass any custom [values](https://github.com/k0rdent/kof/blob/main/charts/kof-storage/values.yaml) to the `kof-storage` chart
    or its subcharts like the [victoria-logs-single](https://docs.victoriametrics.com/helm/victorialogs-single/index.html#parameters),
    add them to the `regional-cluster.yaml` file in the `.spec.config.clusterAnnotations`, for example:
    ```yaml
    k0rdent.mirantis.com/kof-storage-values: |
      victoria-logs-single:
        server:
          replicaCount: 2
    ```

8. Verify and apply the Regional `ClusterDeployment`:
    ```shell
    cat regional-cluster.yaml

    kubectl apply -f regional-cluster.yaml
    ```

9. Watch how the cluster is deployed to AWS until all values of `READY` are `True`:
    ```shell
    clusterctl describe cluster -n kcm-system $REGIONAL_CLUSTER_NAME \
      --show-conditions all
    ```

## Child Cluster

To install KOF on the actual cluster to be monitored,
look through the default values of the [kof-operators](https://github.com/k0rdent/kof/blob/main/charts/kof-operators/values.yaml)
and [kof-collectors](https://github.com/k0rdent/kof/blob/main/charts/kof-collectors/values.yaml) charts,
and apply this example for AWS, or use it as a reference:

1. Set your own value below, verifing [the variables](#regional-cluster):
    ```shell
    CHILD_CLUSTER_NAME=$REGIONAL_CLUSTER_NAME-child1
    echo "$CHILD_CLUSTER_NAME, $REGIONAL_DOMAIN"
    ```

2. Use the up-to-date `ClusterTemplate`, as in:
    ```shell
    kubectl get clustertemplate -n kcm-system | grep aws
    TEMPLATE=aws-standalone-cp-{{{ extra.docsVersionInfo.providerVersions.dashVersions.awsStandaloneCpCluster }}}
    ```

3. Compose the child `ClusterDeployment`:

    For AWS:

    ```shell
    cat >child-cluster.yaml <<EOF
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: $CHILD_CLUSTER_NAME
      namespace: kcm-system
      labels:
        k0rdent.mirantis.com/kof-storage-secrets: "true"
        k0rdent.mirantis.com/kof-cluster-role: child
    spec:
      template: $TEMPLATE
      credential: aws-cluster-identity-cred
      config:
        clusterIdentity:
          name: aws-cluster-identity
          namespace: kcm-system
        controlPlane:
          instanceType: t3.large
        controlPlaneNumber: 1
        publicIP: false
        region: us-east-2
        worker:
          instanceType: t3.small
        workersNumber: 3
    EOF
    ```

    For Azure:

    ```shell
    REGION=AZURE_LOCATION
    AZURE_SUBSCRIPTION_ID=SUBSCRIPTION_ID_SUBSCRIPTION_ID
    TEMPLATE=azure-standalone-cp-{{{ docsVersionInfo.providerVersions.azureStandaloneCpCluster }}}
    cat >child-cluster.yaml <<EOF
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: $CHILD_CLUSTER_NAME
      namespace: kcm-system
      labels:
        kof: collector
    spec:
      template: azure-standalone-cp-{{{ docsVersionInfo.providerVersions.azureStandaloneCpCluster }}}
      credential: azure-cluster-identity-cred
      config:
        clusterIdentity:
          name: azure-cluster-identity
          namespace: kcm-system
        subscriptionID: $AZURE_SUBSCRIPTION_ID
        controlPlane:
          vmSize: Standard_A4_v2
        controlPlaneNumber: 1
        location: $REGION
        worker:
          vmSize: Standard_A4_v2
        workersNumber: 3
        clusterLabels:
          k0rdent.mirantis.com/kof-storage-secrets: "true"
      serviceSpec:
        priority: 100
        services:
          - name: cert-manager
            namespace: kof
            template: cert-manager-{{{ extra.docsVersionInfo.servicesVersions.dashVersions.certManager }}}
            values: |
              cert-manager:
                crds:
                  enabled: true
          - name: kof-operators
            namespace: kof
            template: kof-operators-{{{ extra.docsVersionInfo.kofVersions.kofOperatorsVersion }}}
          - name: kof-collectors
            namespace: kof
            template: kof-collectors-{{{ extra.docsVersionInfo.kofVersions.kofCollectorsVersion }}}
            values: |
              global:
                clusterName: $CHILD_CLUSTER_NAME
              opencost:
                enabled: true
                opencost:
                  prometheus:
                    username_key: username
                    password_key: password
                    existingSecretName: storage-vmuser-credentials
                    external:
                      url: https://vmauth.$REGIONAL_DOMAIN/vm/select/0/prometheus
                  exporter:
                    defaultClusterId: $CHILD_CLUSTER_NAME
              kof:
                logs:
                  username_key: username
                  password_key: password
                  credentials_secret_name: storage-vmuser-credentials
                  endpoint: https://vmauth.$REGIONAL_DOMAIN/vls/insert/opentelemetry/v1/logs
                metrics:
                  username_key: username
                  password_key: password
                  credentials_secret_name: storage-vmuser-credentials
                  endpoint: https://vmauth.$REGIONAL_DOMAIN/vm/insert/0/prometheus/api/v1/write
    EOF
    ```

4. This `ClusterDeployment` uses propagation of its `.metadata.labels`
    to the resulting `Cluster` because there are no `.spec.config.clusterLabels` here.
    Only if you add them, please copy `.metadata.labels` there too.

5. The `kof-operator` discovers the regional cluster by the [location](https://github.com/k0rdent/kof/blob/a71b0524bd86215a37efeb1e478a97279fc90846/kof-operator/internal/controller/clusterdeployment_kof_cluster_role.go#L334-L353) of the child cluster.
    Only if you have more than one regional cluster in the same AWS region / Azure location / etc,
    and you want to connect the child cluster to specific regional cluster,
    add this regional cluster name to the `child-cluster.yaml` file in the `.metadata.labels`:
    ```yaml
    k0rdent.mirantis.com/kof-regional-cluster-name: $REGIONAL_CLUSTER_NAME
    ```

6. The `MultiClusterService` named [kof-child-cluster](https://github.com/k0rdent/kof/blob/d0baccd068f08f0f1d95ae0a26173176d106d284/charts/kof-child/templates/child-multi-cluster-service.yaml)
    configures and installs `cert-manager`, `kof-operators`, and `kof-collectors` charts automatically.
    To pass any custom [values](https://github.com/k0rdent/kof/blob/main/charts/kof-collectors/values.yaml) to the `kof-collectors` chart
    or its subcharts like the [opencost](https://github.com/opencost/opencost-helm-chart/blob/main/charts/opencost/README.md#values),
    add them to the `child-cluster.yaml` file in the `.spec.config`, for example:
    ```yaml
    clusterAnnotations:
      k0rdent.mirantis.com/kof-collectors-values: |
        opencost:
          opencost:
            exporter:
              replicas: 2
    ```
    Note: the first `opencost` key is to reference the subchart,
    and the second `opencost` key is part of its [values](https://github.com/opencost/opencost-helm-chart/blob/main/charts/opencost/README.md#values).

7. Verify and apply the `ClusterDeployment`:
    ```shell
    cat child-cluster.yaml

    kubectl apply -f child-cluster.yaml
    ```

8. Watch while the cluster is deployed to AWS until all values of `READY` are `True`:
    ```shell
    clusterctl describe cluster -n kcm-system $CHILD_CLUSTER_NAME \
      --show-conditions all
    ```
