# NexentaStor How to Add Virtual IP Addresses to HA Cluster Configuration #
There are times when having a single Virtual Internet Protocol Address (VIP) tied to a Service Group of your HA Cluster is not sufficient, and multiple VIPs are required. A good example of this is where you might have dedicated networks for different security levels, or slow and fast networks, etc. Please keep in mind that this is an advanced configuration option, and **failure to configure correctly may lead to loss of HA services**. As such, please follow with caution, and if unsure, engage Support prior to making changes.  

It is possible to configure multiple Virtual IPs for any Service Group, but there are some things to keep in mind.  

* Do not assign VIPs from the same subnet to physical interfaces with different bandwidth capabilities, such as 10GbE and 1GbE. This may result in inbound packets coming from the faster 10GbE interface and leaving via the slower 1GbE.  
* It is acceptable to create an aggregate out of available interfaces and assign a VIP to the aggregate interface.  

Please, complete the following steps via NMC:  

1.  Create a configuration checkpoint to have a known good configuration snapshot, prior to making any changes in case further changes result in a malfunctioning configuration.  

        nmc@clus-a-node-1:/$ setup appliance checkpoint create  
 
2.  Validate current network configuration to determine which interface(s) are available to provision new VIP. Pay close attention to the netmask setting on the interface(s). If a network interface to which the VIP will be assigned is already configured with an IP address, make sure to validate and use the same network mask when configuring the VIP. It is not desirable to have multiple interfaces on the same subnet configured with different network masks.  

        nmc@clus-a-node-1:/$ show network interface  

        ==== Interfaces ====
        lo0: flags=2001000849<UP,LOOPBACK,RUNNING,MULTICAST,IPv4,VIRTUAL> mtu 8232 index 1
                inet 127.0.0.1 netmask ff000000 
        e1000g0: flags=1001000843<UP,BROADCAST,RUNNING,MULTICAST,IPv4,FIXEDMTU> mtu 1500 index 2
                inet 10.10.100.100 netmask fffffc00 broadcast 10.10.103.255
                ether 0:c:29:ed:4a:a8 
        e1000g0:1: flags=1001000843<UP,BROADCAST,RUNNING,MULTICAST,IPv4,FIXEDMTU> mtu 1500 index 2
                inet 10.10.100.101 netmask ff000000 broadcast 10.255.255.255
        e1000g1: flags=1000842<BROADCAST,RUNNING,MULTICAST,IPv4> mtu 1500 index 3
                inet 0.0.0.0 netmask 0 
                ether 0:c:29:ed:4a:b2 
        ...snipped...  

3.  We can obtain further details for interface which will be used for the new VIP by adding <interface name> to command from prior step. Interface must be physically connected to correct network. In our example, we select interface *e1000g1*, and validate its configuration.

        nmc@clus-a-node-1:/$ show network interface e1000g1  

        ==== Interface: e1000g1 ====  
        e1000g1: flags=1000842<BROADCAST,RUNNING,MULTICAST,IPv4> mtu 1500 index 3  
                inet 0.0.0.0 netmask 0  
                ether 0:c:29:ed:4a:b2  
        ==== Interface e1000g1: statistics ====  
        Name  Mtu  Net/Dest      Address        Ipkts  Ierrs Opkts  Oerrs Collis Queue  
        e1000g1 1500 default       0.0.0.0        89235  0     12     0     0      0     

4.  In addition, it is a good idea to validate the routing table on each node. It is prudent to confirm that ALL nodes in the cluster share the same entries in the routing table. Any differences should be investigated.  

        nmc@clus-a-node-1:/$ netstat -nr                                                           

        Routing Table: IPv4
          Destination           Gateway           Flags  Ref     Use     Interface 
        -------------------- -------------------- ----- ----- ---------- --------- 
        default              10.10.100.1          UG        5       2770           
        10.0.0.0             10.10.100.101        U         2          0 e1000g0   
        10.10.100.0          10.10.100.100        U         6      14291 e1000g0   
        127.0.0.1            127.0.0.1            UH       23       5714 lo0  
        ...snipped...  

