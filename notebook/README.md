# Spark Connect in the `spark-notebook` namespace (OpenShift AI 3.5)

YAML and run instructions adapted from Red Hat OpenShift AI Self-Managed 3.5,
*Creating distributed data processing applications with the Kubeflow Spark
Operator*, Chapter 4 ("Run Spark workloads from OpenShift AI workbenches"),
configured for the non-default namespace **`spark-notebook`**.

> **Technology Preview:** Spark Connect on OpenShift AI is a Technology Preview
> feature. It is not supported with production SLAs and is not recommended for
> production use.

## Files

| File | Purpose |
|---|---|
| `01-namespace.yaml`      | `spark-notebook` namespace, labeled `opendatahub.io/dashboard: "true"` (required for the dashboard and the NetworkPolicy) |
| `02-rbac.yaml`           | `ServiceAccount` `spark-operator-spark`, `Role` `spark-role`, `RoleBinding` `spark-role-binding` — must be created manually because the Operator only auto-creates them in `redhat-ods-applications` |
| `03-networkpolicy.yaml`  | `spark-operator-allow-internal` — allows Spark internal ports (7078, 7079, 4040) and workbench gRPC access on 15002 |
| `04-sparkconnect.yaml`   | `SparkConnect` custom resource (server + 2 executors, Spark 4.0.1) |

The Operator auto-creates the server pod and the ClusterIP `Service`; you never
create those by hand.

## Prerequisites

- Red Hat OpenShift AI 3.5 or later installed.
- Cluster administrator privileges and the `oc` CLI.
- The `DataScienceCluster` has both the `sparkoperator` and `workbenches`
  components set to `Managed`. Check with:

  ```shell
  oc get datasciencecluster
  oc get datasciencecluster <dsc-name> \
    -o jsonpath='{.spec.components.sparkoperator.managementState}{"\n"}{.spec.components.workbenches.managementState}{"\n"}'
  ```

  If `sparkoperator` is not `Managed`, enable it (this also enables the
  `SparkConnect` CRD):

  ```shell
  oc patch datasciencecluster <dsc-name> --type merge \
    -p '{"spec":{"components":{"sparkoperator":{"managementState":"Managed"}}}}'
  ```

- Recommended (per the doc): deploy and verify a SparkConnect server in the
  default `redhat-ods-applications` namespace first, to confirm the Operator
  works, before replicating in a non-default namespace.

## Step 1 — Create the namespace

```shell
oc apply -f 01-namespace.yaml
```

The `opendatahub.io/dashboard: "true"` label makes `spark-notebook` show up in
the OpenShift AI dashboard as a Data Science Project and allows workbench
traffic through the NetworkPolicy.

## Step 2 — Create the RBAC resources

```shell
oc apply -f 02-rbac.yaml
```

## Step 3 — Create the NetworkPolicy

```shell
oc apply -f 03-networkpolicy.yaml
```

## Step 4 — Deploy the Spark Connect server

```shell
oc apply -f 04-sparkconnect.yaml
```

(Shortcut: `oc apply -f .` from this directory applies everything at once.)

## Step 5 — Verify the deployment

```shell
# Server pod is Running
oc get pods -n spark-notebook -l sparkoperator.k8s.io/launched-by-spark-operator=true
# NAME                   READY   STATUS    RESTARTS   AGE
# spark-connect-server   1/1     Running   0          30s

# Auto-created ClusterIP Service with ports 7078, 7079, 4040, 15002
oc get svc -n spark-notebook -l sparkoperator.k8s.io/connect-name=spark-connect
# NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                               AGE
# spark-connect-server   ClusterIP   172.30.x.x   <none>        7078/TCP,7079/TCP,4040/TCP,15002/TCP  30s

# RBAC and NetworkPolicy in place
oc get serviceaccount spark-operator-spark -n spark-notebook
oc get role spark-role -n spark-notebook
oc get rolebinding spark-role-binding -n spark-notebook
oc get networkpolicy spark-operator-allow-internal -n spark-notebook
```

**Connection URL for all notebooks** (Operator convention:
`<connect_name>-server.<namespace>.svc.cluster.local`):

