===========================
Infrastructure as a Service
===========================


------------------------------------------------------------------------------
Design and implement tools for configuration management and automated building
------------------------------------------------------------------------------

https://wiki.csc.fi/wiki/CORE/ForgePuppet


---------------------------------------------------------
Create customised and automatically provided Cloud Images
---------------------------------------------------------



-----------------------------------------------------
Automated Cloud Image Creation with Diskimage-Builder
-----------------------------------------------------

Introduction
------------

`Diskimage-Builder <https://github.com/openstack/diskimage-builder>`_ is a dedicated
cloud image building toolkit for OpenStack. It is used to create prepared Linux
disk images with updated packages for cloud environments.

Installation
------------


Redhat/CentOS
*************

Redhat6 support is not available anymore in the upstream repository. It is available
through CSC fork of the upstream repository for CentOS6 and Scientific-Linux6:

`Diskimage-Builder with CentOS6 support <https://github.com/CSC-IT-Center-for-Science/diskimage-builder/tree/centos6>`_

Likewise Scientific-Linux7 is only available through the CSC fork:

`Diskimage-Builder with Scientific-Linux7 support <https://github.com/CSC-IT-Center-for-Science/diskimage-builder/tree/scientific7>`_

For installing Diskimage-Builder on CentOS, the EPEL repository is needed to install the
package *python-pip*.


.. code-block:: bash

   sudo yum install epel-release

Create a dedicated user for image building. The home directory of the dedicated user
should have sufficient disk space, for image creation and image download cache.

Set password-less sudo rights for the dedicated user:

/etc/sudoers.d/imgbuild:

.. code-block:: bash

   imgbuild        ALL=(ALL)       NOPASSWD: ALL


Install software dependencies and clone the GIT repository for Diskimage-Builder
into the home directory of the dedicated user.

.. code-block:: bash

   sudo yum install parted qemu-img kpartx git python python-pip
   git clone https://github.com/openstack/diskimage-builder.git
   cd diskimage-builder


To be able to create CentOS6 and Scientific-Linux6 images,
the CSC fork of Diskimage-Builder has to be used:

.. code-block:: bash

   git clone https://github.com/CSC-IT-Center-for-Science/diskimage-builder.git
   cd diskimage-builder
   git checkout centos6


Likewise for Scientific-Linux7:

.. code-block:: bash

   git clone https://github.com/CSC-IT-Center-for-Science/diskimage-builder.git
   cd diskimage-builder
   git checkout scientific7


Install Diskimage-Builder in the home directory of the imgbuild user:

.. code-block:: bash

   pip install --user -r requirements.txt && python setup.py install --user


Add binaries path to *PATH* environment in .bash_profile:

.. code-block:: bash

   PATH=$PATH:$HOME/bin:~/.local/bin
   export PATH

Create temporary directory in the dedicated users home directory.
This directory will be passed as *TMP_DIR* environment variable.
Otherwise Diskimage-Builder will use */tmp* for image preparation.

.. code-block:: bash

   mkdir temp


Ubuntu
******

.. code-block:: bash

   sudo apt-get install parted qemu-utils kpartx git python python-pip
   git clone https://github.com/openstack/diskimage-builder.git
   cd diskimage-builder
   pip install --user -r requirements.txt && python setup.py install --user


Image Creation
--------------

Creating an image for OpenStack using the centos7 element to install centos7 on the image:

.. code-block:: bash

   disk-image-create --no-tmpfs -o centos7 --image-size 10 vm centos7

Creating an image for OpenStack using centos6 element and a localy provided base image:

.. code-block:: bash

   DIB_LOCAL_IMAGE=base_image_centos6.qcow2 disk-image-create --no-tmpfs -o centos6 \
   --image-size 10 vm centos6

Setting the temporary directory for image preparation:

.. code-block:: bash

   TMP_DIR=~/temp disk-image-create --no-tmpfs -o centos7 --image-size 10 vm centos7


Defining additional packages to be installed inside the new image:

.. code-block:: bash

   disk-image-create --no-tmpfs -o centos7 --image-size 10 vm centos7 -p apache2,mysql-server


Important Parameters
--------------------

Usage:
 
.. code-block:: bash

   disk-image-create [OPTIONS]... [ELEMENTS]...

Options:

=============== ========================================================================
 Parameter       Explanation
