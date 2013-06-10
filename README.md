# Deploying OpenStack with Ansible

- Requires Ansible 1.2
- Expects CentOS/RHEL 6 hosts (64 bit)
- Kernel Version 2.6.32-358.2.1.el6.x86_64


## A Primer into OpenStack Architecture

![Alt text](/images/os_architecture.png "Arch")

As the above diagram depicts, there are several key components that make up the
OpenStack Cloud Infrastructure. A brief description about the services are as
follows:

- Dashboard (Horizon): A Web based GUI that provides an interface for the end
user or Administrator to interact with the OpenStack cloud infrastructure. 

- Nova: The Nova component is primarily responsible for taking API requests
from endusers/admins to create virtual machines and coordinate in
creating/delivering them. The Nova component consists of several
subcomponents/processes which helps Nova deliver the virtual machines. Let's
have a brief look them:

 - nova-api: Runs the API server, which accepts and responds to the
enduser/admin requests.

 - nova-compute: Runs on the compute nodes and interacts with hypervisors to
create/terminate virtual machines.

 - nova-volume: Responsible for providing persistant storage to VMs--it is
gradually being migrated to Cinder.

 - nova-nework: Responsible for networking requirements like adding firewall
rules, creating bridges, etc. This component is also slowly being migrated to
Quantum Services.

 - nova-scheduler: Responsible for choosing the best compute server where the
requested virtual machine should run.

 - nova-console: Responsible for providing the VM's console via VNC to end
users.
   
- Glance: This service provides the endusers/admins with options to add/remove
images to the OpenStack Infrastructure, which would be used to deploy virtual
machines.

- Keystone: The Keystone component is repsonsible for providing identity and
authentication services to all other services that require these services.

- Quantum: Quantum is primarily responsible for providing networks and
connectivity to the Nova instances.

- Swift: The Swift service provides endusers/admins with a higly scalable and
fault-tolerant object store.

- Cinder: Provides permanant disk storage services to Nova instances.


## Ansible's Example Deployment Diagram

![Alt text](/images/os_dep_diagram.png "diagram")

The above diagram shows Ansible playbooks which deploy OpenStack, combining all
the management processes into a single node (controller node) and the compute
service into the other nodes (compute nodes). 

The management services deployed/configured on the controller node are as
follows:

- Nova: Uses MySQL to store VM info and state. The communication between
several Nova components are handled by Apache QPID messaging system.

- Quantum: Uses MySQL as its backend database and QPID for interprocess
communication. The Quantum service uses the OpenVSwitch plugin to provide
network services. L2 and L3 services are taken care by quantum while the
firewall service is provided by Nova.

- Cinder: Uses MySQL as backend database and provides permanant storage to
instaces via lvm and tgtd daemon(ISCSI).

- Keystone: Uses MySQL to store the tenant/user details and provides identity
and authorization service to all other components.

- Glance: Uses MySQL as the backend database, and images are stored on the
local filesystem.

- Horizon: Uses memcached to store user session details and provides UI
interface to endusers/admins.

The playbooks also configure the compute nodes with the following components:

- Nova Compute: The Nova Compute service is configured to use qemu as the
hypervisor and uses libvirt to create/configure/teminate virtual machines.

- Quantum Agent: A Quantum Agent is also configured on the compute node which
uses OpenVSwitch to provide L2 services to the running virtual machines.


## Physical Network Deployment Diagram      

![Alt text](/images/os_phy_network.png "phy_net")

The diagram in Fig 1.a shows a typical OpenStack production network setup which
consists of four networks: A management network for management of the servers,
a data network which would have all the inter-VM traffic flowing through it,
an external network which would faciliate the flow of network traffic destined
to an outside network; this network/interface would be attached to the network
controller. The fourth network is the API network, which would be used by the
endusers to access the APIs for cloud operations.

For the sake of simplicity, the Ansible example deployment simplifies the
network topology by deploying two networks: the first network would handle the
traffic for management/API and data, while the second network/interface would
be for the external access.


### Logical Network Deployment Diagram

![Alt text](/images/os_log_network.png "log_network")

OpenStack provides several logical network deployment options like flat
networks, provider router with private networks, tenant routers with private
networks. The above diagram shows the network deployment carried out by the
Ansible playbooks, which is refered to as provider router with private
networks.

As it can be seen above all the tenants would get their own private networks
which can be one or many. The provider, in this case the cloud admin would have
a dedicated router which would have an external interface for all outgoing
traffic. Incase any of the tenants require external access (internet/other
datacenter networks) they could attach thier private network to the router and
get external access.

     
## Network Under The Hood

![Alt text](/images/os_network_detailed.png "Detailed network")

As described previousely the Ansible example playbooks deploys OpenStack with a
network topology of "Provider Router with private networks for tenants". The
above diagram gives a brief description of the data flow in this network toplogy.

As the first step during deployment the Ansible playbooks create a few OpenVSwitch
software bridges on the controller and compute nodes. They are as follows:


### Controller Node:

