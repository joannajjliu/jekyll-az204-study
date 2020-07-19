---
layout: page
title: Develop Azure Compute Solutions
permalink: /develop-azure-compute-solutions/
---
# AZ-204 Developing Solutions for MS Azure
## Develop Azure Compute Solutions (25 - 30%)
### Implement IaaS Solutions
- Provision VMs
	+ Generalizing a VM:
		0. Use Sysprep to generalize the image, define variables, $vmName, $rgName, $location, $imageName.
		1. Stop-AzVM -ResourceGroupName $rgName -Name $vmName -Force
		2. Set-AzVM -ResourceGroupName $rgName -Name $vmName -Generalized
		3. $vm = Get-AzVM -Name $vmName -ResourceGroupName $rgName
		4. $image = New-AzImageConfig -Location $location -SourceVirtualMachineId $vm.Id
		5. New-AzImage -Image $image -ImageName $imageName -ResourceGroupName $rgName
- Configure VMs for remote access
- Create ARM templates
- Create container images for solutions by using Docker 
	+ Create an ASP.NET Core app in a Docker container from Docker Hub using Azure CLI:
	
	```
	resourceGroup="aspResourceGroup"
	plan="aspAppService"
	appName="aspApp"
	location="WestUS"
	dockerHubContainerPath="company1/asp-app"
	
	az group create --name $resourceGroup --location $location
	
	az appservice plan create --name $plan \
		--resource-group $resourceGroup --location $location --is-linux --sku S1
		
	az webapp create --name $appName --plan $plan \
		--resource-group $resourceGroup
	
	# Configure container image pushed to Docker Hub, to be used by the Web App.
	# To reference an image in Docker Hub registry (default container registry),
	# 	set the dockerHubContainerPath variable with pattern,
	# 	<organization/username>/<image>	
	az webapp config container set \
		--docker-custom-image-name $dockerHubContainerPath \
		--name $appName --resource-group myResourceGroup
	```

+ Push image to a private Docker container registry (azr) using the Docker CLI:
	1. Build the Dockerfile with docker build
		- Creates a new container image, with instructions inside
	2. Tag the image as registry1.azurecr.io/app1
		- To push a container image to ACR you need to tag the image, `<registry name>.azurecr.io/<image name>`
	3. Login in registry1 with az acr login, and push the image to registry1, `az acr login --name myregistry`, or `docker login myregistry.azurecr.io`

	```
	# Log in to a registry
	az acr login --name myregistry
	# or use, docker login myregistry.azurecr.io
	
	# Pull the official Nginx image
	docker pull nginx
	
	# Run the container locally
	docker run -it --rm -p 8080:80 nginx
	
	# Create an alias of the image
	docker tag nginx myregistry.azurecr.io/samples/nginx
	
	# Push the image to your registry
	docker push myregistry.azurecr.io/samples/nginx
	
	# Pull the image from your registry
	docker pull myregistry.azurecr.io/samples/nginx
	
	# Start the Nginx container
	docker run -it --rm -p 8080:80 myregistry.azurecr.io/samples/nginx
	
	# (Optional) Remove the image locally:
	docker rmi myregistry.azurecr.io/samples/nginx
	
	# (Optional) Remove images from your Azure container registry
	az acr repository delete --name myregistry --image samples/nginx:latest
	
	```


	```
	# Set up a docker image with the  
	 aspnetcore runtime environment.  
	 Working directory set to /app, and exposes port 443
	 
	FROM microsoft/dotnet:2.2-aspnetcore-runtime-stretch-slim AS base
	WORKDIR /app
	EXPOSE 443
	
	# Set up an image, to be used to build the application.  
	The myContainerApp.csproj file needs to be built into myContainerApp.dll  
	before it can be deployed in a runtime environment
		
	FROM Microsoft/dotnet:2.2.-sdk-stretch AS build
	WORKDIR /myAppWorkingDir
	RUN dotnet build “myContainerApp.csproj” –c Release –o /app
	
	# Create a publish label for myContainerApp.dll
		
	FROM build AS publish
	RUN dotnet publish “myContainerApp.csproj” –c Release –o /app
	
	# Take the runtime image definition/ base  
	and copies the published version of myContainerApp.dll  
	to the app folder
		
	FROM base AS final
	WORKDIR /app
	COPY –from=publish /app
	
	ENTRYPOINT [“dotnet”, “myContainerApp.dll”]
	```

	+ Another Docker command example:

	```
	FROM Microsoft/dotnet:2.2-aspnetcore-runtime-stretch-slim AS base
	WORKDIR /app
	EXPOSE 80
	EXPOSE 443
	
	FROM Microsoft/dotnet:2.2-sdk-stretch AS build
	WORKDIR /src
	COPY [“HelloDockerTools/HelloDockerTools.csproj”, “HelloDockerTools/”]
	RUN dotnet restore “HelloDockerTools/HelloDockerTools.csproj”
	COPY . .
	WORKDIR “/src/HelloDockerTools”
	RUN dotnet build “HelloDockerTools.csproj” –c Release –o /app
	
	FROM build AS publish
	RUN dotnet publish “HelloDockerTools.csproj” –c Release –o /app
	
	FROM base AS final
	WORKDIR /app
	COPY –from=publish /app
	
	ENTRYPOINT [“dotnet”, “HelloDockerTools.dll”]
	```

