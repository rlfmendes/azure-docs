---
title: Reprotect VMs from Azure to an on-premises site | Microsoft Docs
description: After failover of VMs to Azure, you can initiate a failback to bring VMs back to on-premises. Learn how to reprotect before a failback.
services: site-recovery
author: rajani-janaki-ram
manager: gauravd
ms.service: site-recovery
ms.topic: article
ms.date: 03/05/2018
ms.author: rajanaki
---

# Reprotect machines from Azure to an on-premises site

After [failover](site-recovery-failover.md) of on-premises VMware VMs or physical servers to Azure, the first step in failing back to your on-premises site is to reprotect the Azure VMs that created during failover. This article describes how to do this. 

For a quick overview, watch the following video about how to fail over from Azure to an on-premises site.
> [!VIDEO https://channel9.msdn.com/Series/Azure-Site-Recovery/VMware-to-Azure-with-ASR-Video5-Failback-from-Azure-to-On-premises/player]


## Before you start

If you used a template to create your virtual machines, ensure that each virtual machine has its own UUID for the disks. If the on-premises virtual machine's UUID clashes with that of the master target because both were created from the same template, reprotection fails. Deploy another master target that has not been created from the same template.
- If you are trying to fail back to an alternate vCenter, make sure that your new vCenter and the master target server are discovered. A typical symptom is that the datastores are not accessible or visible in the **Reprotect** dialog box.
- To replicate back to on-premises, you will need a failback policy. This policy get automatically created when you create a forward direction policy. Note that:
    - This policy gets auto associated with the configuration server during creation.
    - This policy is not editable.
    - The set values of the policy are (RPO Threshold = 15 Mins, Recovery Point Retention = 24 Hours, App Consistency Snapshot Frequency = 60 Mins)
- During reprotection and failback, the on-premises configuration server should be running and connected.
- If a vCenter server manages the virtual machines to which you'll fail back make sure that you have the [required permissions](vmware-azure-tutorial-prepare-on-premises.md#prepare-an-account-for-automatic-discovery) for discovery of VMs on vCenter servers.
- Delete snapshots on the master target server before you reprotect. If snapshots are present on the on-premises master target or the virtual machine, reprotection fails. The snapshots on the virtual machine are automatically merged during a reprotect job.
- All virtual machines of a replication group should be of the same operating system type (either all Windows or all Linux). A replication group with mixed operating systems is currently not supported for reprotect and failback to on-premises. This is because the master target should be of the same operating system as the virtual machine and all the virtual machines of a replication group should have the same master target. 
- A configuration server is required on-premises when you fail back. During failback, the virtual machine must exist in the configuration server database. Otherwise, failback is unsuccessful. Make sure that you take regularly scheduled backups of your configuration server. If there's a disaster, restore the server with the same IP address so that failback works.
- Reprotection and failback require a site-to-site VPN to replicate data. Provide the network so that the failed-over virtual machines in Azure can reach (ping) the on-premises configuration server. You might also deploy a process server in the Azure network of the failed-over virtual machine. This process server should also be able to communicate with the on-premises configuration server.
- Make sure you open the following ports for failover and failback.

    ![Ports for failover and failback](./media/vmware-azure-reprotect/failover-failback.png)

## Deploy a process server in Azure

You might need a process server in Azure before you fail back to your on-premises site:
- The process server] receives data from the protected virtual machine in Azure, and sends data to the on-premises site.
- A low-latency network is required between the process server and the protected virtual machine. In general, you need to consider latency when deciding whether you need a process server in Azure:
    - If you have an ExpressRoute connection set up, you can use an on-premises process server to send data because the latency between the virtual machine and the process server is low.
    - However, if you have only an S2S VPN, we recommend deploying the process server in Azure.
    - We recommend using an Azure-based process server during failback. The replication performance is higher if the process server is closer to the replicating virtual machine (the failed-over machine in Azure). For a proof-of-concept, you can use the on-premises process server along with ExpressRoute with private peering.

Deploy as follows:

1. If you need to deploy a process server in Azure, follow [these instructions](vmware-azure-set-up-process-server-azure.md)
2. The Azure VMs will send replication data to the process server. Configure networks so that the Azure VMs can reach the process server.
3. Remember that replication from Azure to on-premises can happen only over the S2S VPN, or over the private peering of your ExpressRoute network. Ensure that enough bandwidth is available over that network channel.

