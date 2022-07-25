## Prerequisite
- WSL2 (Ubuntu 18.04 LTS)
- Visual Studio

## Instructions
- git, 컴파일러 설치
```bash
$ sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses5-dev git bc
```

- Microsoft WSL2 커널 clone
```bash
$ git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
$ cd WSL2-Linux-Kernel
```

- 아래와 같이 menuconfig 설정 후 저장
```bash
$ export KCONFIG_CONFIG=Microsoft/config-wsl
$ make menuconfig
```
```bash
[*] Enable loadable module support

General setup  --->
    [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support

Device Drivers --->
    [*] Multiple devices driver support (RAID and LVM) --->
        <*> Device mapper support
    [*] Block Devices ---> 
        <*> Loopback device support 
    SCSI device support  --->
        <*> SCSI device support --->
        <*> SCSI disk support
        <*> SCSI generic support
        SCSI Transports --->
        <M> iSCSI Transport Attributes
        [*] SCSI low-level drivers  --->
        <M> iSCSI Initiator over TCP/IP     

File systems ---> 
     <*> FUSE (Filesystem in Userspace) support 

[*] Networking support  --->
   Networking options  --->
      [*] TCP/IP networking
```

- dwarves 설치(커널 컴파일시 BTF 생성 에러 제거를 위해 추가함)
```bash
$ sudo apt install dwarves
```

- 커널 컴파일
```bash
$ sudo make KCONFIG_CONFIG=Microsoft/config-wsl -j$(nproc)
```

- iSCSI 모듈 설치
```bash
$ sudo make modules_install
```

- 새로 생성한 커널 이미지(bzImage)를 현재 Windows(host)에 복사
```bash
$ cp ./arch/x86_64/boot/bzImage /mnt/c/Users/현재_윈도우_사용자명/
```

- Windows(host)의 사용자 디렉터리에 “.wslconfig” 파일 생성 후 아래와 같이 작성하여 저장
```bash
[wsl2]
kernel=C:\\Users\\현재_윈도우_사용자명\\bzImage
swap=0
localhostForwarding=true
```

- WSL2 종료 후 재시작
```bash
> wsl -d Ubuntu-18.04 --shutdown 
> wsl -d Ubuntu-18.04
```

- iSCSI module 로드
```bash
$ sudo modprobe -v libiscsi
$ sudo modprobe -v scsi_transport_iscsi
$ sudo modprobe -v iscsi_tcp
$ sudo modprobe -v libiscsi_tcp
```

## WSL2 systemd 활성화
패키지 설치
- daemonize
- dbus-user-session
- fontconfig
```bash
$ sudo apt-get update && sudo apt-get install -yqq daemonize dbus-user-session fontconfig
```

- /usr/sbin/start-systemd-namespace 생성 후 아래 내용 저장
```bash
#!/bin/bash

SYSTEMD_PID=$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')
if [ -z "$SYSTEMD_PID" ] || [ "$SYSTEMD_PID" != "1" ]; then
    export PRE_NAMESPACE_PATH="$PATH"
    (set -o posix; set) | \
        grep -v "^BASH" | \
        grep -v "^DIRSTACK=" | \
        grep -v "^EUID=" | \
        grep -v "^GROUPS=" | \
        grep -v "^HOME=" | \
        grep -v "^HOSTNAME=" | \
        grep -v "^HOSTTYPE=" | \
        grep -v "^IFS='.*"$'\n'"'" | \
        grep -v "^LANG=" | \
        grep -v "^LOGNAME=" | \
        grep -v "^MACHTYPE=" | \
        grep -v "^NAME=" | \
        grep -v "^OPTERR=" | \
        grep -v "^OPTIND=" | \
        grep -v "^OSTYPE=" | \
        grep -v "^PIPESTATUS=" | \
        grep -v "^POSIXLY_CORRECT=" | \
        grep -v "^PPID=" | \
        grep -v "^PS1=" | \
        grep -v "^PS4=" | \
        grep -v "^SHELL=" | \
        grep -v "^SHELLOPTS=" | \
        grep -v "^SHLVL=" | \
        grep -v "^SYSTEMD_PID=" | \
        grep -v "^UID=" | \
        grep -v "^USER=" | \
        grep -v "^_=" | \
        cat - > "$HOME/.systemd-env"
    echo "PATH='$PATH'" >> "$HOME/.systemd-env"
    exec sudo /usr/sbin/enter-systemd-namespace "$BASH_EXECUTION_STRING"
fi
if [ -n "$PRE_NAMESPACE_PATH" ]; then
    export PATH="$PRE_NAMESPACE_PATH"
fi
```

- /usr/sbin/enter-systemd-namespace 생성 후 아래 내용 저장

