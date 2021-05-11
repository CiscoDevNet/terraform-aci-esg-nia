# Cisco ACI Endpoint Security Group module for Network Infrastructure Automation (NIA)

This Terraform module allows users to dynamically create and update **Cisco ACI** Endpoint Security Group by leveraging [Consul](https://www.consul.io/) catalog information.

The **Cisco ACI** fabric can act as a distributed stateless segmentation enforcement device sitting in front of any pool of workloads, regardless of their form-factor. For this module to work, the user should have deployed a Tenant template with Application Profile and VRF. For more information on how to deploy Cisco ACI Endpoint Security Group, please refer to the configuration guide and this [white paper](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/5-x/security/cisco-apic-security-configuration-guide-50x/m-endpoint-security-groups.html?dtid=osscdc000283).

Using this Terraform module in conjunction with **consul-terraform-sync** enables administrators to automatically manage ESGs and ESG selectors based on Consul services definitions and dynamic updates without having to manually reconfigure **Cisco ACI** policies.

#### Note: This Terraform module is designed to be used only with **consul-terraform-sync**.


## Feature
This module supports the following:
* Create and Update Endpoint Security Groups (**fvESg**).
* Create, Update and Delete Endpoint Security Group Selector (**fvEPSelector**).

If there is a missing feature or a bug - [open an issue ](https://github.com/CiscoDevNet/terraform-aci-esg-nia/issues/new).

## Caveats
* All Consul services part of the same "task" must be defined in the same ACI VRF.

## What is consul-terraform-sync?

The **consul-terraform-sync** runs as a daemon that enables a **publisher-subscriber** paradigm between **Consul** and **Cisco ACI** to support **Network Infrastructure Automation (NIA)**. 

* consul-terraform-sync **subscribes to updates from the Consul catalog** and executes one or more automation **"tasks"** with appropriate value of *service variables* based on those updates. **consul-terraform-sync** leverages [Terraform](https://www.terraform.io/) as the underlying automation tool and utilizes the Terraform provider ecosystem to drive relevant change to the network infrastructure. 

* Each task consists of a runbook automation written as a compatible **Terraform module** using resources and data sources for the underlying network infrastructure provider.

Please refer to this [link](https://www.consul.io/docs/nia) for getting started with **consul-terraform-sync**.

## Requirements

| Name | Version |
|------|---------|
| [terraform](https://www.terraform.io/downloads.html) | >= 0.13 |
| [consul-terraform-sync](https://www.consul.io/docs/nia/installation/install) | >= 0.1.0 |
| [consul](https://www.consul.io/downloads) | >= 1.7 |

Links refer to download and installation/instructions for each component.
## Providers

| Name | Version |
|------|---------|
| aci | >= 0.6.0 |  


## Compatibility
This module is meant for use with **consul-terraform-sync >= 0.1.0**, **Terraform >= 0.13** and **Cisco ACI >= 5.0**.

## Usage
In order to use this module, you will need to install **consul-terraform-sync**, create a **"task"** with this Terraform module as a source within the task, and run **consul-terraform-sync**.

The users can subscribe to the services in the consul catalog and define the Terraform module which will be executed when there are any updates to the subscribed services using a **"task"**.

**~> Note:** It is recommended to have the [consul-terraform-sync config guide](https://www.consul.io/docs/nia) for reference.  
1. Download the **consul-terraform-sync** on a node which is highly available (preferably, a node running a consul client).
2. Add **consul-terraform-sync** to the PATH on that node.
3. Check the installation.
   ```
   $ consul-terraform-sync --version
   0.1.0
   Compatible with Terraform ~>0.13.0
   ```
 4. Create a config file **`tasks.hcl`** for consul-terraform-sync. Please note that this just an example. 
 
```terraform
log_level = "debug"

consul {
  address = "172.23.189.57:8500"
}

driver "terraform" {
  log = true
  required_providers {
    aci = {
    source =  "CiscoDevNet/aci"
    version = "0.6.0"
    }
  }
}

provider "aci" {
  alias = "aci1" 
  username = "admin"
  url  = "https://172.31.186.4"
  private_key = "./terraform.key"
  cert_name = "terraform"
}

buffer_period {
  min = "5s"
  max = "20s"
}

task {
  name = "cts-svc-esg"
  description = "Automatically Scale ACI ESGs"
  source = "/Users/anvjain/nia/terraform-nia/terraform-aci-esg-nia"
  providers = ["aci.aci1"]
  services = ["web"]
  variable_files = [ "/Users/anvjain/nia/input.tf"]
}
```
 5. Fill the **`inputs.tf`** file with the required module input and place it in the same directory as **`tasks.hcl`**. Currently the user must specify the **Cisco ACI** Tenant and the the Application Profile where the ESG must be created, the name of the VRF in the same tenant to be associated with the created ESGs and as well as an Endpoint Security Group name prefix. You can use the example below.
 ```terraform
tenant_name              = "common"
application_profile_name = "nia_ap"
vrf_name                 = "default"
esg_prefix               = "nia"
```
 6. Start consul-terraform-sync.
```
$ consul-terraform-sync -config-dir <path_to_configuration_directory>
```
**consul-terraform-sync** will create the appropriate policies in accordance to the Consul catalog.

**consul-terraform-sync** is now subscribed to the Consul catalog. Any updates to the services identified in the task will result in updating the **Cisco ACI** Endpoint Security Group and ESG Selectors.


**~> Note:** If you are interested in how **consul-terraform-sync** works, please refer to this [section](#how-does-consul-terraform-sync-work).




## Inputs:

| Name | Description | Type | Default | Required |
|------|-------------------------------------|------|---------|:--------:|
| aci\_tenant | Cisco ACI Tenant name, e.g., prod_tenant. | `string` | - | yes |
| aci\_application\_profile | Cisco ACI Application Profile name, e.g., prod_ap. | `string` | - | yes |
| aci\_vrf | Cisco ACI VRF name, e.g., prod_vrf. | `string` | - | yes |
| esg\_prefix | Prefix for the Endpoint Security Group that is created when the first service instance is declared in the Consul catalog. The format is `<prefix>-<service-name>-svc` | `string` | - | no |
| services | Consul services monitored by consul-terraform-sync | <pre>map(<br>    object({<br>      id        = string<br>      name      = string<br>      address   = string<br>      port      = number<br>      meta      = map(string)<br>      tags      = list(string)<br>      namespace = string<br>      status    = string<br><br>      node                  = string<br>      node_id               = string<br>      node_address          = string<br>      node_datacenter       = string<br>      node_tagged_addresses = map(string)<br>      node_meta             = map(string)<br>    })<br>  )</pre> | n/a | yes |


## How does consul-terraform-sync work?

There are 2 aspects of consul-terraform-sync.
1. **Updates from Consul catalog:**
In the backend, consul-terraform-sync creates a blocking API query session with the Consul agent indentified in the config to get updates from the Consul catalog.
consul-terraform-sync.
consul-terraform-sync will get an update for the services in the consul catalog when any of the following service attributes are created, updated or deleted. These updates include service creation and deletion as well.
   * service id
   * service name
   * service address
   * service port
   * service meta
   * service tags
   * service namespace
   * service health status
   * node id
   * node address
   * node datacenter
   * node tagged addresses
   * node meta

   
2. **Managing the entire Terraform workflow:**
If a task and is defined, one or more services are associated with the task, provider is declared in the task and a Terraform module is specified using the source field of the task, the following sequence of events will occur:
   1. consul-terraform-sync will install the required version of Terraform.
   2. consul-terraform-sync will install the required version of the Terraform provider defined in the config file and declared in the "task".
   3. A new direstory "nia-tasks" with a sub-directory corresponding to each "task" will be created. This is the reason for having strict guidelines around naming.
   4. Each sub-directory corresponds to a separate Terraform workspace. 
   5. Within each sub-directory corresponding a task, consul-terraform-sync will template a main.tf, variables.tf, terraform.tfvars and terraform.tfvars.tmpl.
      * **main.tf:**
         * This files contains declaration for the required terraform and provider versions based on the task definition. 
         * In addition, this file has the module (identified by the 'source' field in the task) declaration with the input variables.
         * Consul K/V is used as the backend state for fo this Terraform workspace.
      
         Example of generated main.tf:
         ```terraform
         # This file is generated by Consul Terraform Sync.
         #
         # The HCL blocks, arguments, variables, and values are derived from the
         # operator configuration for Sync. Any manual changes to this file
         # may not be preserved and could be clobbered by a subsequent update.

         terraform {
            required_version = "~>0.13.0"
            required_providers {
              aci = {
                source  = "CiscoDevNet/aci"
                version = "0.4.1"
              }
            }
            backend "consul" {
              address = "172.23.189.57"
              gzip    = true
              path    = "consul-terraform-sync/terraform"
            }
          }
          
          provider "aci" {
            cert_name   = var.aci.cert_name
            private_key = var.aci.private_key
            url         = var.aci.url
            username    = var.aci.username
          }
      
         # Automatically Scale ACI Service ESGs
          module "cts-svc-esg" {
            source   = "/Users/anvjain/nia/terraform-aci-esg-nia"
            services = var.services

            application_profile_name = var.application_profile_name
            esg_prefix               = var.esg_prefix
            tenant_name              = var.tenant_name
            vrf_name                 = var.vrf_name
          }
         ```
      * **variables.tf:**
        * This is the variables.tf file defined in the module.
        
         Example of generated variables.tf:
         ```terraform
         #
         # The HCL blocks, arguments, variables, and values are derived from the
         # operator configuration for Sync. Any manual changes to this file
         # may not be preserved and could be clobbered by a subsequent update.

         # Service definition protocol v0
         variable "services" {
          description = "Consul services monitored by Consul NIA"
          type = map(
            object({
              id        = string
              name      = string
              address   = string
              port      = number
              status    = string
              meta      = map(string)
              tags      = list(string)
              namespace = string

              node                  = string
              node_id               = string
              node_address          = string
              node_datacenter       = string
              node_tagged_addresses = map(string)
              node_meta             = map(string)
            })
          )
        }

        variable "tenant_name" {}
        variable "application_profile_name" {}
        variable "vrf_name" {}
        variable "esg_prefix" {}
        ```
      * **terraform.tfvars:**
         * This is the most important file generated by consul-terraform-sync.
         * This variables file is generated with the most updated values from Consul catalog for all the services identified in the task.
         * consul-terraform-sync updates this file with the latest values when the corresponding service gets updated in Consul catalog.
         
         Example of generated terraform.tfvars:
         
         ```terraform
         services = {
          "web-1.ANVJAIN-M-L28H.dc1" : {
            id      = "web-1"
            name    = "web"
            kind    = ""
            address = "10.0.0.162"
            port    = 8080
            meta    = {}
            tags            = ["rails"]
            namespace       = null
            status          = "critical"
            node            = "ANVJAIN-M-L28H"
            node_id         = "6c16cf70-3573-bf40-38b6-ead07f44eab4"
            node_address    = "127.0.0.1"
            node_datacenter = "dc1"
            node_tagged_addresses = {
              lan      = "127.0.0.1"
              lan_ipv4 = "127.0.0.1"
              wan      = "127.0.0.1"
              wan_ipv4 = "127.0.0.1"
            }
            node_meta = {
              consul-network-segment = ""
            }
            cts_user_defined_meta = {}
          }
        }
         ```
      * **Network Infrastructure Automation (NIA) compatible modules are built to utilize the above service variables**.
    6. **consul-terraform-sync** manages the entire Terraform workflow of plan, apply and destroy for all the individual workspaces corrresponding to the defined     "tasks" based on the updates to the services to those tasks.
    
  **In summary, consul-terraform-sync triggers a Terraform workflow (plan, apply, destroy) based on updates it detects from Consul catalog.**
  
