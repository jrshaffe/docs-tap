# Prerequisites for Prisma Scanner (Alpha)

This topic describes prerequisites for installing SCST - Scan (Prisma) from the VMware package repository.

>**Important** This integration is in Alpha, which means that it is still in active
>development by the Tanzu Practices Global Tech Team and might be subject to
>change at any point. Users might encounter unexpected behavior.

## Verify the latest alpha package version

Run the following command to output a list of available tags.

Use the latest version returned in place of the sample version in this topic, such as `0.1.4-alpha.11` in the following output. 

```console
imgpkg tag list -i projects.registry.vmware.com/tanzu_practice/tap-scanners-package/prisma-repo-scanning-bundle | grep -v sha | sort -V

0.1.4-alpha.1  
0.1.4-alpha.2  
0.1.4-alpha.3  
0.1.4-alpha.4  
0.1.4-alpha.5  
0.1.4-alpha.6  
0.1.4-alpha.11  
```

## Relocate images to a registry

VMware recommends relocating the images from VMware Tanzu Network registry to
your own container image registry before installing. The Prisma Scanner is currently in Alpha development phase, and as such, is not packaged as part of the Tanzu Application Platform package and is hosted on the VMware Project Repository instead of TanzuNet. Therefore, if you relocated the Tanzu Application Platform images, you may also wany to relocate the Prisma Scanner package. If you don’t relocate the
images, the Prisma Scanner installation depends on VMware Tanzu Network for
continued operation, and VMware Tanzu Network offers no uptime guarantees. The
option to skip relocation is documented for evaluation and proof-of-concept
only.

For information about supported registeries, please see the documentation.

To relocate images from the VMware Project Registry to your registry:

1. Install Docker if it is not already installed.

2. Log in to your container image registry by running:

    ```console
    docker login MY-REGISTRY
    ```

    Where `MY-REGISTRY` is your own registry.

3. Set up environment variables for installation by running:

    ```console
    export INSTALL_REGISTRY_USERNAME=MY-REGISTRY-USER
    export INSTALL_REGISTRY_PASSWORD=MY-REGISTRY-PASSWORD
    export INSTALL_REGISTRY_HOSTNAME=MY-REGISTRY
    export VERSION=VERSION-NUMBER
    export INSTALL_REPO=TARGET-REPOSITORY
    ```

    Where:

    - `MY-REGISTRY-USER` is the user with write access to MY-REGISTRY.
    - `MY-REGISTRY-PASSWORD` is the password for `MY-REGISTRY-USER`.
    - `MY-REGISTRY` is your own registry.
    - `VERSION` is your Prisma Scanner version. For example, `0.1.4-alpha.11`.
    - `TARGET-REPOSITORY` is your target repository, a directory or repository on `MY-REGISTRY` that serves as the location for the installation files for Prisma Scanner.

5. [Install the Carvel tool imgpkg CLI](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.4/cluster-essentials/deploy.html#optionally-install-clis-onto-your-path-6).

6. Relocate the images with the imgpkg CLI by running:

    ```console
    imgpkg copy -b projects.registry.vmware.com/tanzu_practice/tap-scanners-package/prisma-repo-scanning-bundle:${VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/prisma-repo-scanning-bundle
    ```

> **Note**
> The VMware project repository does not require authentication, so there is no need to perform a docker login for it.

## Add the Prisma Scanner package repository

Tanzu CLI packages are available on repositories. Adding the Prisma Scanning
package repository makes the Prisma Scanning bundle and its packages available
for installation.

> **Note**
> VMware recommends, but does not require, relocating images to a registry for installation. This section assumes that you relocated images to a registry. See the earlier section to fill in the variables.

VMware recommends installing the Prisma Scanner objects in the existing `tap-install` namespace to keep the Prisma Scanner grouped logically with the other Tanzu Application Platform components.

1. Add the Prisma Scanner package repository to the cluster by running:

    ```console
    tanzu package repository add prisma-scanner-repository \
      --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/prisma-repo-scanning-bundle:$VERSION \
      --namespace tap-install
    ```

1. Get the status of the Prisma Scanner package repository, and ensure that the status updates to `Reconcile succeeded` by running:

    ```console
    tanzu package repository get prisma-scanning-repository --namespace tap-install
    ```

    For example:

    ```console
    $ tanzu package repository get prisma-scanning-repository --namespace tap-install
    - Retrieving repository prisma-scanner-repository...
    NAME:          prisma-scanner-repository
    VERSION:       71091125
    REPOSITORY:    index.docker.io/tapsme/prisma-repo-scanning-bundle
    TAG:           0.1.4-alpha.11
    STATUS:        Reconcile succeeded
    REASON:
    ```

1. List the available packages by running:

    ```console
    tanzu package available list --namespace tap-install
    ```

    For example:

    ```console
    $ tanzu package available list --namespace tap-install
    / Retrieving available packages...
      NAME                                                 DISPLAY-NAME                                                              SHORT-DESCRIPTION
      prisma.scanning.apps.tanzu.vmware.com                Prisma for Supply Chain Security Tools - Scan                             Default scan templates using Prisma
    ```

## Prepare the Prisma Scanner configuration

Before installing the Prisma scanner, you'll need to create the configuration and a Kubernetes secret that contains credentials to access Prisma Cloud.  

### Obtain Console URL and Access Keys and Token

The Prisma Scanner supports two methods of authentication:

1) Basic Authentication with API Key and Secret
2) Token Based Authentication

