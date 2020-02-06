# Version 2 - Optimize the template

While we now have an ARM template that deploys our server without any manual installation required there is still room for improvement in a few places

## SSH using public keys is the prefered authentication method

In general, it's recommended to disable SSH access via Internet for VM's. Either enable Just in Time access to only allow SSH from a subnet
However, the Standard_B1s size doesn't support Just in Time access.  

Note: Since I use the the WSL subsystem if I connect to a host using SSH I created a simple shell script to change the firewall setting
````
az network nsg rule update -g mqtt-rg --nsg-name mqtt-nsg --name SSH --access $1
````
You can then call the script and pass either "Allow" or "Deny" as the argument

If you still want/have to use SSH directly you should use a publickey for authentication as that is alot safer than passwords

To allow empty parameters I used 
````
     "adminPublicKey":{ 
         "type":"securestring",
         "defaultValue":"",
         "metadata":{ 
            "description":"The public key for the administrator account of the new VM"
         }
      },
````
then define a variable for the publicKeys property 

https://github.com/martgras/armdemo/blob/cab216ed59cf2a079d411541225db5eeefd4a90a/v2/azuredeploy.json#L231-L238

````
"variables" : {
...
  "sshPublicKeys":{ 
  "publicKeys":[ 
      { 
        "path":"[concat('/home/', parameters('adminUsername'), '/.ssh/authorized_keys')]",
        "keyData":"[parameters('adminPublicKey')]"
      }
    ]
  },
````

If the parameter adminPulicKey is not empty ssh password authentication is disabled and we provision the public key instead.

https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v2/azuredeploy.json#L514-L516

```JSON
"linuxConfiguration":{ 
  "disablePasswordAuthentication":"[if(empty(parameters('adminPublicKey')),'true','false')]",
  "ssh":"[if(not(empty(parameters('adminPublicKey'))),variables('sshPublicKeys'),json('[]'))]"
}
```

For an empty the ARM engine will transform it 
```JSON
"linuxConfiguration":{ 
  "disablePasswordAuthentication":"false",
  "ssh":"[]"
}
```
if a key is provided we get 

```JSON
"linuxConfiguration":{ 
  "disablePasswordAuthentication":"false",
  "ssh":" {
      "publicKeys":[ 
      { 
        "path":"[concat('/home/', parameters('adminUsername'), '/.ssh/authorized_keys')]",
        "keyData":"[parameters('adminPublicKey')]"
      }
    ]}"
}
```

## Make DNS options more flexible

Version 1 of the template requires that you have DNS Zone for the domain you want to use here. 
Because Public IPs have the DNS Label property. With that you get a host name in the '<location>.cloudapp.azure.com' zone automatically we can use the template without a DNS Zone at all 
Instead of creating A/AAA records in our own DNS zone we can create a CNAME instead and point it to the DNS name '<location>.cloudapp.azure.com'

The new version of the template supports more options* 

Possible combinations based on the template parameters:
* custom domain Azure hosted
  * domainzone ( required: resourceId of the Azure DNS zone), hostname(r), alternatename (optional)
* custom domain externally hosted
  * domainzone( required: the domainname e.g. demo.contoso.com),hostname(r), alternatename (o)
* no custom domain
   * domainzone(empty), hostname(r),alternatename (o)

````JSON
// if alternatehostname is not empty use it as the DNSLabel for the public IP's else use the hostname paramter
"publicIpHostname": "[if(empty(parameters('alternatehostname') ), parameters('hostname'),parameters('alternatehostname'))]",
// set the variable to the resourceid of the DNSzone if the value contains one or more slashes '/'  else treat the dnsparameter zone pararamer as a domain name (e.g. demo.contoso.com)
"dnszone-resid": "[if(contains(parameters('dnszone'),'/'),parameters('dnszone'),'')]"

// FQDN used as the hostname on the linux VM
"fqdn": "[concat(parameters('hostname'),'.',parameters('dnszone'))]",
// the fqdn for the public IP (e.g. mgqtt2.francecentral.cloudapp.azure.com)
"publicIpFqdn": "[concat(parameters('publicIpHostname'),'.',parameters('location'),'.','cloudapp.azure.com')]",
// all subject names for the certificate requests 
"certificateSubjectNames": "[if(empty(parameters('dnszone')),variables('publicIpFqdn'),concat(variables('fqdn'),',',variables('publicIpFqdn')))]",
````  

>Note: because certbot validates our cert request if we can serve the validation webpage under publicIpFqdn you even get a certificate for the azure.com domain.

## Support public key authentication and passwords

For enhanced security publickey auhtentication is recommended instead of password. 
A new paramater 'adminPublicKey' is introduced. If it is not empty publicKey authentication is used for SSL and password login via SSL is disabled. 
You can still use the password to logon from the serial console 

To allow empty parameters I used 
```
  "adminPublicKey": {
            "type": "securestring",
            "defaultValue" :  "",
            "metadata": {
                "description": "The public key for the administrator account of the new VM"
            }
```
define a variable for the publicKeys property 
```
"variables" : {
...
  "publicKeys": [
  {
    "path": "[concat('/home/', parameters('adminUsername'), '/.ssh/authorized_keys')]",
    "keyData": "[parameters('adminPublicKey')]"
    }
  ]
}
```
the variable is then used to configure ssh

```
  "osProfile": {
      "computerName": "[parameters('vmName')]",
      "adminUsername": "[parameters('adminUserName')]",
      "adminPassword": "[parameters('adminPassword')]",
      "customData": "[variables('cloudinit')]",
      "linuxConfiguration": {
        "disablePasswordAuthentication": "[if(empty(parameters('adminPublicKey')),'true','false')]",
        "ssh": "[if(not(empty(parameters('adminPublicKey'))),variables('sshPublicKeys'),json('[]'))]"
      }
```

Using variables to customize the deployment is a very powerfull pattern that you will see more om this template

Reference: https://azure.microsoft.com/en-us/blog/create-flexible-arm-templates-using-conditions-and-logical-functions/

## Improve Loadbalancer and NSG rule creation

Another area for improvment are the loadbalancer rules and the NSGs. it's pretty repetive code and needs to be changed in 3 places when changing ports.
Azure copy operations allow a lot more flexibility
[Using a property object in a copy loop](https://docs.microsoft.com/en-us/azure/architecture/building-blocks/extending-templates/objects-as-parameters#using-a-property-object-in-a-copy-loop)

For this project I have to create NSG rules that open the required ports and Loadbalancer rules forwarding the same ports from frontend to backend. 

In v1 this was hardcoded (see https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L241-L359 and https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L392-L480 )

### NSG rules

The new approach relies on this parameter definition

https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v2/azuredeploy.json#L78-L103

```JSON
 "tcpports":{ 
 "type":"array",
 "defaultValue":[ 
  { 
    "port":"22",
    "access":"Deny",
    "name":"SSH"
  },
  { 
     "port":"80",
     "access":"Allow",
     "name":"HTTP"
  },  

 }
 ```

We will use this array of objects for 2 purposes:  Create the NSG rules and the Loadbalancer rules 

https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v2/azuredeploy.json#L239-L258


Again, a variable is used to get most of it done. 
Here "copy" is used to create an array of NSG rules

For more details about "copy" see [Deploy multiple instances of resources - Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/create-multiple-instances)

```JSON
"variables" : {
  "copy":[ 
      { 
        "name":"securityRulesIn",
        "count":"[length(parameters('tcpports'))]",
        "input":{ 
            "name":"[parameters('tcpports')[copyIndex('securityRulesIn')].name]",
            "properties":{ 
              "description":"[parameters('tcpports')[copyIndex('securityRulesIn')].name]",
              "protocol":"Tcp",
              "sourcePortRange":"*",
              "destinationPortRange":"[parameters('tcpports')[copyIndex('securityRulesIn')].port]",
              "sourceAddressPrefix":"*",
              "destinationAddressPrefix":"*",
              "access":"[parameters('tcpports')[copyIndex('securityRulesIn')].access]",
              "priority":"[add(1000,copyIndex('securityRulesIn'))]",
              "direction":"Inbound"
            }
        }
      }
  ],
``` 
What we do here is creating a variables called securityRulesIn 
Then the number of iterations is determined by get the number of elements in the tcpports array. 
Everying within the input element will be processes in the loop. copyIndex('securityRulesIn') is the current value of the counter
In a tradition programming language, its roughly the equivalent of 

````csharp
var securityRulesIn[length(tcpports)]
for (copyIndex = 0 ; copyIndex < length('tcpports')) {
  var newRule
  newRule.name = tcpports[copyIndex].name
  newRule.properties.description =  tcpports[copyIndex].name
  newRule.properties.destionationPortRange =  tcpports[copyIndex].port
  ...
  securityRulesIn.Add ( newRule)
}
````
I need one additional NSG rule for outgoing requests IpV6 requests
After I declared that variable, I combine both arrays using concat

https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v2/azuredeploy.json#L278

```JSON
  "securityRules":"[concat(variables('securityRulesIn'),variables('securityRulesAdditional'))]"
```

We now have all required rules in this variable and can use it in the resource definition 
Because all the hard work was done when constuction the variables this part is very short. We just inject the array of rules

https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v2/azuredeploy.json#L413

```JSON
{ 
    "apiVersion":"2019-11-01",
    "type":"Microsoft.Network/networkSecurityGroups",
    "name":"mqttNsg",
    "location":"[parameters('location')]",
    "properties":{ 
      "securityRules":"[variables('securityRules')]"
    }
},
````

### Loadbalancer Rules

The Loadbalancer rules require a bit more work because we need 2 entries for each port on for Ipv4 and one for Ipv6
See the v1 version to understand what we are going to create https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v1/azuredeploy.json#L241-L359

*Caveat* - I wouldn't consider the following as best practise because it's not that easy to understand without explanation. The main purpose is showing what can be done. 

Before we go into the JSON details it makes sense to explain the 'algorythm' in pseudo code 

````csharp
// we need 12 entries let's loop 2 times the length of the input array
var ipversions[2] = [ "-v4","-v6" ]
var lbfront[2] = ["LBFE-v4","LBFE-v6"]
var lbback[2] = ["LBBAP-v4","LBBAP-v6"]
for (copyIndex = 0 ; copyIndex < 2 * length(tcpports);copyIndex++ )
{
     var soureIndex = copyIndex / 2 ;   // the goal is getting the same index for 2 iterations 
     var ip4voripv6 = copyIndex mod 2  // alternates between 0 and 1  so we get ipv4 if copyIndex is an even numer and 1 if copyIndex is odd
    frontendIPConfiguration.id    =  lbfront[ip4voripv6]"
    backendAddressPool.id    =  lbback[ip4voripv6]
    frontendport = tcpPorts [sourceIndex].port
    backendport = tcpPorts [sourceIndex].port
    name = "lbrule"  + str(tcpPorts [sourceIndex]) + ipversions[ip4voripv6]  // example lbrule80-v4
}
````

and now in JSON 
https://github.com/martgras/armdemo/blob/653fd7f719e5de4fa900acab05965a867c4d85f5/v2/azuredeploy.json#L366-L385

```JSON
"copy":[ 
    { 
      "name":"loadBalancingRules",
      "count":"[mul(length(parameters('tcpports')),2)]",
      "input":{ 
          "properties":{ 
            "frontendIPConfiguration":{ 
                "id":"[resourceId('Microsoft.Network/loadBalancers/frontendIpConfigurations', 'loadBalancer', variables('lbfront')[mod(copyIndex('loadBalancingRules'),2)])]"
            },
            "backendAddressPool":{ 
                "id":"[resourceId('Microsoft.Network/loadBalancers/backendAddressPools', 'loadBalancer', variables('lbback')[mod(copyIndex('loadBalancingRules'),2)])]"
            },
            "protocol":"Tcp",
            "frontendPort":"[parameters('tcpports')[div(copyIndex('loadBalancingRules'),2)].port]",
            "backendPort":"[parameters('tcpports')[div(copyIndex('loadBalancingRules'),2)].port]"
          },
          "name":"[concat('lblRule',parameters('tcpports')[div(copyIndex('loadBalancingRules'),2)].name,variables('ipversions')[mod(copyIndex('loadBalancingRules'),2)])]"
      }
    }
]
```

### Deploy the template

Because we are going to assign a domain label to the public IP Addresses first check if the names are available

````powershell
Test-AzDnsAvailability -DomainNameLabel mgqtt1 -Location Francecentral
False

martgras> Test-AzDnsAvailability -DomainNameLabel mgqtt2  -Location Francecentral
True

New-AzResourceGroupDeployment -Name "deployMqttv2" -ResourceGroupName rg-mqttv2 -TemplateUri 'https://raw.githubusercontent.com/martgras/armdemo/master/v2/azuredeploy.json' `
 -adminUsername mqttboss `
 -adminPassword $vmpwd `
 -adminPublicKey $adminPublicKey `
 -vmName mg-mqtt2 `
 -hostname mgqtt2 `
 -dnszone /subscriptions/89303ea5-a12f-1234-5654-1234566788aa/resourceGroups/rg-demo/providers/Microsoft.Network/dnszones/demo.contoso.com `
 -letsencypt-account 'someone@contoso.com' `

DeploymentName          : deployMqttv2
ResourceGroupName       : rg-mqttv2
ProvisioningState       : Succeeded
Timestamp               : 2/6/2020 2:58:27 PM
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                  Type                       Value
                          ====================  =========================  ==========
                          adminUsername         String                     mqttboss
                          adminPassword         SecureString
                          adminPublicKey        SecureString
                          vmName                String                     mg-mqtt2
                          vmSize                String                     Standard_B1s
                          hostname              String                     mgqtt2
                          alternatehostname     String
                          dnszone               String                     /subscriptions/89303ea5-a12f-1234-5654-1234566788aa/resourceGroups/rg-demo/providers/Microsoft.Network/dnszones/demo.contoso.com
                          letsencypt-account    String                     someone@contoso.com
                          mqttpwd               SecureString
                          location              String                     francecentral
                          tcpports              Array                      [
                            {
                              "port": "22",
                              "access": "Deny",
                              "name": "SSH",
                              "sourceAddressPrefix": "CorpNetPublic"
                            },
                            {
                              "port": "80",
                              "access": "Allow",
                              "name": "HTTP"
                            },
                            {
                              "port": "8883",
                              "access": "Allow",
                              "name": "MQTT-TLS"
                            },
                            {
                              "port": "8083",
                              "access": "Allow",
                              "name": "MQTT-WebSocket"
                            }
                          ]

Outputs                 :
````

### Validate the deployment

#### Test the DNS records **

We created a DNSprefix and supplied a DNS zone therefore we should have 2 DNS names to access the VM mgqtt2.demo.contoso.com and mgqtt2.francecentral.cloudapp.azure.com

````bash
ping -4 mgqtt2.demo.contoso.com
Pinging mgqtt2.demo.contoso.com [40.89.147.208] with 32 bytes of data:

ping -6 mgqtt2.demo.contoso.com
Pinging mgqtt2.demo.contoso.com [2603:1020:800::27] with 32 bytes of data:

ping -4 mgqtt2.francecentral.cloudapp.azure.com
Pinging mgqtt2.francecentral.cloudapp.azure.com [40.89.147.208] with 32 bytes of data:

ping -6 mgqtt2.francecentral.cloudapp.azure.com
Pinging mgqtt2.francecentral.cloudapp.azure.com [2603:1020:800::27] with 32 bytes of data:
````

#### Validate the TLS certifcate

https://www.sslshopper.com/ssl-checker.html#hostname=mgqtt2.francecentral.cloudapp.azure.com:8883

The test should succeed and the certificate should have 2 subject names:  
`Common Name:mgqtt2.demo.contoso.com`  
`SANs:mgqtt2.demo.contoso.com, mgqtt2.francecentral.cloudapp.azure.com`  
`Validity:From 06/02-2020 to 06/05-2020`


## Use Linked templates

For small to medium solutions, a single template is easier to understand and maintain. You can see all the resources and values in a single file. For advanced scenarios, linked templates enable you to break down the solution into targeted components. You can easily reuse these templates for other scenarios.

The disadvantage of linked templates is that you can deploy them from an external location - the policy engine must be able to access the template via HTTP(S). 
While this is great once your templates work as expected development is a bit more complicated because you must replicate your changes to the external location. 
I recommend hosting the template in a github repository you can commit and push changes directly from VSCode.
Be careful not to include confidential data like VM Admin passwords in a public github repo)

[Jump to linked templates](../v2-linked/azuredeploy.json)