- Publish an image to the Azure Container Registry (ACR)
	+ `azr acr import` imports the image into the destination ACR. 
	+ `--name` and `-n` parameters serve the same purpose. They both specify the destination ACR
	+ `-t` or `--image` parameters serves the same purpose; they specify the target image name.
	+ `:latest` is the image tag

	```
	az acr import \
		--name myreg1 \
		--source mcr.microsoft.coWm/windows/servercore:latest \
		-t servercore:latest
	```
	```
	az acr import \
		--name myregistry \
		--source docker.io/library/hello-world:latest \
		--image hello-world:latest
	```
	
	+ Import an image by manifest digest, instead of by tag:

	```
	az acr import \
		--name myregistry \
		--source mysourceregistry.azurecr.io/aci-helloworld@sha256:123456abcdefg
	```

	+ Verify that multiple manifests are associated with an image in ACR:

	```
	az acr repository show-manifests \
		--name myregistry \
		--repository hello-world
	```

	+ `az acr replication` manages geo-replicated regions of ACRs, and not the ACR content.
	+ `az acr update` updates parameters for an ACR, such as enabling an administrative account, or updating tags.
	+ `az acr create` creates a new ACR
	+ `--registry` and `-r` parameters are the same; they both specify a source registry

- Run containers by using Azure Container Instance (ACI):
	+ ACI Container Groups: Ability to host a set of containers on the same host machine; can have two separate container images, maintained by different teams, deployed together. Each deployed container instance shares the resources of the host, and are able to communicate with each other.
	+ Azure Container Registry: Where multiple **container images** are hosted.
	+ Kubernetes Pods: Provides capability to sidecar container instances, similar to ACI Container Groups.
	+ Azure App Servies: Provides platforms to host container images
