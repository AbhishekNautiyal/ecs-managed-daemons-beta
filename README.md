# Amazon ECS Managed Daemons Beta

Amazon ECS is introducing a new **ECS Managed Daemons** service construct for ECS Managed Instances, enabling platform engineers to centrally deploy and update software agents without coordinating with application teams or redeploying workloads. Previously, customers could only run daemons as sidecar containers for ECS Managed Instances, requiring application redeployments and configuration changes every time an agent needed updates—a significant operational burden. This also prevented host-level monitoring for agents due to the multi-task nature of sidecar-based daemons. 

With this launch, ECS introduces daemon services that run as independent processes bound to the container instance lifecycle rather than individual application tasks. When you create or update a daemon service, ECS automatically manages deployments across your ECS Managed Instances, provisioning daemon tasks before placing application tasks and deprovisioning them after all application tasks are removed. This eliminates the need to update shared configurations, or redeploy core applications just to upgrade agents. Platform engineers gain centralized control over daemon lifecycles while application developers maintain full autonomy over their application configurations and deployments.  

Getting started with ECS Managed Daemons is easy. Using the ECS console, navigate to **Daemon task definitions** to create your daemon task definition by specifying your container image, CPU, and memory requirements. Then, go to your desired cluster's **Daemons** tab and click **Create** to deploy the daemon across one or more Managed Instances capacity providers. ECS ensures your daemon task launches first on every provisioned EC2 instance and automatically drains and replaces any running instances, guaranteeing complete daemon coverage across your Managed Instances capacity provider. You can refer to user guide [here](https://github.com/AbhishekNautiyal/ecs-managed-daemons-beta/blob/12dcc5e53c741cfaf1cb690d9d331d07be30d99a/ECS%20Managed%20Daemons%20Console%20User%20Guide.md).

## ADDITIONAL TERMS AND CONDITIONS FOR BETA TEST

These additional terms and conditions supplement the terms and conditions contained in the AWS Customer Agreement, AWS Service Terms, or other agreement with us governing your use of our Services and apply to Company’s participation in the Beta Test (or Tech Preview) for the Beta Service described below. **IF YOU DO NOT AGREE TO THE ADDITIONAL TERMS AND CONDITIONS IN THIS DOCUMENT, YOU MAY NOT USE THE BETA SERVICE.**

**Name of the Beta Service:** 	Amazon ECS Managed Daemons

**Term of Beta Test:**	2/18/2025 to 3/12/2025 (unless shortened or extended by AWS)

**Costs or Charges for use of Beta Services**	For this Beta Service, standard pricing will apply for Company’s use of Amazon ECS.

**Additional Terms and Conditions**	
•	The Beta Service will only be available at certain AWS regions specified by AWS.
•	The Beta Service is only intended for evaluating the Beta Service using development/test workloads. You should not use it for production workloads. 
•	You may experience some disruption as we update the Beta Service with fixes or before general availability.  In addition, Beta Service resources and any data generated during the Beta Service may not be migrated over to any generally available version of the Beta Service and may not persist past the end of the Beta Service term.
•	AWS may change the functionality of the Beta Service during and after the term of the Beta.  Beta Service functionality, features and documentation may change during the Beta and when the Beta Service is made generally available.

