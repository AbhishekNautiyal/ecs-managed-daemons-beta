# ECS Managed Daemons Beta CLI instructions


## 1. Configure AWS CLI for Beta

Add the [ECS beta service model](https://github.com/AbhishekNautiyal/ecs-managed-daemons-beta/blob/main/ecs-test.json) to your CLI :

```
aws configure add-model --service-model file://ecs-test.json --service-name ecs-test
```

Verify the model is loaded:
```
aws ecs-test create-daemon help
```

Configure your CLI to use the CPT region:
```
export AWS_DEFAULT_REGION=af-south-1  # CPT (Cape Town) region
```

Or configure in your AWS CLI config file (~/.aws/config):
```
[profile beta]
region = af-south-1
```

## 2. Necessary IAM Roles and Permissions

The following 2 roles are necessary to use ECS Managed Instances with Daemons:

* An infrastructure role which will be used by AWS to create managed resources on behalf of customers
* An instance profile which will contain an instance role to be assumed by ECS Agent

Note: ECS creates a service-linked-role (SLR) AWSServiceRoleForECSCompute which includes the permissions necessary to manage the resources created for ECS Managed Instances.

2.1 Infrastructure Role

Create an IAM role with the AWS managed policy for ECS Managed Instances:

Using AWS Console:

1. Go to IAM → Roles → Create role
2. Select "AWS service" as trusted entity type
3. Choose "Elastic Container Service" as the service
4. Select "Elastic Container Service" as the use case
5. Attach the managed policy: AmazonECSInfrastructureRolePolicyForManagedInstances
6. Name the role (e.g., ecsInfrastructureRole)

Using AWS CLI:

**Create trust policy file**
```
cat > ecs-infra-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

**Create the role**
```
aws iam create-role \
    --role-name ecsInfrastructureRole \
    --assume-role-policy-document file://ecs-infra-trust-policy.json
```
**Attach the AWS managed policy**
```
aws iam attach-role-policy \
    --role-name ecsInfrastructureRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonECSInfrastructureRolePolicyForManagedInstances
```

The **AmazonECSInfrastructureRolePolicyForManagedInstances** managed policy provides all necessary permissions for ECS to create and manage EC2 resources for Managed Instances on your behalf.

**2.2 Instance Profile**

Create an instance profile and instance role with the AWS managed policy for ECS Managed Instances:

Using AWS Console:

1. Go to IAM → Roles → Create role
2. Select "AWS service" as trusted entity type
3. Choose "EC2" as the service
4. Attach the managed policy: AmazonECSInstanceRolePolicyForManagedInstances
5. Name the role with ecsInstanceRole prefix (e.g., ecsInstanceRole)
6. After creating the role, create an instance profile and attach the role

Using AWS CLI:

**Create trust policy file**
```
cat > ecs-instance-trust-policy.json <<EOF
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF
```
**Create the instance role**
```
aws iam create-role \
    --role-name ecsInstanceRole \
    --assume-role-policy-document file://ecs-instance-trust-policy.json
```

**Attach the AWS managed policy**
```
aws iam attach-role-policy \
    --role-name ecsInstanceRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonECSInstanceRolePolicyForManagedInstances
```

**Create instance profile**
```
aws iam create-instance-profile \
    --instance-profile-name ecsInstanceProfile
```

**Add role to instance profile**
```
aws iam add-role-to-instance-profile \
    --instance-profile-name ecsInstanceProfile \
    --role-name ecsInstanceRole
```

The AmazonECSInstanceRolePolicyForManagedInstances managed policy provides permissions for ECS managed instances to register with ECS clusters and communicate with the ECS service.

Important: The instance role name must include ecsInstanceRole as a prefix to work with the infrastructure role's PassRole permissions.

## 3. Using AWS CLI

### 3.1. Create Cluster

Create an ECS cluster for your daemons:
```
aws ecs-test create-cluster --cluster-name "my-daemon-cluster"
```

### 3.2. Create Capacity Provider

Create a managed instance capacity provider with your IAM roles and network configuration:
```
cat > my-daemon-cp.json <<EOF
{
    "name": "my-daemon-capacity-provider",
    "cluster": "my-daemon-cluster",
    "managedInstancesProvider": {
        "infrastructureRoleArn": "arn:aws:iam::123456789012:role/AmazonECSFMIDefaultInfrastructureRole",
        "instanceLaunchTemplate": {
            "ec2InstanceProfileArn": "arn:aws:iam::123456789012:instance-profile/ecsInstanceProfile",
            "networkConfiguration": {
                "subnets": [
                    "subnet-abcdef01234567",
                    "subnet-bcdefa98765432"
                ],
                "securityGroups": [
                    "sg-0123456789abcdef"
                ]
            },
            "storageConfiguration": {
                "taskVolumeStorageGiB": 100
            }
        }
    }
}
EOF

aws ecs-test create-capacity-provider --cli-input-json file://my-daemon-cp.json
```

### 3.3. Register Daemon Task Definition

Register a task definition for your daemon. Daemons typically run monitoring, logging, or security agents:
```
cat > daemon-taskdef.json <<EOF
{
    "family": "my-daemon-task",
    "containerDefinitions": [
        {
            "name": "daemon-container",
            "image": "public.ecr.aws/docker/library/busybox:latest",
            "essential": true,
            "command": ["sh", "-c", "while true; do echo 'Daemon running'; sleep 30; done"],
            "cpu": 256,
            "memory": 512
        }
    ],
    "cpu": "256",
    "memory": "512"
}
EOF

aws ecs-test register-daemon-task-definition --cli-input-json file://daemon-taskdef.json
```

Valid fields:

* family (required) - Task definition family name
* containerDefinitions (required) - Array of DaemonContainerDefinition objects
* cpu (optional) - Task-level CPU units as string
* memory (optional) - Task-level memory in MiB as string
* taskRoleArn (optional) - IAM role for the task
* executionRoleArn (optional) - IAM role for task execution
* volumes (optional) - Array of DaemonVolume objects
* tags (optional) - Array of Tag objects

### 3.3.1. Daemon Task Definition Parameters

Daemon task definitions support a subset of standard ECS task definition parameters. Key differences and capabilities:

Network Mode:

* Daemons use a special daemon_bridge network mode automatically
* You cannot specify a network mode - it is set automatically by ECS
* All daemons share a network namespace on the instance
* Daemons can communicate with application tasks over the existing bridge network

Privileged Capabilities: On managed instances, daemons support privileged Linux capabilities for system-level operations:
```
{
    "containerDefinitions": [{
        "name": "monitoring-daemon",
        "image": "my-monitoring-agent:latest",
        "privileged": true,
        "linuxParameters": {
            "capabilities": {
                "add": ["SYS_ADMIN", "NET_ADMIN", "SYS_PTRACE", "BPF"]
            }
        }
    }]
}
```

Host Mounts and Volumes: On managed instances, daemons can mount host directories for accessing logs, metrics, and system information:
```
{
    "containerDefinitions": [{
        "name": "log-collector",
        "image": "fluent/fluentd:latest",
        "mountPoints": [{
            "sourceVolume": "var-log",
            "containerPath": "/var/log",
            "readOnly": true
        }]
    }],
    "volumes": [{
        "name": "var-log",
        "host": {
            "sourcePath": "/var/log"
        }
    }]
}
```

Supported Container Parameters:

* image (required) - Container image
* name (required) - Container name
* cpu - CPU units reserved for container
* memory / memoryReservation - Memory limits
* essential - Mark container as essential (default: true)
* command / entryPoint - Override container command
* environment / environmentFiles / secrets - Environment configuration
* privileged - Run container in privileged mode
* user - User to run container as
* workingDirectory - Working directory
* readonlyRootFilesystem - Read-only root filesystem
* mountPoints - Volume mount points
* logConfiguration - Logging configuration (awslogs, splunk, awsfirelens)
* healthCheck - Container health check
* dependsOn - Container dependencies
* ulimits - Resource limits
* systemControls - Kernel parameters
* linuxParameters.capabilities - Linux capabilities (add/drop)
* linuxParameters.initProcessEnabled - Enable init process
* repositoryCredentials - Private registry credentials
* restartPolicy - Container restart policy

Example: CloudWatch Agent Daemon
```
{
    "family": "cloudwatch-agent",
    "taskRoleArn": "arn:aws:iam::123456789012:role/CloudWatchAgentRole",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "cpu": "512",
    "memory": "1024",
    "containerDefinitions": [{
        "name": "cloudwatch-agent",
        "image": "public.ecr.aws/cloudwatch-agent/cloudwatch-agent:latest",
        "essential": true,
        "mountPoints": [{
            "sourceVolume": "proc",
            "containerPath": "/host/proc",
            "readOnly": true
        }, {
            "sourceVolume": "sys",
            "containerPath": "/host/sys",
            "readOnly": true
        }],
        "environment": [{
            "name": "CW_CONFIG_CONTENT",
            "value": "{\"metrics\":{\"namespace\":\"MyApplication/Monitoring\",\"metrics_collected\":{\"cpu\":{\"measurement\":[{\"name\":\"cpu_usage_idle\"}]},\"mem\":{\"measurement\":[{\"name\":\"mem_used_percent\"}]}}}}"
        }],
        "logConfiguration": {
            "logDriver": "awslogs",
            "options": {
                "awslogs-group": "/ecs/cloudwatch-agent",
                "awslogs-region": "af-south-1",
                "awslogs-stream-prefix": "cw-agent"
            }
        }
    }],
    "volumes": [{
        "name": "proc",
        "host": {
            "sourcePath": "/proc"
        }
    }, {
        "name": "sys",
        "host": {
            "sourcePath": "/sys"
        }
    }]
}
```

### 3.4. Create Daemon

Create a daemon that will run on all instances in the capacity provider:
```
cat > create-daemon.json <<EOF
{
    "clusterArn": "arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster",
    "daemonName": "my-monitoring-daemon",
    "daemonTaskDefinitionArn": "arn:aws:ecs:af-south-1:123456789012:daemon-task-definition/my-daemon-task:1",
    "capacityProviderArns": [
        "arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provider"
    ]
}
EOF

aws ecs-test create-daemon --cli-input-json file://create-daemon.json
```

Valid fields:

* daemonName (required) - Name for the daemon
* clusterArn (required) - ARN of the cluster
* daemonTaskDefinitionArn (required) - ARN of the daemon task definition
* capacityProviderArns (required) - Array of capacity provider ARNs
* deploymentConfiguration (optional) - DaemonDeploymentConfiguration object
* tags (optional) - Array of Tag objects
* propagateTags (optional) - Tag propagation setting
* clientToken (optional) - Idempotency token

The daemon will automatically start on all current and future instances in the specified capacity provider.

### 3.5. Verify Daemon Deployment

Check the daemon status:
```
aws ecs-test describe-daemon \
    --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```

List daemons in a cluster:
```
aws ecs-test list-daemons \
    --cluster-arn arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster
```

List daemon deployments:
```
aws ecs-test list-daemon-deployments \
    --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```

Describe daemon deployments:
```
aws ecs-test describe-daemon-deployments \
    --daemon-deployment-arns arn:aws:ecs:af-south-1:123456789012:daemon-deployment/my-daemon-cluster/abc123
```

### 3.6. Run Application Tasks

Now run your application tasks. The daemon tasks will already be running on the instances:
```
cat > app-taskdef.json <<EOF
{
    "family": "my-app-task",
    "containerDefinitions": [
        {
            "name": "app-container",
            "image": "public.ecr.aws/docker/library/nginx:latest",
            "essential": true,
            "memory": 512,
            "cpu": 256,
            "portMappings": [
                {
                    "containerPort": 80,
                    "protocol": "tcp"
                }
            ]
        }
    ],
    "requiresCompatibilities": ["MANAGED_INSTANCES"],
    "networkMode": "awsvpc",
    "cpu": "512",
    "memory": "1024",
    "runtimePlatform": {
        "operatingSystemFamily": "LINUX"
    }
}
EOF

aws ecs-test register-task-definition --cli-input-json file://app-taskdef.json

aws ecs-test run-task \
    --cluster my-daemon-cluster \
    --task-definition my-app-task:1 \
    --capacity-provider-strategy capacityProvider=my-daemon-capacity-provider,weight=1 \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-abcdef01234567],securityGroups=[sg-0123456789abcdef]}"
