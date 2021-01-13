# Airflow KubernetesJobOperator

An airflow operator that executes a task in a kubernetes cluster, given a yaml configuration or an image url.

### If you like it \* it, so other people would also use it.

Contributions are welcome. See [here](https://docs.github.com/en/free-pro-team@latest/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request-from-a-fork).

### Supports

1. Running tasks as Kubernetes Jobs.
1. Running tasks as Kubernetes Pods.
1. Running tasks as multi resource (see example below, similar to `kubectl apply`).
1. Running tasks as custom resources ([help](docs/custom_kinds.md)).
1. Auto detection of kubernetes namespace and config.
1. Pod and participating resources logs -> airflow.
1. Full kubernetes error logs on failure.
1. Integrated operator airflow config, see below.
1. Integrated Jinja2 support for file templates with flag.
1. Tested and working on [google cloud composer](https://cloud.google.com/composer).

### Two operator classes are available

1. KubernetesJobOperator - Supply a kubernetes configuration (yaml file, yaml string or a list of python dictionaries) as the body of the task.
1. KubernetesLegacyJobOperator - Defaults to a kubernetes job definition, and supports the same arguments as the KubernetesPodOperator. i.e. replace with the KubernetesPodOperator for legacy support.

# Install

To install using pip @ https://pypi.org/project/airflow-kubernetes-job-operator,

```shell
pip install airflow_kubernetes_job_operator
```

To install from master branch,

```shell
pip install git+https://github.com/LamaAni/KubernetesJobOperator.git@master
```

To install from a release (tag)

```shell
pip install git+https://github.com/LamaAni/KubernetesJobOperator.git@[tag]
```

# TL;DR

### Example airflow DAG

```python
from airflow import DAG
from airflow_kubernetes_job_operator.kubernetes_job_operator import KubernetesJobOperator
from airflow_kubernetes_job_operator.kubernetes_legacy_job_operator import KubernetesLegacyJobOperator
from airflow.utils.dates import days_ago

default_args = {"owner": "tester", "start_date": days_ago(2), "retries": 0}
dag = DAG("job-tester", default_args=default_args, description="Test base job operator", schedule_interval=None)

job_task = KubernetesJobOperator(
    task_id="from-image",
    dag=dag,
    image="ubuntu",
    command=["bash", "-c", 'echo "all ok"'],
)

body = {"kind": "Pod"}  # The body or a yaml string (must be valid)
job_task_from_body = KubernetesJobOperator(dag=dag, task_id="from-body", body=body)

body_filepath = "./my_yaml_file.yaml" # Can be relative to this file, or abs path.
job_task_from_yaml = KubernetesJobOperator(dag=dag, task_id="from-yaml", body_filepath=body_filepath)

# Legacy compatibility to KubernetesPodOperator
legacy_job_task = KubernetesLegacyJobOperator(
    task_id="legacy-image-job",
    image="ubuntu",
    cmds=["bash", "-c", 'echo "all ok"'],
    dag=dag,
    is_delete_operator_pod=True,
)
```

### Example (multi resource) task yaml

NOTE: that the success/failure of the task is tracked only on the `first` resource, no matter of its kind. Currently native support exists for Pods and Jobs only, though you can always add a [custom resource](docs/custom_kinds.md).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-job # not required. Will be a prefix to task name
  finalizers:
    - foregroundDeletion
spec:
  template:
    metadata:
      labels:
        app: test-task-pod
    spec:
      restartPolicy: Never
      containers:
        - name: job-executor
          image: ubuntu
          command:
            - bash
            - -c
            - |
              #/usr/bin/env bash
              echo "OK"
  backoffLimit: 0
---
apiVersion: v1
kind: Service
metadata:
  name: test-service # not required, will be a prefex to task name.
spec:
  selector:
    app: test-task-pod
  ports:
    - port: 8080
      targetPort: 8080
```

# Configuration

Airflow config extra sections,

```ini
[kubernetes_job_operator]
# The task kube resources delete policy. Can be: Never, Always, IfFailed, IfSucceeded
delete_policy=IfSucceeded
# The default object type to execute with (legacy, or image). Can be: Pod, Job
default_execution_object=Job

# Logs
detect_kubernetes_log_level=True
show_kubernetes_timestamps=False
# Shows the runner id in the log (for all runner logs.)
show_runner_id=False

# Tasks (Defaults)
# Wait to first connect to kubernetes.
startup_timeout_seconds=120
# if true, will parse the body when building the dag. Otherwise only while executing.
validate_body_on_init=False

# Comma seperated list of where to look for the kube config file. Will be added to the top
# of the search list, in order.
kube_config_extra_locations=

```

To set these values through the environment follow airflow standards,

```
export AIRFLOW__[section]__[item] = value
```

# Why would this be better than the [KubernetesPodOperator](https://github.com/apache/airflow/blob/master/airflow/contrib/operators/kubernetes_pod_operator.p)?

The KubernetesJobOperator allows for more execution options thank the KubernetesPodOperator such as multiple resource execution, custom resource executions, creation event and pod log tracking, proper resource deletion after task is complete, on error and when task is cancelled and more.

Further, in the KubernetesPodOperator the monitoring between the worker pod and airflow is done by an internal loop which executes and consumes worker resources. In this operator, an async threaded approach was taken which reduces resource consumption on the worker. Further, since the logging and parsing of data is done outside of the main worker thread the worker is "free" do handle other tasks without interruption.

Finally, using the KubernetesJobOperator you are free to use other resources like the kubernetes [Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) to execute your tasks. These are better designed for use in kubernetes and are "up with the times".

# Kubernetes job operator input variables

argument| default value | description
|---|---|---|
command | None | A list of commands to be sent. i.e. ["echo","ok"]
arguments | None | A list of arguments to be sent to the docker image.
image | None | An image to use. (Overrides the main pod image)
namespace | None | A namespace to use (Overrides/adds a namespace to main resource)
envs | None | A dictionary of envs to add to the main job pod.
body | None | The body of the job to use. Can be string, dictionary.
body_filepath | None | A filepath to the yaml config file. Can use a relative filepath.
image_pull_policy | Always | The kubernetes image pull policy (str)
delete_policy | IfSucceeded | The resources delete policy. Can be one of: Always, Never, IfSucceeded, IfFailed (Str)
in_cluster | None | Use in cluster creds.
config_file | None | Use this kubernetes config file.
get_logs | True | Retrive the executing pod logs (for all resources)
cluster_context | The cluster context name to use.
startup_timeout_seconds | 10 | The max number of seconds to create the job before timeout is called
validate_body_on_init | False | Can be set to true only if jinja is disabled. Process the yaml when the object is created.
enable_jinja| True | Enable jinja on the body (str, or file), and the following args: command, arguments, image, envs, body, namespace, config_file, cluster_context

# Contribution

Are welcome, please post issues or PR's if needed.

# Implementations still missing:

Add an issue (or better submit PR) if you need these.

1. XCom
1. Examples (other than TL;DR)

# Licence

Copyright ©
`Zav Shotan` and other [contributors](../../graphs/contributors).
It is free software, released under the MIT licence, and may be redistributed under the terms specified in [LICENSE](docs/LICENSE).
