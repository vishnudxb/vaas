# Vault-as-a-service on GKE with Terraform


**These configurations require Terraform 0.12+!**

## Feature Highlights

- **Vault HA** - The Vault cluster is deployed in HA mode backed by [Google
  Cloud Storage][gcs]

- **Production Hardened** - Vault is deployed according to the [production
  hardening
  guide](https://www.vaultproject.io/guides/operations/production.html).
  
- **Auto-Init and Unseal** - Vault is automatically initialized and unsealed
  at runtime. The unseal keys are encrypted with [Google Cloud KMS][kms] and
  stored in [Google Cloud Storage][gcs]

- **Full Isolation** - The Vault cluster is provisioned in it's own Kubernetes
  cluster in a dedicated GCP project that is provisioned dynamically at
  runtime. Clients connect to Vault using **only** the load balancer and Vault
  is treated as a managed external service.

- **Audit Logging** - Audit logging to Stackdriver can be optionally enabled
  with minimal additional configuration.


## STEPS

1. Download and install [Terraform][terraform].

1. Download, install, and configure the [Google Cloud SDK][sdk]. You will need
   to configure your default application credentials so Terraform can run. It
   will run against your default project, but all resources are created in the
   (new) project that it creates.

1. Run Terraform:

    ```text
    $ cd terraform/
    $ terraform init
    $ terraform apply
    ```

    This operation will take some time as it:

    1. Creates a new project or specify an existing project
    1. Enables the required services on that project
    1. Creates a bucket for storage
    1. Creates a KMS key for encryption
    1. Creates a service account with the most restrictive permissions to those resources
    1. Creates a GKE cluster with the configured service account attached
    1. Creates a public IP
    1. Generates a self-signed certificate authority (CA)
    1. Generates a certificate signed by that CA
    1. Configures Terraform to talk to Kubernetes
    1. Creates a Kubernetes secret with the TLS file contents
    1. Configures your local system to talk to the GKE cluster by getting the cluster credentials and kubernetes context
    1. Submits the StatefulSet and Service to the Kubernetes API


## Interact with Vault

1. Install Vault on your local machine

   https://www.vaultproject.io/downloads.html

1. Export environment variables:

    Vault reads these environment variables for communication. Set Vault's
    address, the CA to use for validation, and the initial root token.

    ```text
    # Make sure you're in the terraform/ directory
    # $ cd terraform/

    $ export VAULT_ADDR="https://$(terraform output address)"
    $ export VAULT_TOKEN="$(terraform output root_token)"
    $ export VAULT_CAPATH="$(cd ../ && pwd)/tls/ca.pem"
    ```
    For example:

    ```text
    $ export VAULT_ADDR=https://1.1.1.1
    $ export VAULT_TOKEN=xxxxtoken
    $ export VAULT_CAPATH=~/vault/tls/ca.pem    
    ```

1. Run some commands:

    ```text
    $ vault secrets enable -path=secret -version=2 kv
    $ vault kv put secret/foo a=b
    ```

    Sample Error: You need enable the path first otherwise you will get errors like below:

    ```text
    $  vault kv put secret/hello foo=world
        Error making API request.
        URL: GET https://1.1.1.1/v1/sys/internal/ui/mounts/secret/hello
        Code: 403. Errors:
        preflight capability check returned 403, please ensure client's policies grant access to path "secret/hello/"
    ```

    Sample Usage:

    ```text
    $ vault secrets enable -path=secret -version=2 kv

    Success! Enabled the kv secrets engine at: secret/

    $ vault kv put secret/hello foo=world

    Key              Value
    ---              -----
    created_time     2020-01-14T18:41:24.952919365Z
    deletion_time    n/a
    destroyed        false
    version          1

    $ vault kv get secret/hello

    ====== Metadata ======
    Key              Value
    ---              -----
    created_time     2020-01-14T18:41:24.952919365Z
    deletion_time    n/a
    destroyed        false
    version          1

    === Data ===
    Key    Value
    ---    -----
    foo    world

    ```


## Audit Logging

Audit logging is not enabled in a default Vault installation. To enable audit
logging to [Stackdriver][stackdriver] on Google Cloud, enable the `file` audit
device on `stdout`:

```text
$ vault audit enable file file_path=stdout
```

That's it! Vault will now log all audit requests to Stackdriver. Additionally,
because the configuration uses an L4 load balancer, Vault does not need to
parse `X-Forwarded-For` headers to extract the client IP, as requests are
passed directly to the node.

## Additional Permissions

You may wish to grant the Vault service account additional permissions. This
service account is attached to the GKE nodes and will be the "default
application credentials" for Vault.

To specify additional permissions, create a `terraform.tfvars` file with the
following:

```terraform
service_account_custom_iam_roles = [
  "roles/...",
]
```

### GCP Auth Method

To use the [GCP auth method][vault-gcp-auth] with the default application
credentials, the Vault server needs the following role:

```text
roles/iam.serviceAccountKeyAdmin
```

Alternatively you can create and upload a dedicated service account for the
GCP auth method during configuration and restrict the node-level default
application credentials.

### GCP Secrets Engine

To use the [GCP secrets engine][vault-gcp-secrets] with the default
application credentials, the Vault server needs the following roles:

```text
roles/iam.serviceAccountKeyAdmin
roles/iam.serviceAccountAdmin
```

Additionally, Vault needs the superset of any permissions it will grant. For
example, if you want Vault to generate GCP access tokens with access to
compute, you must also grant Vault access to compute.

Alternatively you can create and upload a dedicated service account for the
GCP auth method during configuration and restrict the node-level default
application credentials.


## Cleaning Up

```text
$ terraform destroy
```

## Reference

1. https://cloud.google.com/storage

1. https://cloud.google.com/kubernetes-engine

1. https://cloud.google.com/kms

1. https://cloud.google.com/sdk

1. https://github.com/kelseyhightower/vault-on-google-kubernetes-engine

1. https://cloud.google.com/stackdriver/

1. https://www.terraform.io

1. https://www.vaultproject.io

1. https://www.vaultproject.io/docs/auth/gcp.html

1. https://www.vaultproject.io/docs/secrets/gcp/index.html
