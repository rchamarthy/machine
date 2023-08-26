# machineOS product development

machineOS is a secure container OS for building container-based products that
ship as appliances. A machineOS product, once built, can be deployed as an
appliance on bare metal, VMs, or in the cloud with specific packaging for each
of those environments.

## Appliance based products

Delivering software products with multiple components and services as an
appliance can have several benefits, including:

1. __Simplifying Installation Complexities:__ Appliance-based delivery
simplifies the installation process by providing a pre-configured and
integrated solution.  Users don't need to worry about setting up individual
components or services, which can be time-consuming and error-prone.

1. __Avoiding System-Level Dependencies:__ By encapsulating all necessary
components within the appliance, you reduce the reliance on specific
system-level libraries, versions of shared libraries (like glibc), or shell
versions eliminating potential conflicts that might arise due to differences
in these dependencies between the software and the user's system.

1. __Conflicts and Customer Experience:__ When software assumes specific
dependencies not present on the user's system, it can lead to conflicts and
errors during installation or usage. These errors can frustrate customers and
create a negative experience. Appliance-based delivery helps minimize these
conflicts and provides a smoother experience for users.

1. __Enhanced User Experience:__ Appliance-based solutions can offer a more
consistent and user-friendly experience, as users receive a self-contained
package that works seamlessly out of the box. Improving on time to value for
the customer leads to higher user satisfaction and adoption rates.

1. __Reduced Technical Expertise Required:__ Users may not need advanced
technical skills or knowledge to set up and use an appliance. A more
straightforward software setup widens the potential user base, making the
software accessible to a broader range of customers.

1. __Quick Deployment:__ Appliances can be deployed rapidly, reducing the
time and effort required to get the software up and running. Rapid deployment
is especially valuable for time-sensitive projects or organizations aiming to
streamline their processes.

## Challenges in building appliances

While appliance-based delivery offers many benefits, some challenges exist in
building and maintaining an appliance. These include maintaining the OS (kernel,
libraries, etc.), managing security, monitoring system health, and updates.

machineOS aims to mitigate these challenges by providing tooling to build and
maintain secure container-based appliances, allowing product developers to focus
on their application development:

1. __OS Maintenance:__ Keeping an operating system up to date, secure, and
compatible with various hardware and software components can be complex and
resource-intensive. machineOS provides a standardized base OS that's managed
and maintained, reducing the burden on product teams to handle OS-related
tasks.

1. __security:__ security is paramount, especially for software appliances
containing sensitive data or serving critical functions. machineOS
incorporates Security best practices and regular updates to ensure solid
security foundation, minimizing vulnerabilities and risks.

1. __System Health Monitoring and Management:__ Monitoring the health and
performance of an appliance is crucial for identifying issues and maintaining
optimal performance. machineOS offers built-in monitoring and management
services, simplifying the task of tracking system health and responding to
incidents.

1. __Simple Upgrades:__ Upgrading or updating an appliance can be challenging
and potentially disruptive. machineOS streamlines this process, making it
easier to deliver improvements to the underlying infrastructure without
impacting the product's functionality.

1. __Community and Collaboration:__ machineOS aims to foster a community of
contributors who collaborate on improving the base OS and services. This
shared effort can produce a more robust and well-maintained foundation for
developing software appliances.

This document will describe machineOS constructs that are available to product
developers to build and ship their products.

## Core concepts

machineOS is a container-exclusive operating system purposefully designed to
support solely containerized processes, ensuring natural isolation for
application services. By confining processes within dedicated containers,
machineOS effectively reduce attack vectors, preventing potential
vulnerabilities in a single process from compromising the entire system.

### Trust and Protection

machineOS builds on a foundation of a robust trust model. To trust a system is
to be able to verify that system's integrity and provenance. Trust requires
every artifact that makes the system to be signed and verified.  machineOS uses
a secure provisioning process that is specific to the machine environment. For
example, in the case of a bare-metal machine, the provisioning involves adding
the UEFI shim certificate to the BIOS trust store. Each entity in the process is
responsible for verifying the integrity of the following entity, forming a
strong chain of trust.

__Boot Process:__

1. BIOS verifies the signature of the UEFI shim using the UEFI shim certificate
   in its trust store and loads the shim and hands-off control.
