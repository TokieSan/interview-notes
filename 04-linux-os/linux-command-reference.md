# Linux Command Reference

> Merged from 2024–2025 Meta Production Engineer prep notes. Overlapping copies were deduplicated; the newer, denser notes in this folder take precedence where they disagree.

# Shell & text-processing notes

### Linux Notes

- Assume we have a file with things that we need the first field for (splitted by spaces):
	- To get the first field of the file, you can do awk '{print $1}' access.log 
	- You can alse use "cut" with delimiter of space (-d' ') and picking the first field (-f1): cat access.log |cut -d' ' -f1. 
- $#    Stores the number of command-line arguments that were passed to the shell program.
- $?    Stores the exit value of the last command that was executed.
- $0    Stores the first word of the entered command (the name of the shell program).
- $*    Stores all the arguments that were entered on the command line ($1 $2 ...).
- "$@"  Stores all the arguments that were entered on the command line, individually quoted ("$1" "$2" ...).
- Use the **sed** command to search and replace patterns in files quickly. The basic syntax looks like this: `sed [options] 'subcommand/new_pattern/target_pattern' input_file`
- Here’s an example of a sed command that changes **red** in **colors.txt** and **hue.txt** with **blue**: `sed 's/red/blue' colors.txt hue.txt`
- Learn this to get awk: `awk -F':' '{ total += $2; students[$1] = $2 } END { average = total / length(students); print "Average:", average; print "Above average:"; for (student in students) if (students[student] > average) print student }' score.txt`
- The **cut** command selects specific sections from a file and prints them as a Terminal output, its format is `cut options file` where:
	-f – selects a specific row field.
	-b – cuts the line by a specified byte size.
	-c – sections the line using a specified character.
	-d – separates lines based on delimiters.
- The tee command outputs another command’s results to both the Terminal and a file. It’s helpful if you want to use the data for further processing or backups. Here’s the syntax: `command | tee [options] file_name` If the specified file doesn’t exist, tee will create it. Be careful when using this command since it will overwrite the existing content. To preserve and append existing data, add the -a option.
- fuser

	Display process IDs currently using files or sockets.
	More information: https://manned.org/fuser.
	
	 - Find which processes are accessing a file or directory:
	   fuser path/to/file_or_directory
	
	 - Show more fields (USER, PID, ACCESS and COMMAND):
	   fuser --verbose path/to/file_or_directory
	
	 - Identify processes using a TCP socket:
	   fuser --namespace tcp port
	
	 - Kill all processes accessing a file or directory (sends the SIGKILL signal):
	   fuser --kill path/to/file_or_directory
	
	 - Find which processes are accessing the filesystem containing a specific file or directory:
	   fuser --mount path/to/file_or_directory
	
	 - Kill all processes with a TCP connection on a specific port:
	   fuser --kill port/tcp
   - netstat -ltp: ports connections in use
   - sudo systemctl set-environment MYSQLD_OPTS="--skip-grant-tables --skip-networking"
	   - Here, that's how to run with specific environment set, here I added stuff to ignore missing password

# Essential tools by resource

### Essential Tools & Commands

#### Process Tools
```bash
# Process viewing
ps aux           # List all processes
top/htop         # Real-time process viewer
pstree          # Process tree view
strace          # Trace system calls

# Process Analysis
pidstat         # Per-process statistics
nice/renice     # Adjust process priority
kill/pkill      # Send signals to processes
```

#### Memory Tools
```bash
# Memory viewing
free -m         # Memory usage overview
vmstat         # Virtual memory stats
pmap           # Process memory map
slabtop        # Kernel memory usage

# Memory Analysis
sar -r         # Memory utilization history
/proc/meminfo  # Detailed memory info
```

#### Disk & I/O Tools
```bash
# Disk usage
df -h          # Filesystem usage
du -sh         # Directory size
iostat         # I/O statistics
iotop          # I/O monitoring

# File operations
lsof           # List open files
fuser          # File user processes
```

#### Network Tools
```bash
# Network monitoring
netstat        # Network statistics
ss             # Socket statistics
ip addr        # IP configuration
tcpdump        # Packet capture
iptables       # Firewall management
```

# Sysadmin command reference

### 1. System Management

#### A. Process Management

