# f5-awaf-lab-on-azure

This is a small step by step guide on how to build a F5 AWAF (Advanced Web Application Firewall) lab environment on Microsoft Azure. The purpose of this guide is to provide an easy way for quickly spin up a lab environment which can be used for study or demo purposes.

*Note: The guide was written to be easily followed from any Linux distro. However all of linux-specific commands can be easily converted to other operating systems like Windows or Mac.*

## Overview

This is high-level view about he steps performed by this guide:

1. Create a *Resource Group* ; 
2. Create a *VNET* and 3 *Subnets* on it (management,internal,external) ;
3. Create a *NSG* (with some *rules*) which will be applied to the "vulnerables-apps" VM ;  
4. Create a *Virtual Machine* named **vulnerable-apps**, install the VM *Docker Extension*, and then run on it the script *vulnerable-apps.sh*, which will deploy 4 vulnerables apps (available as container images);
5. Copy all the F5 ARM template files needed to deploy the BIG-IP to the current directory ;
6. Create an *Azure Storage Account* and a *container* which will host the *runtime-init* config file and the *AS3* declaration file (which will be used to deploy the 4 vulnerables apps on BIG-IP);
7. Adjust the *Runtime Init* config file (configure the license key and the AS3 declaration URL):
8. Upload the *Runtime Init* config file and the *AS3* declaration file to the Azure *container* created previously ;
9. Adjust the ARM template parameters file (configure the subnets IDs, SSH key, access restrictions, the URL for the *Runtime Init* config file);
10. Create a new *Resource Group* (which will be used for the BIG-IP deployment) ;
11. Create a BIG-IP *Deployment* using the F5 ARM template files (we will be deploying a Standalone BIG-IP with 3-NICs using BYOL); 
11. Access the newly deployed BIG-IP system and change the admin's password ; 
12. Log in the BIG-IP *Configuration Utility* and check all *Virtual Servers* ; 
13. Access all the vulnerables apps deployed on BIG-IP;

## Vulnerable Apps

These are the vulnerable apps which will be deployed in the lab environment (with the credentials):

1. JuiceShop (credentials should be discovered, this is one of the challenges);
2. DVWA (admin/password);
3. Hackazon (admin/hackmesilly);
4. WebGoat (credentials available on the login page);

## Setting up the lab

1. Create some files on the *secrets* directory:

    ```
    echo "YOUR LICENSE KEY" > secrets/license_key.txt
    echo "YOUR SUBSCRIPTION NAME" > secrets/subscription_name.txt
    ```

2. Generate a SSH key pair (which will be used to access the BIG-IP system):

    ```
    ssh-keygen -f secrets/mykey
    ```

3. Configure some environment variables:

    ```
    export RESOURCEGROUP="f5-awaf-lab"          
    export LOCATION="eastus"             
    export MYIP=$(curl api.ipify.org)
    export SUBSCRIPTION=$(cat secrets/subscription_name.txt)
    export SUBSCRIPTIONID=`az account subscription list --subscription "$SUBSCRIPTION" --query "[0].subscriptionId" 2>/dev/null | tr -d "\""`
    export MYKEY=$(cat secrets/mykey.pub)
    ```

4. Create the main resource group:

    ```
    az group create --location $LOCATION --name $RESOURCEGROUP --subscription "$SUBSCRIPTION"
    ```

5. Create a VNET: 

    ```
    az network vnet create --name $RESOURCEGROUP-vnet --resource-group $RESOURCEGROUP --address-prefixes "10.10.0.0/16"
    ```

6. Create three subnets: 

    ```
    az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $RESOURCEGROUP-vnet --name management --address-prefixes 10.10.1.0/24 --service-endpoints Microsoft.Storage
    az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $RESOURCEGROUP-vnet --name external --address-prefixes 10.10.2.0/24 
    az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $RESOURCEGROUP-vnet --name internal --address-prefixes 10.10.3.0/24
    ```

7. Create a *NSG* (which will be attached to "vulnerable-apps" VM):

    ```
    az network nsg create --name vulnerable-apps-nsg --resource-group $RESOURCEGROUP
    ```

