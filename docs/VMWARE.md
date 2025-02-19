# VMware iSCSI Configuration

This documentation outlines the VMware prerequisites and provides configuration recommendations for the ESXi hypervisors' iSCSI software initiator when used with PetaSAN.

## VMware Prerequisites
- ESXi must une version 6.5 or higher
- LUNs must be formatted in VMFS 6 or higher

## ESXi Configuration

### Network Configuration and iSCSI Initiator Activation

In vCSA:
- In Network view, the distributed switch associated with the cluster containing your ESXi must have one or more virtual networks specific to the iSCSI VLAN (2211 in our example). Several can be created so that each uses a different vmnic physical interface.
- In Hosts and Clusters view, select the hypervisor then Configure / Networking / VMkernel Adapters. Add one or more VMkernel Port Groups with their own IP address for each iSCSI virtual network created above.
- In Hosts and Clusters view, select the hypervisor then Configure / Storage / Storage Adapters. Enable the "iSCSI Software Adapter". In "Network Port Binding" add one Network Port Binding for each VMkernel Port Group.

### SSH and Shell Activation

In vCSA:
- In Hosts and Clusters view, select the hypervisor then Configure / System / Services. Start the ESXi Shell and SSH services.

### iSCSI Initiator Configuration

1. Activate the software adapter:
```
esxcli iscsi software set --enabled=true
```

2. Configure the initiator name:
```
esxcli iscsi adapter set --adapter=$(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1) --name="iqn.1998-01.com.vmware:$(hostname -s)"
```

3. Create VMkernel Port Groups on each ESXi with their IP in 100.74.180.x in iSCSI VLAN.

4. Add vmk to initiator [Adapt X of vmkX in the commands below]:
```
esxcli iscsi networkportal add -n vmk2 -A $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1)
esxcli iscsi networkportal add -n vmk3 -A $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1)
```

5. CHAP authentication configuration:
```
esxcli iscsi adapter auth chap set --direction=uni --authname=XXXXXXXXXXXXXX --secret=XXXXXXXXXXXXXX --adapter=$(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1) --level=required
```

6. Add target to automatic discovery:
```
esxcli iscsi adapter discovery sendtarget add --address=<ip_or_fqdn_of_a_petasan_node> --adapter=$(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1)
```

7. Enable XCOPY:
```
esxcli system settings advanced set --int-value 1 --option /DataMover/HardwareAcceleratedMove
```

8. Set RecoveryTimeout to 10s:
```
esxcli iscsi adapter param set -A $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1) -k RecoveryTimeout -v 10
```

9. Add an SATP rule specific to PetaSAN:
```
esxcli storage nmp satp rule list
esxcli storage nmp satp rule add -s VMW_SATP_ALUA -P VMW_PSP_RR -V PETASAN -M RBD -c tpgs_on -o enable_action_OnRetryErrors -O "iops=1" -e "Ceph iSCSI ALUA RR iops=1 PETASAN"
```

You can identify the -V and -M values (Vendor and Model) announced by PetaSAN with this command:
```
esxcli storage core device list
```

10. Restart the ESXi for the SATP rule to apply to Ceph LUNs.

11. After reboot, verify that action_OnRetryErrors is 'on' for Ceph LUNs:
```
esxcli storage nmp device list
```

12. The ESXi needs some tweaking:
- Increase the queue depth of the software initiator.
```
esxcli system module parameters set -m iscsi_vmk -p "iscsivmk_LunQDepth=512"
```

- Increase iSCSI IO size
```
esxcli system settings advanced set -o /ISCSI/MaxIoSizeKB -i 512
```

- Increase PDU size
```
esxcli iscsi adapter param set -A $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1) --key FirstBurstLength --value 524288
esxcli iscsi adapter param set -A $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1) --key MaxBurstLength --value 524288
esxcli iscsi adapter param set -A $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1) --key MaxRecvDataSegment --value 524288
```

Restart the ESXi.

13. After creating the Datastore in a Datastore cluster, disable SIOC on each Ceph datastore. SIOC intervenes to reduce the Queue Depth of a LUN when the latency threshold of 30 ms (by default) is exceeded. This threshold can be increased to 100ms maximum, which can sometimes be reached on a single IO on Ceph and trigger regulation.

14. Rescan devices and datastores to take into account SIOC deactivation:
```
esxcli storage core adapter rescan --adapter $(esxcli iscsi adapter list | grep iscsi_vmk | cut -d ' ' -f1)
```

15. Increase the "Device Max Queue Depth" and DSNRO of each LUN (defines the maximum queue depth size of this LUN) and the DSNRO (defines the maximum IO that a VM can do on this LUN) of each:
```
for lun in $(esxcli storage core device list | grep -E '^naa.6001405') ; do esxcli storage core device set -d $lun -m 512 ; esxcli storage core device set -d $lun -O 512 ; done
```

This configuration step must be done after adding any new LUN and on each ESXi connecting to it.

16. Configuration Control
Connect via SSH to the hypervisor and do:
```
TERM=xterm esxtop
'u'
```

Pressing the 'u' key displays the LUNs. Check the following columns:
- DQLEN should display 512
- ACTV shows the number of IOs in queue
- QUED should always display 0 under normal conditions. This counter indicates the number of IOs that the initiator has queued.

## Increasing LUN Size
Increasing LUN Size is as simple as adjusting the 'Disk' size in PetaSAN Dashboard.

## Useful Links

### ALUA Configuration:
- https://kb.vmware.com/s/article/1022030?lang=en_US#q=ALUA
- https://cormachogan.com/2015/02/19/vsphere-6-0-storage-features-part-6-action_onretryerrors/
- http://virtualguido.blogspot.com/2016/09/vmware-esxi-claim-rules-unleashed.html

### iSCSI Qdepth for LIO target, ESXi, and vSCSI controllers:
vSCSI: http://virtualguido.blogspot.com/2016/08/vmware-scsi-controller-options.html

Qlogic:
- https://datacore.custhelp.com/app/answers/detail/a_id/1514/~/what-value-should-hosts-with-fibre-channel-hbas-use-for-the-execution-throttle/session/
- https://cormachogan.com/2014/01/09/qlogic-execution-throttle-feature-concerns/

VMware:
- https://kb.vmware.com/s/article/1268
- https://kb.vmware.com/s/article/1267
- https://kb.vmware.com/s/article/1027901
- https://www.teimouri.net/vmware-esxi-queue-depth-overview-configuration-calculation/

Storage Performance Control:
- http://vmconsole.blogspot.com/2014/09/esx-commands-for-checking-storage-and.html

General:
- https://www.settlersoman.com/what-is-storage-queue-depth-qd-and-why-is-it-so-important/
