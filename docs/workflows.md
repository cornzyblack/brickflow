A Workflow is similar to an Airflow dag that lets you encapsulate a set of tasks.

Here is an example of a workflow.
Click the plus buttons to understand all the parts of the workflow file.

```python title="workflow.py"
from datetime import timedelta
from brickflow import (Workflow, Cluster, WorkflowPermissions, User, 
    TaskSettings, EmailNotifications, PypiTaskLibrary, MavenTaskLibrary, JobsParameters)
from brickflow.engine.workflow import WorkflowNotificationSettings

wf = Workflow(  # (1)!
    "wf_test",  # (2)!
    default_cluster=Cluster.from_existing_cluster("your_existing_cluster_id"),  # (3)!

    # Optional parameters below
    schedule_quartz_expression="0 0/20 0 ? * * *",  # (4)!
    timezone="UTC",  # (5)!
    schedule_pause_status="PAUSED",  # (15)!
    default_task_settings=TaskSettings(  # (6)!
        email_notifications=EmailNotifications(
            on_start=["email@nike.com"],
            on_success=["email@nike.com"],
            on_failure=["email@nike.com"],
            on_duration_warning_threshold_exceeded=["email@nike.com"]
        ),
        timeout_seconds=timedelta(hours=2).seconds
    ),
   notification_settings=WorkflowNotificationSettings( # (19)!
      no_alert_for_canceled_runs=True,
      no_alert_for_skipped_runs=True
   ),
    libraries=[  # (7)!
        PypiTaskLibrary(package="requests"),
        MavenTaskLibrary(coordinates="com.cronutils:cron-utils:9.2.0"),
    ],
    tags={  # (8)!
        "product_id": "brickflow_demo",
        "slack_channel": "nike-sole-brickflow-support"
    },
    max_concurrent_runs=1,  # (9)!
    permissions=WorkflowPermissions(  # (10)!
        can_manage_run=[User("abc@abc.com")],
        can_view=[User("abc@abc.com")],
        can_manage=[User("abc@abc.com")],
    ),
    prefix="feature-jira-xxx",  # (11)!
    suffix="_qa1",  # (12)!
    common_task_parameters={  # (13)!
        "catalog": "development",
        "database": "your_database"
    },
    health = { # (16)!
        "metric": "RUN_DURATION_SECONDS",
        "op": "GREATER_THAN",
        "value": 7200
    },
    parameters=[JobsParameters(default="INFO", name="jp_logging_level")],  # (18)!
    queue = True,  # (20)!
)


@wf.task()  # (14)!
def task_function(*, test="var"):
    return "hello world"
```