8. Create some *NSG rules*:

    ```
    az network nsg rule create --resource-group $RESOURCEGROUP --nsg-name vulnerable-apps-nsg --name "SSH" --source-address-prefixes $MYIP --destination-address-prefixes '*' --destination-port-ranges 22 --access Allow --protocol Tcp --priority 100
    az network nsg rule create --resource-group $RESOURCEGROUP --nsg-name vulnerable-apps-nsg --name "juice-shop" --source-address-prefixes $MYIP --destination-address-prefixes '*' --destination-port-ranges 9000 --access Allow --protocol Tcp --priority 101
    az network nsg rule create --resource-group $RESOURCEGROUP --nsg-name vulnerable-apps-nsg --name "dvwa" --source-address-prefixes $MYIP --destination-address-prefixes '*' --destination-port-ranges 9001 --access Allow --protocol Tcp --priority 102
    az network nsg rule create --resource-group $RESOURCEGROUP --nsg-name vulnerable-apps-nsg --name "hackazon" --source-address-prefixes $MYIP --destination-address-prefixes '*' --destination-port-ranges 9002 --access Allow --protocol Tcp --priority 103
    az network nsg rule create --resource-group $RESOURCEGROUP --nsg-name vulnerable-apps-nsg --name "webgoat" --source-address-prefixes $MYIP --destination-address-prefixes '*' --destination-port-ranges 9003 --access Allow --protocol Tcp --priority 104
    ```

9. Create the "vulnerable-apps" VM:

    ```
    az vm create --resource-group $RESOURCEGROUP --name vulnerable-apps --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --nsg vulnerable-apps-nsg --vnet-name $RESOURCEGROUP-vnet --subnet internal --public-ip-sku Standard --private-ip-address 10.10.3.50 --os-disk-delete-option delete --nic-delete-option delete
    ```

10. Get the public IP of "vulnerable-apps" VM:

    ```
    export VULNAPPS_IP=`az vm list-ip-addresses --resource-group $RESOURCEGROUP --name vulnerable-apps -o json --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" | tr -d "\""`
    ```

11. Install the *Docker Extension* on the "vulnerable-apps" VM:

    ```
    az vm extension set --vm-name vulnerable-apps --resource-group $RESOURCEGROUP --name DockerExtension --publisher Microsoft.Azure.Extensions --version 1.1
    ```

12. Deploy the vulnerables apps on the "vulnerables-apps" VM:

    ```
    az vm run-command invoke --resource-group $RESOURCEGROUP --name vulnerable-apps --command-id RunShellScript --scripts "@vulnerable-apps.sh"
    ```

13. Check whether all vulnerables apps were deployed correctly:

    ```
    ssh azureuser@$VULNAPPS_IP "docker ps"
    ```

14. Copy all the F5 ARM template files needed to deploy the BIG-IP (we will be deploying a Standalone BIG-IP with 3-NICs using BYOL):

    ```
    cp f5-azure-arm-template/azuredeploy-existing-network.* .
    cp f5-azure-arm-template/bigip-configurations/runtime-init-conf-3nic-byol.yaml .
    ```

15. Create some environment variables (which will be used to deploy the F5 ARM template):

    ```
    export LICENSEKEY=$(cat secrets/license_key.txt)
    export UNIQUESTRING="bigip-$RANDOM"
    export STORAGEACCOUNT=`echo $UNIQUESTRING | md5sum | sed 's/.//25g'`
    export SUBNET_MANAGEMENT_ID=`az network vnet subnet list --resource-group $RESOURCEGROUP --vnet-name $RESOURCEGROUP-vnet --query "[?name=='management']" | jq ".[] | .id" | tr -d "\""`
    export SUBNET_EXTERNAL_ID=`az network vnet subnet list --resource-group $RESOURCEGROUP --vnet-name $RESOURCEGROUP-vnet --query "[?name=='external']" | jq ".[] | .id" | tr -d "\""`
    export SUBNET_INTERNAL_ID=`az network vnet subnet list --resource-group $RESOURCEGROUP --vnet-name $RESOURCEGROUP-vnet --query "[?name=='internal']" | jq ".[] | .id" | tr -d "\""`
    ```

16. Create an Azure *Storage Account* (with *Default Action* of **Deny**):

    ```
    az storage account create --name $STORAGEACCOUNT --resource-group $RESOURCEGROUP --location $LOCATION --sku Standard_ZRS --encryption-services blob --default-action Deny
    ```

17. Allow access to the Azure *Storage Account* only from the *management* subnet (and from our own Public IP):

    ```
    az storage account network-rule add --resource-group $RESOURCEGROUP --account-name $STORAGEACCOUNT --ip-address $MYIP

    az storage account network-rule add --resource-group $RESOURCEGROUP --account-name $STORAGEACCOUNT --vnet-name $RESOURCEGROUP-vnet --subnet management
    ```

