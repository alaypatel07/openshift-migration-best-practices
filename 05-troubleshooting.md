---
title: Troubleshooting
layout: default
---

# Troubleshooting

This section describes common troubleshooting procedures.

Upstream doc for improving debug experience

- https://github.com/konveyor/enhancements/tree/master/enhancements/debug

Debug flowchart (in progress)

- [MTC Debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0)

## MTC custom resources

The following diagram describes the MTC custom resources (CRs). Each object is a standard Kubernetes CR.

You can manage the MTC resources with the standard create, read, update, and delete operations using the `kubectl` and `oc` clients or directly, using the web interface.

TODO: Need to update diagram with MigAnalytic and MigHook

![CRD Architecture](./images/CRDArch.png)

## Debugging MTC resources using the web console

You can view the resources of a migration plan in the MTC web console:

1. Click the Options menu beside a migration plan and select **View migration plan resources**.

   The migration plan resources are displayed as a tree.

2. Click the arrow of a **Backup** or **Restore** object to view its pods.

3. Click the Copy button of a pod to copy the `oc get` command to your clipboard.

   You can paste the command to the CLI to view the resource details.

4. Click **View Raw** to inspect a pod.

   The resource is displayed in JSON format.

![Resource Debug Kebab Option](./images/ResourceDebugKebabOption.png)

![Debug Tree UI](./images/DebugTree.png)

Typically, the objects that you are interested in depend on the stage at which the
migration failed. The [MTC debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0) provides information about what objects are relevant depending on this failure stage.

Stage migrations have one Backup and one Restore object.

Final migrations have two Backup and two Restore objects. The first Backup object captures the original, unaltered state of the application and its Kubernetes objects. This Backup is the source of truth. Then, the application is quiesced and a second Backup captures the storage-related resources (PVs, PVCs, data).

The first Restore restores these storage objects on the target cluster. The final Restore restores the original application Backup to the target cluster.

## Debugging MTC resources from the command line

The migration debug tree can be viewed from the command line by querying specific label selectors.

- To view all `migmigration` objects associated with the `test` plan:

  ```sh
  $ oc get migmigration -l 'migration.openshift.io/migplan-name=test'
  ```

  **Example output**

  ```
  NAME                                  READY  PLAN  STAGE  ITINERARY  PHASE
  09a8bf20-fdc5-11ea-a447-cb5249018d21         test  false  Final      Completed
  ```

  The columns display the associated plan name, itinerary step, and phase.

- To view `backup` objects:

  ```sh
  $ oc get backup -n openshift-migration
  ```

  **Example output**

  ```
  NAME                                   AGE
  88435fe0-c9f8-11e9-85e6-5d593ce65e10   6m42s
  ```

  Use the same command to view `restore` objects.

- To inspect a `backup` object:

  ```sh
  $ oc describe backup 88435fe0-c9f8-11e9-85e6-5d593ce65e10 -n openshift-migration
  ```

