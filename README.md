# HAI Platform

![Header](https://img.shields.io/badge/-hai--platform-orange)
[![License](https://img.shields.io/badge/license-LGPL-4EB1BB)](https://www.gnu.org/licenses/lgpl-3.0.en.html)

HAI Time-Sharing Scheduling Training Platform, deployable via `docker-compose` or `k8s`, provides:
- Time-sharing scheduling for training tasks
- Training task management
- Jupyter development container management
- [Studio UI](https://github.com/HFAiLab/hai-platform-studio)
- Haienv runtime environment management

## External Dependencies
1. A centralized storage system (e.g., `nfs`, `ceph`, `weka`)
   - Stores user code
   - Stores code execution logs
   - Stores required k8s configurations
2. Kubernetes cluster with all compute nodes joined
3. Recommended: Compute nodes with RDMA support and [rdma-sriov device-plugin](https://github.com/mellanox/k8s-rdma-shared-dev-plugin) installed
   - If unavailable, configure `HAS_RDMA_HCA_RESOURCE: '0'` in `launcher.manager_envs`

## Quick Start
1. Build

    Build all-in-one hai-platform image:

    Note: To include haienv 202207 runtime (with CUDA, Torch) as training image, set `export BUILD_TRAIN_IMAGE=1`. For custom training images, see [Appendix: Database Initialization](#database-initialization) for `train_environment` configuration.
    ```shell
    # Replace IMAGE_REPO with your own repo
    $ IMAGE_REPO=registry.cn-hangzhou.aliyuncs.com/hfai/hai-platform bash one/release.sh
      Build success:
        hai-platform image: registry.cn-hangzhou.aliyuncs.com/hfai/hai-platform:fa07f13
        hai-cli wheels:
          /home/hai-platform/build/hai-1.0.0+fa07f13-py3-none-any.whl
          /home/hai-platform/build/haienv-1.4.1+fa07f13-py3-none-any.whl
          /home/hai-platform/build/haiworkspace-1.0.0+fa07f13-py3-none-any.whl
    ```

    Install hai-cli:
    ```shell
    pip3 install /home/hai-platform/build/hai-1.0.0+fa07f13-py3-none-any.whl
    pip3 install /home/hai-platform/build/haienv-1.4.1+fa07f13-py3-none-any.whl
    pip3 install /home/hai-platform/build/haiworkspace-1.0.0+fa07f13-py3-none-any.whl
    ```

    Prebuilt images and CLI:
    ```shell
    # Base image
    registry.cn-hangzhou.aliyuncs.com/hfai/hai-platform:latest
    # With haienv 202207 runtime
    registry.cn-hangzhou.aliyuncs.com/hfai/hai-platform:latest-202207

    pip3 install hai --extra-index-url https://pypi.hfai.high-flyer.cn/simple --trusted-host pypi.hfai.high-flyer.cn -U
    ```

2. Deploy to Kubernetes

- Get help:
  ```shell
  $ hai-up -h
  Usage:
    hai-up.sh config/run/up/dryrun/down/upgrade [option]
    Commands:
      config:   Print config script
      run/up:   Deploy platform
      dryrun:   Generate config template
      down:     Remove deployment
      upgrade:  Update hai-cli/hai-up

    Options:
      -h/--help:      Show help
      -p/--provider:  k8s/docker-compose (default: k8s)
      -c/--config:    Use environment config file

    Deployment Steps:
      1. Ensure:
         - Kubernetes cluster with LB and ingress
         - Shared filesystem mounted on all nodes
         - Docker/docker-compose installed (for docker-compose provider)
      2. Generate config: "hai-up config > config.sh"
      3. Deploy: "hai-up run -c config.sh"
  ```

Configure environment variables (shared FS path, node groups, training image, mounts, etc.) and deploy:

```shell
hai-up config > config.sh
# Edit config.sh

hai-up run -c config.sh
```
Use hai-cli to initialize and submit jobs:
```shell

HAI_SERVER_ADDR=$(kubectl -n hai-platform get svc hai-platform-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Use token from USER_INFO
hai-cli init ${TOKEN} --url http://${HAI_SERVER_ADDR}

# Python files should be in workspace: ${SHARED_FS_ROOT}/hai-platform/workspace/{user.user_name}
hai-cli python ${SHARED_FS_ROOT}/hai-platform/workspace/$(whoami)/test.py -- -n 1
```
To stop: hai-up down

Appendix: Configuration
Network Ports
Default ports:

80: Web service

5432: PostgreSQL

6379: Redis

8080: Studio UI

Kubernetes Configuration
Requires RBAC permissions for resource creation. Kubeconfig mounted at /root/.kube/config.

Node Groups
Supported groups: TRAINING_GROUP, JUPYTER_GROUP. Configure via:

```shell

export MARS_PREFIX="hai-platform-one"
export TRAINING_GROUP="training"
export JUPYTER_GROUP="jupyter_cpu"
export TRAINING_NODES="cn-hangzhou.172.23.183.227"
export JUPYTER_NODES="cn-hangzhou.172.23.183.226"
```
Mount Points
Default mounts:

```yaml

  - '${HAI_PLATFORM_PATH}/kubeconfig:/root/.kube:ro'
  - '${HAI_PLATFORM_PATH}/log:/high-flyer/log'
  - '${DB_PATH}:/var/lib/postgresql/12/main'
  - '${HAI_PLATFORM_PATH}/redis:/var/lib/redis'
  - '${HAI_PLATFORM_PATH}/workspace/log:${HAI_PLATFORM_PATH}/workspace/log'
  - '${HAI_PLATFORM_PATH}/log/postgresql:/var/log/postgresql'
  - '${HAI_PLATFORM_PATH}/log/redis:/var/log/redis'
  - '${HAI_PLATFORM_PATH}/init.sql:/tmp/init.sql'
  - '${HAI_PLATFORM_PATH}/override.toml:/etc/hai_one_config/override.toml'
Database Configuration
Built-in Database
PostgreSQL and Redis configured via override.toml:
```

```yaml
  [database.postgres]
  host = '${HAI_SERVER_ADDR}'
  port = 5432
  user = '${POSTGRES_USER}'
  password = '${POSTGRES_PASSWORD}'
  [database.redis]
  host = '${HAI_SERVER_ADDR}'
  port = 6379
  password = '${REDIS_PASSWORD}'
```

Database Initialization
Sample init.sql:

```sql
-- Storage mounts
INSERT INTO "storage" (...) VALUES (...);

-- Quota settings
INSERT INTO "quota" (...) VALUES (...);

-- User setup
INSERT INTO "user" (...) VALUES (...);

-- Training environments
INSERT INTO "train_environment" (...) VALUES (...);

-- Node info
INSERT INTO "host" (...) VALUES (...);
```

Platform Configuration
override.toml example:

```toml
[experiment.log.dist]
dir = '${HAI_PLATFORM_PATH}/workspace/log/{user_name}'

[database.postgres]
host = '${HAI_SERVER_ADDR}'

[scheduler]
default_group = '${TRAINING_GROUP}'

[launcher]
api_server = '${HAI_SERVER_ADDR}'
task_namespace = '${TASK_NAMESPACE}'
manager_image = '${BASE_IMAGE}'

[jupyter]
shared_node_group_prefix='${JUPYTER_GROUP}'
```

SSH Configuration
To enable SSH in task containers:

Create initialization scripts for SSH key setup

Mount scripts to /usr/local/sbin/hf-scripts/post_system_init/ via storage table
