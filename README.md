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

export SUBNET="10.14.0.0"
export NETMASK="255.255.255.0"
export GATEWAY="10.14.0.1"
export ADDRESS="10.14.0.10"
export DNS_SERVERS="10.2.1.40"

export HOST_NAME="debian"
export HOST_DOMAIN="cloud.int"

export HOST_USER="debian"
export HOST_PASS=`openssl rand -hex 20`
export HOST_TZ="Brazil/East"
export HOST_AUTHORIZED_KEY="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC4tzrayU6ahMhmWuicy+oFfy//9oB+2EdbbmDfA0d+k3SpYjWVqho64/L+sQIAN0RGBJx42GkbKi8B6AriPw8omLOCk2WSYW3ymEC7n3l32M5T4cLr8LIYwoMOBZkMtRc3H62PrHgDoTJLhUOvT2ewj1SLl7iU5gQuInwPE6jWooIb8R6KMUl31qNpkafCVPz5ovw0iYbDamHQF6sq081Xl39px2345T8TofIAocyBUfCOstmAvPaD9lXIV3j9JmPhAy0oweXpxdPiQzBHXepLh/jrvHrV5ggl2iwmLgF3uzwYdFlQN6eCniBtBEcGqEacb6oP2KHfHer04WIbAMHZ"

# Required only for qemu-kvm
export NAT_INTERFACE="enp4s0"
export INTERFACE="net0"
```

2. Create the preseed file:
```shell
envsubst < preseed.cfg.template > preseed.cfg
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
  -m 4096 \
  -enable-kvm \
  -smp 2 \
  -cpu host \
  -hda disk.img \
  -cdrom ${CUSTOM_ISO_FILE}  \
  -boot d \
  -net nic,model=virtio \
  -net user,hostfwd=tcp::2222-:22


```