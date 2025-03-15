


# Debian automated installation using preseed file

```sh

git clone git@github.com:dorofeevdspb/debian-autoinstall2.git

```

## Image download
проверьте последний релиз

Download [debian 12 netinst iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.9.0-amd64-netinst.iso)

```sh

wget -c https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/\
  debian-12.9.0-amd64-netinst.iso

```
## Pressed file
The preseed file in this repository is provided as an example. Let's use it as a starting point for now.

## Execute the follwing commands:

1. Define the environment variables:
```shell
export ISO_FILE="${HOME}/iso/debian-12.9.0-amd64-netinst.iso"
export PRESED_FILE="preseed.cfg"
export MNT_DIR="${PWD}/mnt"
export CUSTOM_DIR="${PWD}/custom"
export CUSTOM_ISO_FILE="${PWD}/debian-12-custom.iso"

export SUBNET="172.20.0.0"
export NETMASK="255.255.0.0"
export GATEWAY="172.20.0.1"
export ADDRESS="172.20.0.52"
export DNS_SERVERS="172.20.0.1"

export HOST_NAME="debian"
export HOST_DOMAIN="lan"

export HOST_USER="master"
export HOST_PASS=`openssl rand -hex 20`
export HOST_TZ="Europe/Moscow"
export HOST_AUTHORIZED_KEY="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCTLms1Cc3VERlpvERj+9Kd+f6NkK1+Hx3DLgXIeiLIxJ727Ptuf6DO2gSKRkJUnSi0YmvBtU86S6YVZFPM8MzkobC8iYH/UQLLJwJd9zkV7W+DZjn7z/6uOAeTIRCKs/wjgQRl8BheXSt9TQI9dBXtGbw+2aEyIyLp+J2sGwDiAQior+HStGawVheW/jIO/7e/iG5HSh3thfDtD3aVxbUCux27HrYueDsdHZp7hyCXHVpvlk6XsMZHz7Duerr8s3HnnHdw6ntmynnOoC+Icynr3cdZdiUwOoy2x1Ldm9pXcZGp7Uf4u9XiFkOQKlUTez6KgvHS3FHE5qCBQNFLumTTxGldUb8yTFr3l20/gJVtDeTrC+RGHTawJxO35sn7L/UmgSM+zdmtO84plBVkKQveJvCBlWwmcZ7JKgqzI8zjaqJcH3Ue5FZHyzk9YIKk6ksAK0QUio5vaoRUeZETVLqmj9qXbAO6W/uhTzSB9eSTCg6s6kvZ6liQUojmJ/45E+DeYz69RHEZhpsUed0wnJCd+i0EAg7G6mzLy7rDW1rEcZeU9OiT0fE1HPHGYhVqcNTFZHpKASPWqhSEIudQgPHWeVmv8ihZWNqO1sPRmYp3c7XrPKczvdoLNLbBbLEXe7Y6sm4M7K++uYTkH/VRX1ErJ03gFdOVG2NDojS9rpBZEw== dorofeev.d.spb@gmail.com"

# Required only for qemu-kvm
#export NAT_INTERFACE="enp4s0"
#export INTERFACE="net0"
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