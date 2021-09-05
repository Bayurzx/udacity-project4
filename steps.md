# Steps to make this work **locally**
## Prerequisites
- An [azure account](https://azure.microsoft.com/en-us/free/) 
- A code editor
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) 
- Python Installed 
- A redis server either on [Mac](https://medium.com/@petehouston/install-and-config-redis-on-mac-os-x-via-homebrew-eb8df9a4f298), [Linux](https://redis.io/download) or [Windows](https://riptutorial.com/redis/example/29962/installing-and-running-redis-server-on-windows)
- Create a virtual environment *venv*
``` bash
   # Mac/Linux
   python3 -m venv .venv 
   source .venv/bin/activate
   # Windows on Powershell or GitBash terminal
   py -3 -m venv .venv
   .venv\Scripts\activate
```
- Install dependencies 
``` bash
   # Run this command from the parent directory where you have the requirements.txt file
   pip install -r requirements.txt
```

- Completed all todo's
- Start the app with `python main.py`

## Create an Azure VMSS
A bash script setup-script.sh has been provided to automate the creation of the VMSS. You should not need to modify this script.
``` bash
   # Log in to Azure using 
   az login 
   az account list --query "[?isDefault == \`true\`]" # query your current subscription
   # you can set your default subscription with
   az account set -s <subscription id or name>
   # Create a VMSS and related resources. 
   # It uses cloud-init.txt file while running the command "az vmss create" available in the setup-script.sh  
   # The cloud-init.txt will install and start the nginx server (a load balancer) and a few Python packages. 
   chmod +x setup-script.sh # you can confirm that is executable <ls -l setup-script.sh>
   ./setup-script.sh
```
The script above will take a few minutes to create VMSS and related resources

## Application Insights & Log Analytics
- Create an Application Insights resource from [Azure Portal](https://portal.azure.com/#create/Microsoft.AppInsights). It will automatically create a Log Analytics workspace in addition.

- Enable Application Insights monitoring for the VM Scale Set. Make sure to choose the same Log Analytics workspace that you've created in the step above. The Insights deployment will take 10-15 minutes.

- To collect the Logs and Telemetry data, add the reference Application Insights to main.py and specify the instrumentation key. You will need to provide details about the Logging, Metrics, Tracing, and Requests. In addition, add custom event telemetry when 'Dogs' is clicked and when 'Cats' is clicked. Refer to the TODO comments in the main.py file for more details.

> Note that in the main.py file, the configuration related to the Redis Connection will differ in case of deployment to VMSS instance versus deployment to AKS cluster. Rest all code will be the same. Therefore, at this point, you can push your changes to a new branch (say "Deploy_to_VMSS") in your remote so that you can clone it directly inside your VMSS instance. Use the commands like:

``` bash
# Create a new branch locally
git checkout -b Deploy_to_VMSS
# Add, Commit, and Push your changes to the remote
git add -A     
git commit -m "Initial commit for Deploy_to_VMSS branch"
git push --set-upstream origin Deploy_to_VMSS
# git branch --set-upstream-to=origin/Deploy_to_VMSS Deploy_to_VMSS
```
## Deploy to VMSS
- Deploy the application to one of the VMSS instances. Login to one of the VMSS instances, and deploy the application manually.

``` bash
   # Find the port for connecting via SSH 
   az vmss list-instance-connection-info \
      --resource-group deletenow \
      --name bayurzx-vmss 
   # The following command will connect you to your VM. 
   # Replace `[public-ip]` with the public-ip address of your VMSS.
   ssh -p 50000 bayurzx@52.152.143.39
```

- Once you log in to one of the VMSS instances, deploy the application manually:

``` bash
# Clone locally
git clone https://github.com/Bayurzx/udacity-project4.git
cd udacity-project4
# Make sure, you aer in the master branch
git checkout Deploy_to_VMSS

# Update sudo
sudo apt update

# Install Python 3.7
sudo apt install python3.7
# Install pip
sudo -H pip3 install --upgrade pip

# Install and start Redis server. Refer https://redis.io/download for help. 
cd ~
wget https://download.redis.io/releases/redis-6.2.4.tar.gz
tar xzf redis-6.2.4.tar.gz # unzip file
cd redis-6.2.4
make # starts the installation
# Ping your Redis server to verify if it is running. It will return "PONG"
redis-cli ping
# The server will start after make. Otherwise, use
src/redis-server

# __Clone and__ navigate inside the project repo. We need the Flask frontend code
# Install dependencies - necessary Python packages - redis, opencensus, opencensus-ext-azure, opencensus-ext-flask, flask
pip install -r requirements.txt
# Run the app
python3 main.py
```

## Autoscaling VMSS
- Get your vmss
``` bash
az vmss list-instance-connection-info \
  --resource-group deletenow \
  --name bayurzx-vmss
```

- For the VM Scale Set, create an autoscaling rule based on metrics.
*We will use a max count of 4 because if some policy defined for my subscription*
``` bash
# To enable autoscale on a scale set, you first define an autoscale profile
# It sets the default, and minimum, capacity of 2 VM instances, and a maximum of 10
az monitor autoscale create \
  --resource-group deletenow \
  --resource bayurzx-vmss \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale \
  --min-count 2 \
  --max-count 4 \
  --count 2
```

**Create a rule to autoscale out**

```bash
az monitor autoscale rule create \
  --resource-group deletenow \
  --autoscale-name autoscale \
  --condition "Percentage CPU > 70 avg 2m" \
  --scale out 3
```

**Create a rule to autoscale in**
``` bash
az monitor autoscale rule create \
  --resource-group deletenow \
  --autoscale-name autoscale \
  --condition "Percentage CPU < 30 avg 2m" \
  --scale in 1
```

**Generate CPU load on scale set**
- Trigger the conditions for the rule, causing an autoscaling event.
``` bash
ssh bayurzx@52.152.143.39 -p 50000
sudo apt-get update
# install stress and monitor with top or htop
sudo apt-get -y install stress
sudo stress --cpu 10 --timeout 420 &
top
Ctrl-c
exit

# do the same thing for second vm
ssh bayurzx@52.152.143.39 -p 50001
sudo apt-get -y install stress
sudo stress --cpu 10 --timeout 420 &

```

- Watch your VM instances in your scale set. It takes 5 minutes for the autoscale rules to begin the scale-out process
``` bash
watch az vmss list-instances \
  --resource-group deletenow \
  --name bayurzx-vmss \
  --output table
```

- When complete, enable manual scale.

---

# We will continue with `AKS` in another branch called `Deploy_to_AKS`
- Checkout to a new branch
`git checkout -b Deploy_to_AKS`

- Run `docker-compose up -d --build` to build your new image
  - MAke sure you are in `docker-compose.yaml` directory
  - You can also run `docker system prune -a --volumes` to clear all unwanted images not in use
- View the app locally at `http://localhost:8080/`
- Stop the application
`docker-compose down`
- Check if the frontend application is up and running 
``` bash
docker exec -it azure-vote-front bash
ls
```
- Check if the Redis server is running
``` bash
docker exec -it azure-vote-back bash
redis-cli ping # returns pong
```

- Create the AKS cluster
``` bash
   # In your terminal run the following
   az login
   # this to make sure its executable
   chmod +x create-cluster.sh
   # The script below will create an AKS cluster, Configure kubectl to connect to your Kubernetes cluster, and Verify the connection to your cluster
   ./create-cluster.sh
```
> *Make sure to edit the `create-cluster.sh` for non-cloud-lab* 
> Link to cluster is [here](https://github.com/Bayurzx/udacity-project4/blob/master/create-cluster.sh)

# create a Container Registry in Azure to store the image, and AKS can later pull them during deployment to the AKS cluster. Feel free to change the ACR name in place of myacr202109 below.
``` bash
   # Assuming the deletenow resource group is still available with you
   # ACR name should not have upper case letter
   az acr create --resource-group deletenow --name myacr202109 --sku Basic
   # Log in to the ACR
   az acr login --name myacr202109
   # Get the ACR login server name
   # To use the azure-vote-front container image with ACR, the image needs to be tagged with the login server address of your registry. 
   # Find the login server address of your registry
   az acr show --name myacr202109 --query loginServer --output table
   # Associate a tag to the local image. You can use a different tag (say v2, v3, v4, ....) everytime you edit the underlying image. 
   docker tag azure-vote-front:v1 myacr202109.azurecr.io/azure-vote-front:v1
   # Now you will see myacr202109.azurecr.io/azure-vote-front:v1 if you run "docker images"
   # Push the local registry to remote ACR
   docker push myacr202109.azurecr.io/azure-vote-front:v1
   # Verify if your image is up in the cloud.
   az acr repository list --name myacr202109 --output table
   # Associate the AKS cluster with the ACR
   az aks update -n bayurzx-cluster -g deletenow --attach-acr myacr202109

```
# Now, deploy the images to the AKS cluster:
``` bash
   # Get the ACR login server name
   az acr show --name myacr202109 --query loginServer --output table
   # Make sure that the manifest file *azure-vote-all-in-one-redis.yaml*, has `myacr202109.azurecr.io/azure-vote-front:v1` as the image path.  
   # Deploy the application. Run the command below from the parent directory where the *azure-vote-all-in-one-redis.yaml* file is present. 
   kubectl apply -f azure-vote-all-in-one-redis.yaml
   # Test the application at the External IP
   # It will take a few minutes to come alive. 
   kubectl get service azure-vote-front --watch
   # You can also verify that the service is running like this
   kubectl get service
   # Check the status of each node
   kubectl get pods
   # Push your local changes to the remote Github repo, preferably in the Deploy_to_AKS branch

```
If `kubectl apply -f azure-vote-all-in-one-redis.yaml doesn't work` try running
`az aks get-credentials --resource-group deletenow --name bayurzx-cluster` again

## Autoscaling AKS Cluster
- Once the deployment is completed, go to Insights for the cluster. Observe the state of the cluster. Note the number of nodes and the number of containers.

- Create an alert in Azure Monitor to trigger when the number of pods increases over a certain threshold.

- Create an autoscaler by using the following Azure CLI command
``` bash
# kubectl autoscale deployment azure-vote-front --cpu-percent=70 --min=1 --max=10
kubectl autoscale deployment azure-vote-front --cpu-percent=70 --min=1 --max=4
```
  - *We will be using a max of four it seems my azure for students subscription has a restrictive policy for autoscaling*

- Cause load on the system. After approximately 10 minutes, stop the load.

>``` bash
># This didn't cause that much load, so ignore!!! 
>for ((i=1;i<=100;i++)); do   curl -v --header "Connection: keep-alive" "20.185.72.112"; done
>```
  - we create a new deployment with a yaml file called [infinity-call.yaml](https://github.com/Bayurzx/udacity-project4/blob/master/infinity-call.yaml)
    - it basically a [busybox](https://en.wikipedia.org/wiki/BusyBox) that runs a command to keep calling our website
    - run `kubectl get deployments` to confirm its running
    - This is automatically cause load on our cluster
  - After 10 mins run `kubectl scale deploy deployments-simple-deployment-deployment --replicas=0` to scale it to zero this will stop the load
    - Increase replicas to 1 to restart it

- Observe the state of the cluster. Note the number of pods; it should have increased and should now be decreasing.
  - You can observe continuously it with `kubectl get hpa -w`
    - `HPA` means *Horizontal Pod Autoscaling* it only works for deployments set to autoscale `kubectl autoscale deployment`

## Runbook
- Create an Azure Automation Account

- Create a Runbook (Python or Powershell)—either using a script or the UI—that will remedy a problem. It's your choice to mark any event as a "problem", such as average CPU% or memory utlliization crossing a certain threshold is a problem. In this case, think creatively, what you would like the Runbook to do after the average CPU% utlliization of the VMSS crosses a certain threshold. There are numerous options to choose from - vertical/horizontal scale the VMSS, start an existing VM, or any other remedy you choose.
  - Create a RunBook with some powershell scripts
  
``` powershell
workflow Scale-out-VMSS
{
  Param
  (   
    [Parameter(Mandatory=$true)]
    [String]
    $mySubscriptionId,
    [Parameter(Mandatory=$true)]
    [String]
    $myResourceGroup,
    [Parameter(Mandatory=$true)]
    [String]
    $myScaleSet,
    [Parameter(Mandatory=$true)]
    [String]
    $myLocation,
  )
  # define parameters
  # $mySubscriptionId = "8064143e-2180-4b36-89ab-484cbf066722"
  # $myResourceGroup = "deletenow"
  # $myScaleSet = "bayurzx-vmss"
  # $myLocation = "East US"

  # create a scale out rule
  $myRuleScaleOut = New-AzureRmAutoscaleRule `
    -MetricName "Percentage CPU" `
    -MetricResourceId /subscriptions/$mySubscriptionId/resourceGroups/$myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/$myScaleSet `
    -TimeGrain 00:01:00 `
    -MetricStatistic "Average" `
    -TimeWindow 00:05:00 `
    -Operator "GreaterThan" `
    -Threshold 70 `
    -ScaleActionDirection "Increase" `
    -ScaleActionScaleType "ChangeCount" `
    -ScaleActionValue 3 `
    -ScaleActionCooldown 00:05:00
}
```
>Check out [powershell-sample-enable-autoscale](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/scripts/powershell-sample-enable-autoscale)

- Causes the RunBook to be automatically triggered and resolve a problem.
  - Simply follow the *Generate CPU load on scale set* section

- In Azure automation, create an alert rule. An alert rule has the following components:

  - Scope: The target resource you wish to monitor.
  - Condition (or Event) that triggers the alert: For example, whenever the average CPU% utlliization of the target resource is greater than 5%, the automation alert should trigger.
  - Action to be taken: Execute the Runbook to remedy the problem.
  - Cause the problem, such as increase the load, to the flask app on the VM Scale Set.

- Verify the problem is remedied via the Runbook.