5.  Assuming that we do not have entries in */etc/hosts*, we need to update entries on ALL nodes in the cluster. We need to add an entry for the new VIP. Consider being descriptive and using the Service Group name as part of the VIP hostname in the */etc/hosts* file. Command `<setup appliance hosts>` will launch the vi-editor, and allow for modification of the hosts file.   

        We are adding the following entry to /etc/hosts:  
        10.10.100.102   clus-a-pool-a-01

        nmc@clus-a-node-1:/$ setup appliance hosts  

6.  We will next validate /etc/netmasks, and for this VIP we are expecting to use a /22 netmask. Changes to /etc/netmasks should again be made on ALL nodes. Command `<setup appliance netmasks>` will launch the vi-editor, and allow for modification of the hosts file.  
        
        We are adding the following entry to /etc/netmasks:  
        10.10.100.0   255.255.252.0  

        nmc@clus-a-node-1:/$ setup appliance netmasks  

7.  We should now have everything in place to configure the new Virtual IP. We can make this change from any node in the cluster. We will be presented with a a list of cluster groups, and need to select a group to be modified from the list. In this instance we are selecting *clus-1_pool_a*.  
       
        nmc@clus-a-node-1:/$ setup group rsf-cluster clus-alpha-01 vips add 
        Service         :
        clus-1_pool_a 
        -----------------------------------------------------------------------------
        Select HA Cluster service. Navigate with arrow keys (or hjkl), or  
        Ctrl-C to exit.  

8.  Next, we are presented with a configuration VIPs currently in place, and a prompt to add our new VIP. In this case we are going to use hostname **clus-a-pool-a-01**, which we added to /etc/hosts earlier.

        Your current network configuration for service 'clus-1_pool_a':  
        VIP1: clus-a-pool-a  
        clus-a-node-1.homer.lab:e1000g0  
        clus-a-node-2:e1000g0  
        VIP2 Shared logical hostname:  
        -----------------------------------------------------------------------------  
        IPv4 address or hostname that maps to the failover IP interface on all the  
        appliances in the HA Cluster. This IP address or hostname is expected to be  
        used by services such as NFS/CIFS/iSCSI to reliably access shared storage at  
        all times.. Press Ctrl-C to exit.  

9.  At this point we are prompted with a list of interfaces avaialble to us on the system, and as we planned earlier, we will select *e1000g1* from this list. Names and numbers of interfaces will vary from system to system. We are going to see the prompt to select interface for this VIP repeat for each node in the cluster. We are expecting that all nodes will use the same interface.  

        VIP2 Shared logical hostname: clus-a-pool-a-01  
        VIP2 Network interface at clus-a-node-1.homer.lab:  
        e1000g2  e1000g3  e1000g0  e1000g1  

10.  Now we have to specify the mask used for this VIP, and while we already have /etc/masks updated, we are still going to enter the mask here, assurming that if there any changes in the future to the logic of HA configuration, we are already covered and do not have to reconfigure. We are using the same mask as in /etc/netmasks.  

        Verifying logical (failover) IP address '10.10.100.102' ...Success.  
        VIP2 Failover Netmask:  
        -----------------------------------------------------------------------------  
        Optionally, specify IP network mask for the failover interface  
        '10.10.100.102'. The mask includes the network part address and the subnet  
        part. Press Enter to use default netmask: 255.0.0.0.. Press Ctrl-C to exit.  

11. We are prompted with changes that are about to be committed, and we accept these changes. If we are done adding VIPs, we are going to say *Yes* to 'Stop adding VIPs?'.  

        The IP configuration to be used with 'clus-a-pool-a-01' is: 10.10.100.102/255.255.252.0.  
        Please confirm that this configuration is correct ?   Yes  
        Stop adding VIPs?  Yes  
        Success.  

12. At this point we need to validate that the VIP was added, and on the system where this service group is active, we can validate that the VIP is now in service. We can simply ping the VIP to validate its state. We should make sure that we can ping the VIP from ALL nodes in the cluster.  

        nmc@clus-a-node-1:/$ ping clus-a-pool-a-01  
        clus-a-pool-a-01 is alive  