=============== ========================================================================
--no-tmpfs      do not use tmpfs (temp in RAM) but use normal disk temp directory
--image-size    image size in GB for the created image
--image-cache   directory location for cached images (default ~/.cache/image-create)
-a              set the architecture of the image (default: amd64)
-t              set the imagetype of the output image file (qcow2,tar) (default: qcow2)
-p              comma separated list of packages to install in the image
-o              image name of the output image file (default: image)
-x              turn on tracing
=============== ========================================================================

Caching
-------

Located per default in *~/.cache/image-create*, disk-image-create caches images and
packages here. Occasionally a faulty image or element can spoil the cache preventing
further image creation, *rm -rf ~/.cache/image-create* works as a remedy.
 
Elements
--------

Located in *$HOME/.local/share/Diskimage-Builder/elements*, these elements are used to
configure what type of virtual machine is created. A machine can be composed of several
elements. For pre made elements, documentation can be found in the element folder,
in the file *README.md*.

Additional elements can be included by setting the variable *ELEMENTS_PATH* as a
colon-separated path list, pointing to directories with additional elements.

Each VM requires an *operating-system* element, such as *centos6* or *ubuntu* and the *vm*
element for partition and filesystem setup. If an element is an *operating-system* element
is determined in the file *element-provides* inside the elements folder.

The element folder contains everything needed by the element:

 - download URLs for the images
 - configuration for yum or apt and other programs etc. For additional packages
   (installed using the *-p* flag)
 - the portability can be enhanced by mapping package names between distros in a
   *bin/map-packages* file.

 
Operating-System Elements
-------------------------

=============== ========================================================================
Parameter       Explanation
=============== ========================================================================
vm              sets up a partitioned disk and boot loader (mandatory for cloud images) 
ubuntu          create Ubuntu cloud images.                                             
centos6         create CentOS6 cloud images (only available through CSC fork)           
scientific6     create Scientific-Linux6 cloud images (only available through CSC fork) 
centos7         create CentOS7 cloud images                                             
scientific7     create Scientific-Linux7 cloud images (only available through CSC fork) 
fedora          create Fedora cloud images                                              
opensuse        create OpenSUSE cloud images                                            
=============== ========================================================================


CentOS 6
********

In the upstream project of Diskimage-Builder support for CentOS6 has been removed. There
are no base images currently available. To create CentOS6 images, a base image has to be
created and then provided to Diskimage-Builder via *DIB_LOCAL_IMAGE* environment.


Ubuntu
******

By default Diskimage-Builder builds the trusty Ubuntu version. To change the version,
define the following environment:

.. code-block:: bash

   export DIB_RELEASE=saucy


Base Image Creation with KVM
++++++++++++++++++++++++++++

.. code-block:: bash

   sudo virt-install -n centos -r 2048 \
   -l  http://mirror.centos.org/centos/6/os/x86_64/ \
   --graphics vnc --disk path=/mnt/centos-base,size=10 \
   -x "ks=http://192.168.122.1/cloud_base_image_centos6.cfg"

A kickstart file for installing CentOS6 cloud base image can be found at
*scripts/cloud_base_image_centos6.ks*.


Automatic Image Deployment on OpenStack
---------------------------------------

Clone shell scripts and crontab files from GIT repository:

.. code-block:: bash

   git clone https://github.com/CSC-IT-Center-for-Science/diskimage-builder-csc-automation.git .


Install OpenStack glance packages
---------------------------------

For uploading the cloud images to an OpenStack cloud, the glance client package needs to
be installed.

Redhat/CentOS
*************

.. code-block:: bash

   sudo yum install python-glanceclient

Ubuntu
******

.. code-block:: bash

   apt-get install python-glanceclient


Setup a cronjob to execute the script periodically
**************************************************

.. code-block:: bash

   crontab < scripts/crontab


A complete *openrc.sh* with credentials for OpenStack is required at *scripts/openrc.sh*.

On Redhat based systems, sudo is configured to require a tty, and must be disabled in
*/etc/sudoers*:

.. code-block:: bash

   #Defaults    requiretty

Controlling Access in Virtual Infrastructures
---------------------------------------------

Access control for the core Glenna infrastructure. It includes secure access to virtual
machines and technologies for securing hypervisor and guest OS:s.