The steps to configure both are outlined below to allow you to choose which option you use. Note that the token method issued by Prisma Cloud has a expiration of 1 hour, so it will require frequent refreshing.

To obtain your Prisma Compute Console URL and Access Keys and Token. See [Access keys](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/authentication/access_keys) in the Palo Alto Networks documentation.

  >**Note** Generated tokens expire after an hour.

#### Access Key and Secret Authentication

To create a Prisma secret, follow the instructions in the sections below. 

1. Create a Prisma secret YAML file and insert the base64 encoded Prisma API token into the `prisma_token`:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: PRISMA-ACCESS-KEY-SECRET
      namespace: APP-NAME
    data:
      username: BASE64-PRISMA-ACCESS-KEY-ID
      password: BASE64-PRISMA-ACCESS-KEY-PASSWORD
    ```

   Where:
   - `PRISMA-ACCESS-KEY-SECRET` is the name of your Prisma token secret.
   - `APP-NAME` is the namespace you want to use.
   - `BASE64-PRISMA-ACCESS-KEY-ID` is your base64 encoded Prisma Access Key ID.
   - `BASE64-PRISMA-ACCESS-KEY-PASSWORD` is your base64 encoded Prisma Access Key Password.

2. Apply the Prisma secret YAML file by running:

    ```console
    kubectl apply -f YAML-FILE
    ```

   Where `YAML-FILE` is the name of the Prisma secret YAML file you created.

3. Define the `--values-file` flag to customize the default configuration. You
   must define the following fields in the `values.yaml` file for the Prisma
   Scanner configuration. You can add fields to activate or deactivate
   behaviors. You can append the values to this file as shown later in this
   topic. Create a `values.yaml` file by using the following configuration:

    ```yaml
    ---
    namespace: DEV-NAMESPACE
    targetImagePullSecret: TARGET-REGISTRY-CREDENTIALS-SECRET
    prisma:
      url: PRISMA-URL
      basicAuth:
        name: PRISMA-ACCESS-KEY-SECRET
    ```

   Where:

   - `DEV-NAMESPACE` is your developer namespace.
   > **Note** To use a namespace other than the default namespace, ensure that
     the namespace exists before you install. If the namespace does not exist,
     the scanner installation fails.
   - `TARGET-REGISTRY-CREDENTIALS-SECRET` is the name of the secret that contains the
     credentials to pull an image from a private registry for scanning.
   - `PRISMA-URL` is the FQDN of your Twistlock server.
   - `PRISMA-CONFIG-SECRET` is the name of the secret you created that contains the
     Prisma configuration to connect to Prisma. This field is required.

The Prisma integration can work with or without the SCST - Store integration.
The values.yaml file is slightly different for each configuration.

#### Access Token Authentication
1. Create a Prisma secret YAML file and insert the base64 encoded Prisma API token into the `prisma_token`:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: PRISMA-TOKEN-SECRET
      namespace: APP-NAME
    data:
      prisma_token: BASE64-PRISMA-API-TOKEN
    ```

   Where:
 
   - `PRISMA-TOKEN-SECRET` is the name of your Prisma token secret.
    - `APP-NAME` is the namespace you want to use.
    - `BASE64-PRISMA-API-TOKEN` is the name of your base64 encoded Prisma API token.

