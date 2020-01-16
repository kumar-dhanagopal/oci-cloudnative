# Cloud Deployment

## Prerequisites

In order connect the MuShop application with services in Oracle Cloud Infrastructure,
several configurations are necessary. These tenancy configurations will be used to
properly provision and/or connect cloud services: Create a file with the following
information to simplify lookups later:

```yaml
region:       # Region where resources will be provisioned. (ex: us-phoenix-1)
tenancy:      # Tenancy OCID value
user:         # API User OCID value
compartment:  # Compartment OCID value
key:          # Private API Key file path (ex: /Users/jdoe/.oci/oci_key.pem)
fingerprint:  # Public API Key fingerprint (ex: 43:65:2c...)
```

### Compartment

Depending on the tenancy and your level of access, you may want (or need) to
create a Compartment dedicated to this application and the resources allocated.

1. Open Console and navigate to Compartments

    > Governance and Admininstration » Identity » Compartments » `Create Compartment`

1. Specify metadata for the Compartment, and make note of the **OCID**

### API User

You will need a User with API Key access in your tenancy.
This can be your personal user account, or a virtual user specific to usage of
this application.

1. Open Console and navigate to Users

    > Governance and Admininstration » Identity » Users

1. Select _or create_ the user you wish to use

1. If necessary, follow these [instructions](https://docs.cloud.oracle.com/iaas/Content/Functions/Tasks/functionssetupapikey.htm) to create an API key

1. Make note of the following items:
    - User **OCID**
    - API Key **Fingerprint**

### User Policies

If your configured User (with API Key) is **not** a member of the Administrators Group,
then a Group with specific Policies must be created, and the User added as a member.

1. Open Console and navigate to Groups

    > Governance and Admininstration » Identity » Groups » `Create Group`

1. Specify metadata for the Group, and make note of the **NAME**

1. Click the `Add User to Group` button and select your API User

1. Create a Policy with the folliwing statement:

    > Governance and Admininstration » Identity » Policies » `Create Policy`

    ```text
    Allow group <GroupName> to manage all-resources in compartment <CompartmentName>
    ```

<aside class="warning">
   This policy is intentionally broad for the sake of simplicity,
   and is <b>not</b> recommended in most real-world use cases.
   Refer to the <a href="https://docs.cloud.oracle.com/iaas/Content/Identity/Concepts/overview.htm#three">Documentation</a>
   for more on this topic.
</aside>

### Service Limits

Deploying the full application requires services from Oracle Cloud
Infrastructure. Use of these services will be subject to Service Limits in your
tenancy. Check minimum resource availability as follows:

> Check limits in the Console: Governance and Admininstration » Governance » Limits, Quotas, and Usage

| Service | Resource | Requirement |
| -- | -- | -- |
| Autonomous Transaction Processing Database | OCPU Count | `>=1` |
| Streaming | Partition Count | `>=1` |

<aside class="notice">
  This does not include requirements in cases where OKE is used.
</aside>

## Provisioning

Deploying the full application requires cloud backing services from Oracle Cloud Infrastructure.
Provisioning these services may be done manually of course, but can also be done _automatically_ 
through the use of [OCI Service Broker](https://github.com/oracle/oci-service-broker). You are
encouraged to explore each approach.

| [Manual](#manual-steps) | [Automated](#using-service-broker) |
|--|--|
| Provides steps for provisioning and connecting cloud services to the application | Uses OCI Service Broker to _provision_ **and** _connect_ the Autonomous Transaction Processing database |

In all cases, begin by adding tenancy credentials to manage and
connect services from within the cluster. Create a secret containing these
values: 

```shell
kubectl create secret generic oci-credentials \
  --namespace mushop \
  --from-literal=tenancy=<TENANCY_OCID> \
  --from-literal=user=<USER_OCID> \
  --from-literal=region=<USER_OCI_REGION> \
  --from-literal=fingerprint=<PUBLIC_API_KEY_FINGERPRINT> \
  --from-literal=passphrase=<PRIVATE_API_KEY_PASSPHRASE> \
  --from-file=privatekey=<PATH_OF_PRIVATE_API_KEY>
```

> **NOTE:** The passphrase entry is **required**. If you do not have passphrase for your key, just leave empty.

### Manual Steps

Follow the steps outlined below to provision and configure the cluster with cloud service connection details

1. Provision an Autonomous Transaction Processing (ATP) database. The default options will work well, as will an **Always Free** shape if available. Once **RUNNING** download the DB Connection Wallet and configure secrets as follows:
    - Create `oadb-admin` secret containing the database administrator password. Used once for schema initializations.

        ```shell
        kubectl create secret generic oadb-admin \
          --namespace mushop \
          --from-literal=oadb_admin_pw='<DB_ADMIN_PASSWORD>'
        ```
    - Create `oadb-wallet` secret with the Wallet _contents_ using the downloaded `Wallet_*.zip`. The extracted `Wallet_*` directory is specified as the secret file path. Each file will become a key name in the secret data.

        ```shell
        kubectl create secret generic oadb-wallet \
          --namespace mushop \
          --from-file=<PATH_TO_EXTRACTED_WALLET_FOLDER>
        ```
    - Create `oadb-connection` secret with the Wallet **password** and the service **TNS name** to use for connections.

        ```shell
        kubectl create secret generic oadb-connection \
          --namespace mushop \
          --from-literal=oadb_wallet_pw='<DB_WALLET_PASSWORD>' \
          --from-literal=oadb_service='<DB_TNS_NAME>'
        ```

        > Each database has 5 unique TNS Names displayed when the Wallet is downloaded. For a database named `mushopdb` an example would be `mushopdb_TP`.

1. **Optional**: Instead of creating a shared database for the entire application, you may establish full separation of services by provisioning _individual_ ATP instances for each service that requires a database. To do so, repeat the previous steps for each database,and give each secret a unique name, for example: `carts-oadb-admin`, `carts-oadb-connection`, `carts-oadb-wallet`.
    - `carts`
    - `catalogue`
    - `orders`
    - `user`

1. Provision a Streaming instance from the Oracle Cloud Infrastructure [Console](https://console.us-phoenix-1.oraclecloud.com/storage/streaming), and make note of the created Stream `OCID` value. Then create an `oss-connection` secret containing the Stream connection details.

    ```shell
    kubectl create secret generic oss-connection \
      --namespace mushop \
      --from-literal=compartmentId='<COMPARTMENT_OCID>' \
      --from-literal=region='<REGION_NAME>' \
      --from-literal=streamId='<STREAM_OCID>' \
      --from-literal=streamName='<STREAM_NAME>'
    ```

1. Verify the secrets are created and available in the `mushop` namespace:

    ```shell
    kubectl get secret --namespace mushop
    ```

    ```text
    NAME              TYPE      DATA   AGE
    oadb-admin        Opaque    1      3m
    oadb-connection   Opaque    2      3m
    oadb-wallet       Opaque    7      3m
    oci-credentials   Opaque    6      3m
    oss-connection    Opaque    4      3m
    ```

### Using Service Broker

As an alternative to manually provisioning, the included [`provision`](https://github.com/oracle-quickstart/oci-cloudnative/tree/master/deploy/complete/helm-chart/provision)
chart is an application of the open-source [OCI Service Broker](https://github.com/oracle/oci-service-broker)
for _provisioning_ Oracle Cloud Infrastructure services. This implementation utilizes [Open Service Broker](https://github.com/openservicebrokerapi/servicebroker/blob/v2.14/spec.md) in Oracle Container Engine for Kubernetes or in other Kubernetes clusters.

```shell--linux-macos
cd deploy/complete/helm-chart
```

```shell--win
dir deploy/complete/helm-chart
```

1. The Service Broker for Kubernetes requires access credentials to provision and
manage services from within the cluster. Create a secret containing these
values as described [above](#provisioning)

1. Deploy the OCI service broker on your cluster. This is done with the [Oracle OCI Service Broker](https://github.com/oracle/oci-service-broker) helm chart:

    ```shell--helm2
    helm install https://github.com/oracle/oci-service-broker/releases/download/v1.3.2/oci-service-broker-1.3.2.tgz \
      --namespace mushop \
      --name oci-service-broker \
      --set ociCredentials.secretName=oci-credentials \
      --set storage.etcd.useEmbedded=true \
      --set tls.enabled=false
    ```

    ```shell--helm3
    helm install oci-service-broker https://github.com/oracle/oci-service-broker/releases/download/v1.3.2/oci-service-broker-1.3.2.tgz \
      --namespace mushop \
      --set ociCredentials.secretName=oci-credentials \
      --set storage.etcd.useEmbedded=true \
      --set tls.enabled=false
    ```

    > The above command will deploy the OCI Service Broker using an embedded etcd instance. It is not recommended to deploy the OCI Service Broker using an embedded etcd instance and tls disabled in production environments, instead a separate etcd cluster should be setup and used by the OCI Service Broker.

1. Next connect Service Catalog with the OCI Service Broker implementation and create an ATP service instance by installing the included `provision` chart:

    ```shell--helm2
    helm install provision \
      --namespace mushop \
      --name mushop-provision \
      --set global.osb.compartmentId=<compartmentId>
    ```

    ```shell--helm3
    helm install mushop-provision provision \
      --namespace mushop \
      --set global.osb.compartmentId=<compartmentId>
    ```

    > Note that the `oci-credentials` secret was created [previously](#provisioning)

1. It will take a few minutes for the ATP database to provision, and the Wallet binding to become available. Verify `serviceinstances` and `servicebindings` are **READY**:

    ```text
    kubectl get serviceinstances -A
    ```

    ```text
    kubectl get servicebindings -A
    ```

## Deployment

Having completed the [provisioning](#provisioning) steps above, the `mushop` deployment
helm chart is installed using settings to leverage cloud backing services.

1. Make a copy of the [`values-dev.yaml`](https://github.com/oracle-quickstart/oci-cloudnative/blob/master/deploy/complete/helm-chart/mushop/values-dev.yaml) file and store somewhere on your machine as `myvalues.yaml`. Then complete the missing values (e.g. secret names) like the following:

    ```yaml
    global:
      ociAuthSecret: oci-credentials        # OCI authentication credentials secret
      ossStreamSecret: oss-connection       # Name of Stream connection secret
      oadbAdminSecret: oadb-admin           # Name of DB Admin secret
      oadbWalletSecret: oadb-wallet         # Name of Wallet secret
      oadbConnectionSecret: oadb-connection # Name of DB Connection secret
    ```

    > **NOTE:** If it's desired to connect a separate databases for a given service, you can specify values specific for each service, such as `carts.oadbAdminSecret`, `carts.oadbWalletSecret`... 

    <aside class="notice">
    Database secrets (<code>oadb*</code>) may be omitted if using the automated service broker approach.
    </aside>

1. Install the [`mushop`](https://github.com/oracle-quickstart/oci-cloudnative/tree/master/deploy/complete/helm-chart/mushop) application helm chart using the `myvalues.yaml` created above:
    - **OPTION 1:** With cloud services provisioned **manually**:

        ```shell--helm2
        helm install ./mushop \
          --name mushop \
          --namespace mushop \
          --values path/to/myvalues.yaml
        ```

        ```shell--helm3
        helm install mushop ./mushop \
          --namespace mushop \
          --values path/to/myvalues.yaml
        ```
    - **OPTION 2:** When using **OCI Service Broker** (`provision` chart):

        ```shell--helm2
        helm install ./mushop \
          --name mushop \
          --namespace mushop \
          --set global.osb.atp=true \
          --values path/to/myvalues.yaml
        ```

        ```shell--helm3
        helm install mushop ./mushop \
          --namespace mushop \
          --set global.osb.atp=true \
          --values path/to/myvalues.yaml
        ```
1. Wait for deployment pods to become **READY** and init pods to show **COMPLETE**:

    ```shell
    kubectl get pods --watch
    ```

    ```text
    NAME                                                     READY   STATUS      RESTARTS   AGE
    mushop-api-75cd7f4f47-2wzpw                              1/1     Running     0          7m55s
    mushop-carts-56c9f548b5-5rxkz                            1/1     Running     0          7m57s
    mushop-carts-init-1-jprv6                                0/1     Completed   0          7m57s
    mushop-catalogue-85cbfb949b-mcjt9                        1/1     Running     0          7m50s
    mushop-catalogue-init-1-cppf9                            0/1     Completed   0          7m57s
    mushop-edge-7959d6b6f8-vkhgm                             1/1     Running     0          7m55s
    mushop-orders-6586dcb898-qkncx                           1/1     Running     0          7m49s
    mushop-orders-init-1-s4hq7                               0/1     Completed   0          7m57s
    mushop-payment-c7dccd8cc-p7jjq                           1/1     Running     0          7m57s
    mushop-session-5ff4c9557f-df4hh                          1/1     Running     0          7m57s
    mushop-shipping-5db9b4c9ff-wxzbp                         1/1     Running     0          7m55s
    mushop-storefront-f5f988cb-ql8bd                         1/1     Running     0          7m57s
    mushop-stream-6fc9d4749c-sfn2h                           1/1     Running     0          7m50s
    mushop-user-65858dd6cd-d5bjs                             1/1     Running     0          7m57s
    mushop-user-init-1-nsx7p                                 0/1     Completed   0          7m57s
    ```

1. Open a browser with the `EXTERNAL-IP` created during setup, **OR** `port-forward`
directly to the `edge` service resource:

    ```shell
    kubectl port-forward \
      --namespace mushop \
      svc/edge 8000:80
    ```

    > Using `port-forward` connecting [localhost:8000](http://localhost:8000) to the `edge` service

    ```shell
    kubectl get svc mushop-setup-nginx-ingress-controller \
      --namespace mushop-setup
    ```

    > Locating `EXTERNAL-IP` for Ingress Controller. **NOTE** this will be
    [localhost](https://localhost) on local clusters.