```

## 4. Deployment Scenarios

### Scenario A: Deploying Daemon on Empty Capacity Provider

This scenario demonstrates deploying a daemon on a capacity provider with no existing instances. Instances launch when application tasks are scheduled.

Prerequisites:

* Cluster created (section 3.1)
* Capacity provider created (section 3.2)
* No container instances running yet

Steps:

1. Register daemon task definition: aws ecs-test register-daemon-task-definition --cli-input-json file://daemon-taskdef.json
2. Create daemon:

The daemon creation succeeds immediately even though no instances exist yet.
```
aws ecs-test create-daemon \
   --cluster-arn arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --daemon-name my-monitoring-daemon \
   --daemon-task-definition-arn arn:aws:ecs:af-south-1:123456789012:daemon-task-definition/my-daemon-task:1 \
   --capacity-provider-arns arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provider
```
1. Verify daemon created (no instances yet):
```
aws ecs-test describe-daemon \
       --cluster-arn arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
       --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```       

1. Run application task to trigger instance launch:

When the application task is scheduled, ECS launches a container instance. The daemon task starts first, followed by the application task.
```
aws ecs-test run-task \
   --cluster my-daemon-cluster \
   --task-definition my-app-task:1 \
   --capacity-provider-strategy capacityProvider=my-daemon-capacity-provider,weight=1 \
   --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-abc123]}"
