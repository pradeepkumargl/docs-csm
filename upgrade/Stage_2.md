# Stage 2 - Kubernetes Upgrade from 1.19.9 to 1.20.13

**Reminder:** If any problems are encountered and the procedure or command output does not provide relevant guidance, see
[Relevant troubleshooting links for upgrade-related issues](README.md#relevant-troubleshooting-links-for-upgrade-related-issues).

## Stage 2.1

1. (`ncn-m001#`) Run `ncn-upgrade-master-nodes.sh` for `ncn-m002`.

   Follow output of the script carefully. The script will pause for manual interaction.

   ```bash
   /usr/share/doc/csm/upgrade/scripts/upgrade/ncn-upgrade-master-nodes.sh ncn-m002
   ```

   > **`NOTE`** The root password for the node may need to be reset after it is rebooted.

1. Repeat the previous step for each other master node **excluding `ncn-m001`**, one at a time.

## Stage 2.2

### Option 1

1. (`ncn-m001#`) Run `ncn-upgrade-worker-nodes.sh` for `ncn-w001`.

   Follow output of the script carefully. The script will pause for manual interaction.

   ```bash
   /usr/share/doc/csm/upgrade/scripts/upgrade/ncn-upgrade-worker-nodes.sh ncn-w001
   ```

   > **`NOTE`** The root password for the node may need to be reset after it is rebooted.

1. Repeat the previous steps for each other worker node, one at a time.

### Option 2

> **`Tech Preview`**
>>
>> You can also upgrade multiple workers at the same time with comma separated list. Note that in some cases, you can't rebuild all workers in one request. It is system admin's responsibility to make sure a multiple workers request meets following conditions:
>>
>> 1. If your system has more than 5 workers, you can't rebuild `ncn-w001,ncn-w002 and ncn-w003` together in one request. You will need at least two requests (example: rebuild `ncn-w001,ncn-w003,ncn-w004,...` first and then rebuild `ncn-w002,ncn-w003,...`)
>> 2. If your system has more that 5 workers, you can't rebuild all workers that has DVS running in one request.
>
> ```bash
> /usr/share/doc/csm/upgrade/scripts/upgrade/ncn-upgrade-worker-nodes.sh ncn-w001,ncn-w002,ncn-w003
>```
>
>>

## Stage 2.3

By this point, all NCNs have been upgraded, except for `ncn-m001`. In the upgrade process so far, `ncn-m001`
has been the "stable node" -- that is, the node from which the other nodes were upgraded. At this point, the
upgrade procedure pivots to use `ncn-m002` as the new "stable node", in order to allow the upgrade of `ncn-m001`.

1. If the CSM tarball is located on an `rbd` device, then remap that device to `ncn-m002`.

    See [Move an `rbd` device to another node](../operations/utility_storage/Alternate_Storage_Pools.md#move-an-rbd-device-to-another-node).

1. Log in to `ncn-m002` from outside the cluster.

    > **`NOTE`** Very rarely, a password hash for the `root` user that works properly on a SLES SP2 NCN is
    > not recognized on a SLES SP3 NCN. If password login fails, then log in to `ncn-m002` from
    > `ncn-m001` and use the `passwd` command to reset the password. Then log in using the CMN IP address as directed
    > below. Once `ncn-m001` has been upgraded, log in from `ncn-m002` and use the `passwd` command to reset
    > the password. The other NCNs will have their passwords updated when NCN personalization is run in a
    > subsequent step.

   `ssh` to the `bond0.cmn0`/CMN IP address of `ncn-m002`.

1. Authenticate with the Cray CLI on `ncn-m002`.

   See [Configure the Cray Command Line Interface](../operations/configure_cray_cli.md) for details on how to do this.

1. (`ncn-m002#`) Set the `CSM_RELEASE` variable to the **target** CSM version of this upgrade.

   ```bash
   CSM_RELEASE=1.3.0
   CSM_REL_NAME=csm-${CSM_RELEASE}
   ```

1. (`ncn-m002#`) Copy artifacts from `ncn-m001`.

   A later stage of the upgrade expects the `docs-csm` RPM to be located at `/root/docs-csm-latest.noarch.rpm` on `ncn-m002`; that is why this command copies it there.

   ```bash
   mkdir -pv /etc/cray/upgrade/csm/${CSM_REL_NAME} &&
             scp ncn-m001:/etc/cray/upgrade/csm/myenv /etc/cray/upgrade/csm/myenv &&
             scp ncn-m001:/root/output.log /root/pre-m001-reboot-upgrade.log &&
             cray artifacts create config-data pre-m001-reboot-upgrade.log /root/pre-m001-reboot-upgrade.log
   csi_rpm=$(ssh ncn-m001 "find /etc/cray/upgrade/csm/${CSM_REL_NAME}/tarball/${CSM_REL_NAME}/rpm/cray/csm/ -name 'cray-site-init*.rpm'") &&
             scp ncn-m001:${csi_rpm} /tmp/cray-site-init.rpm &&
             scp ncn-m001:/root/docs-csm-*.noarch.rpm /root/docs-csm-latest.noarch.rpm &&
             rpm -Uvh --force /tmp/cray-site-init.rpm /root/docs-csm-latest.noarch.rpm
   ```

1. Upgrade `ncn-m001`.

   ```bash
   /usr/share/doc/csm/upgrade/scripts/upgrade/ncn-upgrade-master-nodes.sh ncn-m001
   ```

## Stage 2.4

Run the following command to complete the upgrade of the `weave` and `multus` manifest versions:

```bash
/srv/cray/scripts/common/apply-networking-manifests.sh
```

## Stage 2.5

Run the following script to apply anti-affinity to `coredns` pods:

```bash
/usr/share/doc/csm/upgrade/scripts/k8s/apply-coredns-pod-affinity.sh
```

## Stage 2.6

Complete the Kubernetes upgrade. This script will restart several pods on each master node to their new Docker containers.

```bash
/usr/share/doc/csm/upgrade/scripts/k8s/upgrade_control_plane.sh
```

> **`NOTE`**: `kubelet` has been upgraded already, ignore the warning to upgrade it.

## Stage completed

All Kubernetes nodes have been rebooted into the new image.

> **REMINDER**: If password for `ncn-m002` was reset during Stage 2.3, then also reset the password
> on `ncn-m001` at this time.

This stage is completed. Continue to [Stage 3](Stage_3.md).
