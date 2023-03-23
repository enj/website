---
reviewers:
- smarterclayton
- enj
title: Using a KMS provider for data encryption
content_type: task
---
<!-- overview -->
This page shows how to configure a Key Management Service (KMS) provider and plugin to enable secret data encryption.
Currently there are two KMS API versions. New integrations should use KMS v2 as it offers significantly
better performance characteristics than v1.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}}

The version of Kubernetes that you need depends on which KMS API version
you have selected. 

- If you selected KMS API v1, any supported Kubernetes version will work fine.
- If you selected KMS API v2, you should use Kubernetes v{{< skew currentVersion >}}
  (if you are running a different version of Kubernetes that also supports the v2 KMS
  API, switch to the documentation for that version of Kubernetes).

{{< version-check >}}

### KMS v1
* Kubernetes version 1.10.0 or later is required

* Your cluster must use etcd v3 or later

{{< feature-state for_k8s_version="v1.12" state="beta" >}}

### KMS v2
{{< feature-state for_k8s_version="v1.27" state="beta" >}}

* For version 1.25 and 1.26, enabling the feature via kube-apiserver feature gate is required: `--feature-gates=KMSv2=true` to configure a KMS v2 provider

* Your cluster must use etcd v3 or later

{{< note >}}

The KMS v2 API and implementation changed in incompatible ways in-between the alpha release in v1.25
and the beta release in v1.27.  Attempting to upgrade between these versions with the alpha feature
enabled will result in data loss.

{{< /note >}}

<!-- steps -->

The KMS encryption provider uses an envelope encryption scheme to encrypt data in etcd.
The data is encrypted using a data encryption key (DEK).
The DEKs are encrypted with a key encryption key (KEK) that is stored and managed in a remote KMS.
With KMS v1, a new DEK is generated for each encryption.
With KMS v2, a new DEK is generated on server startup and when the KMS plugin informs the API server
that a KEK rotation has occurred (see `key_id` section below).
The KMS provider uses gRPC to communicate with a specific KMS plugin over a UNIX domain socket.
The KMS plugin, which is implemented as a gRPC server and deployed on the same host(s)
as the Kubernetes control plane, is responsible for all communication with the remote KMS.

## Configuring the KMS provider

To configure a KMS provider on the API server, include a provider of type `kms` in the
`providers` array in the encryption configuration file and set the following properties:

### KMS v1 {#configuring-the-kms-provider-kms-v1}
* `apiVersion`: API Version for KMS provider. Leave this value empty.
* `name`: Display name of the KMS plugin. Cannot be changed once set.
* `endpoint`: Listen address of the gRPC server (KMS plugin). The endpoint is a UNIX domain socket.
* `cachesize`: Number of data encryption keys (DEKs) to be cached in the clear.
  When cached, DEKs can be used without another call to the KMS;
  whereas DEKs that are not cached require a call to the KMS to unwrap.
* `timeout`: How long should `kube-apiserver` wait for kms-plugin to respond before
  returning an error (default is 3 seconds).

### KMS v2 {#configuring-the-kms-provider-kms-v2}
* `apiVersion`: API Version for KMS provider. Set this to `v2`.
* `name`: Display name of the KMS plugin. Cannot be changed once set.
* `endpoint`: Listen address of the gRPC server (KMS plugin). The endpoint is a UNIX domain socket.
* `timeout`: How long should `kube-apiserver` wait for kms-plugin to respond before
  returning an error (default is 3 seconds).

KMS v2 does not support the `cachesize` parameter. All data encryption keys (DEKs) will be cached in
the clear once the server has unwrapped them via a call to the KMS. Once cached, DEKs can be used
to perform decryption indefinitely without making a call to the KMS.

See [Understanding the encryption at rest configuration](/docs/tasks/administer-cluster/encrypt-data).

## Implementing a KMS plugin

To implement a KMS plugin, you can develop a new plugin gRPC server or enable a KMS plugin
already provided by your cloud provider.
You then integrate the plugin with the remote KMS and deploy it on the Kubernetes master.