- br-int: Known as the integration bridge will have all the interfaces of the
router attached to it.

- br-ext: Known as the external bridge, this will have the interfaces attached
to it that have an external IP address. For example, when an external network
is created and gateway is set, a new interface is created and attached to the
br-ext bridge with the gateway IP. All floating IPs will have an interface
created and attached to this bridge.  A physical NIC is also added to this
bridge so that external traffic flows through this NIC to reach the next
router.


### Compute Node:

- br-int: This integration bridge will have all the interfaces from virtual
machines attached to it.

Apart from the above bridges there will be one more bridge automatically
created by Openstack for communication between computenode and network
controller node.

- br-tun: The tunnel bridge are created on both the compute nodes and the
network controller node, they will have two interface attached to it. 

    - A patch port/interface: This acts like an uplink between the integration
bridge and the tunnel bridge, so that all the traffic at the intergration
bridge will be forwarded to the tunnel bridge.

    - A GRE Tunnel port/interface: A gre tunnel basically encapsulates all the
traffic incoming to its interface with its own IP address and delivers it to
the opposite endpoint. So in our case all the compute nodes will have a tunnel
set up with network controller as its remote endpoint. So the tunnel interface
in the br-tun bridge in compute node delivers all the VM traffic coming from
the br-int interface to the tunnel endpoint interface in the controller's
br-tun bridge.


## Network Operations In Detail

### A private tenant network is created:

When a private network for a tenant is created, the first task is to: 

- Create an interface and assign an IP from that range and attach it to the
br-int bridge on the controller node. This interface will be used by the
dhcp-agent process running on the node to give out dynamic IP addresses to VMs
that get created on this subnet. So as the above diagram shows if the tenant
subnet created is 10.0.0.0/24, an IP of 10.0.0.3 would be added to a vnic and
added to br-int bridge, and this interface would be used by the dnsmasq process
to give IPs to VMs in this subnet.

- Also an virtual interface would be created and added to the br-int bridge,
this interface would typically be given the first ip in subnet created and
would as the default gateway for instances that have an interface attached to
this subnet. So if the subnet created is 10.0.0.0/24 an ip of 10.0.0.1 would be
given to the router interface which would be attached to the br-int bridge.


### A new VM is created:

As per the above diagram, consider that a tenant VM is created and running on
the 10.0.0.0/24 subnet with an IP of 10.0.0.5 on a compute node.

- During the VM creation process the OpenVSwitch agent in the compute node
would have attached the vNIC of the VM to br-int bridge. The br-tun bridge
would have been set up with a tunnel interface with source as the compute node
and the remote endpoint as the network controller.

- A patch interface would have been created between the two bridges so that all
traffic in the br-int bridge are forwarded to the br-tun interface.


### External network is created and gateway set on router:

As the figure above depicts, an external network is used to route the traffic
from the internal/private OpenStack networks to the external network/internet.
External network creation expects a physical interface added to the br-ex
bridge and that physical interface can transfer packets to next hop router for
the external network. As per the above diagram 1.1.1.1 would be an interface on
the next hop router for the OpenStack controller.

- The first thing OpenStack does on external network creation is that it
creates a virtual interface by the name "qg-xxx" on the br-ex bridge and
assigns the second IP on the subnet to this interface. eg: 1.1.1.2

- OpenStack also adds a default route to the controller, which would be the
first IP on the subnet. Eg: 1.1.1.1


### Associate Floating IP to a VM:

Floating IPs are used to give access to the VM from external networks, and to
give external access to the VMs in the private network.

- When a floating IP is associated with a VM, a virtual interface is created in
the br-ex bridge and the public IP is assigned to this interface.

- A Destination NAT is setup in the iptables rules so that the traffic coming
to a particular external IP would be translated to its corresponding private
IP.

- A Source NAT is setup in the iptables so that traffic originating from a
particular private IP is NATted to its corresponding external IP so that it can
access external network.

 
### A Step by Step look a VM in a private network accessing internet:

As per the above diagram, consider the tenant VM is on the 10.0.0.0/24 subnet
and has an IP of 10.0.0.5. An external network is created with a subnet of
1.1.1.0/24 and it attached to the external router. A floating IP is also given
to the tenant's VM: 1.1.1.10. So in case of the VM pinging an external IP, the
sequence of events would be thus.

1. Since the destination is in a external subnet, the VM would try to forward
the packet to its default gateway, which in this case would be 10.0.0.1 which
is a virtual interface in the br-int bridge on the controller node.

2. The packet traverses through the patch interface in the br-int bridge to
the br-tun bridge.

3. The br-tun bridge on the compute node has tunnel endpoint with
192.168.2.42 as source, and remote endpoint as 192.168.2.41 which is the data
interface of controller node. The packet then gets encapsulated in the GRE
packets and get transferred to the network controller node.

4. Upon reaching the br-tun bridge in the network controller, the GRE packet
is stripped and forwarded to the br-int bridge through the patch interface.

