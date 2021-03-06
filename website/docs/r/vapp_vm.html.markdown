---
layout: "vcd"
page_title: "vCloudDirector: vcd_vapp_vm"
sidebar_current: "docs-vcd-resource-vapp-vm"
description: |-
  Provides a vCloud Director VM resource. This can be used to create, modify, and delete VMs within a vApp.
---

# vcd\_vapp\_vm

Provides a vCloud Director VM resource. This can be used to create,
modify, and delete VMs within a vApp.

## Example Usage

```hcl
# System administrator rights are required to connect external network
resource "vcd_network_direct" "direct-external" {
  name             = "net"
  external_network = "my-ext-net"
}
​
resource "vcd_vapp" "web" {
  name = "web"
}
​
resource "vcd_vapp_org_network" "routed-net" {
  vapp_name        = vcd_vapp.web.name
  org_network_name = "my-vdc-int-net"
}
​
resource "vcd_vapp_org_network" "direct-net" {
  vapp_name        = vcd_vapp.web.name
  org_network_name = vcd_network_direct.direct-external.name
}
​
resource "vcd_vapp_network" "vapp-net" {
  name               = "my-vapp-net"
  vapp_name          = vcd_vapp.web.name
  gateway            = "192.168.2.1"
  netmask            = "255.255.255.0"
  dns1               = "192.168.2.1"
  dns2               = "192.168.2.2"
  dns_suffix         = "mybiz.biz"
  guest_vlan_allowed = true
​
  static_ip_pool {
    start_address = "192.168.2.51"
    end_address   = "192.168.2.100"
  }
​
}
​
resource "vcd_vapp_vm" "web1" {
  vapp_name     = vcd_vapp.web.name
  name          = "web1"
  catalog_name  = "my-catalog"
  template_name = "photon-os"
  memory        = 1024
  cpus          = 2
  cpu_cores     = 1
​
  metadata = {
    role    = "web"
    env     = "staging"
    version = "v1"
    my_key  = "my value"
  }
​
  guest_properties = {
    "guest.hostname"   = "my-host"
    "another.var.name" = "var-value"
  }
​
  network {
    type               = "org"
    name               = vcd_vapp_org_network.direct-net.org_network_name
    ip_allocation_mode = "POOL"
    is_primary         = true
  }
}

resource "vcd_independent_disk" "disk1" {
  name         = "logDisk"
  size         = "512"
  bus_type     = "SCSI"
  bus_sub_type = "VirtualSCSI"
}
​
resource "vcd_vapp_vm" "web2" {
  vapp_name     = vcd_vapp.web.name
  name          = "web2"
  catalog_name  = "my-catalog"
  template_name = "photon-os"
  memory        = 1024
  cpus          = 1
​
  metadata = {
    role    = "web"
    env     = "staging"
    version = "v1"
    my_key  = "my value"
  }
​
  guest_properties = {
    "guest.hostname" = "my-hostname"
    "guest.other"    = "another-setting"
  }
​
  network {
    type               = "org"
    name               = vcd_vapp_org_network.routed-net.org_network_name
    ip_allocation_mode = "POOL"
    is_primary         = true
  }
​
  network {
    type               = "vapp"
    name               = vcd_vapp_network.vapp-net.name
    ip_allocation_mode = "POOL"
  }
​
  network {
    type               = "none"
    ip_allocation_mode = "NONE"
  }
​
  disk {
    name        = vcd_independent_disk.disk1.name
    bus_number  = 1
    unit_number = 0
  }
​
}
```

