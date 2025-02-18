# Using PetaSAN's iSCSI GWs with an external Ceph cluster

**Preface: This setup is not supported by PetaSAN.**

The purpose of this documentation is to:
- provide insights regarding the production use of PetaSAN iSCSI GWs with an external Ceph cluster and VMware ESXi clients
- provide a guide [link to documentation] to set up PetaSAN iSCSI GWs with an external Ceph cluster.

# Why choosing PetaSAN's iSCSI implementation?

Going through the points below, you'll find out why we chose PetaSAN's iSCSI implementation over Ceph's native iSCSI implementation for VMware ESXi initiators.

## Looking in the rearview mirror

First, I would first like to extend my gratitude to Mike Christie, Jason Dillaman, Igor Fedotov, Xiubo Li, and everyone who worked on stabilizing the native iSCSI implementation in Ceph (and fixing the Linux kernel) throughout various Ceph and Linux kernel version updates.

We used the community iSCSI implementation for 7 years with VMware ESXi hypervisors, but it came with numerous challenges and little peace of mind. This was primarily due to the gateway blocklist mechanism implemented by OSDs to prevent IO overwriting (sometimes resulting in both GWs being blocklisted), and because each major version upgrade (whether Ceph or kernel) introduced its own set of instability or incompatibiilty issues (with VMware ESXi clients).
Performance was also lacking since the GWs only provided a single active path (AO) to an RBD image. Looking back, I realize that using this access mode in production with VMware ESXi hypervisors, under these conditions, wasn't a wise choice. While we managed to make it work, the effort required to maintain the service over the years was unreasonable.

I had long wanted to test an alternative implementation based on the SUSE kernel but never found the time. I finally got around to testing PetaSAN in July 2023.

## PetaSAN's Key Benefits

PetaSAN implements its own Ceph cluster with various protocol access methods (iSCSI, CIFS, NFS, S3, etc.), manages snapshots, and handles replication both within and between PetaSAN clusters - essentially functioning as a proper SAN. All components deployed by PetaSAN are designed, like Ceph, to provide maximum scalability. It's iSCSI implementation, which incorporates code from SUSE's implementation along with home made patches, is particularly impressive in terms of performance and scalability.

