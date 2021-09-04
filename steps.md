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
- Create an Application Insights resource. It will automatically create a Log Analytics workspace in addition.

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

# Create an Azure RunBook to be executed by an Azure Automation Account.
- Create a RunBook with some powershell scripts
  
``` powershell
# define parameters
$mySubscriptionId = "8064143e-2180-4b36-89ab-484cbf066722"
$myResourceGroup = "deletenow"
$myScaleSet = "bayurzx-vmss"
$myLocation = "East US"

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

```
- Configure an Azure Alert to trigger the RunBook to execute.
  - Create a new alert
  - Select notification type
  - Create an action group
    - Remember to set action type of Automation Runbook and use a user defined `Runbook SOurce`
  - Create

- Cause the RunBook to be automatically triggered and resolve a problem.
  - Simply follow the *Generate CPU load on scale set* section


# We will continue with `AKS` in another branch called `Deploy_to_AKS`
