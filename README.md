# spark-kueue-workbenches

This repository is a demonstration of some of the functionality of the Kubeflow Spark Operator present in OpenShift AI 3.5. It demonstrates 4 things:
1) A simple Spark application showing that it works in RHOAI.
2) A job that takes too many resources for the queue and therefore won't ever be scheduled.
3) Two jobs that together take more resources than the queue allows. Therefore it will run them one at a time. In particular, the entire job must be submitted - it won't partial submit across the executors.
4) Standing up a Spark cluster that can be reached from workbenches. The workbench acts as the client and all of the work takes place on the cluster.

For the most part, you can roll through this repository in the order of the folders. There are a couple of specific exceptions:

* The gang scheduling configuration for Kueue needs to be run as a server side apply
* * `oc apply -f 00-kueue-config/01-kueue-gang-scheduling.yaml --server-side`
* Spark Connect has another README in its folder on how to connect a workbench to the SparkConnect cluster.

## Kueue Borrowing

Folder `05-kueue-borrowing` demonstrates how resources can be borrowed from the shared pool. Run both of them at the same time and watch one of the jobs in `spark-notebook` get preempted for the job in `spark-teamb`.

There is something that bears discussing here about the resources. ClusterQueue nominalQuota is guaranteed. Cohort nominalQuota is *always* shared. So if you have a Cohort quota of 4 and a ClusterQueue quota of 3, technically that ClusterQueue can consume 7 CPU at a time assuming nothing else is pulling resources from the Cohort. This is different from how other resource schedulers operate and put some additional constraints on how you think about configuring your queues. 