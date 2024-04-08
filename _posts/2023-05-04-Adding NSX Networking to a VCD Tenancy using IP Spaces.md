---
title: "Adding NSX Networking to a VCD Tenancy using IP Spaces"
date: 2023-05-04
categories: [vmware]
tags: [vcd, nsx]
image: /images/folder1/nsx-vcd-cover.png
---


## Introduction


IP Spaces is a new IP management service that was introduced in Cloud Director 10.4.1. 

This feature enables Service Providers to have the capability to control and manage IP address scope, ranges, prefixes, and quotas .You can create different IP scopes and assign them across all tenants without any overlapping or duplication.  It also is an improvement when tracking IP usage across tenants. 

Not only are IP Spaces  new to me but NSX integration with VCD is as well so I thought I'd spin up a new tenant from scratch and go from there.

Here is my BoM for this deployment


|   Component    |     Version      |
| :------------: | :--------------: |
| vCenter Server | 8.0.1 (21560480) |
|      ESXI      | 8.0.1 (21495797) |
|      NSX       |       4.1        |
| Cloud Director |      10.4.2      |

Here is what we will go over in this post


Whats been set up already
* Create Network Pool
* Create Provider VDC
* Create Org
* Create Provider Gateway
* Create Private IP Spaces
* Assign IP Space Uplink
* Create OrgVDC
* Create Tenant Gateway
* As a Tenant, create a new network


### Whats been set up already

A working and configured vCenter Server and a working and configured NSX Instance

I have Deployed and configured VCD.  I have added my vcenter server as a compute resource and my NSX Manager as a NSX Resource. 

For this blog post my Provider Gateway (T0 router) is going to be set as Private, this means a T0 needs to be allocated per individual tenant.  This has already been deployed in NSX and configured.  Im also going to enable Route Advertisement on my Networks so I have configured BGP pairing to my physical router. 

![Figure 1: T0 router for my Tenant named Blue](/images/folder1/1-t0.png)


--------
## Network Pool
--------

First up is creating a Network Pool in VCD, this is done in the Provider Portal.

This essentially configures NSX-T backed traffic in VCD to use a specific Geneve backed Transport Zone in NSX.

In the Provider Portal go to Resources >> Cloud Resources >> Network Pools and click **New**

Give your Pool a Name and click Next

![Figure 2: New Network Pool](/images/folder1/2-newnwp.png)

 NSX 4.X is installed so this deployment will use **Geneve Backed** as a Network Pool Type and then click **Next**

![Figure 3: Choose Network Pool Type ](/images/folder1/3-newnptype.png)



Select NSX Instance and then click **Next**
![Figure 4: NSX instance ](/images/folder1/4-newnpprovider.png)


Choose the NSX overlay transport zone thats been configured and then click **Next**
![Figure 5: Choose Transport Zone ](/images/folder1/5-newnptz.png)

Review Settings and then click **Finish**
![Figure 6: Review Settings ](/images/folder1/6-newnpfinish.png)


VCD will now create the Network Pool, this pool is needed to configure your Provider VDC.


--------
## Create Provider VDC
--------

Next is to create a new Provider VDC. 

This can be done in the Provider Portal under Resources >> Cloud Resources >> Provider VDCs and clicking on **New**

Name your Provider VDC and click **Next**
![Figure 7: Name Provider VDC ](/images/folder1/7-pvdcgeneral.png)

Choose your Cloud Resource (vCenter) and click **Next**

![Figure 8: Choose Provider ](/images/folder1/8-pvdcprovider.png)

Choose the vCenter cluster or resource pool that will map to this Provider VDC and click **Next**

![Figure 9: Select Cluster ](/images/folder1/9-pvdcrp.png)

Choose your Sotrage Policy and click **Next**

![Figure 10: Storage Policy ](/images/folder1/10-pvdcstorage.png)

Choose the Geneve Backed Network pool we created in the last step and click **Next**

![Figure 11: Network Pool ](/images/folder1/11-pvdcnwp.png)

Review Settings and then click **Finish**



--------
## Create Org
--------

Next Step is to create an Organization,  At the moment im colour coding all my Tenants so I will call this one 'Blue'

This can be done in the Provider Portal on Resources >> Cloud Resources >> Organizations

![Figure 12: Create Org ](/images/folder1/12-org.png)


--------
## Create Provider Gateway
--------

Next step is to create the Provider Gateway. This is assigning the T0 router that was created in NSX to my VCD tenant named Blue. Assigning a T0 to a specific tenant is done by setting the Provider Gateway to Private which will be done in the steps below. 

