**Educational Booklet**

# Joining Linux to Active Directory Domain (AD) - Step-by-Step Guide

## Audience

IT administrators and system engineers managing Ubuntu and Rocky Linux machines in a Windows Active Directory environment.

---

## Part 1: Active Directory Preparation

### 1.1 Create Organizational Units (Optional)

For better organization, create an OU in Active Directory for Linux clients:

- Open **Active Directory Users and Computers (ADUC)**.
- Right-click the domain (e.g., `example.com`) > **New > Organizational Unit**.
- Name it: `LinuxClients`.

### 1.2 Create a Group for Sudo Access

We will use a group to control sudo access on Linux clients.

- Open **ADUC**.
- Navigate to your desired OU or container.
- Right-click > **New > Group**.
- Group Name: `LinuxAdmins`
- Group Scope: Global
- Group Type: Security
- Click **OK**

### 1.3 Add Users to the Group

- Double-click the `LinuxAdmins` group.
- Go to **Members** tab > click **Add** > type the usernames > click **Check Names** > OK.

---

## Part 2: Preparing Linux Clients

### 2.1 Install Required Packages

#### Ubuntu

```bash
sudo apt update
sudo apt install realmd sssd adcli samba-common krb5-user packagekit -y
```

#### Rocky Linux

```bash
sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools -y
```

### 2.2 Set Hostname and DNS

Make sure hostname is set properly and DNS points to AD server:

```bash
hostnamectl set-hostname linux01.example.com
```

Edit `/etc/resolv.conf` (or configure `NetworkManager`) to use AD DNS:

```
nameserver 192.168.1.10  # IP of AD DNS server
```

### 2.3 Discover the Domain

```bash
realm discover example.com
```

You should see details of the domain.

### 2.4 Join the Domain

```bash
sudo realm join -U administrator example.com
```

- Enter the AD password when prompted.

### 2.5 Verify Domain Join

```bash
realm list
```

Check that the domain appears and `configured = yes`.

---

## Part 3: Configure SSSD and Sudo Access

### 3.1 Allow Only Certain Groups

To allow only `LinuxAdmins`:

```bash
sudo realm permit -g LinuxAdmins@example.com
```

### 3.2 Configure Sudo Access via Group Mapping

Edit the sudoers configuration:

```bash
sudo visudo
```

Add the following line:

```
%LinuxAdmins@example.com ALL=(ALL) ALL
```

> Note: Use `@example.com` for domain-qualified groups.

---

## Part 4: Testing and Verification

### 4.1 Test Login

```bash
su - 'username@example.com'
```

You should log in and have a home directory created.

### 4.2 Test Sudo Privileges

Run a sudo command:

```bash
sudo whoami
```

If the user is in the `LinuxAdmins` group, it should return `root`.

---

## Troubleshooting

- Check DNS and time sync.
- Use `realm leave` to rejoin if needed.
- Check `/var/log/sssd/sssd.log` and `/var/log/messages`.

---

## Conclusion

This setup allows central management of Linux user access and privileges using standard Active Directory tools. It streamlines administration in hybrid environments with both Windows and Linux systems.