1. UEFI shim verifies the integrity of the kernel, the kernel command line,
and the initrd using the embedded kernel certificate and hands-off control.
1. Kernel loads the initrd, and the initrd eventually verifies the integrity of
the RFS using the manifest certificate and pivots the root filesystem,
completing the boot process.

As part of provisioning, the machineOS provisioner programs the hardware or
virtual trust platform module (TPM). The TPM EA policies gates access to the
TPM by storing the measurements of the shim certificate and the kernel into
the TPM's PCR7. Now, the TPM can seal secrets and certificates unsealed by
only a trusted kernel matching the measurements stored in the TPM.

__Product Identity:__

Product identity is an identifier defined by the product designer to identify
the purpose of the machine. This identity is encoded into the machineOS
device identity certificate and is used to determine the role of the
appliance in a multi-appliance cluster. The product identity provides a basis
for defining the CPU, memory, storage, and network devices present in the
machine.

__Device Idenity:__

Device identity is an X.509 certificate and the corresponding private key.
Peer devices in a cluster can use this certificate to verify the device's
identity.

Device identity usually includes the device manufacturer, model, and serial
number. Customers can use this information to sign-off admission of the device
into the cluster securely.

TPM stores the device identity certificate and the key at a well-known NV index,
which can be unsealed and read only by a trusted kernel, preventing tampering of
the device identity.

__Storage Keys:__

A robust set of keys are generated per device and stored in the TPM. These
keys are used to secure sensitive storage volumes created on the device.

1. Machine Configuration Key - A machine configuration key is generated and
stored in the TPM. machineOS stores the system configuration on a LUKS volume
encrypted with the machine configuration key.

1. Storage Volume Key - A storage volume key is generated and stored in the
TPM. Storage management service will create LUKS encrypted volumes using the
storage volume key. Services can store sensitive data in these encrypted
volumes to protect the data at rest.

__Immutable RFS:__

machineOS employs an immutable root filesystem (RFS) that remains mounted in
a read-only state. This immutability extends to the host and all containers,
including application containers, facilitating efficient integrity
verification for the entire system. machineOS mounts a verified RFS upon
boot, and if the RFS is not verified, the system will not boot.

The initrd unseals the TPM on each boot and reads the machine configuration
key. machineOS then uses the machine configuration key to decrypt and mount
`/config` volumes. The `/config` volume contains the system configuration
with the details of the current RFS to be mounted by the system. machineOS
creates a temporary snapshot of the RFS and pivots into that RFS to complete
the rest of the boot process. Creating a new snapshot of the RFS on each boot
ensures that the RFS is always pristine.

### Service

A service is a containerized process that provides a specific functionality.
machineOS manages the service artifacts and their lifecycle. machineOS service
manager will verify all service artifacts before installation. The service
publisher signs all the service artifacts using a key that the machine manifest
certificate can verify. The service meta defines the service ID, vendor,
description, version, and list of privileges required for it to function.

### Authorization

machineOS identifies every service by verifying the service publisher. Each
service publisher is assigned a set of allowed roles and privileges on the
system. Services can then request a subset of these roles allowed for a service
publisher during installation. Identifying services by their publishers allows
for enforcing authorization policies for each service in the system.

__Common Privileges:__

1. MACHINE_USER - Allows reading the machine configuration.
1. MACHINE_ADMIN - Allows writing the machine configuration.
1. NETWORK_ADMIN - Allows configuring the network.
1. NETWORK_USER - Allows privileged network commands that do not alter the
   network state. For example, ping, traceroute, or reading network
   configurations like devices, interfaces, and routes.
1. STORAGE_ADMIN - Allows configuring and managing storage devices. Like CRUD
operations on block devices, configuring LVM for PV, VG, etc.
1. STORAGE_USER - Allows creation and deletion of storage volumes that belong
to the invoking user.
1. SERVICE_ADMIN - Allows managing service lifecycle.
1. SERVICE_USER - Read-only access to service meta and configuration.
1. USER_ADMIN - Allows managing host local users and groups.

Service meta can specify the privileges required, and these privileges can be
verified against the allowed privilege set for a given service publisher
during service installation before the services are assigned these privileges
and enabled.

## System

machineOS is the implementation of carefully studied best practices to build a
security-first container OS for appliances. All system components customize
standard Linux primitives in a specific way that enforces these best practices.
This section will examine machineOS's workflow to build a trusted runtime
environment for application services.

### Configuration