- Azure Kubernetes Service (AKS) is out of scope [they say, but it's not really]

### Create Azure Service Web Apps
Azure App Service Web Apps enables building and hosting of web apps in the programming language of your choice (ASP.NET Core, ASP.NET, Node.JS, PHP, Java) without managing infrastructure. Also offers auto-scaling and high availability, supports both Windows and Linux, and enables automated deployments from Github, Azure DevOps, and any Git repo.

- Create an Azure App Service Web App
	+ Create an Azure App Service Web app, consisting of two containers, configured using a Docker Compose file (docker-compose.yml), using Azure CLI:
	```
	az webapp create --resource-group blogResourceGroup \
	--plan blogServicePlan \
	--name blog \
	#web uses a Docker Compose file for application configuration:
	--multicontainer-config-type compose \
	--multicontainer-config-file docker-compose.yml
	```
	
	* Other commands:
	```
	#use a Kubernetes deployment file as the application configuration
	--multicontainer-config-type kube
	
	#create container web app, using the default
	#  PHP image from Docker Hub
	--docker-custom-image-name php:apache-7
	```
		

	+ Create an Azure App Service app with deployment from Github, using Azure CLI
	```
	# specify github repo
	gitrepo=https://github.com/company1/aspnetcoreapp1
	
	# create resource group, group1
	az group create --location centralus --name group1
	
	# create the linux App Service plan in rg group1, 
	# --sku S1 specifies the Standard pricing tier, and a small VM
	az appservice plan create --name linux-service-plan \
	--resource-group group1 --is-linux --sku S1
	
	# Create the App Service app in linux service plan
	az webapp create --name application1 -- resource-group group1 \
	--plan linux-service-plan

	# Deploy the code from the github repo to application1. --manual-integration performs a one time deploy (without enabling auto syc between source control and the App Service)
	az webapp deployment source config --name application1 --resource-group group1 \
	--repo-url $gitrepo --branch master --manual-integration
	
	```
	+ Remove Resource group and all associated resources:
	```
	az group delete --name myResourceGroup
	```
- Enable diagnostics logging
	+ **AppServiceAppLogs under Diagnostic settings**: exceptions and/or logs written by the application to the environment's logging utility.
	+ **AllMetrics** are collected by agents on the App Service, to report the usage of host resources, such as CPU usage, memory usage, and disk I/O used.
	+ Send to **Log Analytics**: Ability to write Kusto queries, to analyze the logs.
	+ Send logs to a **storage account**, for archiving logs. Storage accounts *cannot* run queries. Logs stored in a storage account need to be restored in an environment before running queries on them. 
	+ Storage of application logging data:
		1. Filesystem storage: supported for both Windows and Linux apps. Is also the **only storage option** supported for Linux apps. Filesystem storage is designed for **short-term logging** and turns itself off after 12 hours.
		2. Blob storage: supported for Windows apps only. When using Blob storage, the storage account **must** be in the same region as the App Service. Blob storage is designed for **long-term storage** of logging information.
- Deploy code to a web app
	+ Create a web app, myCompanyApp1, and deploy the app to a deployment slot, StagingSlot:
	```
	gitRepo = "https://github.com/myCompanyApp1/"
	
	appPlan = "myCompanyAppPlan1"
	
	appName = "myCompanyWebApp1"
	
	slotName = "StagingSlot"
	
	rgName = "myCompanyResourceGroup"

	
	az group create --location centralus --name $rgName
	
	az appservice plan create --name $appPlan --resource-group $rgName -sku P3V2
	
	# automatically creates a slot for the production app
	az webapp create --name $appName --resource-group $rgName --plan $appPlan
	
	# create another slot, "StagingSlot" for testing
	az webapp deployment slot create --name $appName --resource-group $rgName --slot $slotName
	
	az webapp deployment source config --name $appName --resource-group $rgName --slot $slotName \
		--repo-url $gitRepo --branch master --manual-integration
	```
- Configure web app settings including SSL, API, and connection strings
 	+ `New-AzRoleAssignment`: Configure web app 'contribute' members to a security principal, based on RBAC.
	+ `New-AzRoleDefinition`: Create a new RBAC role definition - doesn't control access to the role.
	+ `Set-AzRoleDefintion`: Configure an existing RBAC role.
	+ CORS support for cross-communication between apps of different origins/hosts:
		- Azure App Service api1 (https://api.company1.com) consumed by a SPA, spa1 (https://company1.com): Run the `az webapp cors add --alowed-origins=https://company1.com` command in api1
	+ Users use Azure AD credentials for authentication, and permission levels are based on user Azure AD group membership. You need to provide user group membership information in tokens. The tokens are used in the app to include **security groups and Azure AD roles** only. You create a new Azure AD app manifest in Azure portal. Manifest attribute and value:
		* **Attribute**: groupMembershipClaims
		* **Value**: SecurityGroup
		* ^ **groupMembershipClaims** set at **SecurityGroup**, retrieves group **membership and Azure AD roles** for the group claim. Value **None** returns no user security claims. Value **All** includes distribution groups along with group membership and Azure AD roles.
		* Attribute appRoles: specifies collection of roles that an app may declare.
		* Attribute signInAudience: identify Microsoft accounts supported by the app.
	+ **Application code** executed **reads the local.settings.json** file
		- Require different connection string configurations in the code based on where it is running. Modify the set-up to provide the following features:
			+ Provide only function access to a connection string parameter
			+ Configure separate values for the connection string when running in the local test environment, versus on an Azure Function App
			1. Set the connection string attribute to a setting name in the code.
			2. Use the configuration settings on the Azure function and local.settings.json in the local environment to set the connection string: Code executed in the machine reads from the local.settings.json file. When running on Azure, the configuration is picked up from the values set in the configuration settings section of the function.
		- **host.json** file has configurations defining **how the execution environment is configured;** used by the platform at runtime and **NOT read by the application.**
		- **Global variable** in function startup code, makes the **connection string available to all function** deployed in that server less instance
	+ TLS mutual authentication/ client certificate authentication: Available only via HTTPS (not HTTP). It ensures TLS client certificate from a trusted CA (certificate authority) is available to the app, through:
		- Custom SSL not available in F1 or D1 tiers. Requires scale-up => must be in a non-free tier (i.e. B1, B2, B3 or Production category)
		- Enable client certification:
		```
		az webapp update --set clientCertEnabled=true --name <app_name> --resource-group <group_name>
		```
		- ASP.NET apps only: use `HttpRequestdest.ClientCertificate` property
		- Node.JS and other apps: certificate is passed through the HTTPS request header (as a base64-encoded X-ARR-ClientCert header/PEM string)
	+ Client cookie: contains information passed to the client for local storage


- Implement autoscaling rules, including scheduled autoscaling, and scaling by operational or system metrics:
	+ CPU usage metrics: measure by **App Service Plan**, which contains metrics on VM usage; CPU, Memory, and Disk Queue Length metrics
 		* *CPU time metric* is only of concern in free tiers like **Free** and **Shared** plans. Otherwise, use *CPU percentage* for **Basic, Standard, or Premium** plans.
	+ Scenario: Azure App Service App, myApp1, scales based on the number of messages received in a service bus queue:
		* Configure rule to scale down instances by 1 when the average number of messages in the queue stays equal or below 800/minute over a 10-minute period:
			1. Metric source: Service Bus queue
			2. Metric name: Active Messages Processed/Instance(Avg)
				- vs. Messages Processed/Instance (Avg) includes messages currently in the queue irrespective of whether they are being processed.
				
			3. Time grain:
				- Minimum: returns the least number of messages processed at any  minute, in the duration specified
				- Maximum: returns the maximum number of messages processed at any minute, in the duration specified
				- Sum: adds up the active messages processed in the 10 minutes duration

### Implement Azure Functions

- Implement input and output bindings for a function

- Implement function triggers by using data operations, timers, and webhooks
	+ Control Azure Function timer triggering: NCRONTAB expression fields
	`{second} {minute} {hour} {day} {month} {day-of-week}`
		- "0 0 &ast;/3 &ast; &ast; &ast;" triggers at 0 second, 0 minutes, every third hour
		- "0 &ast;/3 &ast; &ast; &ast; &ast;" triggers every third minute
		- "0 0 3 &ast; &ast; &ast;" triggers at 0 second, 0 minutes, at 3AM, ever day
		- "0 0 0 &ast; &ast; 3" triggers at 0 second, 0 minutes, 12AM, every Wednesday
- Implement Azure Durable Functions:
	+ Durable Functions are an extension of Azure Functions. Used for stateful orchestration of function execution:
		* A durable function app is a solution made up of different Azure functions.
			- Function app project folder structure:
				+ host.json file contains run-time specific configurations applying to the Function App as a whole
				+ Each function.json file contains configurations for that function
				+ SharedCode folder stores shared code
				+ bin folder contains packages and library files that the function app uses
				+ host.json, and function.json files can be edited in the Azure portal's function editor (for small changes). Best practice, and for larger changes, is to use a local development tool, like VS Code.
				```
				FunctionApp
				- host.json
				- MyFirstFunction
					- function.json
					- ...
				- MySecondFunction
					- function.json
					- ...
				- SharedCode
				- bin
				```
		* Four durable function types in Azure Functions: activity, orchestrator, entity, and client
		1. Orchestrator: How actions are executed and the order in which they are executed
		2. Activity: The basic unit of work in a durable function orchestration; they are the functions and tasks orchestrated in the process.
		3. Entity: Operations for reading and updating small pieces of state.
		4. Client: Used to trigger orchestrator and entity functions; neither can be triggered directly using buttons in the Azure portal.
	+ Durable Orchestrator functions are used to orchestrate the execution of other Durable functions within a function app:
		* Execution progress is durable/reliable (is automatically check-pointed when the function "awaits" or "yields"). Local state is never lost when the process recycles or the VM reboots.
		* Can be long-running: secs, days, months, or never-ending
	+ Durable function patterns:
		* **Function chaining**: Series of functions execute in a specific order; the output of one function is applied to the input of the next function.
		```
		[FunctionName("Chaining")]
		public static async Task<object> Run(
			[OrchestrationTrigger] IDurableOrchestrationContext context)
		{
			try
			{
				var x = await context.CallActivityAsync<object>("F1", null);
				var y = await context.CallActivityAsync<object>("F2", x);
				var z = await context.CallActivityAsync<object>("F3", y);
				return await context.CallActivityAsync<object>("F4", z);	
			}
			catch (Exception)
			{
				// Error handling or compensation goes here
			}
		}
		```
		* **Fan in/ Fan out**: Multiple function execute in parallel, than wait for all functions to finish, with aggregation work done on the results returned from the functions.
		```
		[FunctionName("FanOutFanIn")]
		public static async Task Run(
			[OrchestrationTrigger] IDurableOrchestrationContext context)
		{
			var parallelTasks = new List<Task<int>>();
			
			// Get a list of N work items to process in parallel.
			object[] workBatch = await context.CallActivityAsync<object[]>("F1", null);
			for (int i = 0; i < workBatch.Length; i++)
			{
				Task<int> task = context.CallActivityAsync<int>("F2", workBatch[i]);
				parallelTasks.Add(task);
			}
			await Task.WhenAll(parallelTasks);
			
			//Aggregate all N outputs and send the result to F3.
			int sum = parallelTasks.Sum(t => t.Result);
			await context.CallActivityAsync("F3", sum);
		}
		```
		* **Aggregation**: Event data provided in batches by multiple sources over a period of time, to be combined into a single, addressable entity.
		```
		[FunctionName("Counter")]
		public static void Counter([EntityTrigger] IDurableEntityContext ctx)
		{
			int currentValue = ctx.GetState<int>();
			switch (ctx.OperationName.ToLowerInvariant())
			{
				case "add":
					int amount = ctx.GetInput<int>();
					ctx.SetState(currentValue + amount);
					break;
				case "reset":
					ctx.SetState(0);
					break;
				case "get":
					ctx.Return(currentValue);
					break;
			}
		}
		```
		* Using .NET classes and methods:
		```
		public class Counter
		{
			[JsonProperty("value")]
			public int CurrentValue { get; set; }
			
			public void Add(int amount) => this.CurrentValue += amount;
			
			public void Reset() => this.CurrentValue = 0;
			
			public int Get() => this.CurrentValue;
			
			[FunctionName(nameof(counter))]
			public static Task Run([EntityTrigger] IDurableEntityContext ctx)
				=> ctx.DispatchAsync<Counter>();
		}
		```
		* **Async HTTP APIs**: Coordinating the state of long-running operations with external clients.
		```
			> curl -x POST https://myfunc.azurewebsites.net/api/orchestrators/DoWork 
			-H "Content-Length: 0" -i HTTP/1.1 202 Accepted
			Content-Type: application/json
			Location: https://myfunc.azurewebsites.net/runtime/webhooks/durabletask/instances/b79baf67f717453ca9e86c5da21e03ec

			{"id":"b79baf67f717453ca9e86c5da21e03ec", ...}
			
			> curl https://myfunc.azurewebsites.net/runtime/webhooks/durabletask/instances/b79baf67f717453ca9e86c5da21e03ec -i
			HTTP/1.1 202 Accepted
			Content-Type: application/json
			Location: https://myfunc.azurewebsites.net/runtime/webhooks/durabletask/instances/b79baf67f717453ca9e86c5da21e03ec

			{"runtimeStatus":"Running","lastUpdatedTime":"2019-03-16T21:20:47Z", ...}
			
			> curl https://myfunc.azurewebsites.net/runtime/webhooks/durabletask/instances/b79baf67f717453ca9e86c5da21e03ec -i
			HTTP/1.1 200 OK
			Content-Length: 175
			Content-Type: application/json

			{"runtimeStatus":"Completed","lastUpdatedTime":"2019-03-16T21:20:57Z", ...}
		```
		* **Monitoring**: Support to a flexible, recurring process in a workflow, often with a Timer trigger.
		```
		[FunctionName("MonitorJobStatus")]
		public static async Task Run(
			[OrchestrationTrigger] IDurableOrchestrationContext context)
		{
			int jobId = context.GeetInput<int>();
			int pollingInterval = GetPollingInterval();
			DateTime expiryTime = GetExpiryTime();
			
			while (context.CurrentUtcDateTime < expiryTime)
			{
				var jobStatus = await context.CallActivityAsync<string>("GetJobStatus", jobId);
				if (jobStatus == "Completed")
				{
					// Perform an action when a condition is met.
					await context.CallActivityAsync("SendAlert", machineId);
					break;
				}
				
				// Orchestration sleeps until this time.
				var nextCheck = context.CurrentUtcDateTime.AddSeconds(pollingInterval);
				await context.CreateTimer(nextCheck, CancellationToken.None);
			}
			
			// Perform more work here, or let the orchestration end.
		}
		```
		* **Human interaction**: Automated integration with human interaction, using timeouts and compensation logic. For example, an orchestrator can use a durable timer with escalation to request human approval.
		```
		[FunctionName("ApprovalWorkflow")]
		public static async Task Run(
			[OrchestrationTrigger] IDurableOrchestrationContext context)
		{
			await context.CallActivityAsync("RequestApproval", null);
			using (var timeoutCts = new CancellationTokenSource())
			{
				DateTime dueTime = context.CurrentUtcDateTime.AddHours(72);
				Task durableTimeout = context.CreateTimer(dueTime, timeoutCts.Token);
				Task<bool> approvalEvent = context.WaitForExternalEvent<bool>("ApprovalEvent");
				if (approvalEvent == await Task.WhenAny(approvalEvent, durableTimeout))
				{
					timeoutCts.Cancel();
					await context.CallActivityAsync("ProcessApproval", approvalEvent.Result);
				}
				else 
				{
					await context.CallActivityAsync("Escalate", null);
				}
			}
		}
		```