## Example Usage (Override Template Disk)
This example shows how to [change VM template's disk properties](#override-template-disk) when the VM is created.

```hcl
resource "vcd_vapp_vm" "internalDiskOverride" {
  vapp_name     = vcd_vapp.web.name
  name          = "internalDiskOverride"
  catalog_name  = "Boxes"
  template_name = "lampstack-1.10.1-ubuntu-10.04"
  memory        = 2048
  cpus          = 2
  cpu_cores     = 1

  override_template_disk {
    bus_type         = "paravirtual"
    size_in_mb       = "22384"
    bus_number       = 0
    unit_number      = 0
    iops             = 0
    storage_profile  = "*"
  }
}

```

## Example Usage (Wait for IP addresses on DHCP NIC)
This example shows how to use [`network_dhcp_wait_seconds`](#network_dhcp_wait_seconds) with DHCP.

```hcl
resource "vcd_vapp_vm" "TestAccVcdVAppVmDhcpWaitVM" {
  vapp_name     = vcd_vapp.TestAccVcdVAppVmDhcpWait.name
  name          = "brr"
  computer_name = "dhcp-vm"
  catalog_name  = "cat-dserplis"
  template_name = "photon-rev2"
  memory        = 512
  cpus          = 2
  cpu_cores     = 1

  network_dhcp_wait_seconds = 300 # 5 minutes
  network {
    type               = "org"
    name               = vcd_network_routed.net.name
    ip_allocation_mode = "DHCP"
    is_primary         = true
  }
}

resource "vcd_nsxv_ip_set" "test-ipset" {
  name                   = "ipset-with-dhcp-ip"
  ip_addresses           = [vcd_vapp_vm.TestAccVcdVAppVmDhcpWaitVM.network.0.ip]
}
```

## Argument Reference

The following arguments are supported:

* `org` - (Optional; *v2.0+*) The name of organization to use, optional if defined at provider level. Useful when connected as sysadmin working across different organisations
* `vdc` - (Optional; *v2.0+*) The name of VDC to use, optional if defined at provider level
* `vapp_name` - (Required) The vApp this VM belongs to.
* `name` - (Required) A name for the VM, unique within the vApp 
* `computer_name` - (Optional; *v2.5+*) Computer name to assign to this virtual machine. 
* `catalog_name` - (Required) The catalog name in which to find the given vApp Template
* `template_name` - (Required) The name of the vApp Template to use
* `memory` - (Optional) The amount of RAM (in MB) to allocate to the VM
* `cpus` - (Optional) The number of virtual CPUs to allocate to the VM. Socket count is a result of: virtual logical processors/cores per socket. The default is 1
* `cpu_cores` - (Optional; *v2.1+*) The number of cores per socket. The default is 1
* `metadata` - (Optional; *v2.2+*) Key value map of metadata to assign to this VM
* `initscript` (Optional **Deprecated** by `customization.0.initscript`) Script to run on initial boot or with
customization.force=true set. See [Customization](#customization-block) to read more about Guest customization and other
options.
* `storage_profile` (Optional; *v2.6+*) Storage profile to override the default one
* `network_name` - (Optional; **Deprecated** by `network`) Name of the network this VM should connect to.
* `vapp_network_name` - (Optional; v2.1+; **Deprecated** by `network`) Name of the vApp network this VM should connect to.
* `ip` - (Optional; **Deprecated** by `network`) The IP to assign to this vApp. Must be an IP address or
one of `dhcp`, `allocated`, or `none`. If given the address must be within the
  `static_ip_pool` set for the network. If left blank, and the network has
  `dhcp_pool` set with at least one available IP then this will be set with
DHCP.
* `power_on` - (Optional) A boolean value stating if this VM should be powered on. Default is `true`
* `accept_all_eulas` - (Optional; *v2.0+*) Automatically accept EULA if OVA has it. Default is `true`
* `disk` - (Optional; *v2.1+*) Independent disk attachment configuration. See [Disk](#disk) below for details.
* `expose_hardware_virtualization` - (Optional; *v2.2+*) Boolean for exposing full CPU virtualization to the
guest operating system so that applications that require hardware virtualization can run on virtual machines without binary
translation or paravirtualization. Useful for hypervisor nesting provided underlying hardware supports it. Default is `false`.
* `network` - (Optional; *v2.2+*) A block to define network interface. Multiple can be used. See [Network](#network-block) and 
example for usage details. **Deprecates**: `network_name`, `ip`, `vapp_network_name`.
* `customization` - (Optional; *v2.5+*) A block to define for guest customization options. See [Customization](#customization-block)
* `guest_properties` - (Optional; *v2.5+*) Key value map of guest properties
* `description`  - (Computed; *v2.6+*) The VM description. Note: description is read only. Currently, this field has
  the description of the OVA used to create the VM
* `override_template_disk` - (Optional; *v2.7+*) Allows to update internal disk in template before first VM boot. Disk is matched by `bus_type`, `bus_number` and `unit_number`. See [Override template Disk](#override-template-disk) below for details.
* `network_dhcp_wait_seconds` - (Optional; *v2.7+*) Optional number of seconds to try and wait for DHCP IP (only valid
  for adapters in `network` block with `ip_allocation_mode=DHCP`). It constantly checks if IP is present so the time given
  is a maximum. VM must be powered on and _at least one_ of the following _must be true_:
 * VM has Guest Tools. It waits for IP address to be reported by Guest Tools. This is a slower option, but
  does not require for the VM to use Edge Gateways DHCP service.
 * VM DHCP interface is connected to routed Org network and is using Edge Gateways DHCP service (not
  relayed). It works by querying DHCP leases on Edge Gateway. In general it is quicker than waiting
  until Guest Tools report IP addresses, but is more constrained. However this is the only option if Guest
  Tools are not present on the VM.
  


<a id="disk"></a>
## Disk

* `name` - (Required) Independent disk name
* `bus_number` - (Required) Bus number on which to place the disk controller
* `unit_number` - (Required) Unit number (slot) on the bus specified by BusNumber.

<a id="network-block"></a>

## Network

* `type` (Required) Network type, one of: `none`, `vapp` or `org`. `none` creates a NIC with no network attached. `vapp` requires `name` of existing vApp network (created with `vcd_vapp_network`). `org` requires attached vApp Org network `name` (attached with `vcd_vapp_org_network`).
* `name` (Optional) Name of the network this VM should connect to. Always required except for `type` `NONE`. 
* `is_primary` (Optional) Set to true if network interface should be primary. First network card in the list will be primary by default.
* `mac` - (Computed) Mac address of network interface.
* `adapter_type` - (Optional, Computed) Adapter type (names are case insensitive). Some known adapter types - `VMXNET3`,
    `E1000`, `E1000E`, `SRIOVETHERNETCARD`, `VMXNET2`, `PCNet32`.

    **Note:** Adapter type change for existing NIC will return an error during `apply` operation because vCD does not
    support changing adapter type for existing resource.

    **Note:** Adapter with type `SRIOVETHERNETCARD` **must** be connected to a **direct** vApp
    network connected to a direct VDC network. Unless such an SR-IOV-capable external network is
    available in your VDC, you cannot connect an SR-IOV device.

* `ip_allocation_mode` (Required) IP address allocation mode. One of `POOL`, `DHCP`, `MANUAL`, `NONE`:  

  * `POOL` - Static IP address is allocated automatically from defined static pool in network.
  
  * `DHCP` - IP address is obtained from a DHCP service. Field `ip` is not guaranteed to be populated. Because of this it may appear
  after multiple `terraform refresh` operations.  **Note.**
    [`network_dhcp_wait_seconds`](#network_dhcp_wait_seconds) parameter can help to ensure IP is
    reported on first run.

  
  * `MANUAL` - IP address is assigned manually in the `ip` field. Must be valid IP address from static pool.
  
  * `NONE` - No IP address will be set because VM will have a NIC without network.

* `ip` (Optional, Computed) Settings depend on `ip_allocation_mode`. Field requirements for each `ip_allocation_mode` are listed below:

  * `ip_allocation_mode=POOL` - **`ip`** value must be omitted or empty string "". Empty string may be useful when doing HCL
  variable interpolation. Field `ip` will be populated with an assigned IP from static pool after run.
  
  * `ip_allocation_mode=DHCP` - **`ip`** value must be omitted or empty string "". Field `ip` is not
    guaranteed to be populated after run due to the VM lacking VMware tools or not working properly
    with DHCP. Because of this `ip` may also appear after multiple `terraform refresh` operations
    when is reported back to vCD. **Note.**
    [`network_dhcp_wait_seconds`](#network_dhcp_wait_seconds) parameter can help to ensure IP is
    reported on first run.

  * `ip_allocation_mode=MANUAL` - **`ip`** value must be valid IP address from a subnet defined in `static pool` for network.

  * `ip_allocation_mode=NONE` - **`ip`** field can be omitted or set to an empty string "". Empty string may be useful when doing HCL variable interpolation.

<a id="override-template-disk"></a>
## Override template disk
Allows to update internal disk in template before first VM boot. Disk is matched by `bus_type`, `bus_number` and `unit_number`.
Changes are ignored on update. This part isn't reread on refresh. To manage internal disk later please use [`vcd_vm_internal_disk`](/docs/providers/vcd/r/vm_internal_disk.html) resource.
 
~> **Note:** Managing disks in VM is possible only when VDC fast provisioned is disabled.

* `bus_type` - (Required) The type of disk controller. Possible values: `ide`, `parallel`( LSI Logic Parallel SCSI), `sas`(LSI Logic SAS (SCSI)), `paravirtual`(Paravirtual (SCSI)), `sata`. 
* `size_in_mb` - (Required) The size of the disk in MB. 
* `bus_number` - (Required) The number of the SCSI or IDE controller itself.
* `unit_number` - (Required) The device number on the SCSI or IDE controller of the disk.
* `iops` - (Optional) Specifies the IOPS for the disk. Default is 0.
* `storage_profile` - (Optional) Storage profile which overrides the VM default one.


<a id="customization-block"></a>
## Customization

When you customize your guest OS you can set up a virtual machine with the operating system that you want.

vCloud Director can customize the network settings of the guest operating system of a virtual machine created from a
vApp template. When you customize your guest operating system, you can create and deploy multiple unique virtual
machines based on the same vApp template without machine name or network conflicts.

When you configure a vApp template with the prerequisites for guest customization and add a virtual machine to a vApp
based on that template, vCloud Director creates a package with guest customization tools. When you deploy and power on
the virtual machine for the first time, vCloud Director copies the package, runs the tools, and deletes the package from
the virtual machine.

~> **Note:** The settings below work so that all values are inherited from template and only the specified fields are
overridden with exception being `force` field which works like a flag.

* `force` (Optional) **Warning.** `true` value will cause the VM to reboot on every `apply` operation.
This field works as a flag and triggers force customization when `true` during an update 
(`terraform apply`) every time. It never complains about a change in statefile. Can be used when guest customization
is needed after VM configuration (e.g. NIC change, customization options change, etc.) and then set back to `false`.
**Note.** It will not have effect when `power_on` field is set to `false`. See [example workflow below](#example-forced-customization-workflow).
* `enabled` (Optional; *v2.7+*) `true` will enable guest customization which may occur on first boot or if the `force` flag is used.
This option should be selected for **Power on and Force re-customization to work**. For backwards compatibility it is
enabled by default when deprecated field `initscript` is used.
* `change_sid` (Optional; *v2.7+*) Allows to change SID (security identifier). Only applicable for Windows operating systems.
* `allow_local_admin_password` (Optional; *v2.7+*) Allow local administrator password.
* `must_change_password_on_first_login` (Optional; *v2.7+*) Require Administrator to change password on first login.
* `auto_generate_password` (Optional; *v2.7+*) Auto generate password.
* `admin_password` (Optional; *v2.7+*) Manually specify Administrator password.
* `number_of_auto_logons` (Optional; *v2.7+*) Number of times to log on automatically. `0` means disabled.
* `join_domain` (Optional; *v2.7+*) Enable this VM to join a domain.
* `join_org_domain` (Optional; *v2.7+*) Set to `true` to use organization's domain.
* `join_domain_name` (Optional; *v2.7+*) Set the domain name to override organization's domain name.
* `join_domain_user` (Optional; *v2.7+*) User to be used for domain join.
* `join_domain_password` (Optional; *v2.7+*) Password to be used for domain join.
* `join_domain_account_ou` (Optional; *v2.7+*) Organizational unit to be used for domain join.
* `initscript` (Optional; *v2.7+*) Provide initscript to be executed when customization is applied.

## Example of a Forced Customization Workflow

Step 1 - Setup VM:

```hcl
resource "vcd_vapp_vm" "web2" {
  vapp_name     = "${vcd_vapp.web.name}"
  name          = "web2"
  catalog_name  = "Boxes"
  template_name = "windows"
  memory        = 2048
  cpus          = 1

  network {
    type               = "org"
    name               = "net"
    ip                 = "10.10.104.162"
    ip_allocation_mode = "MANUAL"
  }
}
```

Step 2 - Override some VM customization options and force customization (VM will be rebooted during `terraform apply`):

```hcl
resource "vcd_vapp_vm" "web2" {
  #...

  network {
    type               = "org"
    name               = "net"
    ip_allocation_mode = "DHCP"
  }

  customization {
    force                      = true
    change_sid                 = true
    allow_local_admin_password = true
    auto_generate_password     = false
    admin_password             = "my-secure-password"
    # Other customization options to override the ones from template
  }
}
```

Step 3 - Once customization is done, set the force customization flag to false (or remove it) to prevent forcing
customization on every `terraform apply` command:

```hcl
resource "vcd_vapp_vm" "web2" {
  #...
  
  network {
    type               = "org"
    name               = "net"
    ip_allocation_mode = "DHCP"
  }

  customization {
    force                      = false
    change_sid                 = true
    allow_local_admin_password = true
    auto_generate_password     = false
    admin_password             = "my-secure-password"
    # Other customization options to override the ones from template
  }
}
```

## Attribute Reference

The following additional attributes are exported:

* `internal_disk` - (*v2.7+*) A block providing internal disk of VM details. See [Internal Disk](#internalDisk) below for details.
* `disk.size_in_mb` - (*v2.7+*) Independent disk size in MB.

<a id="internalDisk"></a>
## Internal disk

* `disk_id` - (*v2.7+*) Specifies a unique identifier for this disk in the scope of the corresponding VM.
* `bus_type` - (*v2.7+*) The type of disk controller. Possible values: `ide`, `parallel`( LSI Logic Parallel SCSI), `sas`(LSI Logic SAS (SCSI)), `paravirtual`(Paravirtual (SCSI)), `sata`. 
* `size_in_mb` - (*v2.7+*) The size of the disk in MB. 
* `bus_number` - (*v2.7+*) The number of the SCSI or IDE controller itself.
* `unit_number` - (*v2.7+*) The device number on the SCSI or IDE controller of the disk.
* `thin_provisioned` - (*v2.7+*) Specifies whether the disk storage is pre-allocated or allocated on demand.
* `iops` - (*v2.7+*) Specifies the IOPS for the disk. Default is 0.
* `storage_profile` - (*v2.7+*) Storage profile which overrides the VM default one.


## Importing

Supported in provider *v2.6+*

~> **Note:** The current implementation of Terraform import can only import resources into the state. It does not generate
configuration. [More information.][docs-import]

An existing VM can be [imported][docs-import] into this resource via supplying its path.
The path for this resource is made of org-name.vdc-name.vapp-name.vm-name
For example, using this structure, representing a VM that was **not** created using Terraform:

```hcl
resource "vcd_vapp_vm" "tf-vm" {
  name              = "my-vm"
  org               = "my-org"
  vdc               = "my-vdc"
  vapp_name         = "my-vapp"
}
```

You can import such vapp into terraform state using this command

```
terraform import vcd_vapp_vm.tf-vm my-org.my-vdc.my-vapp.my-vm
```

NOTE: the default separator (.) can be changed using Provider.import_separator or variable VCD_IMPORT_SEPARATOR

[docs-import]:https://www.terraform.io/docs/import/

After importing, the data for this VM will be in the state file (`terraform.tfstate`). If you want to use this
resource for further operations, you will need to integrate it with data from the state file, and with some data that
is used to create the VM, such as `catalog_name`, `template_name`.
