# ECS Managed Daemons Console Instructions
Follow these instructions to launch your daemon tasks on ECS Managed Instances using Amazon ECS Console.

## 1. Creating an ECS Cluster with Managed Instances

If you need to create a new ECS cluster, follow the[ Creating a cluster for Amazon ECS Managed Instances ](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-cluster-managed-instances.html) user guide. Once your cluster is created, proceed to create a daemon task definition.

## 2. Creating a Daemon Task Definition
This is the blueprint for your daemon. ECS Managed Daemons use a new daemon task definition resource, distinct from standard ECS task definitions.

**Step 1: Navigate to Daemon Task Definitions**

1. In the ECS console left navigation, click Daemon task definitions
2. Click Create new daemon task definition button

**Step 2: Configure Task Definition Basics**

1. Daemon task definition family: Enter a unique name (e.g., "myDaemon")    * Up to 255 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed
2. Task role (Optional): Choose an IAM role that grants permissions to the applications running in your containers    * This task role allows your application to make API calls to AWS services    * Leave blank if your container doesn't need AWS API access
3. Task execution role: Select ecsTaskExecutionRole    * A task execution IAM role is used by the container agent to make AWS API requests on your behalf    * Required for pulling container images and publishing logs
4. Task size: Specify the amount of CPU and memory to reserve for your daemon    * CPU: 0.5 vCPU (example)    * Memory: 1 GB (example)

**Step 3: Configure Container Details**

1. Container name: Enter a name for your container (e.g., "myDaemon-agent")    * Up to 255 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed
2. Essential container: Select Yes    * Each daemon task definition must have at least one essential container    * If an essential container fails, the entire task is stopped
3. Image URI: Enter the Docker image URI (e.g., "public.ecr.aws/myDaemon/agent:latest")    * You can click Browse ECR images to select from Amazon ECR    * Or use public images from Docker Hub or other registries

**Step 4: Configure Resource Allocation (Optional)**

Container-level CPU and memory limits are different from task-level values. They define how many resources are allocated to the container.

1. CPU: 1 vCPU (example)
2. Memory hard limit: 3 GB (example)    * If container attempts to exceed this memory, the container is terminated
3. Memory soft limit: 1 GB (example)    * Soft reservation of memory for the container

**Step 5: Configure Health Check (Optional)**

Configure container health checks to monitor container health:

1. Health check command: Enter the command to check container health    * Example: CMD-SHELL, curl -f http://localhost/ || exit 1
2. Interval: 30 seconds (time between health checks)
3. Timeout: 5 seconds (time to wait for health check to succeed)
4. Start period: 0 seconds (grace period before health checks start)
5. Retries: 5 (number of consecutive failures before marking unhealthy)

**Step 6: Configure Environment Variables (Optional)**

Add environment variables for your container:

1. Key: Environment variable name (e.g., "MY_KEY")
2. Value type: Choose between:    * Environment variable: Plain text value    * Secrets Manager: Reference to AWS Secrets Manager secret    * Parameter Store: Reference to AWS Systems Manager Parameter Store
3. Value or value from: Enter the value or ARN

**Step 7: Configure Log Collection (Optional)**

1. Check Use log collection to send container logs to a logging destination
2. This uses a default configuration
3. See pricing information on Amazon CloudWatch

**Step 8: Configure Tags (Optional)**

Add tags to help identify and organize your daemon task definitions

**Step 9: Create the Daemon Task Definition**

1. Review all configuration settings
2. Click Create (orange button) to create the daemon task definition.

## 3. Deploy your Daemon Service

Once your daemon task definition is created, you can deploy it as a daemon service which will automatically run one task on each container instance in your cluster.

**Step 1: Navigate to Daemons Tab**
* In the ECS console, click Clusters in the left navigation
* Select your cluster (e.g., "daemon-sanity-check")
* Click the Daemons tab
* Click Create button

**Step 2: Configure Daemon Configuration**
* Daemon task definition family: Select your daemon task definition from the dropdown
* Choose an existing daemon task definition family
* To create a new daemon task definition, go to Daemon task definitions in the left navigation
* Daemon task definition revision: Select the revision to use
* Select from the 100 most recent entries, or enter a revision number
* Leave blank to use the latest revision
* Daemon name: Enter a unique name for this daemon
* Up to 255 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed

**Step 3: Configure Environment**
* Cluster: Pre-populated with your selected cluster (e.g., "daemon-sanity-check")
* Capacity providers: Select one or more Managed Instances capacity providers
* Choose the capacity providers to use for deploying this daemon
* These determine which instance types will run your daemon tasks

**Step 4: Configure Deployment Configuration**
* Drain percentage: Default is 25
* Percentage of tasks to drain during deployment updates
* Controls how many tasks are stopped at once during updates
* Bake time: Default is 3 minutes
* Time to wait after deployment before considering it successful
* Allows time to verify new tasks are healthy before proceeding
* Use CloudWatch alarm(s): Optional checkbox
* Enable to use CloudWatch alarms to monitor deployment health
* Can automatically roll back deployments if alarms trigger

**Step 5: Configure Tags (Optional)**
* Expand Tags - optional section
* Add tags to identify and organize your daemon task definitions
* Format: Key-Value pairs

**Step 6: Create the Daemon**
* Review all configuration settings
* Click Create button
* Wait for the daemon to be created and tasks to deploy

## 4. Running an Application Task to provision EC2 instances and activating the Daemon

After creating the daemon, you must run an application task using the same Managed Instances capacity provider to trigger EC2 instance provisioning and activating the daemon deployment. 
Daemons do not launch instances independently—they require an application task to provision the underlying infrastructure. You can also deploy an ECS service on the Managed Instances Capacity provider

**Step 1: Navigate to Tasks** 
* In your cluster view, click the Tasks tab
* Click Run new task button

**Step 2: Run Task with Managed Instance Capacity Provider**
* Follow the AWS ECS RunTask API reference to run a task
* Important: Select the same Managed Instance capacity provider you used when creating the daemon
* This task triggers the daemon to deploy on your container instances

**Step 3: Observe Task States**
* After running the task, observe the following sequence on the Tasks tab:
* Your requested task will move to Provisioning state
* A new daemon task will automatically start
* Once the daemon task reaches Running state, your requested task will also move to Running state
* This confirms the daemon has been successfully activated on your container instances


## 5. Verifying Daemon Deployment
After creating the daemon:

1. Check Daemon Status: Navigate to your cluster → Daemons tab. Verify daemon shows "Active" status and note the creation timestamp
2. View Running Tasks: Go to the Tasks tab in your cluster. Verify tasks are running with your daemon name. Check that one task is running on each container instance

## 6. Additional Resources

* [ECS Task Placement](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement.html)
* [ECS Troubleshooting Guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/troubleshooting.html)




