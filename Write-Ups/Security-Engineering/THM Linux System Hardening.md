## Physical Security

One of the security principles is Defence-in-Depth. Hence, we should always think in terms of security. It would be best if you prevented potential adversaries from being able to gain physical access to your computer systems. If an intruder can gain physical access to your office, it would be easy to remove the disk drive and take it away. That's a simple attack that requires minimal technical skills.

Let's say you have taken the necessary measures to prevent intruders from taking the disk drive or the whole computer system. Moreover, you have ensured that your system passwords are complex and impossible to guess. If an intruder can access the system physically, it is a non-sophisticates task to use GRUB, a popular Linux bootloader, to reset the root password account. Hence we have the adage "boot access = root access". 

It is evident that we need to ensure physical security for our computer systems; however, in the unlikely event that physical security is breached, we need to provide additional layers of protection. Many BIOS and UEFI firmware allows you to add a boot password. This password will prevent unauthorised users from booting the system. However, this can only be used for personal systems; it won't make sense to use it on servers as this will require someone to be physically present to supply the boot password. 

We can consider adding a GRUB password depending on the Linux system we want to protect. Many tools help achieve that. One tool is `grub2-mkpasswd-pbkdf2`, which prompts you to input your password twice and generates a hash for you. The resulting hash should be added to the appropriate configuration file depending on the Linux distribution. This configuration would prevent unauthorised users from resetting your root password. It will require the user to supply a password to access advanced boot configurations via GRUB, including logging in with root access. 

It is important to note that adding a password for GRUB is not available for systems deployed using cloud service providers; a GRUB password does not make sense as you don't have access to the physical terminal. 

### Q&A

What command can you use to create a password for the GRUB bootloader?

`grub2-mkpasswd-pbkdf2`

What does PBKDF2 stand for?

`Password-Based Key Derivation Function 2`

## Filesystem Partitioning and Encryption

Encryption makes data unreadable without the decryption key. In the scenario where an adversary has complete physical access to your laptop, for instance, by stealing it, we want to ensure that it won't be of any use to them. A disk drive full of encrypted data should be as good as a damaged one. 

There are various software systems and tools that provide encryption to Linux systems. Many modern Linux distributions ship with LUKS (Linux Unified Key Setup). When a partition is encrypted with LUKS, the disk layout would have the following fields:

- **LUKS phdr:** It stands for _LUKS Partition Header._ LUKS phdr stores information about the UUID (Universally Unique Identifier), the used cipher, the cipher mode, the key length, and the checksum of the master key
- **KM:** KM stands for _Key Material_, where we have KM1, KM2, ..., KM8. Each key material section is associated key material section contains a copy of the master key encrypted with a user's password. In other words, we might have the master key encrypted with the first user's password and saved in KM1, encrypted with the second user's password and saved in KM2, and so on.
- **Bulk Data:** This refers to the data encrypted by the master key. The master key is saved and encrypted by the user's password in a key material section. 

LUKS reuses existing block encryption implementations. The pseudocode to encrypt data uses the following syntax:

`enc_data = encrypt(cipher_name, cipher_mode, key, original, original_length)`

As we can see, LUKS works with different ciphers and cipher modes. _Original_ refers to the plaintext data of length, _original_length_. The user-supplied password is used to derive the encryption key; the key is derived using password-based key derive function 2 (PBKDF2).

`key = PBKDF2(password, salt, iteration_count, derived_key_length)`

Using a _salt_ with a hash function repeating an _iteration count_ ensures that the resulting key is secure enough for encryption.

Similarly, to decrypt data and restore the original plaintext, LUKS uses the following syntax:

`original = decrypt(cipher_name, cipher_mode, key, enc_data, original_length)`

Most distributions let you encrypt a drive using a graphical interface. However, if you would like to setup LUKS from the command line, the steps are along these lines:

- Install `cryptsetup-luks`. (You can issue `apt install cryptsetup`, `yum install cryptsetup-luks` or `dnf install cryptsetup-luks` for Ubuntu/Debian, RHEL/Cent OS, and Fedora, respectively.)
- Confirm the partition name using `fdisk -l`, `lsblk` or `blkid`. (Create a partition using `fdisk` if necessary.)
- Set up the partition for LUKS encryption: `cryptsetup -y -v luksFormat /dev/sdb1`. (Replace `/dev/sdb1` with the partition name you want to encrypt.)
- Create a mapping to access the partition: `cryptsetup luksOpen /dev/sdb1 EDCdrive`.
- Confirm mapping details: `ls -l /dev/mapper/EDCdrive` and `cryptsetup -v status EDCdrive`.
- Overwrite existing data with zero: `dd if=/dev/zero of=/dev/mapper/EDCdrive`.
- Format the partition: `mkfs.ext4 /dev/mapper/EDCdrive -L "Strategos USB"`.
- Mount it and start using it like a usual partition: `mount /dev/mapper/EDCdrive /media/secure-USB`.