```
sc://spark-connect-server.spark-notebook.svc.cluster.local:15002
```

## Step 6 — Create a workbench

1. In the OpenShift AI dashboard, go to **Data Science Projects** and open the
   **`spark-notebook`** project (it appears there thanks to the namespace
   label). Alternatively, you can run the workbench in any dashboard-managed
   namespace — the NetworkPolicy's second rule allows workbench pods labeled
   `opendatahub.io/workbenches: "true"` from any `opendatahub.io/dashboard:
   "true"` namespace.
2. Create a workbench with a minimal CPU image, e.g.
   **Jupyter | Minimal | CPU | Python 3.12**. Spark Connect workloads do not
   require GPU images.
3. Workbench pods automatically carry the `opendatahub.io/workbenches: "true"`
   label required by the NetworkPolicy.

## Step 7 — Run PySpark from the notebook

1. In a notebook cell, install the client packages (PySpark **must** match the
   server image's Spark version, 4.0.1):

   ```
   !pip install -q pyspark==4.0.1 pandas pyarrow grpcio grpcio-status zstandard
   ```

   **Restart the notebook kernel** after installation so the new packages load.

2. Connect and run a test workload:

   ```python
   from pyspark.sql import SparkSession

   spark = SparkSession.builder.remote(
       "sc://spark-connect-server.spark-notebook.svc.cluster.local:15002"
   ).getOrCreate()

   data = [("Alice", 34), ("Bob", 45), ("Claire", 66)]*1000
   df = spark.createDataFrame(data, ["Name", "Age"])
   df.filter(df.Age > 40).show()
   ```

   Expected output:

   ```
   +------+---+
   |  Name|Age|
   +------+---+
   |   Bob| 45|
   |Claire| 66|
   +------+---+
   ```

## Troubleshooting quick reference

| Symptom | Check |
|---|---|
| Server pod not `Running` | `oc describe pod spark-connect-server -n spark-notebook` and `oc logs spark-connect-server -n spark-notebook` |
| No Service / can't resolve DNS | `oc get svc -n spark-notebook -l sparkoperator.k8s.io/connect-name=spark-connect`; the Operator creates the Service during reconciliation. From the workbench pod: `nslookup spark-connect-server.spark-notebook.svc.cluster.local` |
| gRPC connection refused / times out | NetworkPolicy blocking 15002. The workbench pod needs label `opendatahub.io/workbenches: "true"` and its namespace needs `opendatahub.io/dashboard: "true"`: `oc get pod <pod> -n <ns> --show-labels` / `oc get namespace <ns> --show-labels` |
| Connection fails in notebook but works via `oc rsh` | Version mismatch or stale packages — restart the notebook kernel; confirm `pyspark==4.0.1` matches `spec.sparkVersion` |
| Executors fail to start | Server log usually shows RBAC errors; confirm `02-rbac.yaml` was applied in `spark-notebook` (the Operator does not create it there automatically) |

Quick connectivity test from inside the workbench pod:

```shell
oc rsh -n <workbench_namespace> <workbench_pod>
python -c "from pyspark.sql import SparkSession; spark = SparkSession.builder.remote('sc://spark-connect-server.spark-notebook.svc.cluster.local:15002').getOrCreate(); print(spark.version)"
```

## Resource footprint and tuning

Default `04-sparkconnect.yaml` footprint: 1 driver (1 CPU / 1 Gi) + 2 executors
(1 CPU / 512 m each). Adjust `spec.executor.instances`, `cores`, and `memory`,
or add a `spec.dynamicAllocation` block for elastic executors. Optional
`spec.sparkConf` / `spec.hadoopConf` maps correspond to `--conf` /
Hadoop-style settings (the Operator prefixes Hadoop keys with `spark.hadoop.`).

## Teardown

```shell
oc delete sparkconnect spark-connect -n spark-notebook   # removes pod + Service
oc delete networkpolicy spark-operator-allow-internal -n spark-notebook
oc delete rolebinding spark-role-binding -n spark-notebook
oc delete role spark-role -n spark-notebook
oc delete serviceaccount spark-operator-spark -n spark-notebook
oc delete namespace spark-notebook                        # optional
```
