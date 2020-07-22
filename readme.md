# Install an App Service with a Private Endpoint with outbound traffic routed through a Firewall and inbound traffic coming through an App Gateway.

## Run the Terraform script
In the /terraform folder are a selection of tf files.  You will need to have terraform installed (I am running v0.12.24 on Powershell Core 7).
run `terraform init` in a powershell prompt and then run the [setup powershell script here](./terraform/setup.ps1), this script runs the `terraform apply` and then deploys the sample app to the website.
The resource prefix and the location are defined as [variables](./terraform/variables.tf) and you should provide new values (particularly for the prefix). 
The resources can be cleaned up by running `terraform destroy`

## What is the script doing?

![What gets deployed!](/diagrams/What%20gets%20deployed.png "What gets deployed")
The script will create 
- 2 resource groups
- 2 virtual networks (peered)
- 4 subnets
- A storage account with table storage accessible via a private endpoint
- A private DNS zone to allow resolution of the storage private endpoint
- An App Service (Website + plan) running a simple node application, public access is restricted through the firewall, network access is allowed through a private endpoint.  (Note: private endpoints for web apps are currently in preview and limited to Premium V2 App Service plans)
- A Firewall that all outbound network traffic is routing through
- An App Gateway that all inbound traffic to the app service is routed through.

## How does the inbound traffic get routed to my website

![Inbound traffic routing](/diagrams/inbound%20calls.png "inbound calls")

1. A request comes into the Application gateway public ip address.  The App Gateway forwards the request onto the website.
2. The website uses access restrictions to prevent access on the public endpoint to only the ip address of the Application Gateway and the user that ran the script.

## How are internet calls from my website routed?

![outbound internet traffic routing](/diagrams/outbound%20calls%20to%20internet.png "outbound internet traffic routing")

The application that gets deployed makes calls to http://httpbin.org/ip to retrieve the outbound ip of the server making the call.
By default a website with a vnet intergration will always go direct for outbound internet calls (using the app service shared outbound ips).  By adding the `WEBSITE_VNET_ROUTE_ALL=1` setting it will use the UDR applied to the subnet.

1. The website has a private endpoint nic allocated to the web subnet.  
2. The web subnet contains a UDR that routes all traffic to the Azure Firewall.
3. The firewall contains application rules that allow traffic to httpbin.org (for the running application), github and npm (for the application build).

The `/ip` route on the website shows the call to httpbin.org/ip is successful and should also return the IP address of the firewall showing the request has been routed.

## How are internet calls to my table storage routed (private endpoint)?

![outbound private endpoint routing](/diagrams/outbound%20calls%20to%20private%20endpoint.png "outbound private endpoint routing")

1. Website requests a MYSTORAGE.table.core.windows.net address.  By default this would resolve to the public ip address (which is locked down)
2. The website is configured to point it's DNS requests at the internal azure DNS Server 168-63-129-16. (https://docs.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16). This uses the new `WEBSITE_DNS_SERVER` app setting in the web app (https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet)
3. The Azure DNS is aware of our Private DNS Zone so forwards the request there.
4. The private DNS zone is configured to resolve the privatelink address to an ip address on the vnet.
5. The web app can now communicate directly with the storage account over the private endpoint ip address.