## Deploy a separate master target server

The master target server receives failback data. By default, the master target server runs on the on-premises configuration server. However, depending on the volume of failed-back traffic, you might need to create a separate master target server for failback. Here's how to create one:

    * [Create a Linux master target server](vmware-azure-install-linux-master-target.md) for failback of Linux VMs. This is required.
    * Optionally create a separate master target server for Windows VM failback. To do this, you run Unified Setup again, and select to create a master target server. [Learn more](physical-azure-set-up-source.md#run-azure-site-recovery-unified-setup).

After you've created a master target server, do the following:

- If the virtual machine is present on-premises on the vCenter server, the master target server needs access to the on-premises virtual machine's VMDK. Access is needed to write the replicated data to the virtual machine's disks. Ensure that the on-premises virtual machine's datastore is mounted on the master target's host with read/write access.
- If the virtual machine is not present on-premises on the vCenter server, the Site Recovery service needs to create a new virtual machine during reprotection. This virtual machine is created on the ESX host on which you create the master target. Choose the ESX host carefully, so that the failback virtual machine is created on the host that you want.
- You cannot use Storage vMotion for the master target server*. This can cause the failback to fail. The virtual machine can't start because the disks are not available to it. To prevent this problem, exclude the master target servers from your vMotion list.
- If a master target undergoes a Storage vMotion task after reprotection, the protected virtual machines disks that are attached to the master target migrate to the target of the vMotion task. If you try to fail back after this, detachment of the disk fails because the disks are not found. It then becomes hard to find the disks in your storage accounts. You need to find them manually and attach them to the virtual machine. After that, the on-premises virtual machine can be booted.
- Add a retention drive to your existing Windows master target server. Add a new disk and format the drive. The retention drive is used to stop the points in time when the virtual machine replicates back to the on-premises site. Following are some criteria of a retention drive. Without these criteria, the drive will not be listed for the master target server.
    - The volume is not used for any other purpose, such as a target of replication.
    - The volume is not in lock mode.
    - The volume is not a cache volume. The master target installation shouldn't exist on that volume. The custom installation volume for the process server and master target is not eligible for a retention volume. When the process server and master target are installed on a volume, the volume is a cache volume of the master target.
    - The file system type of the volume is not FAT or FAT32.
    - The volume capacity is nonzero.
    - The default retention volume for Windows is the R volume.
    - The default retention volume for Linux is /mnt/retention.
- You need to add a new drive if you're using an existing process server/configuration server machine or a scale or a process server/master target server machine. The new drive should meet the preceding requirements. If the retention drive is not present, it doesn't appear in the selection drop-down list on the portal. After you add a drive to the on-premises master target, it takes up to 15 minutes for the drive to appear in the selection on the portal. You can also refresh the configuration server if the drive does not appear after 15 minutes.
- Install VMware tools on the master target server. Without the VMware tools, the datastores on the master target's ESXi host cannot be detected.
- Set the `disk.EnableUUID=true` setting in the configuration parameters of the master target virtual machine in VMware. If this row does not exist, add it. This setting is required to provide a consistent UUID to the virtual machine disk (VMDK) so that it mounts correctly.
- The ESX host on which the master target is created should have at least one VMFS datastore attached to it. If there is none, the **Datastore** input on the reprotect page will be empty, and you can't proceed.
- The master target server cannot have snapshots on the disks. If there are snapshots, reprotection and failback fail.
- The master target cannot have a Paravirtual SCSI controller. The controller can only be an LSI Logic controller. Without an LSI Logic controller, reprotection fails.
- At any given instance, the master target can have atmst 60 disks attached to it. If the number of virtual machines being reprotected to the on-premises master target have a sum total number of disks more than 60, then reprotects to the master target will start failing. Ensure that you have enough master target disk slots or deploy additional master target servers.
    

## Enable reprotection

After a virtual machine boots in Azure, it takes some time for the agent to register back to the configuration server (up to 15 minutes). During this time, you won't be able to reprotect, and an error message indicates that the agent is not installed. If this happens, wait for a few minutes, and then try reprotection again.


1. In **Vault** > **Replicated items**, right-click the virtual machine that's been failed over, and then select **Re-Protect**. You can also click the machine and select **Re-Protect** from the command buttons.
2. Verify the direction of protection, **Azure to On-premises**, is already selected.
3. In **Master Target Server** and **Process Server**, select the on-premises master target server and the process server.  
4. For **Datastore**, select the datastore to which you want to recover the disks on-premises. This option is used when the on-premises virtual machine is deleted, and you need to create new disks. This option is ignored if the disks already exist, but you still need to specify a value.
5. Choose the retention drive.
6. The failback policy is automatically selected.
7. Click **OK** to begin reprotection. A job begins to replicate the virtual machine from Azure to the on-premises site. You can track the progress on the **Jobs** tab. After the reprotection succeeds, the virtual machine will enter a protected state.

Note that:
- If you want to recover to an alternate location (when the on-premises virtual machine is deleted), select the retention drive and datastore that are configured for the master target server. When you fail back to the on-premises site, the VMware virtual machines in the failback protection plan use the same datastore as the master target server. A new virtual machine is then created in vCenter.
- If you want to recover the virtual machine on Azure to an existing on-premises virtual machine, mount the on-premises virtual machine's datastores with read/write access on the master target server's ESXi host.
    ![Re-protect dialog box](./media/vmware-azure-reprotect/reprotectinputs.png)

- You can also reprotect at the level of a recovery plan. A replication group can be reprotected only through a recovery plan. When you reprotect by using a recovery plan, you need to provide the values for every protected machine.
- Use the same master target server to reprotect a replication group. If you use a different master target server to reprotect a replication group, the server cannot provide a common point in time.
- The on-premises virtual machine is turned off during reprotection. This helps ensure data consistency during replication. Do not turn on the virtual machine after reprotection finishes.



## Common issues


- Currently, Site Recovery supports failing back only to a virtual machine file system (VMFS) or vSAN datastore. A NFS datastore is not supported. Due to this limitation, the datastore selection input on the reprotect screen is empty in the case of NFS datastores, or it shows the vSAN datastore but fails during the job. If you intend to fail back, you can create a VMFS datastore on-premises and fail back to it. This failback will cause a full download of the VMDK.
- If you perform a read-only user vCenter discovery and protect virtual machines, protection succeeds, and failover works. During reprotection, failover fails because the datastores cannot be discovered. A symptom is that the datastores aren't listed during reprotection. To resolve this problem, you can update the vCenter credential with an appropriate account that has permissions, and retry the job. 
- When you fail back a Linux virtual machine and run it on-premises, you can see that the Network Manager package has been uninstalled from the machine. This uninstallation happens because the Network Manager package is removed when the virtual machine is recovered in Azure.
- When a Linux virtual machine is configured with a static IP address and is failed over to Azure, the IP address is acquired from DHCP. When you fail over to on-premises, the virtual machine continues to use DHCP to acquire the IP address. Manually sign in to the machine, and set the IP address back to a static address if necessary. A Windows virtual machine can acquire its static IP again.
- If you use either ESXi 5.5 free edition or vSphere 6 Hypervisor free edition, failover succeeds, but failback does not succeed. To enable failback, upgrade to either program's evaluation license.
- If you cannot reach the configuration server from the process server, use Telnet to check connectivity to the configuration server on port 443. You can also try to ping the configuration server from the process server. A process server should also have a heartbeat when it is connected to the configuration server.
- A Windows Server 2008 R2 SP1 server that is protected as a physical on-premises server cannot be failed back from Azure to an on-premises site.
- You can't fail back in the following circumstances:
    - You migrated machines to Azure. [Learn more](migrate-overview.md#what-do-we-mean-by-migration).
    - You moved a VM to another resource group.
    - You deleted the Azure VM.
    - You disabled protection of the VM.
    - You created the VM manually in Azure. The machine should have been initially protected on-premises and failed over to Azure before reprotect.
    - You can only fail to an ESXi host. You can't failback VMware VMs or physical servers to Hyper-V hosts, physical machines, or VMware workstations.


## Next steps

After the virtual machine has entered a protected state, you can [initiate a failback](vmware-azure-failback.md). The failback will shut down the virtual machine in Azure and boot the on-premises virtual machine. Expect some downtime for the application. Choose a time for failback when the application can tolerate downtime.