See [Viewing migration custom resources](https://docs.openshift.com/container-platform/4.6/migration/migrating_3_4/troubleshooting-3-4.html#migration-viewing-migration-crs_migrating-3-4) for more information.

## Accessing more information about Velero Resources using the 'velero' CLI tool

In addition to viewing the Backup and Restore resources, it is possible to obtain more information by invoking the `velero` binary.  This utility will look at a lower level of information stored in the object storage associated with each backup or restore.  This may help to show why a particular resource was not restored or give more context as to why a Velero operation failed.

MTC ships the `velero` binary in the running velero container, a user may access it using:

```sh
  $ oc exec velero-$podname -n openshift-migration -- ./velero --help

  or

  $ oc exec $(oc get pods -n openshift-migration -o name | grep velero) -n openshift-migration -- ./velero --help
```

- `velero {backup|restore} describe $resourceid`
  - `describe`: will provide a summary of warnings and errors Velero saw while processing the action
    - example: `velero backup describe 0e44ae00-5dc3-11eb-9ca8-df7e5254778b-2d8ql`
      - Where '0e44ae00-5dc3-11eb-9ca8-df7e5254778b-2d8ql' is the name of a Velero Backup custom resource

- `velero {backup|restore} logs $resourceid`
  - `logs`: will provide a lower level output of the logs associated to this specific action
    - example: `velero restore logs ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf`
      - Where 'ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf' is the name of a Velero Restore custom resource


### Debugging a 'PartialFailure'

Migrations may display a warning of 'PartialFailure'.  A PartialFailure is when Velero ran into an issue which was unexpected but is not failing the operation.  For example a custom resource was not able to be restored because the Custom Resource Definition is missing or has a wrong version.  Velero will record that the specific Custom Resource could not be restored and process the rest of the items from the Backup.  

When a migration displays a PartialFailure the administrator should use the `velero restore logs` command to see the specifics and determine if manual intervention is needed on a specific resource or if the warning is safe to ignore.

- Note there is a task on the roadmap for improving the experience for debugging Partial Failures:  [MIG-353 Enhance Velero error reporting so problems that cause partial failures (and even full failures) are more visible in structured way](https://issues.redhat.com/browse/MIG-353)


Below are a few examples debugging a failed restore due to a Group Version Kind mismatch.  The failure scenario is available at https://github.com/pranavgaikwad/mtc-breakfix/tree/master/03-Gvk.

- Example looking at the associated 'MigMigration' resource
[full output](https://gist.github.com/jwmatthews/001ff42bf5e712ba2eab92df306ed34e)

```sh
oc get migmigration ccc7c2d0-6017-11eb-afab-85d0007f5a19 -o yaml

status:
  conditions:
  - category: Warn
    durable: true
    lastTransitionTime: "2021-01-26T20:48:40Z"
    message: 'Final Restore openshift-migration/ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf: partially failed on destination cluster'
    status: "True"
    type: VeleroFinalRestorePartiallyFailed
  - category: Advisory
    durable: true
    lastTransitionTime: "2021-01-26T20:48:42Z"
    message: The migration has completed with warnings, please look at `Warn` conditions.
    reason: Completed
    status: "True"
    type: SucceededWithWarnings
```

- Example using `velero restore describe`
[full output](https://gist.github.com/9a3ec8f51e12b84f8bb995286223bdda)
```sh

$ oc exec $(oc get pods -n openshift-migration -o name | grep velero) -n openshift-migration -- ./velero restore describe ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf

.....

Phase:  PartiallyFailed (run 'velero restore logs ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf' for more information)

Errors:
  Velero:     <none>
  Cluster:    <none>
  Namespaces:
    gvk-demo:  error restoring gvkdemoes.konveyor.openshift.io/gvk-demo/gvk-demo: the server could not find the requested resource

...
```

- Example using `velero restore logs`

[full output](https://gist.github.com/jwmatthews/7dc7ed9eb0c4d0611f30675074b9b7d7)
```sh

$ oc exec $(oc get pods -n openshift-migration -o name | grep velero) -n openshift-migration -- ./velero restore logs ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf

...

time="2021-01-26T20:48:37Z" level=info msg="Attempting to restore GvkDemo: gvk-demo" logSource="pkg/restore/restore.go:1107" restore=openshift-migration/ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf
time="2021-01-26T20:48:37Z" level=info msg="error restoring gvk-demo: the server could not find the requested resource" logSource="pkg/restore/restore.go:1170" restore=openshift-migration/ccc7c2d0-6017-11eb-afab-85d0007f5a19-x4lbf

...
```

## Error messages

### CA certificate error when logging in to the MTC console for the first time

The following error message might appear when you log in to the MTC console for the first time:

```
A certificate error has occurred, likely caused by using self-signed CA certificates in one of the clusters. Navigate to the following URL and accept the certificate:
`https://ocp-cluster.com:6443/.well-known/oauth-authorization-server`.

If an "Unauthorized" message appears after you have accepted the certificate, refresh the web page.

To fix this issue permanently, add the certificate to your web browser's trust store.
```

Possible causes are self-signed certificates or network access issues.

Self-signed CA certificates:

- You can navigate to the `oauth-authorization-server` URL and accept the certificate.
- You can add self-signed certificates for the API server, OAuth server, and routes to your web browser's trusted store.

Network access:

- You can inspect the elements of the MTC console with your browser's web inspector to view the network connections.
- MTC 1.3.1 and earlier: The MTC console performs OAuth authentication on the client side.

  The console requires uninterrupted network access to the API server and the OAuth server.

- MTC 1.3.2 and later: OAuth authentication is performed on the backend.

  The console requires uninterrupted network access to the Node.js server, which provides the JavaScript bundle and performs OAuth authentication, and the API server. See [BZ#1878824](https://bugzilla.redhat.com/show_bug.cgi?id=1878824).

### Connection time-out after accepting CA certificate

If you acce[t] a self-signed certificate and a blank page appears, followed by a `Connection has timed out` message, the likely cause is a web proxy blocking access to the OAuth server.

Configure the web proxy configuration to allow access to the `oauth-authorization-server` URL. See [BZ#1890675](https://bugzilla.redhat.com/show_bug.cgi?id=1890675).

## Using `must-gather`

You can use the `must-gather` tool to collect information for troubleshooting or for opening a customer support case on the [Red Hat Customer Portal](https://access.redhat.com/). The `openshift-migration-must-gather-rhel8` image collects migration-specific logs and Custom Resource data that are not collected by the default `must-gather` image.

Run the `must-gather` command on your cluster:
```sh
$ oc adm must-gather --image=openshift-migration-must-gather-rhel8:v1.3.0
```

The `must-gather` tool generates a local directory that contains the collected data.

## Direct volume migration fails to complete

If direct volume migration fails to complete, the most likely cause is that the Rsync transfer pods on the target cluster remain in a `Pending` state.

MTC migrates namespaces with all annotations in order to preserve security context constraints and scheduling requirements. During direct volume migration, MTC creates Rsync transfer pods on the target cluster in the namespaces that were migrated from the source cluster. If the target cluster does not have the same node labels as the source cluster, the Rsync transfer pods cannot be scheduled.

You can check the `migmigration` CR status:
```sh
$ oc describe migmigration 88435fe0-c9f8-11e9-85e6-5d593ce65e10 -n openshift-migration
```

The output displays the following `status` message:
```
Some or all transfer pods are not running for more than 10 mins on destination cluster
```

To resolve this issue, perform the following steps:

1. Obtain the value of the `openshift.io/node-selector` annotation of the migrated namespaces on the source cluster:
```sh
$ oc get namespace -o yaml
```
2. Add the `openshift.io/node-selector` annotation to each migrated namespace on the target cluster:
```yml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/node-selector: "region=east"
...
```
3. Re-run the migration plan.

## Previewing metrics on local Prometheus server

You can use `must-gather` to create a metrics data directory dump from the last day:

```sh
$ oc adm must-gather --image quay.io/konveyor/must-gather:latest -- /usr/bin/gather_metrics_dump
```

You can view the data with a [local Prometheus instance](https://github.com/konveyor/must-gather#preview-metrics-on-local-prometheus-server).

## Performance metrics

For information about the metrics recorded by the MTC controller, see the [`mig-operator` documentation](https://github.com/konveyor/mig-operator/blob/master/docs/usage/Metrics.md#accessing-mig-controller-prometheus-metrics).

This documentation includes [useful queries](https://github.com/konveyor/mig-operator/blob/master/docs/usage/Metrics.md#useful-queries) for performance monitoring.

## Cleaning up a failed migration

### Deleting resources

Ensure that stage pods are cleaned up. If a migration fails during stage or copy, the stage pods are retained to allow debugging. Before retrying a migration, you must delete the stage pods manually.

### Unquiescing an application

If your application was quiesced during migration, you should `unquiesce` it by scaling it back to its initial replica count.

This can be done manually by editing the deployment primitive (`Deployment`, `DeploymentConfig`, etc.) and setting the `spec.replicas` field back to its original, non-zero value:

```sh
$ oc edit deployment <deployment_name>
```

Alternatively, you can scale your deployment with the `oc scale` command:

```sh
$ oc scale deployment <deployment_name> --replicas=<desired_replicas>
```

### Labels for premigration settings

When a source application is quiesced during migration, MTC adds a label indicating the original replica count to the `deployment` resource:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    migration.openshift.io/preQuiesceReplicas: "1"
```

## Deleting the MTC Operator and resources

The following procedure removes the MTC Operator and cluster-scoped resources:

1. Delete the Migration Controller and its resources:

   ```sh
   $ oc delete migrationcontroller <resource_name>
   ```

   Wait for the MTC Operator to finish deleting the resources.

2. Uninstall the MTC Operator:

   - OpenShift 4: Uninstall the Operator in the [web console](https://docs.openshift.com/container-platform/4.6/operators/olm-deleting-operators-from-cluster.html) or by running the following command:

   ```sh
   $ oc delete ns openshift-migration
   ```

   - OpenShift 3: Uninstall the operator by deleting it:
     ```sh
     $ oc delete -f operator.yml
     ```

3. Delete the cluster-scoped resources:
   - Migration custom resource definition:
     ```sh
     $ oc delete $(oc get crds -o name | grep 'migration.openshift.io')
     ```
   - Velero custom resource definition:
     ```sh
     $ oc delete $(oc get crds -o name | grep 'velero')
     ```
   - Migration cluster role:
     ```sh
     $ oc delete $(oc get clusterroles -o name | grep 'migration.openshift.io')
     ```
   - Migration-operator cluster role:
     ```sh
     $ oc delete clusterrole migration-operator
     ```
   - Velero cluster role:
     ```sh
     $ oc delete $(oc get clusterroles -o name | grep 'velero')
     ```
   - Migration cluster role bindings:
     ```sh
     $ oc delete $(oc get clusterrolebindings -o name | grep 'migration.openshift.io')
     ```
   - Migration-operator cluster role bindings:
     ```sh
     $ oc delete clusterrolebindings migration-operator
     ```   
   - Velero cluster role bindings:
     ```sh
     $ oc delete $(oc get clusterrolebindings -o name | grep 'velero')
     ```
