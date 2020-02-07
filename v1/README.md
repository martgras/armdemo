# Deploy a Dual Stack MQTT Server in Azure using an ARM template - Version 1
All deployments are done with nested deployments because we either need the the output of a deployment for another deployment or because we update resources in a different resourcegroup

[createresources](https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L66-L622)
this template creates the network resources, storage and the VM.
in outputs the managed identity is returned. createrbac uses the value 

[getpubip](https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L623-L700)
Used by updatedns to retrieve the IP Addresses assigned to the publicIPs.
Currently public IPv6 Addresses only support the Dynamic allocation mode. That means that we get the IP Address after the publicIP resource has been assigned (to a NIC, LB…) 
Because the LoadBalancer resource needs the publicIP to exists during deployment the public IPs are created before the Loadbalancer. When the deployment is complete all publicIPs are connected to a resource and have a valid IP Address
This show how a nested deployment can be used like a function to return information from a already deployed resource

[updatedns](https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L701-L783)
Creates A and AAA records in the provided DNS Zone.
If you have permissions the DNS Zone can be in a different subscription but in the same tenant. 

[createrbac](https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L784-L834): 
Assigns the network-contributor role to the NSG Firewall Rules
The reason is that the VM only needs to open port 80 while the certificate from Letsencrpyt is requested/renew. Therefore, the VM will open the ports during this process and then close it again 

The order of deployment is 
```
createresources --- getpubip -- updatedns
                +-- createrbac  
```

### 2. Configuring the virtual machine setup
ssh to the VM and follow [How to Install and Secure the Mosquitto MQTT Messaging Broker on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-the-mosquitto-mqtt-messaging-broker-on-ubuntu-18-04-quickstart) :smile:

A better option is defining the configuation in our ARM template. One possibility is the Custom Script Extension for Linux. 
A more portable and comfortable option is levering cloud-init.
Cloud-init allows you to supply a initial configuation (in Yaml format) that is applied when the VM is first booted. See ...
We are supporting [cloud-init for Azure Linux VM](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/using-cloud-init). 
You pass the content of the cloud-init file to the customData property in the ARM template. 

