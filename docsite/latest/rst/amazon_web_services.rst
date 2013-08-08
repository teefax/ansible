Amazon Web Services
===================

.. contents::
   :depth: 2

Introduction
````````````

Ansible contains a number of core modules for interacting with Amazon Web Services (AWS) and other API compatible private clouds, such as Eucalyptus.  (There are other supported cloud systems, but this documentation is about AWS ones).  This page illustrates some use cases for provisioning and managing hosts EC2 resources.

Requirements for the AWS modules are minimal.  All of the modules require and are tested against boto 2.5 or higher. You'll need this Python module installed on the execution host. If you are using Red Hat Enterprise Linux or CentOS, install boto from `EPEL <http://fedoraproject.org/wiki/EPEL>`_:

.. code-block:: bash

    $ yum install python-boto

.. note::

   Authentication with the AWS-related modules is handled by either 
   specifying your access and secret key as ENV variables or passing
   them as module arguments. 

.. note::

   To talk to specific endpoints, the environmental variable EC2_URL
   can be set.  This is useful if using a private cloud like Eucalyptus, 
   exporting the variable as EC2_URL=https://myhost:8773/services/Eucalyptus.
   This can be set using the 'environment' keyword in Ansible if you like.

Provisioning
````````````

The ec2 module provides the ability to provision EC2 instance(s).  Typically the provisioning task will be performed against your Ansible master server as a local_action. There are two ways to provision instances:

* **Non-idempotent provisioning** (default). With non-idempotent provisioning, each time the provision command is run, a new instance is provisioned regardless of any instances that are alredy running.
* **Idempotent provisioning** (enabled with the idempotent_attribute option). By using the idempotent_attribute, the provisioning will ensure that at least count=N instances are running.  If N instance are already running, no new instances will be provisioned.

Non-idempotent provisioning
+++++++++++++++++++++++++++

Each time the following ad-hoc command is run, five instances will be provisioned:

.. code-block:: bash

    ansible localhost -m ec2 -a "count=5 image=ami-6e649707 instance_type=m1.large keypair=mykey group=webservers wait=yes"

In a play, this might look like (assuming the parameters are held as vars)::

    tasks:
    - name: Provision a set of instances
      local_action: ec2 keypair={{mykeypair}} group={{security_group}} instance_type={{instance_type}} image={{image}} wait=true count={{number}}
      register: ec2
                  
By registering the return its then possible to dynamically create a host group consisting of these new instances.  This facilitates performing configuration actions on the hosts immediately in a subsequent play::

    tasks:
    - name: Add all instance public IPs to host group
      local_action: add_host hostname={{ item.public_ip }} groupname=ec2hosts
      with_items: ec2.instances

With the host group now created, the second play in your provision playbook might now have some configuration steps::

    - name: Configuration play
      hosts: ec2hosts
      user: ec2-user
      gather_facts: true

      tasks:
      - name: Check NTP service
        action: service name=ntpd state=started

The method above ties the configuration of a host with the provisioning step.  This isn't always ideal and leads us onto the next section.

Idempotent provisioning
+++++++++++++++++++++++
Idempotent provisioning provides a simple mechanism for maintaing a specified number of instances running in a particular host group.

Using the ec2 inventory plugin it is possible to group hosts by security group, machine image (AMI) or instance tags. Instance tags in particular provide a flexible way of marking instances as belonging to a particular host group.

The following example shows how one can idempotently provision a group of 5 hosts tagged as webservers::

    - local_action: 
	module: ec2 
	keypair: mykey 
	group: webservers
	instance_type: m1.large 
	image: ami-6e649707 
	wait: yes 
	count: 5 
	instance_tags: '{"name":"webserver"}'
	idempotency_attribute: instance_tags

If this play is run when 3 EC2 instances with the tag `'{"name":"webserver"}'` are already running, then only two more will be provisioned in order to bring the total up to 5. If five such instances are already running, then no new instances will be provisioned. If you wanted to refer to this group at some point in the future, then make sure the EC2 inventory plugin is enabled and select the hosts using::

    - hosts: tag_Name_webservers
      gather_facts: false
      sudo: true

      tasks:
      ...

If the above playbook were called `provision_webservers.yml`, then it could be run from the command line using

```bash
ansible-playbook provision_werbservers.yml -i hosts/
```
where the `hosts/` folder contains both files defining host groups, and the `ec2.py` inventory plugin. Only by putting all of these files together in a folder, and specifying that entire folder as the hosts location can locally defined hosts and those provided by the ec2 inventory plugin be combined.