18. Give the appropriate permission to the *Storage Account*:

    ```
    az ad signed-in-user show --query objectId -o tsv | az role assignment create --role "Storage Blob Data Contributor" --assignee @- --scope "/subscriptions/$SUBSCRIPTIONID/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Storage/storageAccounts/$STORAGEACCOUNT"
    ```

19. Create an Azure *Storage Account Container* (which will host the *Runtime Init* config file and the *AS3* declaration file): 

    ```
    az storage container create --account-name $STORAGEACCOUNT --name config --resource-group $RESOURCEGROUP-bigip --auth-mode login --public-access blob
    ```

20. Put your license key in the *Runtime Init* config file:

    ```
    sed -i "s/XXXXX-XXXXX-XXXXX-XXXXX-XXXXXXX/$LICENSEKEY/" runtime-init-conf-3nic-byol.yaml
    ```

21. Put the *AS3* declaration URL in the *Runtime Init* config file:

    ```
    sed -i "s/AS3DECLARATIONURL/https:\/\/$STORAGEACCOUNT.blob.core.windows.net\/config\/vulnerable-apps.json/" runtime-init-conf-3nic-byol.yaml
    ```

22. Check the *Runtime Init* config file:

    ```
    less runtime-init-conf-3nic-byol.yaml
    ```

23. Upload the *Runtime Init* config file to the *storage container* previously created:

    ```
    az storage blob upload --account-name $STORAGEACCOUNT --container-name config --name runtime-init-conf-3nic-byol.yaml --file runtime-init-conf-3nic-byol.yaml --auth-mode login
    ```

24. Upload the *AS3* declaration file to the *storage container* previously created:

    ```
    az storage blob upload --account-name $STORAGEACCOUNT --container-name config --name vulnerable-apps.json --file vulnerable-apps.json --auth-mode login
    ```

25. Configure the ARM template *parameters* file (*azuredeploy-existing-network.parameters.json*):

    ```
    cat azuredeploy-existing-network.parameters.json | jq ".parameters.restrictedSrcAddressApp.value |= \"$MYIP\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json

    cat azuredeploy-existing-network.parameters.json | jq ".parameters.restrictedSrcAddressMgmt.value |= \"$MYIP\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json

    cat azuredeploy-existing-network.parameters.json | jq ".parameters.sshKey.value |= \"$MYKEY\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json

    cat azuredeploy-existing-network.parameters.json | jq ".parameters.bigIpRuntimeInitConfig.value |= \"https://$STORAGEACCOUNT.blob.core.windows.net/config/runtime-init-conf-3nic-byol.yaml\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json
   
    cat azuredeploy-existing-network.parameters.json | jq ".parameters.uniqueString.value |= \"$UNIQUESTRING\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json

    cat azuredeploy-existing-network.parameters.json | jq ".parameters.bigIpExternalSubnetId.value |= \"$SUBNET_EXTERNAL_ID\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json

    cat azuredeploy-existing-network.parameters.json | jq ".parameters.bigIpInternalSubnetId.value |= \"$SUBNET_INTERNAL_ID\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json

    cat azuredeploy-existing-network.parameters.json | jq ".parameters.bigIpMgmtSubnetId.value |= \"$SUBNET_MANAGEMENT_ID\"" > temp.json && mv temp.json azuredeploy-existing-network.parameters.json
    ```
26. Check the ARM template parameters file: 

    ```
    less azuredeploy-existing-network.parameters.json
    ```

27. Accept the Azure Marketplace "License/Terms and Conditions":

    ```
    az vm image terms accept --urn f5-networks:f5-big-ip-byol:f5-big-all-2slot-byol:16.1.000000
    ```

28. Create a new resource group for the BIG-IP deployment:

    ```
    az group create --location $LOCATION --name $RESOURCEGROUP-bigip --subscription "$SUBSCRIPTION"    
    ```


29. Create the *Deployment* (which will deploy the BIG-IP system):

    ```
    az deployment group create --resource-group $RESOURCEGROUP-bigip --name $RESOURCEGROUP-bigip --template-file azuredeploy-existing-network.json --parameters "@azuredeploy-existing-network.parameters.json"
    ```
    **Note:** The deployment could take up to 10 min.