### Enabling the KMS supported by your cloud provider

Refer to your cloud provider for instructions on enabling the cloud provider-specific KMS plugin.

### Developing a KMS plugin gRPC server

You can develop a KMS plugin gRPC server using a stub file available for Go. For other languages,
you use a proto file to create a stub file that you can use to develop the gRPC server code.

#### KMS v1 {#developing-a-kms-plugin-gRPC-server-kms-v1}
* Using Go: Use the functions and data structures in the stub file:
  [api.pb.go](https://github.com/kubernetes/kms/blob/release-{{< skew currentVersion >}}/apis/v1beta1/api.pb.go)
  to develop the gRPC server code 

* Using languages other than Go: Use the protoc compiler with the proto file:
  [api.proto](https://github.com/kubernetes/kms/blob/release-{{< skew currentVersion >}}/apis/v1beta1/api.proto)
  to generate a stub file for the specific language

#### KMS v2 {#developing-a-kms-plugin-gRPC-server-kms-v2}
* Using Go: A high level
  [library](https://github.com/kubernetes/kms/blob/release-{{< skew currentVersion >}}/pkg/service/interface.go)
  is provided to make the process easier.  Low level implementations
  can use the functions and data structures in the stub file:
  [api.pb.go](https://github.com/kubernetes/kms/blob/release-{{< skew currentVersion >}}/apis/v2/api.pb.go)
  to develop the gRPC server code 

* Using languages other than Go: Use the protoc compiler with the proto file:
  [api.proto](https://github.com/kubernetes/kms/blob/release-{{< skew currentVersion >}}/apis/v2/api.proto)
  to generate a stub file for the specific language

Then use the functions and data structures in the stub file to develop the server code.

#### Notes

##### KMS v1 {#developing-a-kms-plugin-gRPC-server-notes-kms-v1}
* kms plugin version: `v1beta1`

  In response to procedure call Version, a compatible KMS plugin should return `v1beta1` as `VersionResponse.version`.

* message version: `v1beta1`

  All messages from KMS provider have the version field set to `v1beta1`.

* protocol: UNIX domain socket (`unix`)

  The plugin is implemented as a gRPC server that listens at UNIX domain socket. The plugin deployment should create a file on the file system to run the gRPC unix domain socket connection. The API server (gRPC client) is configured with the KMS provider (gRPC server) unix domain socket endpoint in order to communicate with it. An abstract Linux socket may be used by starting the endpoint with `/@`, i.e. `unix:///@foo`. Care must be taken when using this type of socket as they do not have concept of ACL (unlike traditional file based sockets). However, they are subject to Linux networking namespace, so will only be accessible to containers within the same pod unless host networking is used.

##### KMS v2 {#developing-a-kms-plugin-gRPC-server-notes-kms-v2}
* KMS plugin version: `v2beta1`

  In response to procedure call Status, a compatible KMS plugin should return `v2beta1` as `StatusResponse.version`,
  "ok" as `StatusResponse.healthz` and a `key_id` (remote KMS KEK ID) as `StatusResponse.key_id`.

  The API server polls the Status procedure call approximately every minute when everything is healthy,
  and more frequently when the plugin is not healthy.  Plugins must take care to optimize this call as it will be
  under constant load.

* Encryption

  The EncryptRequest procedure call provides the plaintext and a UID for logging purposes.  The response must include
  the ciphertext, the `key_id` for the KEK used, and, optionally, any metadata that the KMS plugin needs to aid in
  future DecryptRequest calls (via the `annotations` field).  The plugin must guarantee that any distinct plaintext
  results in a distinct response `(ciphertext, key_id, annotations)`.

  TODO annotations validation

* Decryption

  The DecryptRequest procedure call provides the `(ciphertext, key_id, annotations)` from EncryptRequest and a UID
  for logging purposes.  As expected, it is the inverse of the EncryptRequest call.  Plugins must verify that the
  `key_id` is one that they understand - they must not attempt to decrypt data unless they are sure that it was
  encrypted by them at an earlier time.

* Understanding `key_id`

  The `key_id` is the public, non-secret name of the remote KMS KEK that is currently in use.  It may be logged
  during regular operation of the API server, and thus must not contain any private data.  Plugin implementations
  are encouraged to use a hash to avoid leaking any data.  The KMS v2 metrics take care to hash this value before
  exposing it via the `/metrics` endpoint.

  TODO Plugins must guarantee that the `key_id` returned from Status

  TODO Rotation times

* protocol: UNIX domain socket (`unix`)

  The plugin is implemented as a gRPC server that listens at UNIX domain socket. The plugin deployment should create a file on the file system to run the gRPC unix domain socket connection. The API server (gRPC client) is configured with the KMS provider (gRPC server) unix domain socket endpoint in order to communicate with it. An abstract Linux socket may be used by starting the endpoint with `/@`, i.e. `unix:///@foo`. Care must be taken when using this type of socket as they do not have concept of ACL (unlike traditional file based sockets). However, they are subject to Linux networking namespace, so will only be accessible to containers within the same pod unless host networking is used. 

### Integrating a KMS plugin with the remote KMS

The KMS plugin can communicate with the remote KMS using any protocol supported by the KMS.
All configuration data, including authentication credentials the KMS plugin uses to communicate with the remote KMS, 
are stored and managed by the KMS plugin independently.
The KMS plugin can encode the ciphertext with additional metadata that may be required before sending it to the KMS
for decryption (KMS v2 makes this process easier by providing a dedicated `annotations` field).

### Deploying the KMS plugin

Ensure that the KMS plugin runs on the same host(s) as the Kubernetes master(s).

## Encrypting your data with the KMS provider

To encrypt the data:

1. Create a new `EncryptionConfiguration` file using the appropriate properties for the `kms` provider
to encrypt resources like Secrets and ConfigMaps. If you want to encrypt an extension API that is
defined in a CustomResourceDefinition, your cluster must be running Kubernetes v1.26 or newer.

1. Set the `--encryption-provider-config` flag on the kube-apiserver to point to the location of the configuration file.

1. `--encryption-provider-config-automatic-reload` boolean argument determines if the file set by `--encryption-provider-config` should be automatically reloaded if the disk contents change. This enables key rotation without API server restarts.

1. Restart your API server.

### KMS v1 {#encrypting-your-data-with-the-kms-provider-kms-v1}

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
         - configmaps
         - pandas.awesome.bears.example
       providers:
         - kms:
             name: myKmsPluginFoo
             endpoint: unix:///tmp/socketfile.sock
             cachesize: 100
             timeout: 3s
         - kms:
             name: myKmsPluginBar
             endpoint: unix:///tmp/socketfile.sock
             cachesize: 100
             timeout: 3s
   ```

### KMS v2 {#encrypting-your-data-with-the-kms-provider-kms-v2}

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
         - configmaps
         - pandas.awesome.bears.example
       providers:
         - kms:
             apiVersion: v2
             name: myKmsPluginFoo
             endpoint: unix:///tmp/socketfile.sock
             timeout: 3s
         - kms:
             apiVersion: v2
             name: myKmsPluginBar
             endpoint: unix:///tmp/socketfile.sock
             timeout: 3s
   ```

Setting `--encryption-provider-config-automatic-reload` to `true` collapses all health checks to a single health check endpoint. Individual health checks are only available when KMS v1 providers are in use and the encryption config is not auto-reloaded.

The following table summarizes the health check endpoints for each KMS version:

| KMS configurations        | Without Automatic Reload           | With Automatic Reload  |
| ------------------------- |------------------------------------| -----------------------|
| KMS v1 only               | Individual Healthchecks            | Single Healthcheck     |
| KMS v2 only               | Single Healthcheck                 | Single Healthcheck     |
| Both KMS v1 and v2        | Individual Healthchecks            | Single Healthcheck     |
| No KMS                    | None                               | Single Healthcheck     |

`Single Healthcheck` means that the only health check endpoint is `/healthz/kms-providers`.

`Individual Healthchecks` means that each KMS plugin has an associated health check endpoint based on its location in the encryption config: `/healthz/kms-provider-0`, `/healthz/kms-provider-1` etc.

These healthcheck endpoint paths are hard coded and generated/controlled by the server. The indices for individual healthchecks corresponds to the order in which the KMS encryption config is processed.

At a high level, restarting an API server when a KMS plugin is unhealthy is unlikely to make the situation better.
It can make the situation significantly worse by throwing away the API server's DEK cache.  Thus the general
recommendation is to ignore KMS healthz checks for liveness purposes, i.e. `/livez?exclude=kms-providers`.

Until the steps defined in [Ensuring all secrets are encrypted](#ensuring-all-secrets-are-encrypted) are performed, the `providers` list should end with the `identity: {}` provider to allow unencrypted data to be read.  Once all resources are encrypted, the `identity` provider should be removed to prevent the API server from honoring unencrypted data.

For details about the `EncryptionConfiguration` format, please check the
[API server encryption API reference](/docs/reference/config-api/apiserver-encryption.v1/).

## Verifying that the data is encrypted

Data is encrypted when written to etcd. After restarting your `kube-apiserver`, 
any newly created or updated Secret or other resource types configured in `EncryptionConfiguration` should be encrypted when stored. To verify,
you can use the `etcdctl` command line program to retrieve the contents of your secret data.

1. Create a new secret called `secret1` in the `default` namespace:

   ```shell
   kubectl create secret generic secret1 -n default --from-literal=mykey=mydata
   ```

1. Using the `etcdctl` command line, read that secret out of etcd:

   ```shell
   ETCDCTL_API=3 etcdctl get /kubernetes.io/secrets/default/secret1 [...] | hexdump -C
   ```

   where `[...]` contains the additional arguments for connecting to the etcd server.

1. Verify the stored secret is prefixed with `k8s:enc:kms:v1:` for KMS v1 or prefixed with `k8s:enc:kms:v2:` for KMS v2, which indicates that the `kms` provider has encrypted the resulting data.

1. Verify that the secret is correctly decrypted when retrieved via the API:

   ```shell
   kubectl describe secret secret1 -n default
   ```

   The Secret should contain `mykey: mydata`

## Ensuring all secrets are encrypted

When encryption at rest is correctly configured, resources are encrypted on write.
Thus we can perform an in-place no-op update to ensure that data is encrypted.

The following command reads all secrets and then updates them to apply server side encryption.
If an error occurs due to a conflicting write, retry the command.
For larger clusters, you may wish to subdivide the secrets by namespace or script an update.

```shell
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

## Switching from a local encryption provider to the KMS provider

To switch from a local encryption provider to the `kms` provider and re-encrypt all of the secrets:

1. Add the `kms` provider as the first entry in the configuration file as shown in the following example.

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
       providers:
         - kms:
             apiVersion: v2
             name : myKmsPlugin
             endpoint: unix:///tmp/socketfile.sock
         - aescbc:
             keys:
               - name: key1
                 secret: <BASE 64 ENCODED SECRET>
   ```

1. Restart all `kube-apiserver` processes.

1. Run the following command to force all secrets to be re-encrypted using the `kms` provider.

   ```shell
   kubectl get secrets --all-namespaces -o json | kubectl replace -f -
   ```

## Disabling encryption at rest

To disable encryption at rest:

1. Place the `identity` provider as the first entry in the configuration file: 

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
       providers:
         - identity: {}
         - kms:
             apiVersion: v2
             name : myKmsPlugin
             endpoint: unix:///tmp/socketfile.sock
   ```

1. Restart all `kube-apiserver` processes. 

1. Run the following command to force all secrets to be decrypted.

   ```shell
   kubectl get secrets --all-namespaces -o json | kubectl replace -f -
   ```