1. Workflow definition which constructs the workflow object
2. Define the workflow name
3. The default cluster used for all the tasks in the workflow. This is an all-purpose cluster, but you can also create a job cluster
4. Cron expression in the quartz format
5. Define the timezone for your workflow. It is defaulted to UTC
6. Default task setting that can be used for all the tasks
7. Libraries that need to be installed for all the tasks
8. Tags for the resulting workflow and other objects created during the workflow.
9. Define the maximum number of concurrent runs
10. Define the permissions on the workflow
11. Prefix for the name of the workflow
12. Suffix for the name of the workflow
13. Define the common task parameters that can be used in all the tasks
14. Define a workflow task and associate it to the workflow
15. Define the schedule pause status. It is defaulted to "UNPAUSED"
16. Define health check condition that triggers duration warning threshold exceeded notifications
17. Define timeout_seconds check condition that triggers workflow failure if duration exceeds threshold
18. Define the parameters on workflow level [databricks docs](https://docs.databricks.com/en/jobs/settings.html#job-parameters)
19. Define the notification settings for the workflow
20. To prevent runs of a job from being skipped because of concurrency limits, you can enable queueing for the job

### Clusters

There are two ways to define the cluster for the workflow or a task

#### Using an existing cluster

```python title="existing_cluster"
from brickflow import Cluster

default_cluster=Cluster.from_existing_cluster("your_existing_cluster_id")
```

#### Use a job cluster

```python title="job_cluster"
from brickflow import Cluster

default_cluster=Cluster(
    name="your_cluster_name",
    spark_version='11.3.x-scala2.12',
    node_type_id='m6g.xlarge',
    driver_node_type_id='m6g.xlarge',
    min_workers=1,
    max_workers=3,
    enable_elastic_disk=True,
    policy_id='your_policy_id',
    aws_attributes={
        "first_on_demand": 1,
        "availability": "SPOT_WITH_FALLBACK",
        "instance_profile_arn": "arn:aws:iam::XXXX:instance-profile/XXXX/group/XX",
        "spot_bid_price_percent": 100,
        "ebs_volume_type": "GENERAL_PURPOSE_SSD",
        "ebs_volume_count": 3,
        "ebs_volume_size": 100
    }
)
```

#### Use a job cluster with driver and worker instances from the same pool

```python title="job_cluster_driver_workers_same_pool"
from brickflow import Cluster

default_cluster=Cluster(
    name="your_cluster_name",
    spark_version='11.3.x-scala2.12',
    min_workers=1,
    max_workers=3,
    instance_pool_id="your_instance_pool_id",
    policy_id='policy_id_of_pool',
)
```

#### Use a job cluster with driver and worker instances from different pools

```python title="job_cluster_driver_workers_same_pool"
from brickflow import Cluster

default_cluster=Cluster(
    name="your_cluster_name",
    spark_version='11.3.x-scala2.12',
    min_workers=1,
    max_workers=3,
    instance_pool_id="your_workers_instance_pool_id",
    driver_instance_pool_id="your_driver_instance_pool_id",
    policy_id='policy_id_of_pools',
)
```

### Permissions

Brickflow provides an opportunity to manage permissions on the workflows.
You can provide individual users or to a group or to a ServicePrincipal that can help manage, run or
view the workflows.

Below example is for reference

```python title="manage_permissions"
from brickflow import WorkflowPermissions, User, Group, ServicePrincipal

permissions=WorkflowPermissions(
    can_manage_run=[
        User("abc@abc.com"), 
        Group("app.xyz.team.Developer"), 
        ServicePrincipal("ServicePrinciple_dbx_url.app.xyz.team.Developer")
    ],
    can_view=[User("abc@abc.com")],
    can_manage=[User("abc@abc.com")],
)
```

### Tags

Using brickflow, custom tags can be created on the workflow - but there are also some default tags
that are created while the job is deployed.

The defaults tags that gets automatically attached to the workflow are below

* "brickflow_project_name" : Brickflow Project Name that is referred from the entrypoint.py file
* "brickflow_version" : Brickflow Version that is used to deploy the workflow
* "databricks_tf_provider_version" : Databricks terraform provider version that is used to deploy the workflow
* "deployed_by" : Email id of the profile that is used to deploy the workflow.
   It can be a user or a service principle. Whichever id is used to deploy the workflow, automatically becomes the
   owner of the workflow
* "environment" : Environment to which the workflow is identified to

Use the below reference to define more tags and attach to the workflow. These can be used for collecting various
metrics and build dashboards.

```python title="configure_tags"
tags={
        "product_id": "brickflow_demo",
        "slack_channel": "nike-sole-brickflow-support"
    }
```

### Schedule

Databricks workflows uses Quartz cron expression unlike airflow's unix based cron scheduler.
A typical Quartz cron expression have six or seven fields, seperated by spaces

```text
second minute hour day_of_month month day_of_week year(optional)
```

Below is a sample

```python title="quartz_cron_expression"
schedule_quartz_expression="0 0/20 0 ? * * *"
```

### Tasksettings

Task setting at workflow level can be used to have common setting defined that will be applicable for
all the tasks. Below is a sample that can be used for reference and all the parameters in TaskSettings
are optional

```python title="task_settings"
from datetime import timedelta
from brickflow import TaskSettings, EmailNotifications

default_task_settings=TaskSettings(
   email_notifications=EmailNotifications(
      on_start=["email@nike.com"],
      on_success=["email@nike.com"],
      on_failure=["email@nike.com"],
      on_duration_warning_threshold_exceeded=["email@nike.com"]      
   ),
   timeout_seconds=timedelta(hours=2).seconds,
   max_retries=2,
   min_retry_interval_millis=60000,
   retry_on_timeout=True
)
```

### Libraries

Brickflow allows to specify libraries that are need to be installed and used across different tasks.
There are many ways to install library from different repositories/sources

```python title="libraries"
from brickflow import PypiTaskLibrary, MavenTaskLibrary, StorageBasedTaskLibrary, \
    JarTaskLibrary, EggTaskLibrary, WheelTaskLibrary

libraries=[
   PypiTaskLibrary(package="requests"),
   MavenTaskLibrary(coordinates="com.cronutils:cron-utils:9.2.0"),
   StorageBasedTaskLibrary("s3://..."),
   StorageBasedTaskLibrary("dbfs://..."),
   JarTaskLibrary("s3://..."),
   JarTaskLibrary("dbfs://..."),
   EggTaskLibrary("s3://..."),
   EggTaskLibrary("dbfs://..."),
   WheelTaskLibrary("s3://..."),
   WheelTaskLibrary("dbfs://..."),
]
```

### Common task parameters

Define the common parameters that can be used in all the tasks. Example could be database name, secrets_id etc

```python title="common_task_parameters"
common_task_parameters={
        "catalog": "development",
        "database": "your_database"
    }
```

### Jobs Parameters

You can configure parameters on a job that are passed to any of the job’s tasks that accept key-value parameters, including Python wheel files configured to accept keyword arguments. Parameters set at the job level are added to configured task-level parameters. Job parameters passed to tasks are visible in the task configuration, along with any parameters configured on the task.

You can also pass job parameters to tasks that are not configured with key-value parameters such as JAR or Spark Submit tasks. To pass job parameters to these tasks, format arguments as {{job.parameters.[name]}}, replacing [name] with the key that identifies the parameter.

Job parameters take precedence over task parameters. If a job parameter and a task parameter have the same key, the job parameter overrides the task parameter.

You can override configured job parameters or add new job parameters when you run a job with different parameters or repair a job run.

```python title="jobs_parameters"
    parameters=[JobsParameters(default="INFO", name="jp_logging_level")]
```