```
1. Wait for container instance to register:
```
aws ecs-test list-container-instances \    
    --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster
```
1. Optionally, compare task timestamps to verify daemon started first:

You can compare the startedAt timestamps of tasks on the instance to confirm the daemon task started before the application task.

**List all tasks on the instance**
```
aws ecs-test list-tasks \
   --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --container-instance <instance-arn>
```

**Describe tasks to compare start times**
```
aws ecs-test describe-tasks \
   --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --tasks <daemon-task-arn> <app-task-arn>
```
Cleanup:

**Stop application task**
```
aws ecs-test stop-task \
    --cluster my-daemon-cluster \
    --task <app-task-arn>
```
Instance will drain and terminate automatically after all tasks stop



### Scenario B: Deploying Daemon on Existing Capacity Provider

This scenario demonstrates deploying a daemon on a capacity provider that already has running instances with application tasks.

Prerequisites:

* Cluster created (section 3.1)
* Capacity provider created (section 3.2)
* Application task definition registered (section 3.6)
* Container instances running with application tasks. To set this up:   

**Create service with application tasks**
```
aws ecs-test create-service \       
    --cluster my-daemon-cluster \       
    --service-name my-app-service \       
    --task-definition my-app-task:1 \       
    --desired-count 2 \       
    --capacity-provider-strategy capacityProvider=my-daemon-capacity-provider,weight=1 \       
    --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-abc123]}"