Creating a new Provider Gateway is achieved in the VCD Provider Portal.  Resources >> Cloud Resources >> Provider Gateways, click **New** 

![Figure 13: New Provider Gateway ](/images/folder1/13-pgnew.png)

Choose NSX Manager and click **Next**

![Figure 14: Choose NSX-T ](1/images/folder1/14-pgnsx.png)

Give your Provider Gateway a Name and also choose what VCD IP Management capability will be used.  In this example we will use IP Spaces, click **Next**

![Figure 15: Name Provider Gateway ](/images/folder1/15-pggen.png)

Set Provider Gateway to Private and assign to my Blue Organization, click **Next**

![Figure 16: Make Private ](/images/folder1/16-pgownership.png)

Choose the NSX t0 router allocated for this Org and click **Next**

![Figure 17: Choose T0 ](/images/folder1/17-pgt0.png)

Review Settings and click **Finish**
![Figure 18: Review Settings](/images/folder1/18-pgfinish.png)


--------
## Create Private IP Space
--------

Creating a IP Space will allow the tenant to request a IP Prefix to use within their OrgVDC,  in this example the tenant 'Blue' will be allocated a /16 scope which will then be broken up into 255 /24 blocks. 

Create a IP Space in the Provider Portal.  Resources >> Cloud Resources >> IP Spaces

![Figure 19: New IP Space](/images/folder1/19-ipsnew.png)

This IP Space will be allocated to the tenant Blue only so choose private, assign the appropriate Org and click **Next**
![Figure 20: Make IP Space Private](/images/folder1/20-ipstype.png)

Give the IP Space a name and click **Next**

![Figure 21: Name IP Space ](/images/folder1/21-ipsgeneral.png)

Since im all configured with BGP, **Route Advertisment** will be enabled, this means we also need to set up IP Prefixes instead of IP Ranges. Click **Next**

![Figure 22: Enable Route Advertisement ](/images/folder1/22-ipsnw.png)

Scope is defining the /16 network, click **Next**

![Figure 23: Enter Scope ](/images/folder1/23-ipsscope.png)

Skip IP Ranges and click **Next**

![Figure 24: IP Ranges ](/images/folder1/24-ipsrange.PNG)

Set the IP Prefixes here,  as stated below im going to take that /16 scope and break them up into 255 /24 networks.  click **Next**


**The IP prefixes need to match the scope of the IP Space.**


![Figure 25: IP Prefixes ](/images/folder1/25-ipsprefix.png)

Review Settings and click **Finish**

![Figure 26: Review Settings ](/images/folder1/26-ipsreview.png)

You can now see your IP Space, if you expand IP Prefix you will be able to see all the Sequences. 

![Figure 27: Review Settings ](/images/folder1/27-ss.png)


--------
## Create IP Space Uplink
--------

Since these IP Prefixes are going to be routable to the outside world, A IP Space Uplink needs to be created. This is done by browsing to your Provider Gateway created earlier.

![Figure 28: Create IP Space Uplink ](/images/folder1/28-uplinknew.png)

Under IP Space Uplinks in your Provider Gateway properties, click **New**

![Figure 29: Provider Gateway Properties](/images/folder1/29-uplinkblue.png)

Provide a Tenant Facing Name and click **Next**

![Figure 30: Name Uplink ](/images/folder1/30-uplinkcreate.png)

Choose your IP Space, click **Next**

![Figure 31: Choose IP Space ](/images/folder1/31-uplinkips.png)

Review Settings and click **Finish**

![Figure 32: Review Settings ](/images/folder1/32-uplinkready.png)

You should see your newly created IP Space Uplink now
![Figure 33: Review Settings](/images/folder1/33-uplinksview.png)


--------
## Create OrgVDC
--------
OrgVDC Networks need a OrgVDC so lets create that.

In the Provider Portal under Resources >> Cloud Resources >> Organization VDCs, Click **New**

![Figure 34: New OrgVDC ](/images/folder1/34-orgvdcnew.png)

Name the OrgVDC, click **Next**

![Figure 35: Name OrgVDC ](/images/folder1/35-orgvdcgen.png)

Assign OrgVDC to Organization, click **Next**

![Figure 36: Choose Org ](/images/folder1/36-orgvdcorg.png)

 
Choose Provider VDC, click **Next**

![Figure 37: Provider VDC ](/images/folder1/37-orgvdcpvdc.png)

Choose the preferred Allocation Model, click **Next**


![Figure 38: Allocation Model](/images/folder1/38-orgvdcmodel.png)