2. Apply the Prisma secret YAML file by running:

    ```console
    kubectl apply -f YAML-FILE
    ```

    Where `YAML-FILE` is the name of the Prisma secret YAML file you created.

3. Define the `--values-file` flag to customize the default configuration. You
   must define the following fields in the `values.yaml` file for the Prisma
   Scanner configuration. You can add fields as needed to activate or deactivate
   behaviors. You can append the values to this file as shown later in this
   topic. Create a `values.yaml` file by using the following configuration:

    ```yaml
    ---
    namespace: DEV-NAMESPACE
    targetImagePullSecret: TARGET-REGISTRY-CREDENTIALS-SECRET
    prisma:
      url: PRISMA-URL
      tokenSecret:
        name: PRISMA-CONFIG-SECRET
    ```

   Where:

   - `DEV-NAMESPACE` is your developer namespace.
    > **Note** To use a namespace other than the default namespace, ensure that
      the namespace exists before you install. If the namespace does not exist,
      the scanner installation fails.
   - `TARGET-REGISTRY-CREDENTIALS-SECRET` is the name of the secret that contains the
     credentials to pull an image from a private registry for scanning.
   - `PRISMA-URL` is the FQDN of your Twistlock server.
   - `PRISMA-CONFIG-SECRET` is the name of the secret you created that contains the
     Prisma configuration to connect to Prisma. This field is required.

## Supply Chain Security Tools - Store integration

When using SCST - Store integration, to persist the results
found by the Prisma Scanner, you can enable the SCST -
Store integration by appending the fields to the `values.yaml` file.

The Grype, Snyk, and Prisma Scanner Integrations enable the Metadata Store. To
prevent conflicts, the configuration values are slightly different based on
whether the Grype Scanner Integration is installed or not. If Tanzu Application
Platform is installed using the Full Profile, the Grype Scanner Integration is
installed unless it is explicitly excluded.

If the Grype or Snyk Scanner Integration is installed in the same dev-namespace where the Prisma Scanner is installed:

```yaml
#! ...
metadataStore:
  #! The url where the Store deployment is accessible.
  #! Default value is: "https://metadata-store-app.metadata-store.svc.cluster.local:8443"
  url: "STORE-URL"
  caSecret:
    #! The name of the secret that contains the ca.crt to connect to the Store Deployment.
    #! Default value is: "app-tls-cert"
    name: "CA-SECRET-NAME"
    importFromNamespace: "" #! since both Prisma and Grype/Snyk both enable store, one must leave importFromNamespace blank
  #! authSecret is for multicluster configurations.
  authSecret:
    #! The name of the secret that contains the auth token to authenticate to the Store Deployment.
    name: "AUTH-SECRET-NAME"
    importFromNamespace: "" #! since both Prisma and Grype/Snyk both enable store, one must leave importFromNamespace blank
```

Where:

- `STORE-URL` is the URL where the Store deployment is accessible.
- `CA-SECRET-NAME` is the name of the secret that contains the ca.crt to connect
  to the Store Deployment. Default is `app-tls-cert`.
- `AUTH-SECRET-NAME` is the name of the secret that contains the authentication token to
  authenticate to the Store Deployment.

If the Grype or Snyk Scanner Integration is not installed in the same dev-namespace Prisma Scanner is installed:

```yaml
#! ...
metadataStore:
  #! The url where the Store deployment is accessible.
  #! Default value is: "https://metadata-store-app.metadata-store.svc.cluster.local:8443"
  url: "STORE-URL"
  caSecret:
    #! The name of the secret that contains the ca.crt to connect to the Store Deployment.
    #! Default value is: "app-tls-cert"
    name: "CA-SECRET-NAME"
    #! The namespace where the secrets for the Store Deployment live.
    #! Default value is: "metadata-store"
    importFromNamespace: "STORE-SECRETS-NAMESPACE"
  #! authSecret is for multicluster configurations.
  authSecret:
    #! The name of the secret that contains the auth token to authenticate to the Store Deployment.
    name: "AUTH-SECRET-NAME"
    #! The namespace where the secrets for the Store Deployment live.
    importFromNamespace: "STORE-SECRETS-NAMESPACE"
```

Where:

