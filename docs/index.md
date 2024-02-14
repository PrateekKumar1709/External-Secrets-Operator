# **Secrets Management with External Secrets Operator**

## **Introduction**

Managing secrets in a dynamic, cloud-based environment like Kubernetes can be a security headache. Traditional approaches, like hardcoding them in config files or storing them as Kubernetes Secrets, have vulnerabilities.

But what if there was a way to securely store secrets outside of Kubernetes and automatically inject them into your applications when needed? Enter the ==External Secrets Operator== (ESO), a game-changer for Kubernetes security.

## **What is the External Secrets Operator?**

ESO is a Kubernetes operator that bridges the gap between your desired secret store (e.g., AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) and your Kubernetes cluster. It acts as a mediator, fetching secrets from external providers and injecting them as Kubernetes Secrets into your pods.

## **Architecture**

The External Secrets Operator extends Kubernetes with Custom Resources, which define where secrets live and how to synchronize them. The controller fetches secrets from an external API and creates Kubernetes secrets. If the secret from the external API changes, the controller will reconcile the state in the cluster and update the secrets accordingly.

![Architecture](https://external-secrets.io/latest/pictures/diagrams-high-level-simple.png)

## **Resource Model**

The External Secrets Operator (ESO) simplifies secure secret management in Kubernetes by fetching them from external stores and injecting them into your cluster seamlessly. Here's a breakdown without jargon:

Think of ESO as a bridge:
It connects your Kubernetes cluster to your preferred external secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager).
Instead of storing secrets directly in Kubernetes (risky!), ESO keeps them secure outside.
When your applications need them, ESO acts as a middleman, fetching the secrets and injecting them as Kubernetes Secrets.

### **How does ESO do this magic?**

It uses special resources called Custom Resources (CRs):

- **SecretStore**: Defines how to access your external secrets manager (credentials, etc.).
You can configure multiple SecretStores for different managers or setups.

- **ClusterSecretStore**: Similar to SecretStore, but accessible from any namespace in your cluster.

- **ExternalSecret**: Acts as a blueprint for creating a secret.
It specifies which secret to fetch from the external store and where to put it in Kubernetes.

- **ClusterExternalSecret**: Like ExternalSecret, but can create secrets in multiple namespaces.

## **Installing ESO on K8s**

ESO can be installed using Helm or via an ArgoCD application.

### **Option 1: Helm**

- Add the External Secrets repo

```yaml
helm repo add external-secrets https://charts.external-secrets.io
```

- Install the External Secrets Operator

```yaml
helm install external-secrets \
external-secrets/external-secrets \
-n external-secrets \
--create-namespace \
# --set installCRDs=true
```

### **Option 2: Argo CD**

- Create a YAML file eg.`external-secrets.yaml` with the following file content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
spec:
  source:
    repoURL: https://charts.external-secrets.io
    revision: 0.5.9
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
    createNamespace: true
  helm:
    valueFiles: []
```

- Apply the manifest using the following command:

```yaml
argocd apply -f external-secrets.yaml
```

## **Providers**

External Secrets Operator integrates with a number of providers for secret management. Eg.AWS Secrets Manager, Azure Key Vault, Hashicorp Valut, Kubernetes etc,

This project utilizes **Hashicorp Vault** as the provider for secret management. The KV Secrets Engine is the only one supported by this provider.

Assuming you already have Vault set up and your token is stored securely, here's how to configure ESO to use Vault:

### Create a SecretStore resource in your desired namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://my.vault.server:8200"
      path: "secret"
      # Version is the Vault KV secret engine version.
      # This can be either "v1" or "v2", defaults to "v2"
      version: "v2"
      auth:
        # points to a secret that contains a vault token
        # https://www.vaultproject.io/docs/auth/token
        tokenSecretRef:
          name: "vault-token"
          key: "token"
```

- ==kind==: Specifies the resource type (SecretStore).
- ==metadata.name==: Identifies the SecretStore as vault-backend. 
- ==provider.vault==: Describes the secrets provider as Vault. 
- ==server==: URL of your Vault server (including port, "http://my.vault.server:8200"). 
- ==path==: Specifies the path within Vault where secrets are stored ("secret"). 
- ==tokenSecretRef.name==: Name of the Secret containing the token ("vault-token").
- ==tokenSecretRef.key==: Key within the Secret where the token is stored ("token").


### Create a simple k/v pair at path secret/foo:

```yaml
vault kv put secret/foo my-value=s3cr3t
```

### Create an ExternalSecret
Create a YAML file eg.example-secrets.yaml with the following file content:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
  namespace: default
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: example-sync
  data:
    - secretKey: foobar
      remoteRef:
        key: foo
        property: my-value
```

- ==name & namespace==: Identify the External Secret (vault-example) and where it'll be created (default namespace).
- ==refreshInterval==: How often ESO checks for updates to the external secret (15 seconds).
- ==secretStoreRef==: Points to the SecretStore providing access to the external secret manager (vault-backend).
- ==target==: Specifies the secret to fetch from the external store (example-sync).
- ==data==: Describes how to inject the secret into pods:
- ==secretKey==: Name of the key in the injected Kubernetes Secret (foobar).
- ==remoteRef==: Where to find the secret within the external store:
    - ==key==: Path to the secret (foo).
    - ==property==: Specific value to extract (my-value).

## **Deploy using Argo CD**

- Create a git repository and add the above ExternalSecret manifest to it.
- Sync the ExternalSecret from the git repository to the cluster by creating an app named demo-secret in Argo CD. For this create a YAML file eg.`demo-secrets.yaml` with the following file content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-secret
spec:
  source:
    repoURL: ${GIT_REPO_URL_CREATED_ABOVE}
    path: ${MANIFEST_PATH}
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated: {}
```

You can then view the demo-secret in Argo CD UI.

## **Refrences:**

- [ESO Documentation](https://external-secrets.io/latest/)
- [Vault Documentation](https://www.vaultproject.io/)
- [Argo CD Documentation](https://argoproj.github.io/cd/)
