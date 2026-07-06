**Steps to Attach and Mount a New Disk Permanently (Linux)**

1. **Verify the new disk**

   ```bash
   lsblk
   ```

   Example:

   * `sda` → OS disk
   * `sdb` → New data disk

2. **Create a partition**

   ```bash
   sudo fdisk /dev/sdb
   ```

   Inside `fdisk`:

   * `n` → New partition
   * `p` → Primary
   * `1` → Partition number
   * Press **Enter** twice to use the full disk
   * `w` → Save and exit

   Verify:

   ```bash
   lsblk
   ```

   You should see `sdb1`.

3. **Format the partition**

   ```bash
   sudo mkfs.ext4 /dev/sdb1
   ```

4. **Create a mount point**

   ```bash
   sudo mkdir -p /datadisk
   ```

5. **Mount the disk**

   ```bash
   sudo mount /dev/sdb1 /datadisk
   ```

   Verify:

   ```bash
   df -h
   ```

6. **Get the UUID**

   ```bash
   sudo blkid /dev/sdb1
   ```

7. **Make the mount permanent**
   Edit `/etc/fstab`:

   ```bash
   sudo nano /etc/fstab
   ```

   Add the following line (replace with your UUID):

   ```text
   UUID=<your-uuid> /datadisk ext4 defaults,nofail 0 2
   ```

8. **Validate the fstab entry**

   ```bash
   sudo mount -a
   ```

   If there are no errors, the configuration is correct.

9. **Reboot and verify**

   ```bash
   sudo reboot
   ```

   After reboot:

   ```bash
   df -h
   ```

   Confirm that `/datadisk` is mounted automatically.