The terminal output below shows an example of encrypting a USB flash memory that initially has one partition at `/dev/sdb1` in NTFS format:

```shell-session
root@AttackBox# user@TryHackMe$ sudo lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sdb           8:16   1  14.3G  0 disk 
└─sdb1        8:17   1  14.3G  0 part /run/media/strategos/Strategos
[...]

user@TryHackMe$ sudo blkid
[...]
/dev/sdb1: LABEL="Strategos" BLOCK_SIZE="512" UUID="59402B4A4E12CD06" TYPE="ntfs"

user@TryHackMe$ sudo cryptsetup -y -v luksFormat /dev/sdb1
WARNING: Device /dev/sdb1 already contains a 'ntfs' superblock signature.

WARNING!
========
This will overwrite data on /dev/sdb1 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdb1: 
Verify passphrase: 
Existing 'ntfs' superblock signature on device /dev/sdb1 will be wiped.
Existing 'dos' partition signature on device /dev/sdb1 will be wiped.
Key slot 0 created.
Command successful.

user@TryHackMe$ sudo cryptsetup luksOpen /dev/sdb1 EDCdrive

user@TryHackMe$ sudo mkfs.ext4 /dev/mapper/EDCdrive -L "Strategos USB"
mke2fs 1.46.5 (30-Dec-2021)
[...]
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

user@TryHackMe$ sudo mount /dev/mapper/EDCdrive /media/secure-USB
```

If you want to check the LUKS settings, you can issue the command `cryptsetup luksDump /dev/sdb1`. In the terminal output below, the UUID can be seen, alongside the fact that the cipher used is `aes-xts-plain64`. As for the key, PBKDF2 used SHA 256 with the provided salt for 194180 iterations. 

```shell-session
root@AttackBox# sudo cryptsetup luksDump /dev/sdb1
LUKS header information
Version:        2
Epoch:          3
Metadata area:  16384 [bytes]
Keyslots area:  16744448 [bytes]
UUID:           41199bad-d753-4dce-9284-0a81d27acc95
Label:          (no label)
Subsystem:      (no subsystem)
Flags:          (no flags)

Data segments:
  0: crypt
    offset: 16777216 [bytes]
    length: (whole device)
    cipher: aes-xts-plain64
    sector: 512 [bytes]

Keyslots:
  0: luks2
    Key:        512 bits
    Priority:   normal
    Cipher:     aes-xts-plain64
    Cipher key: 512 bits
    PBKDF:      argon2id
    Time cost:  4
    Memory:     965922
    Threads:    4
    Salt:       bd 45 40 96 27 93 cd fa 50 60 f4 28 d4 d8 b2 bd 
                58 69 72 72 35 2f 26 9c a8 14 ef 91 04 b2 dc cd 
    AF stripes: 4000
    AF hash:    sha256
    Area offset:32768 [bytes]
    Area length:258048 [bytes]
    Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
    Hash:       sha256
    Iterations: 194180
    Salt:       6e e1 70 a4 3f d6 71 44 c8 e6 84 4d 99 51 7d c9 
                49 66 bf 37 61 b8 c3 d2 4e aa f7 25 27 e2 b3 8a 
    Digest:     c1 87 99 a1 d1 7a 05 8a ca cd 13 74 f0 33 ef 3a 
                98 c9 d7 a8 70 93 e2 ac 07 0f 2a 5c 89 f1 18 1d 
```

### Q&A

What does LUKS stand for?

`Linux Unified Key Setup`

What is the flag in the secret vault?

`THM{LUKS_not_LUX}`

## Firewall

Let’s briefly revisit the client/server model before we start. Any networked device can be a client, a server, or both simultaneously. The server offers a service, and the client connects to the server to use it. Examples of servers include web servers (HTTP and HTTPS), mail servers (SMTP(S), POP3(S), and IMAP(S)), name servers (DNS), and SSH servers. A server listens on a known TCP or UDP port number awaiting incoming client connection requests. The client initiates the connection request to the listening server, and the server responds to it.

