# ebs-nvme-mapping

POC of EBS NVMe mapping using udev and scripting to mount in fstab

## Scenario

A Nitro-based instance (t3.medium) is being created via Cloudformation template with 3 additional block devices manifested in BlockDeviceMapping.

The 3 block devices will be given `/dev/nvme1n1`, `/dev/nvme2n1`, and `/dev/nvme3n1` names. However, devices can respond to driver discovery in a different order in subsequent instance starts, which causes the device name to change.

To get around this issue, we can use `ebsnvme-id` utility to identify the correct `DeviceName` of every block devices, create filesystem/partition, and then use the partition UUID to mount the volume in `/etc/fstab`.

## How-To

To get the UUID of the partition, used `blkid` command.

```
export nvme1uuid=$(blkid | grep nvme1n1 | awk '{ print $2 }' | sed 's/"//g')
export nvme2uuid=$(blkid | grep nvme2n1 | awk '{ print $2 }' | sed 's/"//g')
export nvme3uuid=$(blkid | grep nvme3n1 | awk '{ print $2 }' | sed 's/"//g')
```

To identify the correct `DeviceName` of a block device, use `ebsnvme-id` command.

```
export nvme1attachment=$(ebsnvme-id /dev/nvme1n1 -u | sed 's/ //g')
export nvme2attachment=$(ebsnvme-id /dev/nvme2n1 -u | sed 's/ //g')
export nvme3attachment=$(ebsnvme-id /dev/nvme3n1 -u | sed 's/ //g')
```

After correct attachment is identified, put the partition UUID in fstab. This is done for every block devices.

```
case "$nvme1attachment" in
"sdf")
    echo "$nvme1uuid /mnt/disk1 xfs defaults 0 0" >> /etc/fstab
    ;;
"sdg")
    echo "$nvme1uuid /mnt/disk2 xfs defaults 0 0" >> /etc/fstab
    ;;
"sdh")
    echo "$nvme1uuid /mnt/disk3 xfs defaults 0 0" >> /etc/fstab
    ;;
esac
```

The rest of the test is optional but would help to ensure consistent device mount point using older naming convention (/dev/sdf, sdg, sdh...). For this, we use udev rules. However, the OS will need to be restarted in order for udev to take effect and generate symlinks.

```
# Write udev rule so there will be /dev/sd* symlinks to the correct NVMe devices order
echo 'KERNEL=="nvme[0-9]*n[0-9]*", ENV{DEVTYPE}=="disk", ATTRS{model}=="Amazon Elastic Block Store", PROGRAM="/sbin/ebsnvme-id -u /dev/%k", SYMLINK+="%c"' > /etc/udev/rules.d/70-ec2-nvme-devices.rules

# Restart so the udev rules in place and symlinks created
shutdown -r now
```

## Cloudformation template

The `template.yaml` file creates an EC2 instance with Elastic IP. The logic to automatically mount the correct device/partition into the correct mount point is provided in the EC2 instance's `UserData`.