# Terraform Template to create Multiple Instance of ASAv in a single location

## Preinstallation

First step is to set up the Linux server to install Terrraform and Azure CLI.

- Install Terraform on Ubuntu : [Doc](https://www.terraform.io/docs/cli/install/apt.html)

- Install Azure CLI on Ubuntu : [Doc](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt)

Make sure to login to Azure using below command to use Azure as a provider in Terraform template

`az login --use-device-code`

The template has been tested on :

- Terraform Version =  0.14.7
- Hashicorp AzureRM Provider = 2.56.0

Using this Terraform template, n instances of ASA will be deployed in Azure based on the user requirement.

- one new VPC with four subnets (1 Management networks, 2 data networks)
- Routing table attachment to each of these subnets.
- Public IP attachment to the Management subnet.
- One External and One Internal Load Balancer.

The variables should be defined with a value in the `terraform.tfvars` file before using the templates.

### Variables

| Variable | Meaning |
| --- | --- |
| `location = "centralindia"` | Resorce group Location in Azure |
| `prefix = "cisco-ASAv"` | This would prefix all the component with this string |
| `source-address = "*"` | Limit the Management access to specific source IPs |
| `IPAddressPrefix = "10.10"` | All the IP Address segment will use this as prefix with .0,.1,.2 and .3 as the 3rd octet |
| `Version = "915.1.1"` | ASA Version to be deployed - Please validate the correct version using - `az vm image list --offer asav --all` |\
| `VMSize = "Standard_D3_v2"` | Size of the ASAv to be deployed |
| `RGName = "cisco-ASAv-RG"` | Resource Group Name |
| `instancename = "cisco-ASAv"` | Instance Name and properties of ASAv |
| `instances = 2` | Number of ASAvs to be deployed |
| `username = "cisco"` | Username to login to ASA |
| `password = "P@$$w0rd1234"` | Password to login to ASA |

## Deployment Procedure

1) Clone or Download the Repository
2) Input the values in the terraform ".tfvars" for variables in .tf
3) Initialize the providers and modules, go to the specific terraform folder from the cli

    ```bash
        cd xxxx
        terraform init 
    ```

4) submit the terraform plan

    ```bash
       terraform plan -out <filename>
    ```

5) verify the output of the plan in the terminal; if everything is fine, then apply the plan

    ```bash
        terraform apply <out filename generated earlier>
    ```

6) Check the output and confirm it by typing "yes."

## Post Deployment Procedure

1) SSH to the instance by using ssh cisco@PublicIP

2) configure each of the interface based on the subnet the interface belongs to

    An example is attached below,

    ```asa
        !
        interface GigabitEthernet0/0
        no shutdown
        nameif dmz
        security-level 50
        ip address dhcp
        !
        interface GigabitEthernet0/1
        no shutdown
        nameif outside
        security-level 100
        ip address dhcp
        !
    ```

3) Configure ACL to the outside interface

    ```asa
        access-list testacl extended permit ip any any
        access-group testacl in interface outside
        !
    ```

4) Configure the routes and enable ssh
  
    ```asa
        route outside 0.0.0.0 0.0.0.0 10.10.1.1 3
        route inside 10.10.4.0 255.255.255.0 10.10.2.1 3
        route outside 168.63.129.16 255.255.255.255 10.10.1.1 3
        route inside 168.63.129.16 255.255.255.255 10.10.2.1 4
        !
        ! For probe to check health status
        ssh 0.0.0.0 0.0.0.0 outside
        ssh 0.0.0.0 0.0.0.0 inside
        !
    ```