Without a firewall, a client can communicate with any server without restrictions; moreover, a client can function as a server and listen for incoming connections from other clients. In other words, if an attacker manages to exploit a vulnerability on a system without a firewall in place, the attacker could use the exploit to listen on a chosen port number on the victim’s machine and connect to it without any restrictions.

Setting up a firewall offers many security benefits. First and foremost, firewall rules provide fine control over which packets can leave your system and which packets can enter your system. Consequently, firewall rules help mitigate various security risks by controlling network traffic between devices. More importantly, firewall rules can be devised to ensure that no client can act as a server. In other words, an attacker cannot start a reachable listening port on a lab machine; the exploit can start a listening port, but the firewall will prevent all incoming connection attempts.

A host-based firewall is a piece of software installed on a system we want to protect. Unlike a network-based firewall, the host-based firewall restricts network packets to and from a single host. The firewall has two main functions:

- What can enter? Allow or deny packets from entering a system.
- What can leave? Allow or deny packets from leaving a system.

Imposing rules on the packets entering and leaving a system will significantly improve our security posture.

### Linux Firewalls

The first Linux firewall was a packet filtering firewall, i.e., a stateless firewall. A stateless firewall can inspect certain fields in the IP and TCP/UDP headers to decide upon a packet but does not maintain information about ongoing TCP connections. As a result, a packet can manipulate a few TCP flags to appear as if it is part of an ongoing connection and evade certain restrictions. Current Linux firewalls are stateful firewalls; they keep track of ongoing connections and restrict packets based on specific fields in the IP and TCP/UDP headers and based on whether the packet is part of an ongoing connection.

The IP header fields that find their way into the firewall rules are:

1. Source IP address
2. Destination IP address

The TCP/UDP header fields that are of primary concern for firewall rules are:

1. Source TCP/UDP port
2. Destination TCP/UDP port

It is worth noting that it is impossible to allow and deny packets based on the process but instead on the port number. If you want the web browser to access the web, you must allow the respective ports, such as ports 80 and 443. This limitation differs from MS Windows’ built-in firewall, which can restrict and allow traffic per application.