```    
**Wait for service deployment to stabilize**
```
aws ecs-test wait services-stable \       
    --cluster my-daemon-cluster \       
    --services my-app-service       
```

Steps:

1. Verify existing instances and tasks:

**List container instances**
```
aws ecs-test list-container-instances \    
    --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster
```
**List running tasks**
```
aws ecs-test list-tasks \    
    --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \    
    --desired-status RUNNING
   ``` 

1. Register daemon task definition: aws ecs-test register-daemon-task-definition --cli-input-json file://daemon-taskdef.json
2. Create daemon:

Daemon creation triggers a rolling deployment to all existing instances. ECS drains instances running the old daemon revision (or no daemon) and starts the new daemon task on each instance. This ensures daemon tasks are updated without disrupting application tasks. One daemon task runs per container instance.
```
aws ecs-test create-daemon \
   --cluster-arn arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --daemon-name my-monitoring-daemon \
   --daemon-task-definition-arn arn:aws:ecs:af-south-1:123456789012:daemon-task-definition/my-daemon-task:1 \
   --capacity-provider-arns arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provide
   ```



Deployment Configuration Parameters:

You can customize the deployment behavior using the --deployment-configuration parameter:

* drainPercent (0.0-100.0) - Percentage of instances to drain simultaneously during deployment. Higher values speed up deployments but may impact capacity.
* bakeTimeInMinutes (Integer) - Time to wait after deploying to an instance before moving to the next batch. Allows monitoring for issues before continuing the rollout.
* alarms (DaemonAlarmConfiguration) - CloudWatch alarms to monitor during deployment. If alarms trigger, the deployment can be automatically rolled back.

**Example with deployment configuration: **
```
aws ecs-test create-daemon \        
    --cluster-arn arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \        
    --daemon-name my-monitoring-daemon \        
    --daemon-task-definition-arn arn:aws:ecs:af-south-1:123456789012:daemon-task-definition/my-daemon-task:1 \        
    --capacity-provider-arns arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provider \        
    --deployment-configuration '{"drainPercent":20.0,"bakeTimeInMinutes":5}'
   ```
 

1. Monitor the deployment:

Track the deployment progress using describe-daemon-deployments. The deployment status will transition from PENDING → IN_PROGRESS → SUCCESSFUL.

**List deployments for the daemon**
```
aws ecs-test list-daemon-deployments \
   --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```

**Describe specific deployment to check status**
```
aws ecs-test describe-daemon-deployments \
   --daemon-deployment-arns <deployment-arn
