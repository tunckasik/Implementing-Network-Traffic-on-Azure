# Implementing Network Traffic

## Scenario

Instead of creating the mesh topology, I tested the managing network traffic targeting Azure virtual machines in the hub and spoke network topology.
This testing needs to include implementing connectivity between spokes by relying on user defined routes that force traffic to flow via **the hub**, as well as traffic distribution across virtual machines by using Azure Load Balancer (layer 4) and Azure Application Gateway (layer 7).

>**Note**: In the ARM template, I generated a total of 8 vCPUs available in the Standard_Dsv3 series in the East US region.

## Objectives

+ Step 1: Provision the lab environment
+ Step 2: Configure the hub and spoke network topology
+ Step 3: Test transitivity of virtual network peering
+ Step 4: Configure routing in the hub and spoke topology
+ Step 5: Implement Azure Load Balancer
+ Step 6: Implement Azure Application Gateway

## Steps

#### Step 1: Provision the environment with ARM-template

In this step, I deployed four virtual machines. The first two will reside in a hub virtual network, while each of the remaining two will reside in a separate spoke virtual network.

Steps:
1. Created Recource Group with **assign-05-rg1**, while opening Cloud Shell
1. Describing a variable for the Resource Group:
    
    ```powershell
    $rgName = 'assign-05-rg1'
    ```
