ha-cluster-pacemaker
=========

Role for configuring and expanding basic pacemaker cluster on CentOS/RHEL 6/7/8 and Fedora 28/29/30/31/32 systems.

Requirements
------------

RHEL: It is expected that machines will already be registered and subscribed for access to 'High Availability' or 'Resilient storage' channels.

Fedora: Don't forget to set `ansible_python_interpreter=/usr/bin/python3` for Fedora hosts as shown in example inventory at the end of this README. On Fedora systems this role uses Python 3.

CentOS/RHEL 8: Don't forget to set `ansible_python_interpreter=/usr/libexec/platform-python` for CentOS/RHEL 8 hosts as shown in example inventory at the end of this README. On CentOS/RHEL 8 systems this role uses python from system (Python 3)

This role requires at least version `2.9` of `python-jinja2` library. When this role is run from RHEL/CentOS 7 system you may encounter issue with old `python-jinja2` package described in Issue #6. To get the updated version of python-jinja2 and its dependencies you can use following RPM repository - https://copr.fedorainfracloud.org/coprs/ondrejhome/ansible-deps-el7/.

pcs-0.10 on Fedora 29+ and CentOS/RHEL 8 is supported when using 'ondrejhome.pcs-modules-2' version 20.0.0 or later.

Ansible version 2.9.10 and 2.9.11 will fail with error `"'hostvars' is undefined"` when trying to configure remote nodes. This applies only when there is at least one node with `cluster_node_is_remote=True`.