##### 1. Process Monitoring
```bash
# View processes
ps aux                    # All processes
ps -ef                    # Full format
ps -u username           # User processes
top                      # Real-time process viewer
htop                     # Enhanced top

# Process information
pidof process_name       # Get process ID
pgrep process_name       # Advanced process ID search
pstree                   # Process tree
strace command           # Trace system calls

# Process control
kill -l                  # List signals
kill PID                 # Send SIGTERM
kill -9 PID             # Send SIGKILL
killall process_name     # Kill by name
pkill pattern           # Kill by pattern
nice -n 10 command      # Run with nice value
renice 10 PID           # Change nice value
```

##### 2. System Resources
```bash
# Memory usage
free -h                  # Display memory usage
vmstat 1                 # Virtual memory stats
swapon -s               # Swap usage

# CPU information
lscpu                   # CPU information
uptime                  # Load average
mpstat 1               # CPU statistics

# Disk usage
df -h                   # Disk space usage
du -sh directory       # Directory size
iostat                 # I/O statistics
iotop                  # I/O monitoring
```

#### B. Service Management

##### 1. Systemd Services
```bash
# Service control
systemctl start service_name
systemctl stop service_name
systemctl restart service_name
systemctl reload service_name
systemctl status service_name

# Service configuration
systemctl enable service_name     # Start at boot
systemctl disable service_name    # Don't start at boot
systemctl is-enabled service_name # Check if enabled

# System control
systemctl poweroff               # Shutdown
systemctl reboot                # Restart
systemctl suspend               # Suspend
systemctl hibernate             # Hibernate

# Service files
/etc/systemd/system/           # System service units
/lib/systemd/system/           # Vendor service units
```

##### 2. Init.d Services (Legacy)
```bash
# Service control
/etc/init.d/service_name start
/etc/init.d/service_name stop
/etc/init.d/service_name restart
service service_name start     # Alternative syntax

# Configuration
update-rc.d service_name defaults    # Enable service
update-rc.d service_name remove      # Disable service
```

#### C. Package Management

##### 1. APT (Debian/Ubuntu)
```bash
# Repository management
apt update                      # Update package list
apt upgrade                     # Upgrade packages
apt full-upgrade               # Upgrade with removals

# Package operations
apt install package_name       # Install package
apt remove package_name       # Remove package
apt purge package_name        # Remove with config
apt autoremove               # Remove unused deps
apt search pattern           # Search packages
apt show package_name        # Show package info

# Package files
/etc/apt/sources.list        # Repository list
/var/cache/apt/archives/     # Package cache
```

##### 2. YUM/DNF (RHEL/CentOS)
```bash
# Repository management
yum update                   # Update packages
yum check-update            # Check for updates

# Package operations
yum install package_name    # Install package
yum remove package_name     # Remove package
yum search pattern         # Search packages
yum info package_name      # Package information
yum provides filename      # Find package for file

# Repository files
/etc/yum.repos.d/         # Repository configs
/var/cache/yum/           # Package cache
```

### 2. Network Management

#### A. Network Configuration

##### 1. Interface Management
```bash
# Interface information
ip addr show              # Show interfaces
ip link show              # Link status
ifconfig                  # Traditional tool

# Interface configuration
ip addr add 192.168.1.10/24 dev eth0   # Add IP
ip addr del 192.168.1.10/24 dev eth0   # Remove IP
ip link set eth0 up                    # Enable interface
ip link set eth0 down                  # Disable interface

# Network testing
ping host                 # ICMP echo
traceroute host          # Trace route
mtr host                 # Advanced traceroute
```

##### 2. Routing
```bash
# Route management
ip route show            # Show routing table
ip route add default via 192.168.1.1   # Add default route
ip route del default                   # Delete default route

# Advanced routing
ip rule show             # Show routing rules
ip route show table 100  # Show specific table
```

#### B. Firewall Management

##### 1. IPTables
```bash
# Table management
iptables -L              # List rules
iptables -F              # Flush rules
iptables-save           # Save rules
iptables-restore        # Restore rules

# Rule management
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # Allow HTTP
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT    # Allow network
iptables -P INPUT DROP                           # Default policy

# Common rules
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # SSH
```

##### 2. UFW (Uncomplicated Firewall)
```bash
# Basic operations
ufw enable              # Enable firewall
ufw disable             # Disable firewall
ufw status             # Show status

# Rule management
ufw allow 80/tcp       # Allow HTTP
ufw deny 23/tcp        # Deny Telnet
ufw allow from 192.168.1.0/24    # Allow network
```

#### C. Network Services

##### 1. SSH
```bash
# SSH client
ssh user@host          # Connect to host
ssh -p 2222 user@host  # Connect to specific port
ssh-keygen            # Generate key pair
ssh-copy-id user@host # Copy public key

# SSH server config
/etc/ssh/sshd_config  # Server configuration
systemctl restart sshd # Restart server

# SSH tunneling
ssh -L 8080:localhost:80 user@host    # Local forward
ssh -R 8080:localhost:80 user@host    # Remote forward
```