30. Get the BIG-IP management Public IP:

    ```
    export BIGIP=`az network public-ip show --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-mgmt-pip0 -o json --query "ipAddress" | tr -d "\""`
    ```

31. Log in the BIG-IP and change the admin's password:

    ```
    ssh -i secrets/mykey admin@$BIGIP
    ```
    ```
    modify /auth user admin password "F5training@123"
    ```
    ```
    save sys config
    ```
    ```
    quit
    ```
32. Log in the BIG-IP Configuration Utility and check the configuration (*echo $BIGIP*):

    ![Virtual Server List](https://github.com/pedrorouremalta/f5-awaf-lab-on-azure/blob/master/images/f5-awaf-lab-on-azure-01.png)

    ![Pool List](https://github.com/pedrorouremalta/f5-awaf-lab-on-azure/blob/master/images/f5-awaf-lab-on-azure-02.png)

33. Create some Public IPs (one for each vulnerable app):

    ```
    az network public-ip create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-juiceshop --sku Standard
    
    az network public-ip create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-dvwa --sku Standard
    
    az network public-ip create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-hackazon --sku Standard
    
    az network public-ip create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-webgoat --sku Standard
    ```

34. Get the Public IPs created for each vulnerable app:

    ```
    export JUICESHOP_PUBLIC_IP=`az network public-ip show --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-juiceshop -o json --query "ipAddress" | tr -d "\""`

    export DVWA_PUBLIC_IP=`az network public-ip show --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-dvwa -o json --query "ipAddress" | tr -d "\""`

    export HACKAZON_PUBLIC_IP=`az network public-ip show --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-hackazon -o json --query "ipAddress" | tr -d "\""`

    export WEBGOAT_PUBLIC_IP=`az network public-ip show --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-pip-webgoat -o json --query "ipAddress" | tr -d "\""`
    ```

35. Create some *IP configurations* on the external NIC of the BIG-IP system (which will create one private address for each vulnerable app and associate it with their respective public IP created previously):

    ```
    az network nic ip-config create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-ipconfig-juiceshop --nic-name $UNIQUESTRING-nic1 --private-ip-address 10.10.2.20 --resource-group $RESOURCEGROUP-bigip --public-ip-address $UNIQUESTRING-external-pip-juiceshop

    az network nic ip-config create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-ipconfig-dvwa --nic-name $UNIQUESTRING-nic1 --private-ip-address 10.10.2.21 --resource-group $RESOURCEGROUP-bigip --public-ip-address $UNIQUESTRING-external-pip-dvwa

    az network nic ip-config create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-ipconfig-hackazon --nic-name $UNIQUESTRING-nic1 --private-ip-address 10.10.2.22 --resource-group $RESOURCEGROUP-bigip --public-ip-address $UNIQUESTRING-external-pip-hackazon

    az network nic ip-config create --resource-group $RESOURCEGROUP-bigip --name $UNIQUESTRING-external-ipconfig-webgoat --nic-name $UNIQUESTRING-nic1 --private-ip-address 10.10.2.23 --resource-group $RESOURCEGROUP-bigip --public-ip-address $UNIQUESTRING-external-pip-webgoat 
    ``` 

36. Access the vulnerables apps:

    ```
    echo -e "\n\nJUICESHOP - http://$JUICESHOP_PUBLIC_IP/\nDVWA - http://$DVWA_PUBLIC_IP/\nHACKAZON - http://$HACKAZON_PUBLIC_IP/\nWEBGOAT - http://$WEBGOAT_PUBLIC_IP/WebGoat\n\n"
    ```

##  Cleaning up the lab

1. Log in the BIG-IP, revoke the license and quit (so the license can be used again later):

    ```
    ssh -i secrets/mykey admin@$BIGIP
    ```
    ```
    revoke /sys license
    ```
    ```
    quit
    ```

2. Delete the BIG-IP deployment and the resource group:

    ```
    az deployment group delete --resource-group $RESOURCEGROUP-bigip --name $RESOURCEGROUP-bigip
    az group delete --name $RESOURCEGROUP-bigip --yes
    ```

3. Delete the main resource group:

    ```
    az group delete --name $RESOURCEGROUP --yes
    ```

4. Delete some files:

    ```
    rm azuredeploy-existing-network.json
    rm azuredeploy-existing-network.parameters.json
    rm runtime-init-conf-3nic-byol.yaml
    rm secrets/mykey secrets/mykey.pub
    ```