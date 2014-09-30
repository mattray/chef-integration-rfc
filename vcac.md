Chef and VMware's vCloud Automation Center have the ability to support operations and development with a unified view of workflows. Chef provides configuration policy and orchestration at scale while vCAC provides governance, workflows and reporting. Blending GUI and CLI-based workflows together allows us to speed adoption of more automated workflows while maintaining visibility of our infrastructure.

We have found that Chef/vCAC shops want to maintain support for both GUI and CLI-based workflows, so we want to ensure that they have the optimum experience. We will highlight where these intrations could be done, then prioritize based on customer feedback.

# vCloud Automation Center WebUI

The initial touch points are within vCloud Automation Center's Application Services (formerly Application Director). This approach would work well if you were creating a self-service portal for users with no Chef exposure.

## Chef-Managed Service

There will be a "Chef-Managed Service" that provides a base Chef client for Red Hat 5 and 6, Ubuntu LTS 12.04 and 14.04 and Windows on 64-bit platforms.

The Service will have Properties for the Chef server URL, organization and validation key as well as the node's environment and run list. We may want to add the option to include the chef-client cookbook (https://supermarket.getchef.com/cookbooks/chef-client) and expose the check-in interval as a Property.

For the Service's Actions:

- INSTALL installs the chef-client
- CONFIGURE writes the client.rb and installs the validation.pem (this could be stored within vCAC's storage)
- START, UPDATE and ROLLBACK call chef-client with options for the environment, run list and loading a JSON file
- TEARDOWN could have the node delete itself and its client from the Chef server with knife

This is very similar to what was demonstrated at ChefConf 2014 (https://www.getchef.com/blog/2014/04/28/chef-vmware-integration/). Working with VMware, we would ideally make this a first-class Service instead of manually creating an "Other" Service like we had at ChefConf. Windows support and TEARDOWN Actions would be new as well using vCAC's ability to manage files to store the validation.pem and possibly the client installers. Chef Solo and/or Zero support could be supported if we didn't want to use a full Chef server.

## Applications and Imported Chef Services

The Chef-Managed Service could be used within an Application in multiple configurations, setting the run list to differentiate the Services. The Service would only bind to the supported operating system Templates.

We could sync the Chef cookbooks and roles to create Chef Services dynamically with new "knife vcac" commands called periodically or with native API calls. Services from cookbooks would be named combining their name and version (ie. "COOKBOOK-VERSION") and since roles are unversioned they could be named "ROLE". Options for syncing only the latest and preventing modification of existing versions could be supported. Platforms supported from the cookbook's metadata would restrict which operating system Templates could be bound. Attributes files from the cookbooks could populate Properties fields with their defaults with the option of overriding them or we could support only explicitly exported attributes from the metadata (not widely used, but supported). Roles would populate Properties for the run list (read-only) and attributes.

The recipes from the cookbooks will be exposed in Properties, defaulting to 'default' but the other choices could be exposed in a drop-down. The use of single cookbooks and roles as Services is less useful if we cannot apply multiple Services to a Template, but that will be a known limitation if true. The base Chef-Managed Service doesn't have this limitation, since it may alter its run list.

Properties from cookbooks and roles that are changed in the UI would be passed back into the node through a JSON file at override precedence to ensure proper application. This was done in the ChefConf demo with an Application Component, but ideally it would be incorporated into the Service so it would not be directly exposed.

Environments for nodes would default to "_default". The environments could be synced to vCAC with a "knife vcac" command or API call made periodically. The environments would populate a drop-down menu under the Service's Properties for choosing whichone the node will belong to.

If we are supporting Chef Solo and/or Zero we will need to copy the roles and cookbooks into the nodes as they are instantiated (similar to how Chef Container works with Docker). We would keep the cookbooks and roles within vCloud Application Center's storage for ease of deployment. If the integration proves popular, we could investigate using the Berkshelf API backed by vCAC's storage eventually.

## Deployments

The Chef-based Services are bound to operating system Templates within Applications and provisioned by vCAC into a Deployment. When Applications are used as Deployments, Properties from the underlying Services may be further customized. This may be where we want to allow users to set their Chef nodes' Chef server details, environments or other customizations. Depending on the workflow styles, we could lock down users' access to the underlying Chef data so the Deployments are the only interaction they have with the underlying Chef-managed infrastructure.

## Chef Data in/from vCAC

There may be places in the WebUI that we could add additional data from the Chef server. APIs exist to push or pull additional data from the Chef Server, the use cases have yet to be identified. Similarly for purposes of reporting, we may wish to add an Event Handler to our Chef client runs that pushes additional data into vCAC or vCAC data into the Chef Analytics platform.

# External Application Service Automation with Chef

vCloud Automation Center 6.0 added a [REST API](http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-FBE98671-7140-4FA6-B9C7-57FF748842C9.html) that we should be able to use to build automated workflows outside of the WebUI. These workflows will still take advantage of vCAC's governance and management capabilities, but make it easier to build CI pipelines and give developers faster access to virtualized resources.

Chef users expect to automate platforms with knife, Test Kitchen and Chef Metal. Since there does not appear to be a Ruby SDK available for the API, we will need to write a new Ruby library that wraps the API calls used by Chef, published to Rubygems with an Apache 2 license. This library will not cover the entire API, only the Resources managed by Chef.

## knife-vcac

knife-vcac will be the initial integration point for vCloud Automation Center, providing the commands called by vCAC for syncing catalogs and eventually providing commands for external management. Currently it appears that creating Applications and Deployments is not supported, we may have to pre-create Applications and Deployments and use the API to change out the underlying Services and schedule the Deployment of them. Still investigating.

### Configuration

In order to communicate with the vCAC API you will need to tell Knife your vCAC server URL, your username and password. The easiest way to accomplish this is to create these entries in your `knife.rb` file (you may also use shell environment variables):

    knife[:vcac_url] = 'https://ApplicationServicesServerIP:8443'
    knife[:vcac_username] = 'Your vCAC username'
    knife[:vcac_password] = 'Your vCAC password'

### knife vcac cookbook sync

Synchronizes the cookbooks from a Chef repository (or Berksfile) into vCAC as Services.

The following additional options will be supported:

- `--latest-only`: deletes any cookbook/services that are not the latest version. This will probably require some sort of metadata or naming pattern applied to Services on vCAC, so it may not be feasible.
- `--environment ENVIRONMENT`: restrict to versions specified in an Environment
- `-W` or `--why-run`: enable whyrun mode to preview changes
- `--push`: upload cookbooks to vCAC storage

### knife vcac role sync

Synchronizes the roles from a Chef repository into vCAC as Services.

The following additional options will be supported:
- `-W` or `--why-run`: enable whyrun mode to preview changes

### knife vcac environment sync

Synchronizes the environments from a Chef repository into vCAC.

The following additional option will be supported:
- `-W` or `--why-run`: enable whyrun mode to preview changes

### knife vcac service list

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-0514F40F-225E-43E5-B2A4-1B4F8D35D9CE.html

### knife vcac service create

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-0514F40F-225E-43E5-B2A4-1B4F8D35D9CE.html

### knife vcac service delete

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-0514F40F-225E-43E5-B2A4-1B4F8D35D9CE.html

### knife vcac application list

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-4482A2ED-86CB-4D41-B53F-BAEFB7ED27E7.html

### knife vcac application create

Unsupported? "You can create an application in the Application Services user interface."

### knife vcac application delete

Unsupported?

### knife vcac deployment list

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-6270D810-E255-4A4D-BE75-9BD0DB4AAEAC.html

### knife vcac deployment create

Unsupported?

### knife vcac deployment teardown

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-31E03255-27F3-4810-A21B-2E3C77648995.html

### knife vcac deployment delete

Supported by the API.
http://pubs.vmware.com/vCAC-61/index.jsp#com.vmware.vcac.appservices.all.doc/GUID-31E03255-27F3-4810-A21B-2E3C77648995.html

## test-kitchen-vcac

Test Kitchen support will be added after knife-vcac development is providing useful functionality. Test Kitchen is currently single-instance only, so deployments of single nodes bound to operating system templates should prove analogous to how it currently behaves.

## chef-metal-vcac

Chef Metal provides libraries for managing large numbers of machines and infrastructure with Chef recipes. Once knife-vcac provides useful functionality the code may be ported to Chef Metal as Resources.