Configure the Resource Allocation settings, click **Next**
![Figure 39: Resource Allocation](/images/folder1/39-orgvdcfex.png)

 
Add Storage Policies and Quotas, click **Next**

![Figure 40: Storage Policies](/images/folder1/40-orgvdcsp.png)

Choose the Network Pool, click **Next**
![Figure 41: Network Pool ](/images/folder1/41-orgvdcnp.png)

Review Settings, click **Finish**

![Figure 42: Review Settings](/images/folder1/42-orgvdcready.png)

You should now see your NSX-T backed OrgVDC
![Figure 43: NSX-T backed OrgVDC ](/images/folder1/43-orgvdclist.png)


--------
## Create Tenant Edge Gateway
--------

Create Edge Gateway for Tenant Blue, This will provision a T1 Gateway in NSX

Created in the Provider Portal under Resources >> Cloud Resources >> Edge Gateways

Click **New**

![Figure 44: New Edge Gateway ](/images/folder1/44-egwview.png)

Choose the OrgVDC, click **Next**

![Figure 45: OrgVDC ](/images/folder1/45-egworgvdc.png)

Name Edge Gateway and enable IP Spaces, click **Next**

![Figure 46: Name Edge Gateway ](/images/folder1/46-egwgen.png)

Choose your Provider Gateway, click **Next**
![Figure 47: Provider Gateway](/images/folder1/47-egwpgw.png)

This is dependant on the NSX configuration but, in this instance the edge cluster linked to the provider T0 will be used, click **Next**

![Figure 48: Edge Cluster ](/images/folder1/48-egwec.png)
Review Settings and click **Finish**

![Figure 49: Review Settings ](/images/folder1/49-egwcomplete.png)


In NSX you will see the Edge Gateway deployed as a T1 Router linked to the T0 Provider Gateway
![Figure 50: Edge Gateway Summary ](/images/folder1/50-egwnsx.png)

--------
## Modify Tenant Default Right Bundle
--------

To allow Tenants to create Networks via an IP Space prefix, the Tenant Default Rights Bundle needs to be modified.

![Figure 51: Tenant Default Right Bundle](/images/folder1/51-tenrb.png)


--------
## Create New Network as Tenant
--------

A tenant can now log into the Tenant Portal and create their own L3 Routable Network using IP Spaces. 

Under Networking, click **New**

![Figure 52: New Network](/images/folder1/52-nwtenant.png)

Choose your orgVDC, click **Next**


 **If there is a need to present a network to multiple OrgVDCs, a Datacenter Group can be created. **



![Figure 53: OrgVDC](/images/folder1/53-nwtscope.png)

Choose Routed, click **Next**

![Figure 54:Routed ](/images/folder1/54-nwttype.png)

Specifiy the Edge Gateway, click **Next**

![Figure 55: Edge Gateway](/images/folder1/55-nwtedge.png)

Name your Network, under Gateway CIDRs request a IP Prefix from your IP Space

![Figure 56:  Request a IP Prefix from your IP Space](/images/folder1/56-nwtgen.png)

VCD will then get a Sequence from the IP Space and assign to this network

![Figure 57: IP Prefix Allocation ](/images/folder1/57-nwtallocate.png)



![Figure 58: IP Prefix Confirmation](/images/folder1/58-nwtassigned.png)

Skip Static IP Pools, click **Next**

![Figure 59: Static IP Pools](/images/folder1/59-nwtstatic.png)

Skip DNS, click **Next**

![Figure 60: DNS](/images/folder1/60-nwtdns.png)

Review Settings, click **Next**

![Figure 61: Review Settings](/images/folder1/61nwt-ready.png)


The Network is now visible in the Blue Tenancy, the tenant is now able to build VMs and applications to assign to this network.

![Figure 62: Network](/images/folder1/62-nwtverify.png)


--------
## NSX and BGP Verification
--------
viewing the IP Space usage and allocation can be done in the Tenant Portal under Networking >> IP Spaces

![Figure 63: IP Space usage and allocation](/images/folder1/63-lease.png)

In NSX, you can now see your network created as an Overlay Segment.

![Figure 64: Overlay Segment](/images/folder1/64-sumnsx.png)

As promised, this network is routable and is showing on my physical router via BGP. 

![Figure 65: BGP Route ](/images/folder1/65-sumbgp.png)

--------
## Summary
--------


IP Spaces is a VCD IP management system that allows providers to assign IP interfaces to customers without any duplication and overlapping. 

Another benefit I can see here is if the Provider provides public IP addresses to customers where they can request and release as needed.


Thank you for reading
