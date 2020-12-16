# Create a loadbalancer with 2 VMs.

## Set the variables:


```
RG="433-8a4eaf4e-create-a-standard-load-balancer-with"
LOC="eastus"
NSG="nsg1-if37m"
VNET="lab-VNet1"
SNET="default"
```

## Create NIC 

```
az network nic create \
--resource-group $RG \
--location $LOC \
--name myNicVM1 \
--vnet-name $VNET \
--subnet $SNET \
--network-security-group $NSG

az network nic create \
--resource-group $RG \
--location $LOC \
--name myNicVM2 \
--vnet-name $VNET \
--subnet $SNET \
--network-security-group $NSG
```

## Create the cloud-init.txt to install the requirements by default


```
#cloud-config
package_upgrade: true
packages:
  - nginx
  - nodejs
  - npm
write_files:
  - owner: www-data:www-data
  - path: /etc/nginx/sites-available/default
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection keep-alive;
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
  - owner: azureuser:azureuser
  - path: /home/azureuser/myapp/index.js
    content: |
      var express = require('express')
      var app = express()
      var os = require('os');
      app.get('/', function (req, res) {
        res.send('Hello World from host ' + os.hostname() + '!')
      })
      app.listen(3000, function () {
        console.log('Hello world app listening on port 3000!')
      })
runcmd:
  - service nginx restart
  - cd "/home/azureuser/myapp"
  - npm init
  - npm install express -y
  - nodejs index.js

```

Create both VMS, new public IP, LB

```
az vm create \
--resource-group $RG \
--location $LOC \
--name myVM1 \
--nics myNicVM1 \
--image UbuntuLTS \
--generate-ssh-keys \
--custom-data cloud-init.txt \
--zone 1 \
--no-wait

az vm create \
--resource-group $RG \
--location $LOC \
--name myVM2 \
--nics myNicVM2 \
--image UbuntuLTS \
--generate-ssh-keys \
--custom-data cloud-init.txt \
--zone 2 \
--no-wait

az network public-ip create \
--resource-group $RG \
--location $LOC \
--name myPublicIP \
--sku Standard

az network lb create \
--resource-group $RG \
--location $LOC \
--name myLoadBalancer \
--sku Standard \
--public-ip-address myPublicIP \
--frontend-ip-name myFrontEnd \
--backend-pool-name myBackEndPool

az network lb probe create \
--resource-group $RG \
--lb-name myLoadBalancer \
--name myHealthProbe \
--protocol tcp \
--port 80

az network lb rule create \
--resource-group $RG \
--lb-name myLoadBalancer \
--name myHTTPRule \
--protocol tcp \
--frontend-port 80 \
--backend-port 80 \
--frontend-ip-name myFrontEnd \
--backend-pool-name myBackEndPool \
--probe-name myHealthProbe \
--disable-outbound-snat true
```

## Get the NIC name from Azure Portal

```
NIC="<NIC_NAME>"
```

## Join both VM to the loadbalancer
```
az network nic ip-config address-pool add \
--address-pool myBackEndPool \
--ip-config-name ipconfig1 \
--nic-name myNicVM1 \
--resource-group $RG \
--lb-name myLoadBalancer

az network nic ip-config address-pool add \
--address-pool myBackEndPool \
--ip-config-name ipconfig1 \
--nic-name myNicVM2 \
--resource-group $RG \
--lb-name myLoadBalancer
```

## Get the publicIP for the loadbalancer
```
az network public-ip show \
--resource-group $RG \
--name myPublicIP \
--query [ipAddress] \
--output tsv
```