5. The 10.0.0.1 interface intercepts the packet and based on the routing
rules forwards the packet to appropriate interface, eg., 1.1.1.2.

6. The SNAT rule on the post-routing chain of iptables NATs the source IP
from 10.0.0.5 to 1.1.1.10 and the packet is sent out to next hop router:
1.1.1.1.

7. On receiving a reply, the pre-routing chains does a destination NAT and
the 1.1.1.10 is changed to 10.0.0.5 and follows the same route to reach the VM.

 
## Deploying OpenStack with Ansible

As discussed above this example deploys OpenStack in an all-in-one model, where
all the management services are deployed on a single host (controller node) and
the hypervisors (compute nodes) are deployed on the other nodes. 


#### Prerequisites:

- CentOS version 6.4 (64-bit)
- Kernel Version: 2.6.32-358.2.1.el6.x86_64 
- Controller Node: Should have an extra NIC (for external traffic routing) and
an extra disk or partition for Cinder Volumes.

Before the playbooks are run please make sure that the following parameters in
group_vars/all are suited to your environment:

- quantum_external_interface: eth2 # The nic that would be used for external
traffic. Make sure there is no IP assigned to it and it is in enabled state.

- cinder_volume_dev: /dev/vdb # An additional disk or a partition that would be
used for cinder volumes. Please make sure that it is not mounted or any other
volume groups are created on this disk.

- external_subnet_cidr: 1.1.1.0/24 # The subnet that would be used for external
network access. If you are just testing, any subnet should be fine, and the VMs
should be accesible from the controller using this external IP. If you need the
Ansible host to access the VMs using this public IP, make sure Ansible host has
an IP in this segment and that interface is on the same broadcast domain as the
second interface on the network controller.


### Operation:

Once the prerequisites are satisfied and the group variables are changed to
suit your environment, modify the inventory file 'hosts' to match your
environment. Here's an example inventory:

        [openstack_controller]
        openstack-controller

        [openstack_compute]
        openstack-compute

Run the playbook to deploy the OpenStack Cloud:

        ansible-playbook -i hosts site.yml

Once the playbooks complete you can check the deployment by logging into the
Horizon console http://controller-ip/dashboard. The credentials for login would
be as defined in group_vars/all. The state of all services can be verified by
checking the "system info" tab.

![Alt text](/images/os_sysinfo.png "sysinfo")


### Uploading an Image to OpenStack

Once OpenStack is deployed, an image can be uploaded to Glance for VM creation
using the following command. This uploads a cirros image from a an URL directly
to Glance.

Note: if your external network is not a valid network, this download of image
from the Internet might not work as OpenStack adds a default route to your
controller to the first IP of external network. So please make sure you have
the right default route set in your controller.

        ansible-playbook -i hosts playbooks/image.yml -e "image_name=cirros image_url=https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img"

We can verify the image status by checking the "images" tab in Horizon.

![Alt text](/images/os_images.png "Images")


### Create a Tenant and their Private Network

The following command shows how to create a tenant and a private network. The
following command creates a tenant by the name of "tenant1" and creates an admin
tenant by the name of "t1admin" and also provisions a private network/subnet with
a cidr of 2.2.2.0/24 for their VMs.

        ansible-playbook -i hosts playbooks/tenant.yml -e "tenant_name=tenant1 tenant_username=t1admin tenant_password=abc network_name=t1net subnet_name=t1subnet subnet_cidr=2.2.2.0/24 tunnel_id=3"

The status of the tenant and the network can be verified from Horizon. The
tenant list would be available in the "Projects" tab.

![Alt text](/images/os_projects.png "projects")

The network topology would be visible in the "Projects -> Network Topology" tab.

![Alt text](/images/os_networks.png "networks")


### Creating a VM for the Tenant

To create a VM for a tenant we can issue the following command. The command
creates a new VM for tenant1. The VM is attached to the network created above,
t1net, and the Ansible user's public key is injected into the node so that
the VM is ready to be managed by Ansible.

        ansible-playbook -i hosts playbooks/vm.yml -e "tenant_name=tenant1 tenant_username=t1admin tenant_password=abc network_name=t1net vm_name=t1vm flavor_id=6 keypair_name=t1keypair image_name=cirros" 

Once created, the VM can be verified by going to the "Instances" tab in Horizon.

![Alt text](/images/os_instances.png "Instances")

We can also get the console of the VM from the Horizon UI by clicking on the
"Console" drop down menu.

![Alt text](/images/os_vnc.png "console")


### Managing OpenStack VMs Dynamically from Ansible

There are multiple ways to manage the VMs deployed in OpenStack via Ansible.

1. Use the inventory script. While creating the virtual machine pass the
metadata paramater to the vm specifying the group to which it should belong,
and the inventory script would generate hosts arranged by their group name.

2. If the requirement is to deploy and manage the virtual machines using the
same playbook, this can be achieved by writing a play to create virtual
machines and add those VMs dynamically to the inventory using 'add_host'
module. Later, write another play targeting group name defined in the add_host
module and implement the tasks to be carried out on those new VMs.

