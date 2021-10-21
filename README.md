```bash
#!/bin/bash

if [ -z "$GUI_TYPE" ] ; then
  echo "Variable 'GUI_TYPE' not set, using default.."
  GUI_TYPE="ubuntu"
fi

if [ "$GUI_TYPE" == "kubuntu" ] ; then
  GUI_TYPE="sddm kubuntu-desktop"
elif [ "$GUI_TYPE" == "ubuntu" ] ; then
  GUI_TYPE="gdm3 ubuntu-gnome-desktop ubuntu-software"
else
  echo "Possible value of variable 'GUI_TYPE' is 'ubuntu' or 'kubuntu'.."
  exit
fi

echo "GUI_TYPE: $GUI_TYPE"

if [ -z "$DISK_TYPE" ] ; then 
  echo "Variable 'DISK_TYPE' not set, using default.."
  DISK_TYPE="nvme0"
fi

if [[ "$DISK_TYPE" =~ ^sd[a-z]{1}$ ]] ; then
  DISK_TYPE="/dev/$DISK_TYPE"
elif  [[ "$DISK_TYPE" =~ ^nvme[0-9]{1}$ ]] ; then
  DISK_TYPE="/dev/${DISK_TYPE}n1"
else
  echo "Possible value of variable 'DISK_TYPE' is 'nvmeX' or 'sdX'.. check README.md"
  exit
fi

echo "DISK_TYPE: $DISK_TYPE"

apt-get update -y
apt-get install -y mkisofs xorriso gzip cpio rsync wget p7zip-full

mkdir -p "$BASE_DIR"

cp "download/image.iso" "$BASE_DIR"

UBUNTU_ISO="$BASE_DIR/image.iso" && touch "$UBUNTU_ISO"
UBUNTU_MBR="$BASE_DIR/ubuntu.mbr" && touch "$UBUNTU_MBR"
UBUNTU_ISO_DIR="$BASE_DIR/ubuntu" && mkdir -p "$UBUNTU_ISO_DIR"
BUILD_DIR="$BASE_DIR/build" && mkdir -p "$BUILD_DIR"
TEMP_DIR="$BASE_DIR/temp" && mkdir -p "$TEMP_DIR"
CUSTOM_ISO_DIR="$BASE_DIR/custom" && mkdir -p "$CUSTOM_ISO_DIR"
CUSTOM_ISO="$BASE_DIR/$1" && touch "$CUSTOM_ISO"

TXT_CFG="$BUILD_DIR/txt.cfg"
GRUB_CFG="$BUILD_DIR/boot/grub/grub.cfg"

TXT_LINE="append vga=788 initrd=initrd.gz \-\-\- quiet"
GRUB_LINE="linux\t\/linux \-\-\- quiet"

PRESEED="\/preseed.cfg"
APPEND_LINE="auto=true priority=critical file=$PRESEED hostname=changeme"

7z x "$UBUNTU_ISO" -o"$UBUNTU_ISO_DIR"
rm -rf "$UBUNTU_ISO_DIR"/\[BOOT\]
rsync -av "$UBUNTU_ISO_DIR/" "$BUILD_DIR"
dd if="$UBUNTU_ISO" bs=1 count=446 of="$UBUNTU_MBR"

sed -i "s/$GRUB_LINE/$GRUB_LINE $APPEND_LINE/g" $GRUB_CFG
sed -i "s/$TXT_LINE/$TXT_LINE $APPEND_LINE/g" $TXT_CFG

cp "$BUILD_DIR/initrd.gz" "$TEMP_DIR"
mkdir -p "$TEMP_DIR/initrd"
(cd "$TEMP_DIR/initrd" && gunzip -c "$BUILD_DIR/initrd.gz" | cpio -id)

cat > "$TEMP_DIR/initrd/preseed.cfg" << 'EndOfMessage'
d-i debian-installer/language string en
d-i debian-installer/country string US
d-i debian-installer/locale string en_US.UTF-8

d-i keyboard-configuration/layout select us
d-i keyboard-configuration/variant select English (US)
d-i keyboard-configuration/modelcode string pc105
d-i localechooser/supported-locales multiselect en_US.UTF-8, pl_PL.UTF-8

d-i netcfg/choose_interface select auto
d-i netcfg/link_wait_timeout string 15
d-i netcfg/dhcpv6_timeout string 15
d-i netcfg/dhcp_timeout string 15
#d-i netcfg/hostname string hostname
#d-i netcfg/get_hostname string hostname
#d-i netcfg/get_domain string hostdomain

d-i mirror/country string manual
d-i mirror/http/hostname string pl.archive.ubuntu.com
d-i mirror/http/directory string /ubuntu
d-i mirror/suite string focal

d-i user-setup/allow-password-weak boolean true

# mkpasswd -m sha-512 -S $(pwgen -ns 16 1) mypassword
#d-i passwd/root-login boolean true
#d-i passwd/root-password-crypted password $6$rdp2Pq9okmQm$mrIjThChupd4A9zbT4CIk9YXbhmFPWhobNsUk7bKApMtdWxaWJDhRcbB0cXUBvxbDZGxz2uOrRa1ga/Z1a29H1
d-i passwd/make-user boolean true
# d-i passwd/user-fullname string ChangeMe
# d-i passwd/username string changeme
d-i passwd/user-uid string 1001
d-i passwd/user-default-groups string sudo
d-i passwd/user-password-crypted password $6$xqjeJ9TzDByw1chl$.9XDlX63uzgpmvE2GM89oapRBZoZgKelPeGqguIFW3RPJtNRqZ.p8a3SblPAOTZYnz7f.KBhLA/2vk0EUNpFt/

d-i clock-setup/utc boolean true
d-i time/zone string EU/Warsaw
d-i clock-setup/ntp boolean true
d-i clock-setup/ntp-server string 0.pl.pool.ntp.org

d-i partman-auto/init_automatically_partition select biggest_free
d-i partman/unmount_active boolean true
d-i partman-auto/disk string DISK_TYPE
d-i partman-crypto/weak_passphrase boolean true
d-i partman-auto/method string crypto
d-i partman-efi/non_efi_system boolean true
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-auto-lvm/guided_size string max
d-i partman-auto-lvm/new_vg_name string crypt
d-i partman-auto/choose_recipe select root-encrypted
d-i partman-auto/expert_recipe string                         \
      root-encrypted ::                                       \
              1 1 1 free                                      \
                      $bios_boot{ }                           \
                      method{ biosgrub }                      \
              .                                               \
              1024 150 1024 fat32                             \
                      $iflabel{ gpt }                         \
                      $reusemethod{ }                         \
                      $gptonly{ }                             \
                      $primary{ }                             \
                      method{ efi } format{ }                 \
                      use_filesystem{ } filesystem{ fat32 }   \
              .                                               \
              1024 100 1024 ext4                              \
                      $gptonly{ }                             \
                      $primary{ } $bootable{ }                \
                      method{ format } format{ }              \
                      use_filesystem{ } filesystem{ ext4 }    \
                      mountpoint{ /boot }                     \
              .                                               \
              8192 200 -1 ext4                                \
                      $lvmok{ } lv_name{ root }               \
                      in_vg { crypt }                         \
                      $gptonly{ }                             \
                      $primary{ }                             \
                      method{ format } format{ }              \
                      use_filesystem{ } filesystem{ ext4 }    \
                      mountpoint{ / }                         \
              .
d-i partman/default_filesystem string ext4
d-i partman-partitioning/no_bootable_gpt_biosgrub boolean false
d-i partman-partitioning/no_bootable_gpt_efi boolean false
d-i partman-basicfilesystems/no_mount_point boolean false
d-i partman-basicfilesystems/choose_label string gpt
d-i partman-basicfilesystems/default_label string gpt
d-i partman-partitioning/choose_label string gpt
d-i partman-partitioning/default_label string gpt
d-i partman/choose_label string gpt
d-i partman/default_label string gpt
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

d-i pkgsel/updatedb boolean true
d-i pkgsel/upgrade select full-upgrade
d-i pkgsel/language-packs multiselect en, pl
d-i pkgsel/update-policy select unattended-upgrades

d-i cdrom-detect/eject boolean true
d-i finish-install/reboot_in_progress note

d-i preseed/late_command string in-target apt-get update -y ; \
                                in-target apt-get upgrade -y ; \
                                in-target apt-get dist-upgrade -y ; \
                                in-target apt-get install -y GUI_TYPE ; \
                                in-target apt-get install -y openssh-server build-essential lvm2 cryptsetup ; \
                                in-target apt-get install -y sudo gdebi wget perl gawk sed python3 vim curl gnupg gnupg2 nmap traceroute git htop ; \
                                in-target apt-get --fix-broken install -y ; \
                                cp /init.sh /target/tmp/init.sh ; \
                                chmod 755 /target/tmp/init.sh ; \
                                in-target /bin/bash /tmp/init.sh ;
EndOfMessage

sed -i "s/GUI_TYPE/$GUI_TYPE/g" "$TEMP_DIR/initrd/preseed.cfg"
sed -i "s|DISK_TYPE|$DISK_TYPE|g" "$TEMP_DIR/initrd/preseed.cfg"

cat > "$TEMP_DIR/initrd/init.sh" << "MultiEndOfMessage"

MultiEndOfMessage

chmod 644 "$TEMP_DIR/initrd/init.sh"
chmod 644 "$TEMP_DIR/initrd/preseed.cfg"

(cd "$TEMP_DIR/initrd" && find . | cpio -o -H newC | gzip) > "$TEMP_DIR/initrd.gz"

mv "$TEMP_DIR/initrd.gz" "$BUILD_DIR"

chmod 444 "$BUILD_DIR/initrd.gz"

xorriso -as mkisofs -r -V "ISO Image" \
  -cache-inodes -J -l \
  -isohybrid-mbr "$UBUNTU_MBR" \
  -c boot.cat \
  -b isolinux.bin \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -eltorito-alt-boot \
  -e boot/grub/efi.img \
  -no-emul-boot -isohybrid-gpt-basdat \
  -o "$CUSTOM_ISO" \
  "$BUILD_DIR"
```
