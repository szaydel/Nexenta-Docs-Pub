# NexentaStor Auto-sync based Disaster Recovery Strategy #


## Overview: ##
It is critical that you plan your Disaster Recovery (DR) strategy at a level that addresses your operational processes and business logic. It is not always enough to have a storage-centric recovery strategy. There is a huge diversity of envoronments, and each with its own unique set of constraints. Knowing and understanding these constraints is key to developing an appropriate strategy to your organization. Depending upon various service-level agreements, solution that may be appropriate to another business may be completely unacceptable to your organizarion.

It is critically important to understand where storage fits in your overall DR strategy, and plan availability of storage that is in line with availability requirements of the application. It is critically important to define a **Recovery Point Objective** (RPO), for the various applications within your production environment.


Within NexentaStor the Auto-sync plugin is your primary tool for building a DR architecture for your storage. Fundementally, with Auto-sync we have the ability to move data from our Production to our DR environment, and back from DR to Production. This framework is what enables you to build a solution where storage is replicated between datacenters, whether in full or in part. Auto-sync allows you to maintain an asynchonous relationship between your two datacenters, and should a disaster ever occur, with Auto-sync we can disrupt our established replication services, and present production data at the DR site in a matter of moments. Asynchronous nature of Auto-sync does mean that your primary datacenter will be slightly ahead of your DR site, but surely not by much. With increased frequency of replication your sites could be within a few minutes of each other at all times.


## Necessary Considerations: ##
When planning out a DR strategy with Auto-sync as the replication mechanism, it is important to think about a number of items, all of which will necessarily contribute to complexity of your recovery strategy.

* Basic concepts of Auto-sync and their implications on your decisions
* Amount of delta and synchronization frequency objective
* Recovery Point Objective on a per application basis
* Protocols deployed in the Environment
* Organization and structure of shares
* Different tiers of data and necessity for replication

### Auto-sync Basic Concepts ###
Auto-sync is an asynchronous block-level replication mechanism, based on replicating data in the form of snapshots between your source (normally production) and your target (normally DR). By the nature of ZFS, consistency of data in any given snapshot is assured, because all data is written in a transactional manner. Each transaction is atomic, and will either entirely succeed or fail. Be default, we transfer differences, i.e. delta between previous and latest snapshots of a given dataset. Differences are block-specific, which means that if only a single block changed in a file, we only transfer the block that changed, and not the entire file. Each dataset is typically either a share or a ZVOL. Depending upon the amount of data change on any given share, more or less time will be required to transfer that delta to a remote appliance. What this means to you is that some delta in the data will always exist between your DR system, and your source. Rate of change, utilization of drives in the pool(s), bandwidth of available links between source and target, number of Auto-sync services and frequency of scheduled Auto-sync services will all play a role in defining how far behind your DR system will be behind your production system.

It is very important to understand that datasets, be they filesystems or ZVOLs, which need to be point-in-time consistent should all be part of the same Auto-sync service. So what does this mean in practical terms? Most environments are of sufficient complexity, where consistency of data between multiple shares may be critical. For example, you may have an application which consists of a group of servers all of which perform the same function, but data produced happens to be spread across a number of filesystems. Each time your application processes a piece of data there may be an update to one or multiple databases and at the same time there might be a change to flat application files referencing the update made to the database. In most cases these actions are strictily sequential and their relationship has to be maintained in order for the application to maintain state. If say data in the databases is ahead of the application data, transactions may fail because in one place transaction is already registered, but not so in the other. Understanding these relationships is critical to your Auto-sync strategy. Each Auto-sync service will potentially depend upon completion of already-running services, before it will start. There are too many things that factor into this, but at the highest level, there is absolutely no guarrantee that two or more Auto-sync services scheduled to begin at the same time will all start at the same time, meaning that between Auto-sync services data may not be consistent.



 that with Auto-sync creating multiple Auto-sync services for data which 


 The very first area that you should define, before designing your Auto-sync strategy is the amount of time that your replcation site is 

### Recovery Point Objective ###


### Deployed Protocols ###
Complexity of environment with regard to protocol diversity is a factor which we cannot ignore. Keep in mind that while we are replicating data with Auto-sync, we are not replicating configuration information that is not part of the ZFS properties for each filesystem or virtual block device(s) that are being replicated.

For example, while information necessary to share NFS is stored as a property on a filesystem, which 


and focus of making sure that storage does not become a real constraint. Obviously, it is difficult for it not to be a constraint since your applications are as useful without data as a car with no wheels. That said, having an environment where storage is flexible and your applications do not enforce very specific requirements from storage will make for a more simplified and realistic DR strategy.