- `STORE-URL` is the URL where the Store deployment is accessible.
- `CA-SECRET-NAME` is the name of the secret that contains the `ca.crt` to connect to the Store Deployment. Default is `app-tls-cert`.
- `STORE-SECRETS-NAMESPACE` is the namespace where the secrets for the Store Deployment live. Default is `metadata-store`.
- `AUTH-SECRET-NAME` is the name of the secret that contains the authentication token to authenticate to the Store Deployment.

**Without Supply Chain Security Tools - Store Integration:** If you don’t want
to enable the SCST - Store integration, explicitly deactivate the integration by
appending the following fields to the values.yaml file that is enabled by
default:

```yaml
# ...
metadataStore:
  url: "" # Configuration is moved, so set this string to empty
```

## Prepare the ScanPolicy

To prepare the ScanPolicy, follow the instructions in the sections below.

### Sample ScanPolicy using Prisma Policies

The following sample ScanPolicy allows you to control whether the SupplyChain passes or fails based on the compliance and vulnerability rules configured in the Prisma Compute Console.

The policy reads the `complianceScanPassed` and `vulnerabilityScanPassed` fields returned from Prisma scanner output to control the results of the scan.

```yaml
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  name: prisma-scan-policy
  labels:
    'app.kubernetes.io/part-of': 'enable-in-gui'
spec:
  regoFile: |
    package main
    
    import future.keywords.in
    import future.keywords.if
    
    failedPrismaComplianceOrVulnerabilityChecks(metadata) {
      x := false in cast_set(metadata.properties.property)
      x
    }

    deny[msg] {
      failedPrismaComplianceOrVulnerabilityChecks(input.bom.metadata)
      vulnerabilityMessages := { message |
        components := {e | e := input.bom.components.component} | {e | e := input.bom.components.component[_]}
        some component in components
        vulnerabilities := {e | e := component.vulnerabilities.vulnerability} | {e | e := component.vulnerabilities.vulnerability[_]}
        some vulnerability in vulnerabilities
        ratings := {e | e := vulnerability.ratings.rating.severity} | {e | e := vulnerability.ratings.rating[_].severity}
        formattedRatings := concat(", ", ratings)
        message := sprintf("Vulnerability - Component: %s CVE: %s Severity: %s", [component.name, vulnerability.id, formattedRatings])
      }
      complianceMessages := { message |
        compliances := {e | e := input.bom.metadata.component.compliances.compliance} | {e | e := input.bom.metadata.component.compliances.compliance[_]}
        some compliance in compliances
        message := sprintf("Compliance - %s \\\nId: %s Severity: %s Category: %s", [compliance.title, compliance.id, compliance.severity, compliance.category])
      }
      combinedMessages := complianceMessages | vulnerabilityMessages
      some message in combinedMessages
      msg := message
    }

```

### Sample ScanPolicy using Local Policies

The following sample ScanPolicy allows you to control whether the SupplyChain passes or fails based on the Prisma Scanner CycloneDX vulnerability results returned from the Prisma Scanner.

```yaml
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  name: prisma-scan-policy
  labels:
    'app.kubernetes.io/part-of': 'enable-in-gui'
spec:
  regoFile: |
    package main

    # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
    notAllowedSeverities := ["Critical", "High", "UnknownSeverity"]
    ignoreCves := []

    contains(array, elem) = true {
      array[_] = elem
    } else = false { true }

    isSafe(match) {
      severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
      some i
      fails := contains(notAllowedSeverities, severities[i])
      not fails
    }

    isSafe(match) {
      ignore := contains(ignoreCves, match.id)
      ignore
    }

    deny[msg] {
      comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
      some i
      comp := comps[i]
      vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
      some j
      vuln := vulns[j]
      ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
      not isSafe(vuln)
      msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
    }
```

Apply the YAML:

```console
kubectl apply -n $DEV-NAMESPACE -f SCAN-POLICY-YAML
```

Where:

- `DEV-NAMESPACE` is the name of the developer namespace you want to use.
- `SCAN-POLICY-YAML` is the name of your SCST - Scan YAML.

## Install Prisma Scanner

After all prerequisites are completed, install the Prisma Scanner. See [Install another scanner for Supply Chain Security Tools - Scan](install-scanners.hbs.md).

## Known Limits

- OpenShift is not supported
