# 黑群晖安装问题

## 环境
1. 常用工具:   
   * telnet 工具 putty (下载: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)  
   * ssh 工具 FinalShell (下载: https://www.hostbuf.com/t/988.html)  
   * sftp 工具 WinSCP (下载: https://winscp.net/eng/index.php)
   * 文本编辑工具 Notepad3 (下载: https://github.com/rizonesoft/Notepad3/releases) (抵制Notepad++)   
   * 镜像写盘工具 Rufus (下载: https://rufus.ie/zh/)
   * 磁盘管理工具 Diskgenius (下载: 不敢说)

2. 常用链接: 
   * GitHub Proxy: 
     * https://ghproxy.com/
   * 传文件: 
     * `curl -skL --insecure -w '\n' --upload-file output.zip https://transfer.sh`
   * 查找: 
     * http://find.synology.cn/
     * http://find.synology.com/
   * 下载: 
     * https://archive.synology.cn/download/Os/DSM
     * https://archive.synology.com/download/Os/DSM
   * 型号列表: 
     * https://kb.synology.cn/zh-cn/DSM/tutorial/What_kind_of_CPU_does_my_NAS_have
     * https://kb.synology.com/zh-hk/DSM/tutorial/What_kind_of_CPU_does_my_NAS_have
   * RAID计算: 
     * https://www.synology.cn/zh-cn/support/RAID_calculator
     * https://www.synology.com/zh-tw/support/RAID_calculator

## 群晖系统
1. ssh 开启 root 权限
    ```
    sudo -i
    sed -i 's/^.*PermitRootLogin.*$/PermitRootLogin yes/' /etc/ssh/sshd_config  
    synouser --setpw root xxxxxx  # xxxxxx 为你要设置的密码
    reboot
    ```
2. 群晖系统下挂载引导盘
    ```
    sudo -i
    echo 1 > /proc/sys/kernel/syno_install_flag
    ls /dev/synoboot*    # 正常会有 /dev/synoboot  /dev/synoboot1  /dev/synoboot2  /dev/synoboot3
    # 挂载第1个分区
    mkdir -p /tmp/synoboot1 
    mount /dev/synoboot1 /tmp/synoboot1 
    ls /tmp/synoboot1/
    # 挂载第2个分区
    mkdir -p /tmp/synoboot2
    mount /dev/synoboot2 /tmp/synoboot2
    ls /tmp/synoboot2/
    ```
3. 群晖 opkg 包管理
    ```
    wget -O - https://bin.entware.net/x64-k3.2/installer/generic.sh | /bin/sh
    /opt/bin/opkg update
    /opt/bin/opkg install rename
    ```

## 群晖引导态
1. 开启 telnet / ssh
  * telnet 用户: root 密码: null (无)
  * 任何引导在安装界面打开 http://\<ip\>:5000/webman/start_telnet.cgi 可开启 telnet.
  * 传统引导包含 early-telnet 会自动打开 telnet  
  * 传统引导包含 misc 会自动打开 web ttyd, (http://\<ip\>:7681 链接)
  * arpl 编译前执行以下命令编译以后会自动打开 telnet
    ```
    [ -z "$(grep "inetd" files/board/arpl/overlayfs/opt/arpl/ramdisk-patch.sh)" ] && sed -i '/# Build modules dependencies/i\# Enable Telnet\necho "inetd" >> "${RAMDISK_PATH}/addons/addons.sh"\n' files/board/arpl/overlayfs/opt/arpl/ramdisk-patch.sh
    ```
2. 串口
  * 默认群晖引导日志输出到 COM0(ttyS0), 使用串口工具监听 COM0 即可.
  * 注意: Vmware 虚拟机无法监听COM0, 所以 Vmware 要 在引导时修改cmdline中的 "console=ttyS0,115200n8" 为 "console=ttyS1,115200n8" 改到COM1 即可.


## 调试
  ```
  # 磁盘相关
  fdisk -l                                      # 查看硬盘信息 
  ls /sys/block/                                # 查看块设备  
  ls /sys/block/sd*                             # 查看识别的 sata 硬盘  
  ls /sys/block/sata*                           # 查看识别的 sata 硬盘  
  ls /sys/block/nvme*                           # 查看识别的 nvme 硬盘  
  cat /sys/block/sd*/device/syno_block_info     # 查看识别的 sata 硬盘挂载点 (非设备树(dtb)的型号)  
  cat /sys/block/sata*/device/syno_block_info   # 查看识别的 sata 硬盘挂载点 (设备树(dtb)的型号)  
  cat /sys/block/nvme*/device/syno_block_info   # 查看识别的 nvme 硬盘挂载点  
  # 日志相关
  dmesg                                         # 内核日志
  cat /var/log/linuxrc.syno.log                 # 引导态下启动日志
  cat /var/log/messages                         # 引导态下操作日志
  ```

1. 关于硬盘的问题
  * 引导态下查看磁盘信息是否与实际相符, 如果不符基本确定磁盘扩展卡的驱动存在问题,
  * 如果引导态处识别到了, 但是系统内没有识别到，请检查 map三个参数/dtb文件.

  * 引导状态下格盘
    ```
    fdisk -l   # 找到需要格式化的磁盘, 比如 /dev/sata2
    echo -e "m\nd\n1\nm\nd\n2\nm\nd\n3\nm\nw" > fdisk.txt; fdisk /dev/sata2 < fdisk.txt > /dev/null; rm fdisk.txt
    ```