On a Linux system, a solution such as [SELinux](https://github.com/SELinuxProject) or [AppArmor](https://www.apparmor.net/) can be used for more granular control over processes and their network access. For example, we can allow only the `/usr/bin/apache2` binary to use ports 80 and 443 while preventing any other binary from doing so on the underlying system. Both tools enforce access control policies based on the specific process or binary, providing a more comprehensive way to secure a Linux system.

### Netfilter

At the very core, we have netfilter. The netfilter project provides packet-filtering software for the Linux kernel 2.4.x and later versions. The netfilter hooks require a front-end such as `iptables` or `nftables` to manage.

In the following examples, we use different front-ends to netfilter in order to allow incoming SSH connections to the SSH server on our Linux system. As shown in the figure below, we want our SSH server to be accessible to anyone on the Internet with an SSH client.

### iptables

As a front-end, iptables provides the user-space command line tools to configure the packet filtering rule set using the netfilter hooks. For filtering the traffic, iptables has the following default chains:

- **Input**: This chain applies to the packets incoming to the firewall.
- **Output**: This chain applies to the packets outgoing from the firewall.
- **Forward** This chain applies to the packets routed through the system.

Let’s say that we want to be able to access the SSH server on our system remotely. For the SSH server to be able to communicate with the world, we need two things:

1. Accept incoming packets to TCP port 22.
2. Accept outgoing packets from TCP port 22.

Let’s translate the above two requirements into `iptables` commands:

`iptables -A INPUT -p tcp --dport 22 -j ACCEPT`

- `-A INPUT` appends to the INPUT chain, i.e., packets destined for the system.
- `-p tcp --dport 22` applies to TCP protocol with destination port 22.
- `-j ACCEPT` specifies (jump to) target rule ACCEPT.

`iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT`

- `-A OUTPUT` append to the OUTPUT chain, i.e., packets leaving the system.
- `-p tcp --sport 22` applies to TCP protocol with source port 22.

Let’s say you only want to allow traffic to the local SSH server and block everything else. In this case, you need to add two more rules to set the default behaviour of your firewall:

- `iptables -A INPUT -j DROP` to block all incoming traffic not allowed in previous rules.
- `iptables -A OUTPUT -j DROP` to block all outgoing traffic not allowed in previous rules.

In brief, the rules below need to be applied in the following order:

```shell-session
root@AttackBox# iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
iptables -A INPUT -j DROP
iptables -A OUTPUT -j DROP
```

In practice, you should flush (delete) previous rules before applying new ones. This action can be achieved using `iptables -F`.

Example front-ends to iptables can be divided into:
	- Command-line Interface (CLI) front-ends, such as firewalld and ufw
	- Graphical User Interface (GUI) front-ends, such as fwbuilder
### nftables

nftables is supported in Kernel 3.13 and later, adding various improvements over iptables, particularly in scalability and performance.

Unlike iptables, nftables start with no tables or chains. We need to add the necessary tables and chains before adding rules. To begin, we will create a table, `fwfilter`.

`nft add table fwfilter`

- `add` is used to add a table. Other commands include `delete` to delete a table, `list` to list the chains and rules in a table, and `flush` to clear all chains and rules from a table.
- `table TABLE_NAME` is used to specify the name of the table we want to create or work on.

In our newly created table, `fwfilter`, we will add an _input_ chain and an _output_ chain for incoming and outgoing packets, respectively.

- `nft add chain fwfilter fwinput { type filter hook input priority 0 \; }`
- `nft add chain fwfilter fwoutput { type filter hook output priority 0 \; }`

The above two commands add two chains to the table `fwfilter`:

- `fwinput` is the input chain. It is of type `filter` and applies to the input hook.
- `fwoutput` is the output chain. It is of type `filter` and applies to the output hook.

With the two chains created within our table, we can add the necessary rule to allow SSH traffic. The following two rules are added to the table `fwfilter` to the chains `fwinput` and `fwoutput`, respectively:

- `nft add fwfilter fwinput tcp dport 22 accept` accepts TCP traffic to the local system’s destination port 22.
- `nft add fwfilter fwoutput tcp sport 22 accept` accepts TCP traffic from the local system’s source port 22.

We can check the shape of the `fwfilter` table using the command `nft list table fwfilter`:

```shell-session
root@AttackBox# sudo nft list table fwfilter
table ip fwfilter {
    chain fwinput {
        type filter hook input priority filter;
        tcp dport 22 accept
    }

    chain fwoutput {
        type filter hook output priority filter;
        tcp sport 22 accept
    }
}
```

### UFW

UFW stands for uncomplicated firewall. Let’s see how it stands for its promise of being _uncomplicated_. We will allow SSH traffic. This firewall rule can be achieved through one of the following commands:

`ufw allow 22/tcp`

It configures the firewall to `allow` traffic to TCP port 22. We can confirm our settings with the command `ufw status`:

```shell-session
root@AttackBox# sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
```

### Firewall Policy

Before configuring a firewall, you need to decide upon the firewall policy. You might be the decision maker regarding the firewall policy or an enforcer of an existing security policy that covers firewall configuration. It all depends on the system you are protecting.

We will not go into security policies as this is outside the scope of this room. We will mention that the two main approaches are:

- Block everything and allow certain exceptions.
- Allow everything and block certain exceptions.

Each of the above two approaches has its advantages and disadvantages. Blocking everything with a limited set of exceptions would provide tighter and better security; however, it might cause inconvenience to the users depending on the situation.

Let’s consider the following example. You are responsible for configuring the (host) firewall installed on the university computers. In this example, the academic institution has decided to block all outgoing and incoming traffic except for DNS, HTTP, and HTTPS traffic. In firewall terms, that’s allowing UDP port 53 and TCP ports 80 and 443. This policy should allow browsing the Internet over HTTP and HTTPS; however, if one of the websites uses a non-standard HTTP or HTTPS port, it will be blocked. Dealing with these exceptions will create a challenge; keeping the firewall rules organised and properly documented is tricky as the number of exceptions grows over time.

### Q&A

There is a firewall running on the Linux VM. It is allowing port 22 TCP as we can ssh into the machine. It is allowing another TCP port; what is it?

`12526`

What is the allowed UDP port?

`14298`

## Remote Access

Providing remote access to a system is a very convenient way to access your system and files when you are not physically present at the target system’s keyboard. However, this also means that you are voluntarily providing a service that attackers will target. Common attacks include:

1. Password sniffing
2. Password guessing and brute-forcing
3. Exploiting the listening service

### Protecting Against Password Sniffing

Remote access can be achieved through many different protocols and services. Although all modern systems use encrypted protocols, such as the SSH protocol, for remote access, older systems might still use cleartext protocols, such as the Telnet protocol.
