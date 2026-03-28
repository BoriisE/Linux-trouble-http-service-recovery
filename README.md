# Restoring and Launching the `trouble` HTTP Service

## Project Overview

In this task, the goal was to restore a broken virtual machine, locate and mount the logical volume containing the `trouble` service binary, fix several system-level issues, and finally make the HTTP service accessible in a browser.

According to the task description, the `trouble` executable is located on the filesystem of the `sysadmin` logical volume inside the `opt` volume group, and the final result should be successful access to the service through a web browser. The structure and sequence of steps in this README are based on the original practice assignment.

---

## Objective

Recover the system after multiple intentionally introduced failures and make sure that the `trouble` service:

- starts without errors;
- listens on the required port;
- is reachable externally through a reverse proxy;
- opens in a browser and returns the final flag.

---

## What Was Done

### 1. Fixing the System Boot

At boot, the system dropped into `initramfs` with the error:

```bash
ALERT! /dev/sda2 does not exist
```

Actions taken:

- checked the available block devices;
- identified the correct root partition as `/dev/vda2`;
- manually corrected the GRUB boot parameter to `root=/dev/vda2`;
- booted the system using the correct root device.

### 2. Resetting the Root Password

Since normal login was not possible, the standard recovery method using `init=/bin/bash` was used.

Actions taken:

- remounted the root filesystem in read-write mode;
- changed the `root` password;
- resumed normal initialization;
- after login, updated the bootloader configuration with `update-grub`.

### 3. Disk Analysis and Package Manager Recovery

After logging in, it was determined that the required data was stored in LVM on `/dev/vda3`.

Problems discovered at this stage:

- LVM utilities were missing;
- `/etc/apt/sources.list` was broken;
- later it was also confirmed that both networking and DNS were misconfigured.

Actions taken:

- restored a valid `sources.list`;
- identified the OS codename as `jammy`;
- prepared the system to install the required packages.

### 4. Restoring Network Connectivity

The `eth0` interface was not receiving an IP address.

Cause:

A wrong `MAC` address had been specified in the netplan configuration, so the interface could not receive its DHCP settings.

Actions taken:

- checked the actual MAC address of the interface;
- fixed `/etc/netplan/50-cloud-init.yaml`;
- applied the configuration with `netplan apply`;
- confirmed successful IPv4 address assignment.

This part of the task, including the netplan, DHCP, and MAC address issue, is described in the original assignment text. 

### 5. Fixing DNS Resolution

After restoring network connectivity, packages still could not be installed because hostname resolution was broken.

Actions taken:

- verified basic network reachability by IP;
- used `host`, `dig`, and `getent` for diagnostics;
- found incorrect settings in `nsswitch.conf`;
- restored the correct `hosts` line:

```bash
hosts: files dns
```

After that, DNS resolution started working normally, and package installation became possible again. This follows directly from the task text, where `getent` and the `hosts: files dns` line are used as the validation point for the resolver fix. 

### 6. Recovering LVM and the Filesystem

After installing `lvm2`, the inactive logical volume `/dev/opt/sysadmin` was discovered.

Actions taken:

- activated the logical volume with `lvchange -ay /dev/opt/sysadmin`;
- attempted to mount it;
- detected filesystem corruption;
- checked the superblock;
- identified the filesystem type as `XFS`;
- repaired it with `xfs_repair`;
- successfully mounted it to `/opt/sysadmin`.

The original task explicitly mentions activating the volume, checking the superblock, using `xfs_repair`, and locating the binary at `/opt/sysadmin/bin/trouble`.

### 7. Preparing and Starting the Service

Once the filesystem was restored, the file `/opt/sysadmin/bin/trouble` became accessible.

Actions taken:

- created a dedicated service user `trouble`;
- granted execute permissions to the binary;
- created a `systemd` unit file named `trouble.service`;
- ran `daemon-reload`;
- attempted to start the service.

### 8. Configuring SSH Access

For easier дальнейшая debugging and administration, SSH access was configured.

Actions taken:

- installed `openssh-server`;
- started the `ssh` service;
- added a public key to the `authorized_keys` file of the `trouble` user;
- configured passwordless `sudo` via `/etc/sudoers.d/trouble`.

### 9. Debugging Runtime Errors

Even after that, the service still did not start correctly, so step-by-step runtime diagnostics were performed.

#### Error 1: Too many open files

Using `strace`, the following error was identified:

```bash
EMFILE (Too many open files)
```

Actions taken:

- checked the limit with `ulimit -n`;
- reviewed `/etc/security/limits.conf`;
- increased file descriptor limits for the `trouble` user.

#### Error 2: lock on `/locks/lockfile.lock`

The next issue was related to `flock`.

Actions taken:

- found the process holding the lock file;
- identified the `watcher` process with `lsof`;
- stopped the process.

#### Error 3: Port 8080 already in use

After fixing the previous issues, the service still could not `bind` because port `8080` was already occupied.

Actions taken:

- checked listening ports with `ss -nltp`;
- identified the process using port `8080`;
- confirmed it was being restarted by `systemd`;
- stopped the conflicting `echo` service;
- started `trouble` again.

In the practice description, these three troubleshooting stages are given in the same order: `strace` with `EMFILE`, the `/locks/lockfile.lock` issue, then the `8080` port conflict and stopping the `echo` service.

### 10. Publishing the Service Through Nginx

The final issue was that the service was only listening on the loopback interface and was not directly accessible from a browser.

Actions taken:

- installed `nginx`;
- created a reverse proxy configuration;
- forwarded external traffic from port `8081` to `127.0.0.1:8080`.

Nginx configuration:

```nginx
server {
    listen 8081;
    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

This exact completion method — using Nginx to proxy an external request to the local `trouble` service — is explicitly described in the original task text.

---

## Tools and Technologies Used

- Ubuntu 22.04
- GRUB2
- initramfs
- LVM (`lvm2`)
- XFS (`xfsprogs`)
- APT
- Netplan
- DNS utilities: `host`, `dig`, `getent`
- `systemd`
- `strace`
- `lsof`
- `ss`
- OpenSSH
- Nginx

---

## Result

As a result, the virtual machine was fully recovered, the damaged storage and configuration issues were fixed, the `trouble` service was launched successfully, and external access was provided through Nginx reverse proxying.

This project demonstrates practical skills in:

- Linux troubleshooting;
- boot recovery;
- network and DNS debugging;
- LVM and filesystem recovery;
- service management with `systemd`;
- runtime error analysis;
- reverse proxy configuration with Nginx.