Role Variables
--------------

  - user used for authorizing cluster nodes

    ```
    cluster_user: 'hacluster'
    ```

  - password for user used for authorizing cluster nodes

    ```
    cluster_user_pass: 'testtest'
    ```

  - group to which cluster user belongs (should be 'haclient')

    ```
    cluster_group: 'haclient'
    ```

  - name of the cluster

    ```
    cluster_name: 'pacemaker'
    ```

  - configuration of firewall for clustering, NOTE in RHEL/Centos 6 this replaces iptables configuration file!
  
    ```
    cluster_firewall: true
    ```

  - enable cluster on boot on normal (not pacemaker_remote) nodes
  
    ```
    cluster_enable_service: true
    ```

  - configure cluster with fence_xvm fencing device ?
    This will copy /etc/cluster/fence_xvm.key to nodes and add fencing devices to cluster
    NOTE: you need to define 'vm_name' in the inventory for each cluster node
  
    ```
    cluster_configure_fence_xvm: true
    ```

  - configure cluster with fence_vmware_soap fencing device ?
    This will install fence_vmware_soap fencing agent and configure it. When this is enabled you
    have to specify 3 additional variables with information on accessing the vCenter.
    NOTE: You also need to define 'vm_name' in the inventory for each cluster node specifying the name or UUID of VM
    as seen on the hypervisor or in the output of `fence_vmware_soap -o list` command.
    ```
    cluster_configure_fence_vmware_soap: false
    fence_vmware_ipaddr: ''
    fence_vmware_login: ''
    fence_vmware_passwd: ''
    ```
    You can optionally change the additional attributes passed to fence_vmware_soap using the variable `fence_vmware_options`.
    By default this variable enables encryption but disables validation of certificates.
    ```
    fence_vmware_options: 'ssl="1" ssl_insecure="1"'
    ```

  - configure cluster with fence_kdump fencing device ?
    This starts kdump service and defines the fence_kdump stonith devices.
    NOTE: if the kdump service is not started this won't work properly or at all
  
    ```
    cluster_configure_fence_kdump: false
    ```

  - (RHEL/CentOS) enable the repositories containing needed packages
    ```
    enable_repos: true
    ```

  - (RHEL only) enable the extended update (EUS) repositories containint packages needed
    ```
    enable_eus_repos: false
    ```

  - (RHEL only) enable the SAP Solutions update service (E4S) repositories containint packages needed
    ```
    enable_e4s_repos: false
    ```

  - (RHEL only) enable Beta repositories containint packages needed
    ```
    enable_beta_repos: false
    ```

  - (RHEL only) type of enable repositories, note that E4S repos have only 'ha' type available
    - ha - High-Availability
    - rs - Resilient Storage
    ```
    repos_type: 'ha'
    ```

  - (RHEL only) custom_repository allows enabling an arbitrarily named repository to be enabled.
    RHEL8 repo names can be found at http://downloads.redhat.com/redhat/rhel/rhel-8-beta/rhel-8-beta.repo
    ```
    custom_repository: ''

    ```

  - (CentOS only) install the needed packages from the CD-ROM media available at /dev/cdrom
    ```
    use_local_media: false
    ```

  - Enable or disable PCSD web GUI. By default the role keeps the default of installation means that
    PCSD web GUI is disabled on CentOS/RHEL 6.X and enabled on CentOS/RHEL 7.X. `true` or `false` can
    be passed to this variable to make sure that PCSD web GUI is enabled or disabled.
    ```
    enable_pcsd_gui: 'nochange'
    ```

  - Cluster transport protocol. By default this role will use what is default for give OS.
    For CentOS/RHEL 6.X this means 'udp' (UDP multicast) and for CentOS/RHEL 7.X this means 'udpu'
    (UDP unicast). This variable accepts following options: `default`, `udp` and `udpu`.
    ```
    cluster_transport: 'default'
    ```

  - Allow adding nodes to existing cluster when used with ondrejhome.pcs-modules-2 v16 or newer.
    ```
    allow_cluster_expansion: false
    ```

  - Cluster network interface. If specified the role will map hosts to IPv4 addresses from this interface.
    By default the IPv4 addresses from `ansible_default_ipv4` are used. For exmaple to use IPv4 addresses
    from interface `ens8` use `cluster_net_iface: 'ens8'`. Interface must exists on all cluster nodes.
    ```
    cluster_net_iface: ''
    ```

  - Redundant network interface. If specified the role will setup a corosync redundant ring using the default IPv4 from this interface.
    Interface must exist on all cluster nodes.
    ```
      rrp_interface: ''
    ```
    NOTE: you can define this variable either in defaults/main.yml, in this case the same rrp_interface name is used for all hosts in the hosts file.
          Either you specify an interface for each host present in the hosts file: this allows to use a specific interface name for each host (in the case they dont have the same interface name). Also note that instead of defining rrp_interface for a host, you can define rrp_ip: in this case this alternate ip is used to configure corosync RRP (this IP must be different than the host' default IPv4 address). This allows to use an alternate ip belonging to the same primary interface.


  - Whether to add hosts to /etc/hosts. By default an entry for the hostname
    given by `cluster_hostname_fact` is added for each host to `/etc/hosts`.
    This can be disabled by setting `cluster_etc_hosts` to `false`.

    ```
    cluster_etc_hosts: true
    ```

  - Which Ansible fact to use as the hostname of cluster nodes. By default this
    role uses the `ansible_hostname` fact as the hostname for each host. In
    some environments it may be useful to use the Fully Qualified Domain Name
    (FQDN) `ansible_fqdn` or node name `ansible_nodename`.

    ```
    cluster_hostname_fact: "ansible_hostname"
    ```

  - Whether the node should be setup as a remote pacemaker node. By default
    this is `false`, and the node will be a full member of the Pacemaker
    cluster.  Pacemaker remote nodes are not full members of the cluster, and
    allow exceeding the maximum cluster size of 32 full members. Note that
    remote nodes are supported by this role only on EL7 and EL8.

    ```
    cluster_node_is_remote: false
    ```

Example Playbook
----------------

Example playbook for creating cluster named 'test-cluster' enabled on boot, with fence_xvm and firewall settings

    - hosts: servers
      roles:
         - { role: 'ondrejhome.ha-cluster-pacemaker', cluster_name: 'test-cluster' }

Example for creating cluster named 'test-cluster' without configuring firewalling and without fence_xvm.
For cluster to get properly authorize it is expected that firewall is already configured or disabled.

    - hosts: servers
      roles:
         - { role: 'ondrejhome.ha-cluster-pacemaker', cluster_name: 'test-cluster', cluster_firewall: false, cluster_configure_fence_xvm: false }

Example playbook for creating cluster named 'vmware-cluster' with fence_vmware_soap fencing device.

    - hosts: servers
      vars:
        fence_vmware_ipaddr: 'vcenter-hostname-or-ip'
        fence_vmware_login: 'vcenter-username'
        fence_vmware_passwd: 'vcenter-password-for-username'
      roles:
         - { role: 'ondrejhome.ha-cluster-pacemaker', cluster_name: 'vmware-cluster', cluster_configure_fence_xvm: false, cluster_configure_fence_vmware_soap: true }

Example playbook for creating cluster named 'test-cluster' where `/etc/hosts` is not modified:

    - hosts: servers
      roles:
         - { role: 'ondrejhome.ha-cluster-pacemaker', cluster_name: 'test-cluster', cluster_etc_hosts: false }

Inventory file example for CentOS/RHEL and Fedora systems.

    [cluster-el]
    192.168.22.21 vm_name=fastvm-centos-7.6-21
    192.168.22.22 vm_name=fastvm-centos-7.6-22
    [cluster-fedora]
    192.168.22.23 vm_name=fastvm-fedora28-23 ansible_python_interpreter=/usr/bin/python3
    192.168.22.24 vm_name=fastvm-fedora28-24 ansible_python_interpreter=/usr/bin/python3
    [cluster-rhel8]
    192.168.22.25 vm_name=fastvm-rhel-8.0-25 ansible_python_interpreter=/usr/libexec/platform-python
    192.168.22.26 vm_name=fastvm-rhel-8.0-26 ansible_python_interpreter=/usr/libexec/platform-python
    [cluster-el-rrp]
    192.168.22.27 vm_name=fastvm-centos-7.6-21 rrp_interface=ens6
    192.168.22.28 vm_name=fastvm-centos-7.6-22 rrp_ip=192.168.22.29

Inventory file example with two full members and two remote nodes:

    [cluster]
    192.168.22.21 vm_name=fastvm-centos-7.6-21
    192.168.22.22 vm_name=fastvm-centos-7.6-22
    192.168.22.23 vm_name=fastvm-centos-7.6-23 cluster_node_is_remote=True
    192.168.22.24 vm_name=fastvm-centos-7.6-24 cluster_node_is_remote=True

Video examples of running role with defaults for:
  - CentOS 7.6 installing CentOS 7.6 two node cluster: https://asciinema.org/a/226466
  - Fedora 29 installing Fedora 29 two node cluster: https://asciinema.org/a/226467

License
-------

GPLv3

Author Information
------------------

WARNING: this is alpha-version quality proof-of concept role that still needs some polishing and is using custom modules 
         to interact with pacemaker through python. This is suitable for testing purposes only.

To get in touch with author you can use email ondrej-xa2iel8u@famera.cz or create a issue on github when requesting some feature.