Unfortuantly you cannot embedd the content directly. It has to be converted to a one-line string. (Replace all crlf or lf's with \n and esscape any quotes - see)
(BTW: A nice trick is leveraging the #include statement in cloud-config where you can include parts from URL)

See the cloud-init documentation to learn about the configuation settings 
Essentially all updates are installed. The required packages are installed. And files are created on the host: mosquitto config and 2 shellscripts that open/close port 80. 
Then az cli setup is started followed by apt cleanup (remove packages that are not required anymore)
certbot then requests the certificate for this VM from Letsencrypt. After that port 80 on the Azure Firewall is closed.
A cron task to renew the certificat before expiry (60 days) is created automatically.
Finally, the VM reboots if required 
Note $host,$fqdn ,$rg - these placeholder are replaced by the ARM template with the actually values 

See ["[replace(replace(replace('#cloud-config\nhostname: $host\n...','$fqdn',variables('fqdn')),'$rg',variables('rg')),'$host',parameters('vmName'))]"](https://github.com/martgras/armdev/blob/ce2cee59a3261c4899f5ba789de88c08dc325e0c/mqtt/v1/azuredeploy.json#L146)


````YAML
#cloud-config
hostname: $host
package_upgrade: true
packages:
  - curl
  - apt-transport-https
  - lsb-release
  - gnupg
  - mosquitto
  - mosquitto-clients
  - certbot
write_files:
  - owner: root:root
    path: /etc/mosquitto/conf.d/default.conf
    content: |
      allow_anonymous true
      password_file /etc/mosquitto/passwd

      listener 1883 localhost

      listener 8883
      certfile /etc/letsencrypt/live/$fqdn/cert.pem
      cafile /etc/letsencrypt/live/$fqdn/chain.pem
      keyfile /etc/letsencrypt/live/$fqdn/privkey.pem

      listener 8083
      protocol websockets
      certfile /etc/letsencrypt/live/$fqdn/cert.pem
      cafile /etc/letsencrypt/live/$fqdn/chain.pem
      keyfile /etc/letsencrypt/live/$fqdn/privkey.pem
  - owner: root:root
    path: /etc/letsencrypt/renewal-hooks/pre/enablehttp.sh
    content: |
      az login --identity
      az network nsg rule update -g $rg --nsg-name mqttNsg --name HTTP --access Allow
      sleep 30

  - owner: root:root
    path: /etc/letsencrypt/renewal-hooks/post/disablehttp.sh
    content: |
      az login --identity
      az network nsg rule update -g $rg --nsg-name mqttNsg --name HTTP --access Deny
      systemctl restart mosquitto
runcmd:
  - chmod +x /etc/letsencrypt/renewal-hooks/pre/enablehttp.sh
  - chmod +x /etc/letsencrypt/renewal-hooks/post/disablehttp.sh
  - curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
  - apt autoremove -y
  - certbot certonly --non-interactive --agree-tos -m $email --standalone --preferred-challenges http -d $fqdn
  - /etc/letsencrypt/renewal-hooks/post/disablehttp.sh
  - if [ -f /var/run/reboot-required ]; then reboot ; fi
````



**Note**

The customer-data value  for cloud-init must be a one liner - which is a pain to work with directly 
Therefore I'm using powershell to convert dualstack.yaml to single line value suited for JSON and copies it to the clipboard

````powershell
Install-Module -Name ClipboardText  ## Powershell has no command for clipboard
$Content = Get-Content -raw -path $cloudinit
$oneline =  $Content.Replace('\','\\').Replace("`r`n","\n").Replace("`n","\n").Replace('"','\"')
Set-ClipboardText $oneline
$oneline

Or all in one line

(Get-Content -raw -path .\dualstack.yaml).replace('\','\\').Replace("`r`n","\n").Replace("`n","\n").Replace('"','\"')  | Set-ClipboardText ; Get-ClipboardText
````


**Don't forget to disable SSH when done (Change NSG rule for SSH to Deny)**



**Side node - Be carefull when updating API Versions - things may break**  
>Even when you set the allocation method to *static*, you cannot specify the actual IP address assigned to the *public IP resource*. Instead, it gets allocated from a pool of available IP addresses in the Azure location the resource is created in.
>
>While testing this template I wanted to check if my API Version are up to date. Not that it matters but I updated them anyways and after some time I noticed that my 'createresources' template failed 
> 
>The error is Resource Microsoft.ManagedIdentity/Identities 'default' failed with message '{  "error": {
>    "code": "ResourceNotFound",
>    "message": "The Resource 'Microsoft.Compute/virtualMachines/mg-mqtt1' under resource group 'rg-mqttv1' was not found."
>  }
> 
>The error happened after I changed the API version for Microsoft.Resources/deployments from 2017-05-10 to  2019-08-01. Once I reverted to the older version the error went away.
>
>ARM has changed the behaviors of extension resources in template deployment since API version 2019-05-01. This changed how "reference" function works on >extension resources too. In short, you need to update your template and use this pattern when you reference an MSI resource
>
>`"tenantId": "[reference(concat('Microsoft.Compute/virtualMachineScaleSets/',  variables('vmName')), variables('vmApiVersion'), 'Full').Identity.tenantId]"`

>Essentially the differences are:
>- you reference the VM resource instead of the MSI
>- use an api-version for VM
>- use 'Full' option in reference
>- get the "Identity" as a property in the VM resource
>
>The MSI is an implicit resource created with VM and ARM template deployment couldn't build dependency on it. If you "reference" the MSI resource, ARM assume >the resource exists there outside your template, and it tries to GET it immediately when the deployment starts. At that time, the VM was not created yet, and >you get a 404. When you "reference" the VM resource instead, ARM will wait till VM is deployed (with MSI) before it tries to "GET" the VM and retrieve the >"identity" property.
>
>The better fix was using the latest API Version and switching from 
````JSON
"outputs": {
  "principalId": {
  "type": "string",
  "value": "[reference(concat(resourceId('Microsoft.Compute/virtualMachines', parameters('vmName')),'/providers/Microsoft.ManagedIdentity/Identities/default'),'2015-08-31-PREVIEW').principalId]"
  }
}
````
>to
````JSON
"outputs": {
  "principalId": {
  "type": "string",
  "value": "[reference(concat('Microsoft.Compute/virtualMachines/', parameters('vmName')),'2019-07-01','Full').identity.principalId]"
  }
}
````
>

### Deploy the template 




````powershell
New-AzResourceGroup -Location francecentral -Name rg-mqttv1

$vmpwd = ConvertTo-SecureString -AsPlainText -Force  'vjui.mIuzi.921!753'
$adminPublicKey=ConvertTo-SecureString -AsPlainText -Force 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCGsxMlYySzXjwmkX6JWi/LZ+RVXgXzkr+cfGGG3ENgPY3G3Zk4x31ODnPU/CHecy6fSdT8Fdge8hik1n/GHoHtalCp3jYyL2cX59427dc6mwCA8o5ovbWe2bSqdzEffApOILerFsHmw+mHreHStAMk5PJO7gauk9Zfr++71tveGKTqNMZbL5eJfiHRTT+S05v7tJYNPzNiQjVyr8JgDdrisbHrnhr5NE4h2Y1xg6Mfon9qxuOnjEs2ny090SUpgI6iAIHjxV8mX2vQcLSa/xyu+IRzrdWH2fY7zhjKHIdVn+wSZtaee/TblDEmD2/qO9V2YrSOmqopkdfpVoXR1skR'

New-AzResourceGroupDeployment -Name "deployMqttv1" -ResourceGroupName rg-mqttv1 -TemplateUri 'https://raw.githubusercontent.com/martgras/armdemo/master/v1/azuredeploy.json' `
 -adminUsername mqttboss `
 -adminPassword $vmpwd  `
 -adminPublicKey $adminPublicKey `
 -vmName mg-mqtt1  `
 -hostname mgqtt1  `
 -dnszone-resid /subscriptions/89303ea5-a12f-1234-5654-1234566788aa/resourceGroups/rg-demo/providers/Microsoft.Network/dnszones/demo.contoso.com  `
 -letsencypt-account 'someone@contoso.com'


DeploymentName          : deployMqttv1
ResourceGroupName       : rg-mqttv1
ProvisioningState       : Succeeded
Timestamp               : 2/6/2020 2:11:31 PM
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                  Type                       Value
                          ====================  =========================  ==========
                          adminUsername         String                     mqttboss
                          adminPassword         SecureString
                          adminPublicKey        SecureString
                          vmName                String                     mg-mqtt1
                          hostname              String                     mgqtt1
                          letsencypt-account    String                     someone@contoso.com
                          dnszone-resid         String                     /subscriptions/89303ea5-a12f-1234-5654-1234566788aa/resourceGroups/rg-demo/providers/Microsoft.Network/dnszones/demo.contoso.com
                          location              String                     francecentral
                          vmSize                String                     Standard_B1s
````

*Note*:  this template needs a ssh public key. The deployment works with the sample key given here but of course can't be used to sign in because you don't have a private key for it. See [Create and manage SSH keys for authentication to a Linux VM in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed)


### Validate the deployment


#### Watch the cloud-init steps on the Serial Console 
If you open the VM directly after the deployment is completed and select "Serial Console" you can see how the cloud-init script is processed (takes ~5 minutes)


#### Test the DNS records **
````bash
ping -4 mgqtt1.demo.contoso.com
Pinging mgqtt1.demo.contoso.com [52.143.187.46] with 32 bytes of data:

ping -6 mgqtt1.demo.contoso.com
Pinging mgqtt1.demo.contoso.com [2603:1020:800::26] with 32 bytes of data:
````

#### Mqtt setup 

##### Linux

If you are using Linux or WSL  you can use mosquitto_pub and mosquitto_sub 
The --capath parameter is required to tell the mosquitto clients to use TLS for the connection
depending on your distribution the location of the certificate files may vary

Subscribe to the broker 
```sh
  mosquitto_sub -h mgqtt1.demo.contoso.com -t '#' -d -u user1 -P YTQWs9qtPgeTiyDaQLDI --capath /etc/ssl/certs/
```

Publish a message
```sh
mosquitto_pub -h mgqtt1.demo.contoso.com -t 'test' -m "Hi there" -u user1 -P YTQWs9qtPgeTiyDaQLDI --capath /etc/ssl/certs/
```

##### Windows

For Windows MQTT Explorer is a nice tools - you can download the portable (no setup required) version from https://github.com/thomasnordquist/MQTT-Explorer/releases/download/v0.3.5/MQTT-Explorer-0.3.5.exe
You can also download it from the [Microsoft Store](https://www.microsoft.com/store/apps/9PP8SFM082WD?ocid=badge)

##### Android / IoS

I am using the MQTT broker with [OwnTracks](https://owntracks.org/booklet/) on my phone to send a message when I'm leaving my house to turn off lights and heating.

There are a couple of MQTT clients in Googles PlayStore or Apples App Store 



## Challenge 
 * Create a keyvault and get the values for adminPassword and adminPublicKey from vault secrets  
   Hint : You only have to modify/use a template paramater file 
*  Configure OwnTracks to send a message when you leave a defined Zone   





## Next step
[Jump to version 2](../v2/README.md)
