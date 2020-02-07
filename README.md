# Introduction 
Demo project to show a few advanced ARM Template Features

The sample templates deploy a dual stack (ipv4/v6) MQTT broker in Azure.  
Of course, it's only meant as a demo and not a blueprint for a productive system.  
Nevertheless, it should create a fully functional mqtt broker that can be used for further experiments. 

## Goal: Create a mqtt broker in Azure.  
I want to connect several IoT devices to a mqtt broker and connect with my on-prem OpenHab server.

### Requirements 
* IPv6 support - many IoT devices are working in a ipv6 environment. 
* TLS support with a public CA provided certificate. 
* Easy to deploy. 
* Minimize cost. 

### Non-Requirements 
Performance and scalability it out of scope here.  
[Vernemq](https://vernemq.com/) is probably a good choice - but there are more. 

## Alternatives  
* Azure IoT Hub is also a mqtt broker but has some limitations. See [Communicate with your IoT hub using the MQTT protocol](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-mqtt-support) 
* Host the mqtt broker on-prem - possible but requires open firewall ports and/or VPN connections and dynamic DNS hassles. 
* [Azure Containers](https://docs.microsoft.com/azure/containers/) : a good choice if you have a higher workload than my 10 devices. 
For a mqtt broker running 24/7 with no load Azure Containers is more expensive than a B1s VM ( ~ 5 Euros per month )

## High level 
* A Standard_B1s VM running Ubuntu Server 18.04 LTS. 
* Mosquitto as the mqtt broker. See https://mosquitto.org/ for detailed information. 
* Automatic certificate deployment from [Letsencrypt](https://letsencrypt.org).


### Steps to configure a VM
1. Set up a Ubuntu 18.04 VM 
2. Create a DNS entry pointing to this VM 
3. Configure Ubuntu
   * Install updates. 
   * Install mosquitto and mosquitto clients. 
   * Install certbot. 
   * Install az cli. 
   * Run certbot to get a TLS certificate from Letsencrypt. 
   * Configure mosquitto to use the certificates. 
   * Create a mosquitto user. 

### 1. VM configuration

You currently can’t create a VM with IPv6 and IPv4 support in the portal. 
The gist is that you need a load balancer because you can't directly attach a public ipv6 address to a NIC (yet). 
The deployment consists of a load balancer that has both a public IPv4 and a Ipv6 address. 
Ports 22,80 and 8883 are routed to the single member of the backend. 
(If more than one instance is desired the cloud-init script must be modified to avoid that each instance requests a new certificate) 

The VM template is based on [Deploy an IPv6 dual stack application with Basic Load Balancer in Azure - Template](https://docs.microsoft.com/en-us/azure/virtual-network/ipv6-configure-template-json)

To support IPv4 and IPv6, a load balancer is used in front of the VM. The load balancer forwards all traffic to a single VM in the backend. 

The VM opens port 80 on demand using az cli, therefore a managed identity is created for the VM and a RBAC assignment for this managed identity is deployed. 


## Prerequisits
* Azure subscription 
* for V1 access to a Azure hosted DNS Zone
* optional - Keyvault secrets for passwords and SSH keys

## Implementation 

There are (currently) 3 variants of the template.  
Compare v1 and v2 to better understand how *copy* and *if* is used to replace parts of the template that are hardcoded in v1

### Version 1

The ARM template is quite straight forward without many advanced features. 
All deployments are done with nested deployments because we either need the output of a deployment for another deployment or because we update resources in a different resourcegroup. It also shows how a nested template is used as a function to return values from a previous deployment.  
[Jump to Version1](v1/README.md)

### Version 2
This version is a bit more flexible. It allows more customization of the deployment through parameters and shows how to use concepts like the copy operator, if and conditionals and how variables can be used to build a parts of resource properties.  
[Jump to Version 2](v2/README.md)

### Version 2 - Linked
This is the V2 version of the template but uses linked templates instead of inline nesting.  
[Jump to code](v2-linked/)

## Challenges 

* Use this template to create a [Managed Application](https://docs.microsoft.com/en-us/azure/azure-resource-manager/managed-applications/)
* Create an Azure Policy that ensures that port 22 is not open to the internet
* Add the [VMAccess extension for Linux](https://github.com/Azure/azure-linux-extensions/tree/master/VMAccess) to the template