##### 2. DNS
```bash
# DNS queries
dig domain           # DNS lookup
nslookup domain     # Name resolution
host domain         # Simple lookup

# DNS configuration
/etc/resolv.conf    # DNS resolver config
/etc/hosts         # Static host entries
```

### 3. Storage Management

#### A. Disk Management

##### 1. Partition Management
```bash
# Disk information
fdisk -l            # List partitions
parted -l           # List partitions
lsblk              # Block device info

# Partition creation
fdisk /dev/sda     # Interactive partitioning
parted /dev/sda    # GNU parted

# File systems
mkfs.ext4 /dev/sda1    # Create ext4 fs
mkfs.xfs /dev/sda2     # Create XFS fs
fsck /dev/sda1         # Check filesystem
```

##### 2. LVM (Logical Volume Management)
```bash
# Physical volumes
pvcreate /dev/sda1     # Create PV
pvdisplay              # Show PVs

# Volume groups
vgcreate vg0 /dev/sda1    # Create VG
vgdisplay                 # Show VGs

# Logical volumes
lvcreate -L 10G -n lv0 vg0    # Create LV
lvdisplay                     # Show LVs
```

#### B. Mount Management

##### 1. Mount Operations
```bash
# Mount points
mount /dev/sda1 /mnt           # Mount filesystem
umount /mnt                    # Unmount filesystem
mount -a                       # Mount all in fstab

# Auto-mounting
/etc/fstab                    # Filesystem table
# Example entry:
# /dev/sda1 /mnt ext4 defaults 0 0
```

##### 2. Storage Monitoring
```bash
# Disk usage
df -h               # Filesystem usage
du -sh /path       # Directory size
ncdu /path         # Interactive disk usage

# IO monitoring
iostat             # IO statistics
iotop              # IO top
```

### 4. Security Management

#### A. User Management

##### 1. User Operations
```bash
# User management
useradd username    # Create user
usermod -a -G group username    # Add to group
passwd username     # Set password
userdel username    # Delete user

# User information
id username         # User info
who                # Who is logged in
w                  # Who is logged in with details
```

##### 2. Permission Management
```bash
# File permissions
chmod 755 file     # Change mode
chown user:group file    # Change owner
chgrp group file        # Change group

# Special permissions
chmod u+s file     # Set SUID
chmod g+s file     # Set SGID
chmod +t directory # Set sticky bit
```

#### B. System Security

##### 1. SELinux/AppArmor
```bash
# SELinux
getenforce                # Get SELinux mode
setenforce 1             # Set enforcing
semanage port -a -t http_port_t -p tcp 8080 # Add port

# AppArmor
aa-status               # AppArmor status
aa-enforce /etc/apparmor.d/profile # Enforce profile
```

##### 2. Auditing
```bash
# Audit system
auditctl -w /etc/passwd -p wa    # Watch file
ausearch -f /etc/passwd          # Search audit log
aureport                         # Audit report

# System logs
journalctl                      # System journal
tail -f /var/log/syslog        # System log
```

### 5. Performance Tuning

#### A. System Tuning

##### 1. Kernel Parameters
```bash
# Sysctl management
sysctl -a                       # Show all parameters
sysctl vm.swappiness=60        # Set parameter
/etc/sysctl.conf               # Permanent config

# Memory management
echo 3 > /proc/sys/vm/drop_caches    # Clear caches
```

##### 2. Resource Limits
```bash
# Process limits
ulimit -a                      # Show limits
/etc/security/limits.conf      # Permanent limits

# System resources
/proc/sys/                     # System parameters
```

This guide covers essential Linux system administration tasks with practical examples and commands. Each section includes common operations and configurations administrators need to manage Linux systems effectively.

# Emergency commands

> ⚠️ These are destructive. Know them, and know why you'd hesitate: `drop_caches` throws away useful cache and causes an I/O storm; SysRq-`b` reboots without syncing disks.

### Common Emergency Commands
```bash
# Kill runaway process
kill -9 $(pidof process_name)

# Clear page cache
echo 3 > /proc/sys/vm/drop_caches

# Emergency reboot
echo b > /proc/sysrq-trigger

# Network reset
systemctl restart networking

# Process list when unresponsive
ps aux w >> /var/log/ps-snapshots/$(date +%F-%T)
```