```
   


1. Verify daemon tasks deployed to existing instances:     

**List tasks on each instance** 
```

aws ecs-test list-tasks \ 
    --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
    --container-instance <container-instance-arn>
```

**Describe daemon tasks**
```
aws ecs-test describe-tasks \
    --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
    --tasks <daemon-task-arn-1> <daemon-task-arn-2>
```
1. Verify daemon count matches instance count:

The number of daemon tasks should equal the number of container instances. When new instances launch, daemon tasks automatically deploy to them.

**Count instances**
```
aws ecs-test list-container-instances \
   --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --query 'length(containerInstanceArns)'
```
**Count daemon tasks**
```
aws ecs-test list-tasks \
   --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --started-by daemon/<daemon-name> \
   --query 'length(taskArns)
```   

**Verify daemon is in ACTIVE state: **   
```
aws ecs-test describe-daemon \
    --cluster-arn arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
    --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```
# Check daemon task status on each instance
aws ecs-test describe-tasks \
    --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
    --tasks <daemon-task-arn> \
    --query 'tasks[0].{Status:lastStatus,Health:healthStatus,StartedAt:startedAt}'
    


Scenario C: Adding Capacity Provider to Existing Daemon

This scenario demonstrates adding a new capacity provider to an existing daemon. The daemon will automatically deploy to instances in the new capacity provider.

Prerequisites:

* Scenario B completed (daemon running on first capacity provider)
* Second capacity provider created and associated with cluster

Steps:

1. Create second managed instances capacity provider:

aws ecs-test create-capacity-provider \
    --name my-daemon-capacity-provider-2 \
    --capacity-provider-type MANAGED_INSTANCES \
    --managed-instances-configuration "instanceType=c5.large,instanceRole=arn:aws:iam::123456789012:instance-profile/ecsInstanceRole,subnetIds=[subnet-abc123,subnet-def456],securityGroupIds=[sg-abc123],vpcId=vpc-abc123"

# Associate with cluster
aws ecs-test put-cluster-capacity-providers \
    --cluster my-daemon-cluster \
    --capacity-providers my-daemon-capacity-provider my-daemon-capacity-provider-2 \
    --default-capacity-provider-strategy capacityProvider=my-daemon-capacity-provider,weight=1

1. Create service on second capacity provider:

Launch application tasks to create instances in the second capacity provider.
```
aws ecs-test create-service \
   --cluster my-daemon-cluster \
   --service-name my-app-service-2 \
   --task-definition my-app-task:1 \
   --desired-count 2 \
   --capacity-provider-strategy capacityProvider=my-daemon-capacity-provider-2,weight=1 \
   --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-abc123]}"
