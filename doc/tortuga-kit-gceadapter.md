## Google Compute Engine resource adapter

### Overview

Google Compute Engine support is enabled in Tortuga through the installation and activation of the Google Compute Engine resource adapter kit.

The Google Compute Engine resource adapter kit provides a resource adapter that can be used to perform the following functions on the Google Compute Engine platform:

* Add/delete node (virtual machine) instances
* Run a Tortuga installer node from within Google Compute Engine
* Run a Tortuga installer node from outside Google Compute Engine (also known
  as *hybrid* mode)

The Google Compute Engine resource adapter maps each virtual machine to a
Tortuga compute node. It also enables *cloud bursting* when used in conjunction
with the Tortuga Simple Policy Engine.

### Installing the Google Compute Engine resource adapter kit

Use `install-kit` to install the Google Compute Engine resource adapter kit:

    install-kit kit-gceadapter-6.3.0-0.tar.bz2

Once installed, the "management" component is enabled on the Tortuga installer
as follows:

    enable-component -p gceadapter-6.3.0-0 management-6.3
    /opt/puppetlabs/bin/puppet agent --onetime --no-daemonize

Using the Google Cloud Platform Console, create and download credentials to be
used with the Tortuga Google Compute Engine resource adapter. Refer to the
Google documentation "[Manage APIs in the Cloud Platform
Console](https://support.google.com/cloud/answer/6326510)" for information on
setting up API keys. Tortuga is capable of using either a P12 key file or a
JSON authentication file, both of which are available through the Google Cloud
Platform credentials management console.

When using a P12 key file, it is necessary to configure the settings `key` and
`service_account_email`. When using a JSON authentication file, these values
are provided and only the setting `json_keyfile` is required.

It is *recommended* to copy the P12 key file or JSON authentication file to
`$TORTUGA_ROOT/config`. If not copying either the P12 key file or JSON
authentication file to `$TORTUGA_ROOT/config`, it is necessary to specify the
full file path to the options `key` or `json_keyfile`, respectively.

Configure the Google Compute Engine resource adapter using the `adapter-mgmt`
command-line interface.

    adapter-mgmt create --resource-adapter gce --profile default \
        --setting default_ssh_user=centos \
        --setting image_url=<image_url> \
        --setting json_keyfile=<filename of json authentication file> \
        --setting network=default \
        --setting project=<project name> \
        --setting startup_script_template=startup_script.py \
        --setting type=n1-standard-1 \
        --setting zone=us-east1-b

To enable support for point-to-point VPN (ie. for a hybrid cluster
installation), add the following setting:

    adapter-mgmt update --resource-adapter gce --profile default \
        --setting vpn=true

Refer to the section "Google Compute Engine resource adapter configuration
reference" below for further information.

### Google Compute Engine resource adapter configuration reference

| Setting                 | Description                                 |
|-------------------------|---------------------------------------------|
| zone                    | Zone in which compute resources are created. Zone names can be obtained from Console or using `gcloud compute regions list` |
| key                     | Filename/path of P12 key file as provided by Google Compute Platform |
| json_keyfile            | Filename/path of JSON authentication file as provided by Google Compute Platform |
| service_account_email   | Email address as provided by Google Compute Platform |
| type                    | Virtual machine type. For example, "n1-standard-1" |
| network                 | Name of network where virtual machines will be created |
| project                 | Name of Google Compute Engine project |
| vpn                     | Set to "true" to enable OpenVPN point-to-point VPN for hybrid installations (default is "false") |
| startup_script_template | Filename of "bootstrap" script used by Tortuga to bootstrap compute nodes |
| image_url               | URL of Google Compute Engine image to be used when creating compute nodes. This URL can be obtained from the Google Compute Engine console or through the `gcloud` command-line interface <sup>\*</sup> |
| default_ssh_user        | Username of default user on created VMs. 'centos' is an appropriate value for CentOS-based VMs. |
| tags                    | Keywords (separated by spaces) |
| vcpus                   | Number of virtual CPUs for specified virtual machine type |

<sup>*</sup> Use the following `gcloud` command-line to determine the value for `image_url` for CentOS 7:

    gcloud compute images describe \
        $(gcloud compute images list -r 'centos-7.*' --format='value(name)') \
        --project centos-cloud --format='value(selfLink)'

### Creating Google Compute Engine hardware profile

Create a default Google Compute Engine-enabled hardware profile:

    create-hardware-profile \
        --xml-file=$(find $TORTUGA_ROOT/kits -name gceHardwareProfile.tmpl.xml) --name execd

Map the newly created hardware profile to an existing software profile or
create new software profile as necessary.

Nodes can then be added using the `add-nodes` command-line interface.

### Google Compute Engine firewall rules

All nodes within the Tortuga-managed environment on Google Compute Engine must
be unrestricted access to each other. This is the Google Compute Platform
default.

Nodes accessed externally (through SSH) minimally have SSH, and optionally,
OpenVPN ports enabled.

### Google Compute Engine resource adapter usage

#### Supported Node Operations

The Google Compute Engine resource adapter supports the following Tortuga node
management commands:

- `activate-node`
- `add-nodes`
- `delete-node`
- `idle-node`
- `reboot-node`
- `transfer-node`
- `shutdown-node`
- `startup-node`

The Google Compute Engine resource adapter *does not* support the following
node operation commands as they do not make sense within the context of
cloud-based compute nodes:

- `checkpoint-node`
- `migrate-node`

#### Adding Nodes

Nodes are added using the Tortuga `add-nodes` command. Specifying an Google
Compute Engine-enabled hardware profile (hardware profile with resource adapter
set to `gce`) automatically causes Tortuga to use the Google Compute Engine
resource adapter to manage the nodes.

For example, the following command-line will add 4 Google Compute Engine nodes
to the software profile `execd` and hardware profile `execd`:

    add-nodes --count 4 --software-profile execd \
        --hardware-profile execd

See Advanced Topics for additional information about enabling support for
creating preemptible virtual machines.

### Advanced Topics

#### Instance type to VCPU mapping {#instance_mapping_gce}

The Google Compute Engine platform does not provide the ability to
automatically query VM size metadata, so it is necessary to provide a mapping
mechanism.

This mapping is contained within the comma-separted value formatted file
`$TORTUGA_ROOT/config/gce-instance-sizes.csv` to allow Tortuga to
automatically set UGE exechost slots.

This file can be modified by the end-user. The file is the GCE VM size (ie.
`n1-standard-1`) followed by a comma and the number of VCPUs for that instance
type. Some commonly used instance type to VCPUs mappings are included in the
default installation.

#### Enabling support for preemptible virtual machines

Tortuga supports Google Compute Engine [preemptible virtual
machines](https://cloud.google.com/preemptible-vms/) through a standalone
"helper" daemon in Tortuga called `gce_monitord`. This daemon must be
enabled/started after configuring the Google Compute Engine resource adapter.

`gce_monitord` will poll Google Compute Engine resources every 60s monitoring
preemptible virtual machines that may have been terminted by Google Compute
Engine. These nodes will be automatically removed from the Tortuga-managed
cluster.

**Note:** `gce_monitord` will *only* monitor Google Compute Engine VM instances created/launched by Tortuga.

Enable support for preemptible virtual machines:

1. Configure Google Compute Engine resource adapter
1. Enable and start `gce_monitord`

    RHEL/CentOS 7

        systemctl enable gce_monitord
        systemctl start gce_monitord

    RHEL/CentOS 6

        chkconfig gce_monitord on
        service gce_monitord start

Once preemptible support has been enabled, add nodes to Tortuga using the
"--extra-arg preemptible" option. For example:

    add-nodes --software-profile execd --hardware-profile execd \
        --extra-arg preemptible --count 6

This command would add 6 preemptible nodes to the "execd" hardware profile and
"execd" software profile.

#### Google Compute Engine VM image requirements

All custom virtual machine images must conform to the guidelines set by Google
Compute Platform. The "startup script" mechanism (enabled by default in
Google-provided images) is used by Tortuga to bootstrap compute instances. This
mechanism must be preserved for custom, non-Google provided images.

\newpage

[Google Compute Engine]: https://cloud.google.com/compute           "Google Compute Engine"