machineOS stores its configuration in an encrypted volume and mounts it at
`/config` on the host. This volume is created at the first install of the OS and
is encrypted using the machine configuration key. machineOS configuration
service is a versioned store that captures the system configuration and the
configuration of all the services installed on the system. It supports
versioning, rollbacks, snapshots, and update transactions. Note that, given the
service configuration and the container images that make up the system, a
machineOS operator can restore the system to a known previous state.

The images, including the RFS, are immutable, and the configuration store is the
only state that keeps track of the runtime configuration of the system.
Immutable RFS and versioned configuration are the architectural invariants of a
machineOS system. The ability to quickly restore a device to a particular state
snapshot is a powerful construct in clustered applications. Restoring a device
allows application state replication to recover the application soon, restoring
the cluster and application health.

### Boot Process

machineOS boot process ensures that the system starts in a trusted and verified
state. mos binary in the initrd calculates the target RFS from the system
configuration and mounts it as /.

The root file system packages a single service called the machineOS service
manager, which is responsible for bootstrapping all the services in the system
as containers. For details, refer to service management
[section](# Service Management).

Every installation of machineOS MUST have a set of default services. Building
these functions as just another set of services will allow them to be customized
or replaced in the odd situation where the default service implementation may
not meet the product requirement for that function.

| System Service Name         | Optional   |
|:--------------------------- |:----------:|
| Configuration Service       | No         |
| Bootstrap Service           | No         |
| Service Manager             | No         |
| System Update Service       | No         |
| Storage Service             | No         |
| Network Service             | Yes        |
| Host Agent Service          | Yes        |
| Host Local CA Service       | Yes        |
| User Authentication Service | Yes        |
| Log Service                 | Yes        |
| SSH Service                 | Yes        |
| NTP Service                 | Yes        |
| DNS Client Service          | Yes        |

Subsequent sections describe these services.

## Configuration Service

Configuration Service (CS) is responsible for managing the configuration of
every service in the system. CS stores service configuration as a git-managed
directory in `/config` volume. This model allows modification of multiple
service configurations and the system configuration as a single git commit,
allowing versioning and audit of configuration.

CS stores each service configuration as a set of arbitrary files under
`/config/<service-id>` directory, and every change to the configuration is
captured as a git commit. These files are then mounted into the service
container at `/etc/services/<service-id>/config` as read-only files. The service
manager will issue create/delete/update events to manage service-specific
configuration files as part of the service lifecycle.

## Bootstrap Service

Bootstrap service is the initial configuration input service that assists in
gathering the minimal system configuration that the user has to provide to Have
the appliance boot properly and become accessible. This service is usually
environment-specific and needs to be configured by the product designer specific
to the appliance and the target deployment environment.

### Console Bootstrap Service

Console bootstrap service will prompt the user for the initial system
configuration via the console. The service then validates the input and writes
the configuration into the service configuration store.

### KVM Bootstrap Service

TODO: Discuss if we can mount a configuration file from the host into the guest
OS as a mechanism to bootstrap the system.

### OVFTOOL Bootstrap Service

This flavor of the bootstrap service reads the configuration from the OVF device
mounted into the guest OS and writes the configuration into the service.

### AMI Bootstrap Service

This flavor of the bootstrap service reads the configuration from the aws config
mounted into the guest OS and writes the configuration into the service.

### Azure Bootstrap Service

This flavor of the bootstrap service reads the configuration from the azure
config mounted into the guest OS and writes the configuration into the service.

### GCP Bootstrap Service

This flavor of the bootstrap service reads the configuration from the GCP config
mounted into the guest OS and writes the configuration into the service.

## Service Manager

machineOS service manager (SM) is responsible for managing the lifecycle of
services.  The service manager is responsible for installing, starting,
stopping, and uninstalling all other services on the system. SM can install
services via a local OCI layout or a remote OCI registry.

### Artifacts

machineOS service is an OCI container image and accompanying artifacts. All the
service images and artifacts are signed using the machineOS manifest key and
verified by the service manager before installation.

1. __Image:__ Service image is an OCI container image that contains the RFS of
the container includes the binaries and libraries required to run the
service.
1. __Manifest:__ Service manifest is a YAML document that describes the
service meta required to install and manage the service.
1. __Signatures:__ All artifacts need to be signed; the signature itself is
another OCI artifact.
1. __certificate:__ Service certificate that identifies the service publisher
that signed all service artifacts. This certificate needs to be signed by
machineOS manifest key.

Example of a service manifest:

```yaml
service:
  name: zot
  vendor: cisco
  description: Zot is an OCI container registry
  version: 0.1.0
  tags:
    - registry
    - oci
  compatibility: 
    machine: 
      architecture: x86_64 
      versions:
        - 0.2.*
        - 0.1.3
    service:
      versions:
        - 0.2.*
        - 0.1.4
    privileges:
      - STORAGE_USER
      - SERVICE_USER
    artifacts:
      image: zothub.io/cisco/zot:0.1.0
      image-signature: zothub.io/cisco/zot:0.1.0.sig
      manifest: zothub.io/cisco/zot:0.1.0.manifest
      manifest-signature: zothub.io/cisco/zot:0.1.0.manifest.sig 
      certificate: zothub.io/cisco/zot:0.1.0.cert 
    entry-point: /usr/bin/zotd -f /etc/services/zot/config/zotd.conf
    storage:
      - mount: /data
        type: SSD
        size: 100G
        fs: ext4
        encrypt: false
      - mount: /logs
        type: SSD
        size: 1G
        fs: ext4
        encrypt: false
      - mount: /tmp
        type: ramdisk
        fs: tmpfs 
        size: 1G 
    cgroup: 
      cpu: 2000 
      memory: 2G 
    network:
      ns: host
```

The above example shows a service manifest for a service called zot.

__vendor:__ name of the service publisher unique across all vendor names.
The vendor name identifies the service publisher, and machine administrators
can create a host policy to define this publisher's allowed privileges.

__name:__ Name of the service to identify the service; this must be unique
for a given vendor.

__version:__ Version of the service. The expected convention for versioning
is semver.

__tags:__ list of tags to group and search for services.

__compatibility:__ Defines the compatibility rules for the service. Service
needs to be compatible with the machineOS version and architecture for
install and in case of updates, it needs to be consistent with the previous
versions of itself.

__privileges:__ list of privileges required by the service.

__artifacts:__ list of artifacts that make up the service.

__entry-point:__ Entry point for the service that will be executed on
service startup.

__storage:__ list of storage volumes required by the service.

__cgroup:__ Cgroup limits for the service.

__network:__ Network namespace for the service.

### Install

To install a new service into the system, the service manager performs the
following actions:

1. Pull the service manifest signature and the service manifest
1. Verify that the machineOS manifest key signs the service certificate
1. Verify the manifest signature using the manifest certificate
1. Verify if the service is compatible with the running machine and any previous
instances of the service itself
1. Verify if the privileges requested by this service are allowed for the
signing authority that signed the manifest
1. Pull the service image signature and the service image defined in the
manifest
1. Verify the image signature using the manifest certificate
1. Make an entry into the machineOS configuration for the service and its state
as installed

SM assigns a service-id per service instance in the form of
`<vendor>-<name>-<version>-<instance-id>,` allowing for multiple versions and
instances of the same service to exist on a single host.

### Start

To start a service, the service manager performs the following actions:

1. Check if the storage volumes required by the service are available, if not
create them
1. Create an instance-specific temporary service identity certificate using the
host local CA and mount it RO into the service container at
`/etc/services/<service>/certs` and recreate it on every service restart
1. Mount the service meta into the service container at
`/etc/services/<serviec>/meta` as read-only files
1. Mount the service ID into the service container at
`/etc/services/<service>/id` as a read-only file
1. Mount the service configuration files, if any, into the service container at
`/etc/services/<service>/config` as read-only files
1. Check if the lxc container exists with the correct cgroup limits, storage
1. Check if the systemd service is created and enabled; if not, create and
enable
1. Start the systemd service

If the service is already running, this command is a no-op. Note that service
start is idempotent, and every service restart invokes all steps required to set
up service prerequisites. Retrying service start is allowed to mitigate
transient failures. For example, the system admin can delete an existing service
or add more storage and retry with the start command if storage is unavailable.

### Stop

To stop a service, the service manager performs the following actions:

1. Stop the service using systemd

### Uninstall

To uninstall a service, the service manager performs the following actions:

1. Check that the service is in a stopped state
1. Mark the service deleted in the machineOS configuration
1. disable and remove the systemd service file
1. Remove the storage volumes associated with the service
1. Remove the LXC container

### Clean

1. Check that the service is in a stopped state
1. Remove the storage volumes associated with the service

Restarting the service will create the required storage volumes, and the service
has a clean slate.

### Update

Updating the service involves the following workflow:

1. Install the next version of the service. Note that an existing version can
still be installed and active.
1. Stop the existing version of the service
1. Start the new version of the service. Note that the current version's storage
must be attached to the new version so that the new version does not have any
state loss.

## System Update Service

System update is special compared to updating services, as the boot process has
to be updated consistently. The system update service follows this workflow:

1. Check if all the latest installed services, active or otherwise, are
compatible with the new machineOS version. Compatibility checks help guarantee
that these services are still valid on reboot after a system update.
1. Update the machineOS configuration to record the new machineOS version and
the RFS.
1. When a reboot happens, the new kernel, initrd, and the RFS will be used to
bring up the system.

## Storage Service

machineOS storage service is responsible for discovering the storage devices,
and managing them. machineOS product designed MUST provide a storage policy for
the appliance that will define the storage devices expected in the machine, how
to pool/partition these devices, and create volume groups for the services to
request storage volumes from.

### Storage Policy

Storage policy is a yaml file that describes the storage devices and the
configuration:

```yaml
storage:
  devices:
    - type: SSD
      min-size: 100G
      max-size: 1T
      count: 1
      attachment: RAID
    - type: HDD
      min-size: 1T
      max-size: 2T
      count: 4
      attachment: RAID
    - type: nvme
      min-size: 1T
      max-size: 2T
      count: 1
      attachment: PCIE
    - type: SSD
      min-size: 100G
      max-size: 250G
      count: 1
      attachment: ATA
  volume-groups:
    - name: vg0-ssd
      devices:
        - raid-ssd1
    - name: vg0-nvme
      devices:
        - pcie-nvme1
    - name: vg0-hdd1
      devices:
        - raid-hdd1
    - name: vg0-hdd2
      devices:
        - raid-hdd2
    - name: vg0-hdd3
      devices:
        - raid-hdd3
    - name: vg0-hdd4
      devices:
        - raid-hdd4
```

The above configuration tells the storage service to expect 1 SSD, 4
HDDs, and 1 NVMe disks. Storage service can then assign storage volume
requests to these volume groups. Note that any arbitrary grouping is
possible. On finding a new device, the storage service will wipe it and
partition it with a unique GUID that can be referred to by the name. The
naming convention used to identify the device is
`<attachment>-<type><count>,` and can be used to assign devices to
specific volume groups.

## Network Service

Network service is responsible for configuring the network devices
present in the system. In addition, the network service provides a way
to configure the iptable rules on the host to allow for port forwarding
or to define firewall rules.

The network configuration follows the `netplan` format to define the
network.

__TODO:__ Discuss how to configure iptables.

## Host Agent Service

The host agent service monitors the host CPU, memory, storage, and device health
and provides APIs for the report. This service also offers privileged operations
like host reboot, power-off, configuration-wipe, re-bootstrap, etc.

Host service configuration records all host operations as requests, and the
service will act on them as a one-shot request. For example, if the host agent
service gets a reboot request, it will first record it as a configuration change
and then reboot the host. On reboot, the host agent service will check if the
last boot time is greater than the recorded reboot request time, and it will
assume that the operation is complete and will not act again.

This approach will allow for auditing all host operations as part of the
auditing the service's configuration.

## Host Local CA Service

Host local CA service initializes a self-signed CA certificate and key for the
host. This CA certificate signs service-specific certificates and establishes a
strong identity for the services. Services can then use these certificates to
establish mutual TLS connections and enforce service-specific authorization
policies. By default, services do not trust peer services, implementing a secure
service-service communication. The idea is even if the service is on the host.
It must be assigned an identity and specific privileges to access other
services.

## User Authentication Service

User authentication service provides local user authentication for interactive
users on the system. This service provides primitives to add/delete/modify users
and groups on the system. It will provide APIs to verify user credentials to
create user sessions by other services like SSH or product-specific user
interfaces.

## Log Forward Service

Log forward service provides syslog and journald forwarders to forward logs to
remote systems for management and monitoring of the appliances.

## SSH Service

SSH service allows interactive users to log in and manage the appliance. This
service is optional and can be disabled if remote login is not required.

## NTP Service

NTP service can sync the system time with a remote NTP server. The service
configuration will define the servers to sync with. This service will provide
APIs to query the system time and the NTP sync status.

## DNS Client Service

DNS client service provides a way to configure external DNS services, hostnames,
and domain names for all the processes on the appliance.

## System Clean Workflow

In certain recovery cases, the entire system configuration can be corrupt, and
the system must restore the appliance to the first boot state, also called the
factory state. An operational command can record the clean request with the
configuration service. The entire configuration is dropped on the next reboot,
making the system bootstrap again as if it were a fresh appliance.

## System Restore Workflow

Given that we have a canonical model for maintaining the entire system config in
one place. Defining a backup and restore procedure for quick system recovery to
a known last good state will now be possible. Importing a known backup is simply
applying the patch on the configuration store and rebooting the system.

## Putting it all together

Let's look at the use case to build a bare-metal appliance for
[zot](https://zotregistry.io). The product developer will follow these steps.

__Hardware Specifications:__

1. 24-core AMD64 CPU
1. 64GB of memory
1. 1x240GB ATA SSD
1. 16x2TB 10k RPM RAID HDD
1. 1x450GB RAID SSD
1. 1x25GB Intel 810 Fiber NIC

__PID:__ ZOT-SERVER-M1

__Machine Keys:__

Generate the machine keys for the appliance:

1. UEFI Shim Certificate
1. Kernel Certificate
1. Manifest Certificate
1. Zot Product Certificate

__Note__ that zot product certificate is signed by the machine manifest
certificate.

__Storage Policy:__

```yaml
storage:
  devices:
    - type: SSD
      min-size: 400G
      max-size: 1T
      count: 1
      attachment: RAID
    - type: HDD
      min-size: 2T
      max-size: 4T
      count: 16
      attachment: RAID
    - type: SSD
      min-size: 100G
      max-size: 250G
      count: 1
      attachment: ATA
  volume-groups:
    - name: vg0-ssd
      devices:
        - raid-ssd1
    - name: vg0-hdd
      devices:
        - raid-hdd1
        - raid-hdd2
        - raid-hdd3
        - raid-hdd4
        - raid-hdd5
        - raid-hdd6
        - raid-hdd7
        - raid-hdd8
        - raid-hdd9
        - raid-hdd10
        - raid-hdd11
        - raid-hdd12
        - raid-hdd13
        - raid-hdd14
        - raid-hdd15
        - raid-hdd16
```

__Network Policy:__

__TODO:__ Insert netplan config here.

__Factory Provisioning:__

1. BIOS Configuration
1. BIOS customization with UEFI Shim Certificate
1. Define a secure process to create the Device Identity Certificate
1. TPM Provisioning process

__Console Bootstrap:__

Capture the minimal information required to bootstrap the machine, like
the initial root password and IP connectivity.

__zot service configuration:__

```yaml
service:
  name: zot
  vendor: cisco
  description: Zot is an OCI container registry
  version: 0.1.0
  tags:
    - registry
    - oci
  compatibility: 
    machine: 
      architecture: x86_64 
      versions:
        - 0.2.*
        - 0.1.3
    service:
      versions:
        - 0.2.*
        - 0.1.4
    privileges:
      - STORAGE_USER
      - SERVICE_USER
    artifacts:
      image: zothub.io/cisco/zot:0.1.0
      image-signature: zothub.io/cisco/zot:0.1.0.sig
      manifest: zothub.io/cisco/zot:0.1.0.manifest
      manifest-signature: zothub.io/cisco/zot:0.1.0.manifest.sig 
      certificate: zothub.io/cisco/zot:0.1.0.cert 
    entry-point: /usr/bin/zotd -f /etc/services/zot/config/zotd.conf
    storage:
      - mount: /data
        type: HDD
        size: 32T
        fs: ext4
        encrypt: false
      - mount: /logs
        type: SSD
        size: 1G
        fs: ext4
        encrypt: false
      - mount: /tmp
        type: ramdisk
        fs: tmpfs 
        size: 1G 
    cgroup: 
      cpu: 27000 
      memory: 60G 
    network:
      ns: host
```

That's it; the product developer can now build the appliance and ship it!

## Conclusion

machineOS builds on a robust trust model to provide a secure container
OS and a set of reusable standard services. These batteries included but
swappable approach allows product designers to focus on their
application development while still having the ability to customize
almost every aspect of the system as needed by the product.