```

**Wait for service to stabilize**
```
aws ecs-test wait services-stable \
   --cluster my-daemon-cluster \
   --services my-app-service-2
```



1. Update daemon to include second capacity provider:

Add the second capacity provider to the daemon. This triggers deployment to all instances in the new capacity provider.
```

    aws ecs-test update-daemon \
       --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon \
       --daemon-task-definition-arn arn:aws:ecs:af-south-1:123456789012:daemon-task-definition/my-daemon-task:1 \
       --capacity-provider-arns arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provider arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provider-2
```

1. Monitor the deployment:

**List deployments to find the new deployment ARN**
```
aws ecs-test list-daemon-deployments \
    --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```

**Monitor deployment status**
```
aws ecs-test describe-daemon-deployments \
    --daemon-deployment-arns <deployment-arn>
```

**1. Verify daemon tasks on both capacity providers:**
Use the totalRunningInstanceCount field from the deployment to verify all daemon tasks are running.

**Check deployment running instance count**
```
aws ecs-test describe-daemon-deployments \
   --daemon-deployment-arns <deployment-arn> \
   --query 'daemonDeployments[0].targetDaemonRevision.totalRunningInstanceCount'
```

**List instances to verify count matches**
```
aws ecs-test list-container-instances \
   --cluster arn:aws:ecs:af-south-1:123456789012:cluster/my-daemon-cluster \
   --query 'length(containerInstanceArns)
```
   

# 6. Daemon Updates and Rollback

## 6.1. Updating Daemon Task Definition

To update a daemon with a new task definition:

**Register new task definition revision**
```
aws ecs-test register-daemon-task-definition --cli-input-json file://daemon-taskdef-v2.json
```
**Update daemon (requires ARNs based on UpdateDaemonRequest model)**
```
aws ecs-test update-daemon \
    --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon \
    --daemon-task-definition-arn arn:aws:ecs:af-south-1:123456789012:daemon-task-definition/my-daemon-task:2 \
    --capacity-provider-arns arn:aws:ecs:af-south-1:123456789012:capacity-provider/my-daemon-capacity-provider
```

The update will trigger a rolling deployment across all instances.

# 7. Cleanup

## 7.1. Delete Daemon
```
aws ecs-test delete-daemon \
    --daemon-arn arn:aws:ecs:af-south-1:123456789012:daemon/my-daemon-cluster/my-monitoring-daemon
```

Wait for daemon tasks to stop before proceeding.

## 7.2. Delete Capacity Provider
```
aws ecs-test delete-capacity-provider \
    --capacity-provider my-daemon-capacity-provider
```

Note: You must delete all daemons and services using the capacity provider first.


## 7.3. Delete Cluster
```
aws ecs-test delete-cluster --cluster my-daemon-cluster
```



# 8. API Reference

Daemon-Specific APIs

* create-daemon - Create a new daemon
* update-daemon - Update daemon task definition or configuration
* delete-daemon - Delete a daemon
* describe-daemons - Get daemon details
* list-daemons - List all daemons in a cluster
* list-daemon-deployments - List deployments for a daemon
* describe-daemon-deployments - Get deployment details