More information on [PetaSAN's website](https://www.petasan.org)

## VMware ESXi clients

### Resiliency

First and foremost, the ESXi resilience to access loss and/or inability to read/write IO to the cluster is excellent. 

Below is a synthesis of my observations:

- When some paths fail, IOs are automatically routed through other active paths.
- Shutting down the backend interfaces (Ceph cluster side) on an iSCSI gateway has very little influence on the I/O within the test VM (fio). The I/O requests are rejected and take another available path. The inaccessible paths are not marked 'Dead' on the ESXi side. The PetaSAN cluster moves the IPs of the portals (1 LUN = 2+ IPs) to another iSCSI gateway.
- Shutting down the front-end interfaces (ESXi side) on an iSCSI gateway has very little influence on the I/O within the test VM. The iSCSI connections are dropped, and the inaccessible paths are marked 'Dead' on the ESXi side. The PetaSAN cluster does not move the portal IP to another iSCSI gateway.
- The abrupt shutdown of an iSCSI gateway has little to no influence on the I/O within the VM. The inaccessible paths are marked 'Dead,' and the I/O continues on the other iSCSI gateways. The PetaSAN cluster moves the portal IP to another iSCSI gateway, and the paths become 'active' again. When the crashed gateway comes back online, the portal IP does not return to the revived gateway. It must be reassigned (path reassignment) from the PetaSAN WebUI.
- At no point does the ESXi lose connectivity to the storage (except when all gateways carrying an access IP to the LUN are stopped).
- Moving an IP associated with a LUN has little to no influence on the I/O within the VM. The failover is very quick (~2 seconds).
- Stopping I/O on the Ceph cluster side (ceph osd pause) does not cause the paths to be marked 'Dead.' Only the I/O is stopped and resumes when the I/O is unblocked on the Ceph cluster side (ceph osd unpause). The ESXi does not notice anything and does not become unavailable trying over and over to recover datastore access. Simply, the I/O within the VM is stopped. It resumes immediately when the I/O is unblocked on the Ceph cluster side, as long as the freeze period doesn't exceed 180s (max wait time for most kernels on filesystem unavailability, IIRC).
- PGs peering on the Ceph cluster side has an impact on the I/O rate within the VM (4k with 32 parallel requests, so high workload), but the impact remains low even on the small 12 OSDs cluster used for the test. The impact of peering is almost imperceptible from a VM perspective on our large (+600 OSDs) production cluster.

The performance obtained during the fio tests is excellent, and especially, the ESXi access seems robust and persists regardless of storage disruptions, without the need for manual intervention.

### Multi-pathing and Performance

Paths are balanced across all PetaSAN GWs (providing excellent scalability), and the path selection policy (PSP) used for VMware Datastores access is "Round Robin" with path changes at **each** IO, maximizing performance.

I estimate the performance to be roughly 5 to 20 times better than the community iSCSI implementation, even on a single active path. While specific performance figures wouldn't be meaningful here, as they would naturally depend on the number of OSDs and drive types in the underlying Ceph cluster, trust me when I say that PetaSAN's performance is impressive, at least compared to native's Ceph iSCSI implementation, even on a single data IO path.

All operations performed by VMware's iscsi_vmk software initiator - such as device/path rescanning, VMFS formatting, VMFS browsing - are also significantly faster with PetaSAN GWs. These same operations were notably slow with the community iSCSI implementation.

### Web UI and operations

PetaSAN's web UI allows for both automatic and manual path balancing across GWs, and even enables completely clearing paths from a GW for hardware maintenance purposes. All of this works smoothly and transparently, though the execution speed (and WebUI display) could be improved, particularly when dealing with numerous 'Disks'/LUNs at a time. You can also schedule snapshots of 'Disks' (RBD images) and easily provision a datastore from an RBD snapshot to recover a damaged VM - all through the WebUI.
I've successfully used this feature a few times. While it's also possible to configure replication of a PetaSAN 'Disk' to another PetaSAN cluster's 'Disk', I haven't tested this functionality yet.

When a gateway goes offline, its paths (portals) are automatically redistributed to the surviving gateways. However, I see two areas for improvement regarding path balancing:

- When two GWs are operational, all paths to a single RBD image shouldn't be allowed to route through just one GW.
- PetaSAN should consistently maintain path balance between different GWs. As of July 2023, this isn't the case - when a GW goes down, its paths are redistributed to other GWs, but when it comes back online, those paths don't automatically return to the restored GW, which requires surveillance (monitoring) and manual intervention as of now.

## Final Choice

Based on this positive experience, I decided to replace the native iSCSI implementation of Ceph with PetaSAN's iSCSI GWs in our production environment, running on our external Ceph cluster (600+ OSDs) - even though PetaSAN doesn't support this kind of setup (with external Ceph clusters). Since August 2023, we have successfully operated 135 VMs on PetaSAN, totaling approximately 75TB, with excellent performance and reliability.

We started with PetaSAN GWs as virtual machines (on dedicated ESXi) and decided to stick with this setup even though we had 2 hardware nodes available for the following reasons:

- The performance offered by virtualized PetaSAN gateways was more than sufficient in our context and still is (each GW uses 2x 10Gbits/s NICs for front (ESXi) trafic and 2x 10Gbits/s NICs for back (Ceph cluster) trafic).
- Moving VMs between hypervisors does not cause any paths unavailability between hypervisors and iSCSI GWs.
- VMs offer more operational flexibility when it comes to replacing or updating GWs (eg. updates, etc.) withouth the need to buy some new hardware and reduces the risk of impacting prodution.
- The hardware setup required a 3rd node (with same number of interfaces) that we didn't have to ensure the quorum of PetaSAN operated by Consul.

## Setup guide

You'll find steps to install and configure PetaSAN's iSCSI GWs with an external Ceph cluster following this link [link to documentation].

## Special thanks

I want to specifically acknowledge Maged Mokhtar (PetaSAN) and his team for their swift investigation and resolution of various issues I encountered during PetaSAN testing (non-functioning VMware VAAI XCOPY, a kernel memory leak, and LIO queue_depth mismatch). Maged was incredibly accessible and responsive, providing the necessary fixes (2 modules and 1 kernel) at remarkable speed! This level and promptness of support was unprecedented in my experience.

## Support

As of Feb. 2025, **don't expect official support from PetaSAN** when using PetaSAN iSCSI GWs with an external Ceph cluster in production.

If you're looking for some information on how PetaSAN works internaly, you can find some on [PetaSAN's forums](https://www.petasan.org/forums/)
If you're looking for some information on using PetaSAN with an external Ceph cluster, look in the ceph-user list [archives](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/) and/or [subscribe](https://lists.ceph.io/postorius/lists/ceph-users.ceph.io/) to the ceph-user list and ask questions there.