```bash
#!/bin/bash

if [ "$UID" != 0 ]; then
    echo "You need to run $0 through sudo"
    exit 1
fi

SYSTEMD_PID="$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')"
if [ -z "$SYSTEMD_PID" ]; then
    /usr/sbin/daemonize /usr/bin/unshare --fork --pid --mount-proc /lib/systemd/systemd --system-unit=basic.target
    while [ -z "$SYSTEMD_PID" ]; do
        SYSTEMD_PID="$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')"
    done
fi

if [ -n "$SYSTEMD_PID" ] && [ "$SYSTEMD_PID" != "1" ]; then
    if [ -n "$1" ] && [ "$1" != "bash --login" ] && [ "$1" != "/bin/bash --login" ]; then
        exec /usr/bin/nsenter -t "$SYSTEMD_PID" -a \
            /usr/bin/sudo -H -u "$SUDO_USER" \
            /bin/bash -c 'set -a; source "$HOME/.systemd-env"; set +a; exec bash -c '"$(printf "%q" "$@")"
    else
        exec /usr/bin/nsenter -t "$SYSTEMD_PID" -a \
            /bin/login -p -f "$SUDO_USER" \
            $(/bin/cat "$HOME/.systemd-env" | grep -v "^PATH=")
    fi
    echo "Existential crisis"
fi
```

- 권한 추가
```bash
$ sudo chmod +x /usr/sbin/enter-systemd-namespace
```

- sudo visudo 입력 후 sudoers에 추가
```markdown
Defaults        env_keep += WSLPATH
Defaults        env_keep += WSLENV
Defaults        env_keep += WSL_INTEROP
Defaults        env_keep += WSL_DISTRO_NAME
Defaults        env_keep += PRE_NAMESPACE_PATH
%sudo ALL=(ALL) NOPASSWD: /usr/sbin/enter-systemd-namespace
```

- /etc/bash.bashrc 의 맨 아래에 추가
```markdown
sudo sed -i 2a"# Start or enter a PID namespace in WSL2\nsource /usr/sbin/start-systemd-namespace\n" /etc/bash.bashrc
```

- Windows(Host)에 환경변수 추가
```bash
cmd.exe /C setx WSLENV BASH_ENV/u
cmd.exe /C setx BASH_ENV /etc/bash.bashrc
```

- open-iscsi 설치
```bash
$ sudo apt install open-iscsi
```

- iscsi initiator 시작
```bash
$ sudo /etc/init.d/open-iscsi start
```

## WSL2에 USB 연결
- Windows(Host)에서 iSCSIConsole 프로젝트(https://github.com/TalAloni/iSCSIConsole) 다운로드 후 빌드
- 관리자 권한으로 iSCSIConsole 실행
![iSCSI Console Main 화면](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/99f681f4-7459-4a05-99a7-9df6d6593629/Untitled.png)
iSCSI Console Main 화면

- iSCSI Console에서 Target USB 추가 (Add Target →Add Physical Disk)
![Target USB 추가](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b29b08d-7809-4638-b556-8903df539a9c/Untitled.png)
Target USB 추가

![추가된 Target 확인](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dbe2f4dc-5a7e-4a6b-bc6b-02494ff043d1/Untitled.png)
추가된 Target 확인

- 저장장치 탐색 (매 WSL2 부팅 시 마다 환경변수 설정 필요)
```bash
$ export WSLHOSTIP=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
$ sudo iscsiadm -m discovery -t st -p $WSLHOSTIP

172.26.240.1:3260,-1 iqn.1991-05.com.microsoft:target1

$ sudo iscsiadm -m node

172.26.240.1:3260,-1 iqn.1991-05.com.microsoft:target1
```

- iscsi 연결
```bash
$ sudo iscsiadm -m node --targetname "iqn.1991-05.com.microsoft:target1" --portal "$WSLHOSTIP:3260" --login

Logging in to [iface: default, target: iqn.1991-05.com.microsoft:target1, portal: 172.26.240.1,3260] (multiple)
Login to [iface: default, target: iqn.1991-05.com.microsoft:target1, portal: 172.26.240.1,3260] successful.
```

- usb 연결 확인
```bash
$ sudo fdisk -l /dev/sd*
$ lsblk
```

- iscsi 연결 해제
```bash
$ sudo iscsiadm -m node --targetname "iqn.1991-05.com.microsoft:target1" --portal "$WSLHOSTIP:3260" --logout
```

## References
- [WSL2 및 Ubuntu 설치 : 네이버 블로그 (naver.com)](https://blog.naver.com/PostView.naver?blogId=chcbaram&logNo=222525998696)
- https://github.com/jovton/USB-Storage-on-WSL2
- [Running Snaps on WSL2 (Insiders only for now) - snapd - snapcraft.io](https://forum.snapcraft.io/t/running-snaps-on-wsl2-insiders-only-for-now/13033)
- https://github.com/TalAloni/iSCSIConsole
