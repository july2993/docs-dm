---
title: Replicate Data Using Data Migration
summary: Use the Data Migration tool to replicate the full data and the incremental data.
aliases: ['/docs/tidb-data-migration/dev/replicate-data-using-dm/']
---

# Replicate Data Using Data Migration

This guide shows how to replicate data using the Data Migration (DM) tool.

## Step 1: Deploy the DM cluster

It is recommended to deploy the DM cluster using DM-Ansible. For detailed deployment, see [Deploy Data Migration Using DM-Ansible](deploy-a-dm-cluster-using-ansible.md).

You can also deploy the DM cluster using binary for trial or test. For detailed deployment, see [Deploy Data Migration Cluster Using DM Binary](deploy-a-dm-cluster-using-binary.md).

> **Note:**
>
> - For database passwords in all the DM configuration files, use the passwords encrypted by `dmctl`. If a database password is empty, it is unnecessary to encrypt it. See [Encrypt the upstream MySQL user password using dmctl](deploy-a-dm-cluster-using-ansible.md#encrypt-the-upstream-mysql-user-password-using-dmctl).
> - The user of the upstream and downstream databases must have the corresponding read and write privileges.

## Step 2: Check the cluster information

After the DM cluster is deployed using DM-Ansible, the configuration information is like what is listed below.

- The configuration information of related components in the DM cluster:

    | Component | Host | Port |
    |------| ---- | ---- |
    | dm_worker1 | 172.16.10.72 | 8262 |
    | dm_worker2 | 172.16.10.73 | 8262 |
    | dm_master | 172.16.10.71 | 8261 |

- The information of upstream and downstream database instances:

    | Database instance | Host | Port | Username | Encrypted password |
    | -------- | --- | --- | --- | --- |
    | Upstream MySQL-1 | 172.16.10.81 | 3306 | root | VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU= |
    | Upstream MySQL-2 | 172.16.10.82 | 3306 | root | VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU= |
    | Downstream TiDB | 172.16.10.83 | 4000 | root | |

- The configuration in the DM-master process configuration file `{ansible deploy}/conf/dm-master.toml`:

    ```toml
    # Master configuration.

    [[deploy]]
    source-id = "mysql-replica-01"
    dm-worker = "172.16.10.72:8262"

    [[deploy]]
    source-id = "mysql-replica-02"
    dm-worker = "172.16.10.73:8262"
    ```

    > **Note:**
    >
    > The `{ansible deploy}` in `{ansible deploy}/conf/dm-master.toml` indicates the directory where DM-Ansible is deployed. It is the directory configured in the `deploy_dir` parameter.

## Step 3: Create data source

1. Write MySQL-1 related information to `conf/source1.yaml`:

    ```yaml
    # MySQL1 Configuration.
    
    source-id: "mysql-replica-01"
    # Indicates whether GTID is enabled.
    enable-gtid: true
    
    from:
      host: "172.16.10.81"
      user: "root"
      password: "VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU="
      port: 3306
    ```

2. Execute the following command in the terminal, and use `dmctl` to load the MySQL-1 data source configuration to the DM cluster:

    {{< copyable "shell-regular" >}}

    ```bash
    ./bin/dmctl --master-addr=127.0.0.1:8261 operate-source create conf/source1.yaml
    ```

3. For MySQL-2, modify the relevant information in the configuration file and execute the same `dmctl` command.

## Step 4: Configure the data migration task

The following example assumes that you need to replicate all the `test_table` table data in the `test_db` database of both the upstream MySQL-1 and MySQL-2 instances, to the downstream `test_table` table in the `test_db` database of TiDB, in the full data plus incremental data mode.

Copy the `{ansible deploy}/conf/task.yaml.example` file and edit it to generate the `task.yaml` task configuration file as below:

```yaml
# The task name. You need to use a different name for each of the multiple tasks that
# run simultaneously.
name: "test"
# The full data plus incremental data (all) replication mode.
task-mode: "all"
# The downstream TiDB configuration information.
target-database:
  host: "172.16.10.83"
  port: 4000
  user: "root"
  password: ""

# Configuration of all the upstream MySQL instances required by the current data migration task.
mysql-instances:
-
  # The ID of upstream instances or the migration group. You can refer to the configuration of `source_id` in the "inventory.ini" file or in the "dm-master.toml" file.
  source-id: "mysql-replica-01"
  # The configuration item name of the block and allow lists of the name of the
  # database/table to be replicated, used to quote the global block and allow
  # lists configuration that is set in the global block-allow-list below.
  block-allow-list: "global"  # Use black-white-list if the DM's version <= v2.0.0-beta.2.
  # The configuration item name of Mydumper, used to quote the global Mydumper configuration.
  mydumper-config-name: "global"

-
  source-id: "mysql-replica-02"
  block-allow-list: "global"  # Use black-white-list if the DM's version <= v2.0.0-beta.2.
  mydumper-config-name: "global"

# The global configuration of block and allow lists. Each instance can quote it by the
# configuration item name.
block-allow-list:                     # Use black-white-list if the DM's version <= v2.0.0-beta.2.
  global:
    do-tables:                        # The allow list of upstream tables to be replicated.
    - db-name: "test_db"              # The database name of the table to be replicated.
      tbl-name: "test_table"          # The name of the table to be replicated.

# Mydumper global configuration. Each instance can quote it by the configuration item name.
mydumpers:
  global:
    extra-args: "-B test_db -T test_table"  # The extra Mydumper argument. Since DM 1.0.2, DM automatically generates the "--tables-list" configuration. For versions earlier than 1.0.2, you need to configure this option manually.
```

## Step 5: Start the data migration task

To detect possible errors of data migration configuration in advance, DM provides the precheck feature:

- DM automatically checks the corresponding privileges and configuration while starting the data migration task.
- You can also use the `check-task` command to manually precheck whether the upstream MySQL instance configuration satisfies the DM requirements.

For details about the precheck feature, see [Precheck the upstream MySQL instance configuration](precheck.md).

> **Note:**
>
> Before starting the data migration task for the first time, you should have got the upstream configured. Otherwise, an error is reported while you start the task.

1. Come to the dmctl directory `/home/tidb/dm-ansible/resources/bin/`.

2. Run the following command to start dmctl.

    ```bash
    ./dmctl --master-addr 172.16.10.71:8261
    ```

3. Run the following command to start the data migration tasks.

    ```bash
    # `task.yaml` is the configuration file that is edited above.
    start-task ./task.yaml
    ```

    - If the above command returns the following result, it indicates the task is successfully started.

        ```json
        {
            "result": true,
            "msg": "",
            "workers": [
                {
                    "result": true,
                    "worker": "172.16.10.72:8262",
                    "msg": ""
                },
                {
                    "result": true,
                    "worker": "172.16.10.73:8262",
                    "msg": ""
                }
            ]
        }
        ```

    - If you fail to start the data migration task, modify the configuration according to the returned prompt and then run the `start-task task.yaml` command to restart the task.

## Step 5: Check the data migration task

If you need to check the task state or whether a certain data migration task is running in the DM cluster, run the following command in dmctl:

```bash
query-status
```

## Step 7: Stop the data migration task

If you do not need to migrate data any more, run the following command in dmctl to stop the task:

```bash
# `test` is the task name that you set in the `name` configuration item of
# the `task.yaml` configuration file.
stop-task test
```

## Step 8: Monitor the task and check logs

Assuming that Prometheus, Alertmanager, and Grafana are successfully deployed along with the DM cluster deployment using DM-Ansible, and the Grafana address is `172.16.10.71`. To view the alert information related to DM, you can open <http://172.16.10.71:9093> in a browser and enter into Alertmanager; to check monitoring metrics, go to <http://172.16.10.71:3000>, and choose the DM dashboard.

While the DM cluster is running, DM-master, DM-worker, and dmctl output the monitoring metrics information through logs. The log directory of each component is as follows:

- DM-master log directory: It is specified by the `--log-file` DM-master process parameter. If DM is deployed using DM-Ansible, the log directory is `{ansible deploy}/log/dm-master.log` in the DM-master node.
- DM-worker log directory: It is specified by the `--log-file` DM-worker process parameter. If DM is deployed using DM-Ansible, the log directory is `{ansible deploy}/log/dm-worker.log` in the DM-worker node.
- dmctl log directory: It is the same as the binary directory of dmctl.