1. Creating the three virtual networks and four Azure VMs into them by using the ARM template and its parameter files:

   ```powershell
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/assign-5-template.json `
      -TemplateParameterFile $HOME/assign-5-parameters.json
   ```
   >**Note**: The mentioned json files are also added in this repository
   
1. To be able to see our generated topology of the Virtual Networks; on the Cloud Shell pane, I installed the Network Watcher extension on the Azure VMs deployed:

   ```powershell
   $rgName = 'assign-05-rg1'
   $location = (Get-AzResourceGroup -ResourceGroupName $rgName).location
   $vmNames = (Get-AzVM -ResourceGroupName $rgName).Name

   foreach ($vmName in $vmNames) {
     Set-AzVMExtension `
     -ResourceGroupName $rgName `
     -Location $location `
     -VMName $vmName `
     -Name 'networkWatcherAgent' `
     -Publisher 'Microsoft.Azure.NetworkWatcher' `
     -Type 'NetworkWatcherAgentWindows' `
     -TypeHandlerVersion '1.4'
   }
   ```

#### Step 2: Configuring the hub and spoke network topology

In this step, configured local peering between the virtual networks which previously deployed in order to create a hub and spoke network topology.

  > **Note**: The template is used for deployment of the three virtual networks. This information is to keep in mind to do not overlap.
  Vnet-1: 10.60.0.0/22, Vnet-2: 10.62.0.0/22, Vnet-3: 10.63.0.0/22.
   
1. Added a peering with the following settings:

    | Setting | Value |
    | --- | --- |
    | This virtual network: Peering link name | **assign-5-vnet01_to_assign-5-vnet2** |
    | Traffic to remote virtual network | **Allow (default)** |
    | Traffic forwarded from remote virtual network | **Block traffic that originates from outside this virtual network** |
    | Virtual network gateway | **None (default)** |
    | Remote virtual network: Peering link name | **assign-5-vnet2_to_assign-5-vnet01** |
    | Virtual network deployment model | **Resource manager** |
    | I know my resource ID | disabled |
    | Virtual Machine | **assign-5-vnet2** |
    | Traffic to remote virtual network | **Allow (default)** |
    | Traffic forwarded from remote virtual network | **Allow (default)** |
    | Virtual network gateway | **None (default)** |

    > **Note**: This step establishes two local peerings - one from assign-5-vnet01 to assign-5-vnet2 and the other from assign-5-vnet2 to assign-5-vnet01. Same thing done for assign-5-vnet3. By completing setting up the hub and spoke topology (with two spoke virtual networks).

#### Step 3: Tested transitivity of virtual network peering

In this step, I tested transitivity of virtual network peering by using Network Watcher for **assign-5-vm2 (10.62.0.4)** and **assign-5-vm3 (10.63.0.4)**.

Verified that the status for both is **Reachable**. Review the network path and note that the connection was direct, with no intermediate hops in between the VMs.

#### Step 4: Configured routing in the hub and spoke topology

In this step, I configured and tested routing between the two spoke virtual networks by enabling IP forwarding on the network interface of the **assign-5-vm0** virtual machine, enabling routing within its operating system, and configuring user-defined routes on the spoke virtual network.

1. Setting **IP forwarding** to **Enabled**.

   > **Note**: This setting is required in order for **assign-5-vm0** to function as a router, which will route traffic between two spoke virtual networks.

1. Configured operating system of the **assign-5-vm0** virtual machine to support routing.

1. In the Azure portal, navigating to the **assign-5-vm0**, clicking **Run command**, and, in the list of commands, clicking **RunPowerShellScript**.

1. On the **Run Command Script** blade, typing the following to install the Remote Access Windows Server role:

   ```powershell
   Install-WindowsFeature RemoteAccess -IncludeManagementTools
   ```
1. On the **Run Command Script** blade, type the following to install the Routing role service.

   ```powershell
   Install-WindowsFeature -Name Routing -IncludeManagementTools -IncludeAllSubFeature

   Install-WindowsFeature -Name "RSAT-RemoteAccess-Powershell"

   Install-RemoteAccess -VpnType RoutingOnly

   Get-NetAdapter | Set-NetIPInterface -Forwarding Enabled
   ```

1. Generating and configuring user defined routes(UDRs) on the spoke virtual networks.

1. Creating a route table with the following settings:

    | Setting | Value |
    | --- | --- |
    | Subscription | the name of the subscription that used in this lab |
    | Resource group | **assign-05-rg1** |
    | Location | the name of the Azure region in which you created the virtual networks |
    | Name | **assign-5-rt23** |
    | Propagate gateway routes | **No** |

1. After having a route table, adding a new route with the following settings:

    | Setting | Value |
    | --- | --- |
    | Route name | **assign-5-route-vnet2-to-vnet3** |
    | Address prefix destination | **IP Addresses** |
    | Destination IP addresses/CIDR ranges | **10.63.0.0/20** |
    | Next hop type | **Virtual appliance** |
    | Next hop address | **10.60.0.4** |

1. Associating the route table **assign-5-rt23** with the following subnet:

    | Setting | Value |
    | --- | --- |
    | Virtual network | **assign-5-vnet2** |
    | Subnet | **subnet0** |

1. Creating a route table, a new route and associate it as above mentioned. Only changing names and destination IP addrress prefix will be **10.63.0.0/20**

It is time to check connectivity between 'assign-5-vm2' and 'assign-5-vm3' through **Network Watcher - Connection troubleshoot**.
    > **Note**: This is expected, since the traffic between spoke virtual networks is now routed via the virtual machine located in the hub virtual network, which functions as a **router**.

    Here is the topology of the network:

![After-Task-4-Network-Topology](https://user-images.githubusercontent.com/101068723/200074043-2ed71d0b-cc63-47f6-a68f-910fb73537e1.png)


#### Step 5: Implement Azure Load Balancer

In this step, I implemented an Azure Load Balancer in front of the two Azure virtual machines in the hub virtual network.

1. Creating a load balancer with the following settings:

    | Setting | Value |
    | --- | --- |
    | Subscription | the name of the subscription that used in this lab |
    | Resource group | (create new) **assign-05-rg4** |
    | Name | **assign-5-lb4** |
    | Region | name of the Azure region into which you deployed all other resources in this lab |
    | SKU  | **Standard** |
    | Type | **Public** |
	  | Tier | **Regional** |

1. On the **Frontend IP configuration** tab, used the following settings:

    | Setting | Value |
    | --- | --- |
    | Name | any unique name |
  	| IP version | IPv4 |
  	| IP type | IP address |
    | Public IP address | **Create new** |
    | Name | **assign-5-pip4** |
  	| Availability zone | **No Zone** |

1. On the **Backend pools** tab, used the following settings

    | Setting | Value |
    | --- | --- |
    | Name | **assign-5-lb4-be1** |
    | Virtual network | **assign-5-vnet01** |
	  | Backend Pool Configuration | **NIC** |
    | IP Version | **IPv4** |
	  | Click **Add** to add a virtual machine |  |
    | assign-5-vm0 | **check the box** |
    | assign-5-vm1 | **check the box** |


1. On the **Inbound rules** tab, adding a load balancing rule with the following settings:

    | Setting | Value |
    | --- | --- |
    | Name | **assign-5-lb4-lbrule1** |
    | IP Version | **IPv4** |
    | Frontend IP Address | **assign-5-pip4** |
    | Backend pool | **assign-5-lb4-be1** |
	  | Protocol | **TCP** |
    | Port | **80** |
    | Backend port | **80** |
	  | Health probe | **Create new** |
    | Name | **assign-5-lb4-hp1** |
    | Protocol | **TCP** |
    | Port | **80** |
    | Interval | **5** |
    | Unhealthy threshold | **2** |
	  | Close the create health probe window | **OK** |
    | Session persistence | **None** |
    | Idle timeout (minutes) | **4** |
    | TCP reset | **Disabled** |
    | Floating IP | **Disabled** |
	  | Outbound source network address translation (SNAT) | **Recommended** |

#### Step 6: Implement Azure Application Gateway

In this step, I implemented an Azure Application Gateway in front of the two Azure virtual machines in the spoke virtual networks.

1. Adding an additional subnet as the following settings in the Virtual Netwrok called **assign-5-vnet01** :

    | Setting | Value |
    | --- | --- |
    | Name | **subnet-appgw** |
    | Subnet address range | **10.60.3.224/27** |

    > **Note**: This subnet is used by the Azure Application Gateway instances, which I deployed as below step. The Application Gateway requires a dedicated subnet of /27 or larger size.

1. Creating a Application Gateway; on the **Basics** tab with the following settings:

    | Setting | Value |
    | --- | --- |
    | Subscription | the name of the subscription that used in this lab|
    | Resource group | **assign-05-rg1** |
    | Application gateway name | **assign-5-appgw5** |
    | Region | name of the Azure region into which you deployed all other resources in this lab |
    | Tier | **Standard V2** |
    | Enable autoscaling | **No** |
	  | Instance count | **2** |
	  | Availability zone | **None** |
    | HTTP2 | **Disabled** |
    | Virtual network | **assign-5-vnet01** |
    | Subnet | **subnet-appgw (10.60.3.224/27)** |

1. For the Frontends, the following settings:

    | Setting | Value |
    | --- | --- |
    | Frontend IP address type | **Public** |
    | Public IP address| **Add new** | 
	  | Name | **assign-5-pip5** |
	  | Availability zone | **None** |

1. Adding a backend pool, the following settings:

    | Setting | Value |
    | --- | --- |
    | Name | **assign-5-appgw5-be1** |
    | Add backend pool without targets | **No** |
    | IP address or FQDN | **10.62.0.4** | 
    | IP address or FQDN | **10.63.0.4** |

    > **Note**: The targets represent the private IP addresses of virtual machines in the spoke virtual networks **assign-5-vm2** and **assign-5-vm3**.

1. Adding a routing rule in configuration with the following settings:

    | Setting | Value |
    | --- | --- |
    | Rule name | **assign-5-appgw5-rl1** |
    | Priority | **10** |
    | Listener name | **assign-5-appgw5-rl1l1** |
    | Frontend IP | **Public** |
    | Protocol | **HTTP** |
    | Port | **80** |
    | Listener type | **Basic** |
    | Error page url | **No** |

1. In the Backend targets tab added the following settings:

    | Setting | Value |
    | --- | --- |
    | Target type | **Backend pool** |
    | Backend target | **assign-5-appgw5-be1** |
	  | Backend settings | **Add new** |
    | Backend settings name | **assign-5-appgw5-http1** |
    | Backend protocol | **HTTP** |
    | Backend port | **80** |
    | Additional settings | **take the defaults** |
    | Host name | **take the defaults** |

    > **Note**: Targeting virtual machines on multiple virtual networks is not a common configuration, but it is meant to illustrate the point that Application Gateway is capable of targeting virtual machines on multiple virtual networks (as well as endpoints in other Azure regions or even outside of Azure), unlike Azure Load Balancer, which load balances across virtual machines in the same virtual network.


#### Review

With this assignment-5, I learnt:

+ Provisioning the environment with ARM-template
+ Configuring the hub and spoke network topology
+ Testing transitivity of virtual network peering
+ Configuring routing in the hub and spoke topology
+ Implementing Azure Load Balancer
+ Implementing Azure Application Gateway