Like any other IT system, openstack based cloud deployments are vulnerable to security threats. The openstack security guideline identifies four elements in the security domain - users, applications, servers or networks that share common trust requirements and expectations within a system. Of which we can roughly cataegorize applications, servers and networks as virtual infrastructure. In this section we focus on how access control can be done to such virtual infrastructure so that only authenticated and authorized users or services get access. 

The underlying virtualization technology with in openstack provides enterprise-level capabilities in any openstack deployment. As capabilities like scalability, efficency, uptime and the likes are controlled by it; important virtual infrastrcutre within openstack are also managed by the virtualization technology in use. The openstack security guideline and the Hypervisor Support matrix (https://wiki.openstack.org/wiki/HypervisorSupportMatrix) present detailed information about the differnt supported virtualization tools and thier access control capabilities. However such capabilities vary from one version of the tool to another or one openstack release to another; so we focus more on general issues related to the security of virtual infrastructures.

In virtualization the hypervisor has higher responsibility to secure the entire virtualization environment. Hypervisors have to be secure enough to block attacks directed to the hypervisor itself, the host and/or the guest OS. Attackers usually try to break out of a guest OS so that they can gain access to the hypervisor, other guest OSs, or the underlying host. Compromise of the hypervisor can give control of all of its guest Oss to the attacker which makes it a primary target. Guest OS isolation can be one preventive mehanism but Guest OSs are often not completely isolated from each other and from the host OS because that would affect some necessary functionalities. For example, many hosted virtualization solutions provide mechanisms called guest tools through which a guest OS can access files, directories, the copy/paste buffer, and other resources on the host OS or another guest OS. These communication mechanisms can inadvertently serve as an attack vector, such as transmitting malware or permitting an attacker to gain access to particular resources.  However, bare metal virtualization software does not offer such sharing capabilities and this makes them a preferred choice when considering such requirements.

In additon to guest-OS isolation, hypervisors mitigate attacks through the guest OS, by guest OS monitoring and image and snapshot management mechanisms. We discuss these mechanims briefly in the following sub-sections. 

Hypervisor Security
*******************
Hypervisors are responsible for various tasks within a virtualized environment and they can become a single point of failure during compromise. Due consideration shall be given to the security of hypervisors and  we extract the following important recommendations from the NIST document on how to secure hypervisors:

- Most hypervisor software currently only use passwords for access control; this may be too weak for some organizations’ security policies and may require the use of compensating controls, such as a separate authentication system used for restricting access to the host on which the virtualization management system is installed.
- Hypervisors can be managed in different ways, with some hypervisors allowing management through multiple methods. It is important to secure each hypervisor management interface, both locally and remotely accessible. The capability for remote administration can usually be enabled or disabled in the virtualization management system. If remote administration is enabled in a hypervisor, access to all remote administration interfaces should be restricted by a firewall. Also, hypervisor management communications should be protected. One option is to have a dedicated management network that is separate from all other networks and that can only be accessed by authorized administrators. Management communications carried on untrusted networks must be encrypted using FIPS-approved methods, provided by either the virtualization solution or a third-party solution, such as a virtual private network (VPN) that encapsulates the management traffic.
- Because of the hypervisor’s level of access to and control over the guest OSs, limiting access to the hypervisor is critical to the security of the entire system. The access options vary based on hypervisor type. Most bare metal hypervisors have access controls to the system. Typically, the access method is just username and password, but some bare metal hypervisors offer additional controls such as hardware token-based authentication to grant access to the hypervisor’s management interface. On some systems, there are different levels of authorization, such as allowing some users to view logs but not be able to change any settings or interact directly with the guest OSs. These view-only user accounts allow auditors and others to have sufficient access to meet their needs without reducing overall security.
- In contrast to bare metal solutions, hosted virtualization products rarely have hypervisor access controls: anyone who can launch an application on the host OS can run the hypervisor. The only access control is whether or not someone can log into the host OS. Because of this wide disparity in security, organizations should have security policies about which guest OSs can be run from bare metal hypervisors and which can be run from hosted virtualization hypervisors. Further, organizations running bare metal hypervisors should  - Install all updates to the hypervisor as they are released by the vendor. Most hypervisors have features that will check for updates automatically and install the updates when found. Centralized patch management solutions can also be used to administer updates.
- Restrict administrative access to the management interfaces of the hypervisor. Protect all management communication channels using a dedicated management network or the management network communications is authenticated and encrypted using FIPS 140-2 validated cryptographic modules.
- Synchronize the virtualized infrastructure to a trusted authoritative time server.
- Disconnect unused physical hardware from the host system. For example, a removable disk drive might be occasionally used for backups, but it should be disconnected when not actively being used for backup or restores. Disconnect unused NICs from any network.
- Disable all hypervisor services such as clipboard- or file-sharing between the guest OS and the host OS unless they are needed. Each of these services can provide a possible attack vector. File sharing can also be an attack vector on systems where more than one guest OS share the same folder with the host OS.
- Consider using introspection capabilities to monitor the security of each guest OS. If a guest OS is compromised, its security controls may be disabled or reconfigured so as to suppress any signs of compromise. Having security services in the hypervisor permits security monitoring even when the guest OS is compromised.
- Consider using introspection capabilities to monitor the security of activity occurring between guest OSs. This is particularly important for communications that in a non-virtualized environment were carried over networks and monitored by network security controls (such as network firewalls, security appliances, and network IDPS sensors).
- Carefully monitor the hypervisor itself for signs of compromise. This includes using self-integrity monitoring capabilities that hypervisors may provide, as well as monitoring and analyzing hypervisor logs on an ongoing basis.
- It is also important to provide physical access controls for the hardware on which the virtualization system runs. For example, hosted hypervisors are typically controlled by management software that can be used by anyone with access to the keyboard and mouse. Even bare metal hypervisors require physical security: someone who can reboot the host computer that the hypervisor is running on could alter some of the security settings for the hypervisor. It is also important to secure the external resources that the hypervisor uses, particularly data on hard drives and other storage devices.
- There are additional recommendations for hosted virtualization solutions for server virtualization. Hosted virtualization exposes the system to more threats because of the presence of a host OS. To increase the security of the host OS, minimize the number of applications other than the hypervisor that are ever run on the system. All unneeded applications should be removed. Those that remain should be restricted as much as possible to prevent malware from being inadvertently installed on the system. For example, a web browser is often used to download updates to the hypervisor, and also to read instructions and bulletins about the hypervisor. If the computer is intended to be exclusively used to run the hosted hypervisor, the web browser should have as many settings as possible adjusted to their highest security level.
- Because hosted virtualization systems are run under host OSs, the security of every guest OS relies on the security of the host OS. This means that there should be tight access controls to the host OS to prevent someone from gaining access through the host OS to the virtualization system and possibly changing its settings or modifying the guest OSs.
- There have been some concerns in the security community about designing hypervisors so that they cannot be detected by attackers. The motivation for this was to provide an additional layer of security that is invisible to the attacker, thus preventing successful attacks against the hypervisor and the host OS underneath it. One of the basic principles of security design, however suggests that the the design of a security solution shall not depend on its secrecy. In addition, hypervisors have various characteristics that permit attackers to detect their presence. Detection techniques include checking for hypervisor artifacts in processes, file system, registry, and memory; checking for hypervisor-specific processor instructions or capabilities; and checking for hypervisor-specific virtual hardware devices. These detection techniques are hypervisor implementation-dependent. Although hypervisor detection can be deterred by a vendor modifying the hypervisor’s implementation or hiding its identifiable software artifacts, it is not possible to completely hide all characteristics. When planning their virtualization security, organizations shall not assume that attackers will not be able to detect the presence of a hypervisor or the product type and version.


Host Security (what host security entails and recommended practices)
*************

Without virtualization, services are usually deployed on dedicated host for isolation. In virtualization, a single host can be used multiple VMs each running an OS and one or more services. This is one of the advantages of virtualization but it at the same time makes the security of the host more critical as a compromise can affect multiple services. The increase in the number of services running on a single host increases the attack vector and a compromise on any of the running services can open a security hole on the other services sharing the same host. A holistic view of the system operations as well as security solutions is required for any of our actions on the host as well as the VMs and services running on it. Apart from that security solutions and policies that apply for any host in the organization’s environment can be applied on hosts involved in a virtualized environment. 

Guest OS Security (what guest security entails and recommended practices) 
*****************

Virtualization can be taken as an additional layer of security when it comes to guest OS security. In a non-virtualized environment, a compromise in the underlying OS can affect the hardware resources in the host machine. However, in virtualization access to hardware resources by the guest OS is mediated through hypervisors. Compromise in the guest OS cannot directly lead to misuse or compromise of the underlying hardware in the host machine. But as described in the next section, Guest OSs are often not completely isolated from each other and from the host OS. Which makes a compromise on the guest OS as a means to gain access to other guest OSs, the host OS and the hypervisor. We suggest the following recommendations that are extracted from the NIST … document

- Organizations that have security policies that cover network shared storage should apply those policies to shared disks in virtualization systems.
- Follow the recommended practices for managing the physical OS, e.g., time synchronization, log management, authentication, remote access, etc.
- Install all updates to the guest OS promptly. All modern OSs have features that will automatically check for updates and install them.
- Back up the virtual drives used by the guest OS on a regular basis, using the same policy for backups as is used for non-virtualized computers in the organization.
- In each guest OS, disconnect unused virtual hardware. This is particularly important for virtual drives (usually virtual CDs and floppy drives), but is also important for virtual network adapters other than the primary network interface and serial and/or parallel ports.
- Use separate authentication solutions for each guest OS unless there is a particular reason for two guest OSs to share credentials.
- Ensure that virtual devices for the guest OS are associated only with the appropriate physical devices on the host system, such as the mappings between virtual and physical NICs.

If a guest OS on a hosted virtualization system is compromised, that guest OS can potentially infect other systems on the same hypervisor. The most likely way this can happen is that both systems are sharing disks or clipboards. If such sharing is turned on in two or more guest OSs, and one guest OS is compromised, the administrator of the virtualization system needs to decide how to deal with the potential compromise of other guest OSs. Two strategies for dealing with this situation are:

- Assume that all guest OSs on the same hardware have been compromised. Revert each guest OS to a known-good image that was saved before the compromise.
- Investigate each guest OS for compromise, just as one would during normal scanning for malware. If malware is found, follow the organization’s normal security policy.

The first method assumes that guest OSs are different than “regular” systems, while the second assumes that the organization’s current security policy is sufficient and should be applied to all systems in the same manner.


Guest OS Isolation
******************

In virtualization the hypervisor is responsible for managing guest OS access to hardware (e.g., CPU, memory, storage). The hypervisor partitions these resources so that each guest OS can access its own resources but cannot encroach on the other guest OSs’ resources or any resources not allocated for virtualization use. This prevents unauthorized access to resources and also helps prevent one guest OS from injecting malware into another, such as infecting a guest OS’s files or placing malware code into another guest OS’s memory. Separately, partitioning can also reduce the threat of denial of service conditions caused by excess resource consumption in other guest OSs on the same hypervisor.

Another motivation for isolating guest OSs from each other and the underlying hypervisor and host OS is the mitigation of side-channel attacks. These attacks exploit the physical properties of hardware to reveal information about usage patterns for memory access, CPU use, and other resources. A common goal of these attacks is to reveal cryptographic keys. These attacks are considered difficult, usually requiring direct physical access to the host.

Guest OS Monitoring
*******************

The hypervisor is fully aware of the current state of each guest OS it controls. As such, the hypervisor may have the ability to monitor each guest OS as it is running, which is known as introspection. Introspection can provide full auditing capabilities that may otherwise be unavailable. Monitoring capabilities provided through introspection can include network traffic, memory, processes, and other elements of a guest OS. For many virtualization products, the hypervisor can incorporate additional security controls or interface with external security controls and provide information to them that was gathered through introspection. Examples include firewalling, intrusion detection, and access control. Many products also allow the security policy being enforced through hypervisor-based security controls to be moved as a guest OS is migrated from one physical host to another.

Network traffic monitoring is particularly important when networking is being performed between two guest OSs on the host or between a guest OS and the host OS. Under typical network configurations, this traffic does not pass through network-based security controls, so host-based security controls should be used to monitor the traffic instead.

Image and Snapshot Management
*****************************

Image and snapshot management is an often neglected part of cloud environments and big organizations need some standard routines on this as the number of images and snapshots can go very high and problems that arise due to unpatched or vulnerable images can increase the risk of security attacks on organizations' IT infrastructure. The NIST Guide to Security for Full Virtualization Technologies suggests the recommendations shown below.

Creating guest machine images and snapshots does not affect the vulnerabilities within them, such as the vulnerabilities in the guest OSs, services, and applications. However, images and snapshots do affect security in several ways, some positive and some negative, and they also affect IT operations.

Note that one of the biggest security issues with images and snapshots is that they contain sensitive data (such as passwords, personal data, and so on) just like a physical hard drive. Because it is easier to move around an image or snapshot than a hard drive, it is more important to think about the security of the data in that image or snapshot. Snapshots can be more risky than images because snapshots contain the contents of RAM memory at the time that the snapshot was taken, and this might include sensitive information that was not even stored on the drive itself.

An operating system and applications can be installed, configured, secured, and tested in a single image and that image then distributed to many hosts. This can save considerable time, providing additional time for the contents of the image to be secured more effectively, and also improve the consistency and strength of security across hosts. However, because images can be distributed and stored easily, they need to be carefully protected against unauthorized access, modification, and replacement. Some organizations need to have a small number of known-good images of guest OSs that differ, for example, based on the application software that is installed.

As the use of server and desktop virtualization as well as cloud computing infrastructure grows within an organization, the management of images can become a significant challenge. Some virtualization products offer management solutions that can examine stored images and update them as needed, such as applying patches and making security configuration changes, but other products offer no way of applying updates other than loading each image. For these products, the longer an image is stored without running it, the more vulnerabilities it is likely to contain when it is loaded again. It may be necessary to track all images and ensure that each non-archival image is periodically updated. Tracking images may also be a significant problem, particularly if users and administrators are able to create their own images. These images may also not be secured properly, especially if they are not based on a security baseline (e.g., the one provided by a different pre-secured image). This could increase the risk of compromise.

Another potential problem with increasing the use of virtualization in particular is the proliferation of images, also known as sprawl. It is easy to create a new image — it can often be done in just a few minutes, albeit without any consideration of security—so unnecessary images may be created and run. Each additional image running is another potential point of compromise for an attacker. Also, each additional image is another image that has to have its security maintained. Therefore, organizations should minimize the creation, storage, and use of unnecessary images. Organizations should consider implementing formal image management processes that govern image creation, security, distribution, storage, use, retirement, and destruction, particularly for server virtualization. Similar consideration should be given to snapshot management. In some cases, organizations have policies to not allow storage of snapshots because of the risk of malware from infected systems being stored in snapshots and later reloaded.

Image management can provide significant security and operational benefits to an organization. For example, if the contents of an image become compromised, corrupted, or otherwise damaged, the image can quickly be replaced with a known good image. Also, snapshots can serve as backups; permitting the rapid recovery of information added to the guest OS since the original image was deployed. One of the drawbacks associated with this type of backup is that incremental or differential backups of the system may not be feasible unless those backups are supported by the hypervisor. If a modification is made to the guest OS after a snapshot has been captured, the original snapshot will not include the modification. So, a new snapshot will need to be applied. Because of this, snapshot management needs to be considered as part of image management.

Image files can be monitored to detect unauthorized changes to the image files; this can be done by calculating cryptographic checksums for each file as it is stored, then recalculating these checksums periodically and investigating the source of any discrepancies. Image files can also be scanned to detect rootkits and other malware that, when running, conceal themselves from security software present within the guest OS.


Secure Virtualization Planning and Deployment (recommended security practices in the plan and deployment of virtualization)
*********************************************

A critical aspect of deploying a secure virtualization solution is careful planning prior to installation, configuration, and deployment. This helps ensure that the virtual environment is as secure as possible and in compliance with all relevant organizational policies. Based on the NIST SP 800-64 standard on Security Considerations in the Information System Development Life Cycle we suggest the following 5 phases for a secure virtualization environment planning and deployment.  Please refer the document for a more detailed description of each of these phases.

Phase 1: Initiation. This phase includes the tasks that an organization should perform before it starts to design a virtualization solution. These include identifying needs for virtualization, providing an overall vision for how virtualization solutions would support the mission of the organization, creating a high-level strategy for implementing virtualization solutions, developing virtualization policy, identifying platforms and applications that can be virtualized, and specifying business and functional requirements for the solution.

Phase 2: Planning and Design. In this phase, personnel specify the technical characteristics of the virtualization solution and related components. These include the authentication methods and the cryptographic mechanisms used to protect communications. At the end of this phase, solution components are procured.

Phase 3: Implementation. In this phase, equipment is configured to meet operational and security requirements, installed and tested as a prototype, and then activated on a production network. Implementation includes altering the configuration of other security controls and technologies, such as security event logging, network management, and authentication server integration.

Phase 4: Operations and Maintenance. This phase includes security-related tasks that an organization should perform on an ongoing basis once the virtualization solution is operational, including log review, attack detection, and incident response.

Phase 5: Disposition. This phase encompasses tasks that occur when a virtualization solution is being retired, including preserving information to meet legal requirements, sanitizing media, and disposing of equipment properly.


Security architecture and specification
---------------------------------------

Develop the specification and architecture of the security components in Glenna

The Openstack foundation has identified the following core points on the main reasons why we need federated identity in cloud services and the challenges associated with it [http://docs.openstack.org/security-guide/identity/federated-keystone.html]:

- Provisioning new identities often incurs some security risk. It is difficult to secure credential storage and to deploy it with proper policies. A common identity store is useful as it can be set up properly once and used in multiple places. With Federated Identity, there is no longer a need to provision user entries in Identity service, since the user entries already exist in the IdP’s databases.
- This does introduce new challenges around protecting that identity. However, this is a worthwhile tradeoff given the greater control, and fewer credential databases that come with a centralized common identity store.
- It is a burden on the clients to deal with multiple tokens across multiple cloud service providers. Federated Identity provides single sign on to the user, who can use the credentials provided and maintained by the user’s IdP to access many different services on the Internet.
- Users spend too much time logging in or going through ‘Forget Password’ workflows. Federated identity allows for single sign on, which is easier and faster for users and requires fewer password resets. The IdPs manage user identities and passwords so OpenStack does not have to.
- Too much time is spent administering identities in various service providers.
- The best test of interoperability in the cloud is the ability to enable a user with one set of credentials in an IdP to access multiple cloud services. Organizations, each using its own IdP can easily allow their users to collaborate and quickly share the same cloud services.
- Removes a blocker to cloud brokering and multi-cloud workload management. There is no need to build additional authentication mechanisms to authenticate users, since the IdPs take care of authenticating their own users using whichever technologies they deem to be appropriate. In most organizations, multiple authentication technologies are already in use.

Security overview in Federated Cloud Infrastructure (what makes federated cloud different when it comes to security)
***************************************************

Security in any IT system entails the Confidentiality, Integrity and Availability of the system as a whole and its individual components (data and infrastructure). Confidentiality ensures that information is accessible only to authorized people and if unauthorized entitles get access to data, they shouldn´t get any valuable information out of it. With integrity a system shall block un-authorized modification of data. The third pillar deals with making systems available for users. So that the system functions properly in which ever circumstance. These basic three pillars still apply to Cloud Computing Systems and the change is in how one can keep the Confidentiality, Integrity and Availability of systems. 

Unlike the traditional computing environments, cloud computing has brought its own specialties for the good of systems running on it. For instance the resource sharing, scalability, on-demand resource pooling and the likes are interesting characteristics in cloud computing.  When it comes to security however each characteristics brings new challenges in the way one can keep the CIA of cloud computing environments.

In the context of Glenna, the challenges in implementing security services like authentication and authorization are eased because of the federated authentication scheme. In such environments, the requirements to keep the confidentiality and integrity of data are more or less handled by existing authentication and authorization systems. The Kalmar2 and EduGAIN federated authentication systems have both been used in various systems within the academic and research environments and well tested. The 4 national level Identity Providers (IdPs) namely FEIDE, SWAM-ID, HAKA and WAYF are also currently being used in the 5 Nordic counties and in continuous development to deal with associated security risks. It will then be a matter of identifying in what way Glenna can use this existing systems rather than dealing with the security aspect. But when it comes to individual cloud service providers involved in the Federation, a due consideration of the security aspects of the respective systems and infrastructures is needed.

As the individual cloud service providers also use own local authentication and authorization mechanisms, a security analysis to identify the strength and weaknesses of local systems will be very helpful. A chain is as strong as its weakest link and a security problem in any of the services can induce a general security problem to the entire federation.

Apart from keeping the confidentiality and integrity of systems and data, availability shall also be given due emphasis. Currently Business Continuity, Disaster Recovery and Resilience are becoming important characteristics of IT services including the cloud and we suggest some approaches to attain such characteristics in the services involved in Glenna Federation.

Based on what we have experienced so far there are systems and solutions in the Nordics that can aid the security of the Glenna Federation and we try to come up with security solutions that can be added on top of the existing ones and makes the entire system more secure.  In the next section we go through the suggested security architecture and security components specifications for Glenna.

Security architecture and specifications for Glenna (suggested security architecture, detailed specifications of its security components)
***************************************************

The three Cloud Service Models (IaaS, PaaS and SaaS) have different levels of extensibility and security responsibility for the cloud service provider. According to the CSA description:

- IaaS: More extensibility and less security responsibility to the provider
- SaaS: Less extensibility and high security responsibility to the provider
- PaaS: in between the two service models in both extensibility and security responsibility.

Before going to the details of security in these three service models we first describe what the service models entail in cloud computing. NIST defines IaaS as “The capability provided to the consumer to provision processing, storage, networks and other fundamental computing resources where the consumer is able to deploy and run arbitrary software which can include operating systems and applications. The consumer does not manage or control the underlying cloud infrastructure but has control over operating systems, storage, deployed applications and possibly limited control of select networking components (e.g. host firewalls)”.  In this service model the security responsibility is divided between the service provider and the customer, which makes it difficult to identify a clear demarcation. Added to that, security breaches can also be highly damaging as the customers have access to the infrastructure.

The spectrum of IaaS vendors is very wide, in which some offer large full data-center-style infrastructure replication while others offer more end-user-centric services, such as simple data storage (e.g. Amazon Simple Storage Service S3 or Dropbox) [ ]. Currently the services we are considering in Glenna mainly go to these category as the three countries Norway, Sweden and Finland are focusing on IaaS and the other two Denmark and Iceland focus on Storage service which is also considered as IaaS as per the definition above.
In SaaS it’s possible to implement a Pay per use or free for use licensing model in which the customer is subscribed to a complete cloud hardware, software and maintenance package. In this service model it´s not hard to realize the limited access customers have to the underlying infrastructure, which puts the main security responsibility on the service provider. 

PaaS is similar to SaaS, but the service is an entire application development environment, not just the use of an application. PaaS solutions differ from SaaS solutions in that they provide a cloud-hosted virtual development platform, accessible via a Web browser [ ]. NIST describes PaaS as - “The capability provided to the consumer is to deploy onto the cloud infrastructure consumer-created or acquired applications created using programming languages and tools supported by the provider. The customer does not manage or control the underlying cloud infrastructure including network, servers, operating systems, or storage,  but has control over the deployed applications and possibly application hosting environment configurations.” The main target customers of PaaS delivery are developers and some example services include Google App Engine and Sales Force PaaS service at force.com.

As mentioned in the beginning of this section for each service delivery model the security responsibility level of customers and service providers varies.  But in any of the models the service providers have responsibility and in this document we give due emphasis to recommend security solutions and approaches to the service providers involved in Glenna. In addition to the usual cloud service delivery model, the federation induces its challenge in security. To mitigate with such challenges, we first suggest security architecture for Glenna that mainly covers how federation can be implemented in a secure manner. Then the subsequent chapters will cover other important security aspects in the cloud.

The way the federation works, differs based on the way resource authorization is decided. For instance, if all authenticated users can gain access to available cloud resources with no limitation every individual can be considered autonomously and authentication through Kalmar2 or eduGAIN would suffice. We shall also consider how billing is implemented in such scenario. As the individual users are autonomously authenticated and authorized to cloud resources, the users may be billed directly or the institute they are affiliated to can be billed. In this scenario, we can only identify to which institute the user belongs to based on the user account. There is no other group or project information that can be taken into consideration for billing. 

The second scenario involves consideration of users’ affiliation to institutes, groups or projects. In such cases, authorization decision to resources takes the group membership of users into consideration and there can also be resource limitation depending on the resource level the groups, projects or institutes are subscribed. The billing in this scenario shall be directly attached to individual users or the groups/project/institute they belong to.


Security Analysis, Test and Evaluation
--------------------------------------

This task will carry out the necessary analysis, testing and evaluation of the security
mechanisms for Glenna.