Note that the value of the `idempotency_attribute` option can also be `image`, `group` (security group), `group_id` (security group id) or `client-token`. The `client-token` is set using the `id` option.


Advanced Usage
``````````````

Host Inventory
++++++++++++++

Once your nodes are spun up, you'll probably want to talk to them again.  The best way to handle his is to use the ec2 inventory plugin.

Even for larger environments, you might have nodes spun up from Cloud Formations or other tooling.  You don't have to use Ansible to spin up guests.  Once these are created and you wish to configure them, the EC2 API can be used to return system grouping with the help of the EC2 inventory script. This script can be used to group resources by their security group or tags. Tagging is highly recommended in EC2 and can provide an easy way to sort between host groups and roles. The inventory script is documented `in the API chapter <http://www.ansibleworks.com/docs/api.html#external-inventory-scripts>`_.

You may wish to schedule a regular refresh of the inventory cache to accommodate for frequent changes in resources:

.. code-block:: bash
   
    ./ec2.py --refresh-cache

Put this into a crontab as appropriate to make calls from your Ansible master server to the EC2 API endpoints and gather host information.  The aim is to keep the view of hosts as up-to-date as possible, so schedule accordingly. Playbook calls could then also be scheduled to act on the refreshed hosts inventory after each refresh.  This approach means that machine images can remain "raw", containing no payload and OS-only.  Configuration of the workload is handled entirely by Ansible.  

Pull Configuration
++++++++++++++++++

For some the delay between refreshing host information and acting on that host information (i.e. running Ansible tasks against the hosts) may be too long. This may be the case in such scenarios where EC2 AutoScaling is being used to scale the number of instances as a result of a particular event. Such an event may require that hosts come online and are configured as soon as possible (even a 1 minute delay may be undesirable).  Its possible to pre-bake machine images which contain the necessary ansible-pull script and components to pull and run a playbook via git. The machine images could be configured to run ansible-pull upon boot as part of the bootstrapping procedure. 

More information on pull-mode playbooks can be found `here <http://www.ansibleworks.com/docs/playbooks2.html#pull-mode-playbooks>`_.

(Various developments around Ansible are also going to make this easier in the near future.  Stay tuned!)

Use Cases
`````````

This section covers some usage examples built around a specific use case.

Example 1
+++++++++

    Example 1: I'm using CloudFormation to deploy a specific infrastructure stack.  I'd like to manage configuration of the instances with Ansible.

Provision instances with your tool of choice and consider using the inventory plugin to group hosts based on particular tags or security group. Consider tagging instances you wish to managed with Ansible with a suitably unique key=value tag.

Example 2
+++++++++

    Example 2: I'm using AutoScaling to dynamically scale up and scale down the number of instances. This means the number of hosts is constantly fluctuating but I'm letting EC2 automatically handle the provisioning of these instances.  I don't want to fully bake a machine image, I'd like to use Ansible to configure the hosts.

There are two approaches to this use case.  The first is to use the inventory plugin to regularly refresh host information and then target hosts based on the latest inventory data.  The second is to use ansible-pull triggered by a user-data script (specified in the launch configuration) which would then mean that each instance would fetch Ansible and the latest playbook from a git repository and run locally to configure itself.

Example 3
+++++++++

    Example 3: I don't want to use Ansible to manage my instances but I'd like to consider using Ansible to build my fully-baked machine images.

There's nothing to stop you doing this. If you like working with Ansible's playbook format then writing a playbook to create an image; create an image file with dd, give it a filesystem and then install packages and finally chroot into it for further configuration.  Ansible has the 'chroot' plugin for this purpose, just add the following to your inventory file::

    /chroot/path ansible_connection=chroot

And in your playbook::

    hosts: /chroot/path

Pending Information
```````````````````

In the future look here for more topics.

Using Ansible's S3 module
+++++++++++++++++++++++++

these modules are documented on the module page, more walk throughs coming soon

Using Ansible's Elastic Load Balancer Support
+++++++++++++++++++++++++++++++++++++++++++++

these modules are documented on the module page, more walk throughs coming soon

Using Ansible's Cloud Formation Module
++++++++++++++++++++++++++++++++++++++

these modules are documented on the module page, more walk throughs coming soon

.. seealso::

   :doc:`examples`
       Examples of basic commands
   :doc:`playbooks`
       Learning ansible's configuration management language
   `Mailing List <http://groups.google.com/group/ansible-project>`_
       Questions? Help? Ideas?  Stop by the list on Google Groups
   `irc.freenode.net <http://irc.freenode.net>`_
       #ansible IRC chat channel

