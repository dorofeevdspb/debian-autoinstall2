# Debian automated installation using preseed file

## Image download
Download [debian 12 netinst iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.7.0-amd64-netinst.iso)

## Pressed file
The preseed file in this repository is provided as an example. Let's use it as a starting point for now.

## Execute the follwing commands:

1. Define the environment variables:
```shell
export ISO_FILE="${HOME}/iso/debian-12.4.0-amd64-netinst.iso"
export PRESED_FILE="preseed.cfg"
export MNT_DIR="${PWD}/mnt"
export CUSTOM_DIR="${PWD}/custom"
export CUSTOM_ISO_FILE="${PWD}/debian-12-custom.iso"
```

2. Cleanup and create the directories:
```shell
if [ -d ${MNT_DIR} ]; then sudo rmdir ${MNT_DIR}; fi
if [ -d ${CUSTOM_DIR} ]; then sudo rmdir ${CUSTOM_DIR}; fi
mkdir -pv ${MNT_DIR}
mkdir -pv ${CUSTOM_DIR}
```

3. Mount iso and copy the files to the custom directory:
```shell
sudo mount -o loop ${ISO_FILE} ${MNT_DIR}
sudo rsync -av ${MNT_DIR}/ ${CUSTOM_DIR}/
sudo umount ${MNT_DIR}
group=`groups`
sudo chown -R ${USER}:${group} ${CUSTOM_DIR}
```

4. Execute the next commands to prepare and create a customized iso image:
```shell
chmod +w -R ${CUSTOM_DIR}/install.amd/
gunzip ${CUSTOM_DIR}/install.amd/initrd.gz
echo preseed.cfg | cpio -H newc -o -A -F ${CUSTOM_DIR}/install.amd/initrd
gzip ${CUSTOM_DIR}/install.amd/initrd
chmod -w -R ${CUSTOM_DIR}/install.amd/
sudo chmod +w ${CUSTOM_DIR}/isolinux/menu.cfg
sudo chmod +w ${CUSTOM_DIR}/isolinux/spkgtk.cfg
sudo sed -i 's|timeout 300|timeout 10|g' ${CUSTOM_DIR}/isolinux/spkgtk.cfg
sudo sed -i 's|ontimeout /install.amd/vmlinuz vga=788 initrd=/install.amd/gtk/initrd.gz speakup.synth=soft --- quiet|ontimeout /install.amd/vmlinuz vga=788 initrd=/install.amd/initrd.gz --- quiet|g' ${CUSTOM_DIR}/isolinux/spkgtk.cfg
sudo sed -i 's|otherwise speech synthesis will be started in|otherwise automated install will be started in|g' ${CUSTOM_DIR}/isolinux/spkgtk.cfg
sudo chmod -w ${CUSTOM_DIR}/isolinux/menu.cfg
sudo chmod -w ${CUSTOM_DIR}/isolinux/spkgtk.cfg
cd ${CUSTOM_DIR}
chmod +w md5sum.txt
find -follow -type f ! -name md5sum.txt -print0 | xargs -0 md5sum > md5sum.txt
chmod -w md5sum.txt
cd ..
```

5. Create the iso image:
```shell
sudo genisoimage -r -J -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o ${CUSTOM_ISO_FILE} ${CUSTOM_DIR}
group=`groups`
sudo chown -R ${USER}:${group} ${CUSTOM_ISO_FILE} 
```

6. Execute a test:
```shell
qemu-img create disk.img 40G
qemu-system-x86_64 \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -hda disk.img \
  -cdrom ${CUSTOM_ISO_FILE}  \
  -boot d \
  -net nic,model=virtio \
  -net user,hostfwd=tcp::2222-:22
```