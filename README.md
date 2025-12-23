# Hybrid-Enterprise-identity-Automation-Lab
A cross-platform infrastructure lab integrating Red Hat Enterprise Linux 9 with Windows Server 2019 Active Directory. Features centralized identity management, RBAC, and automated "Patch Tuesday" updates via Ansible.

## Technologies Used
- **Windows Server 2019**: Domain Controller (DC) setup with Active Directory.
- **RHEL 9**: Linux server joined to AD for hybrid identity.
- **Ansible**: Cross-platform automation for tasks like patching, user management, and reboots.
- **PowerShell**: Enabling RDP, WinRM for remote management, and AD configurations.
- **Other Tools**: SSSD for Linux-AD integration, Kerberos for authentication, WinRM for Ansible-Windows communication.

## Project Phases

### Phase 1: Build the Windows Domain Controller
- Deploy a Windows Server 2019 VM with static IP (e.g., 192.XXX.X.XX).
<img width="1398" height="1004" alt="Windows_IP" src="https://github.com/user-attachments/assets/e36a1214-8a60-408e-a8bc-657e618db6aa" />

- Enable RDP using PowerShell commands for secure remote access:
  ```powershell
  Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
  Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 1
  Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
  
#### Step2: Install Domain Controller (AD DS) and DNS Server
**Install AD DS:**
  - Open **Server Manager:** **Add Roles and Features**.
  - Select **Active Directory Domain Services** (AD DS).
  - Click Next/Install.
<img width="1070" height="768" alt="AD_DS" src="https://github.com/user-attachments/assets/9ce667be-0692-4f9d-9db4-a5cc537506b7" />

**Promote:**

<img width="732" height="537" alt="Promote_DC" src="https://github.com/user-attachments/assets/992cfd1c-0df0-45d8-a014-1372a5a100c3" />

- Add new forest with Domain name: GOVLAB.LOCAL
- Make sure you enable DNS
- Set your DSRM password (remember this!).
- Click Next/Install and let it reboot.
#### Step 3: Create GovTech-Style Users (RBAC)
<img width="1160" height="733" alt="linuxadmins_group" src="https://github.com/user-attachments/assets/c6d393fd-cf06-4c79-b36f-9f854bd3b172" />

- Open Active Directory Users and Computers (ADUC).
- Right-click domain > New > Organizational Unit (OU) named IT_Admin.
- Create admin user inside:
  - `linux_admin` (admin account).
- Create a Group named linuxadmins and add linux_admin to it.
#### Step 4: Harden with GPO
<img width="925" height="562" alt="GPM" src="https://github.com/user-attachments/assets/3766dd07-6490-4392-8bf4-9b6efd5fbd17" />

- Open **Group Policy Management**.
- Right-click `Default Domain Policy` > Edit.
- Navigate to: `Computer Config -> Policies -> Windows Settings -> Security Settings -> Account Policies -> Password Policy`.
- Set **Minimum password length** to `14 characters` (STIG Requirement).

### Phase 2: Join RHEL to Active Directory (Hybrid Identity Bridge)
#### Step 1: Configure DNS on RHEL system to point to Windows Domain Controler

<img width="715" height="246" alt="govlab_ping" src="https://github.com/user-attachments/assets/36c492f9-b043-419f-a296-c67ecc485052" />

- On your RHEL VM (ansible5), check your DNS:
    `cat /etc/resolv.conf`
- Action: It MUST point to your Windows DC (192.XXX.X.XX) to find the domain.
  - `nmcli con mod <interface_name (ex: enp6s18)> ipv4.dns "192.XXX.X.XX"`
  - `nmcli con up <interface_name (ex: enp6s18)>`
- **Test:** `ping govlab.local` (If this fails, stop and fix DNS.)
#### Step 2: Install AD Dependencies
`sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools -y`
#### Step 3: Join the Domain
- Discover the domain to ensure it’s reachable:
    - `realm discover GOVLAB.LOCAL`      
- Join the domain (use the Administrator account):
    - `realm join GOVLAB.LOCAL -U Administrator`
- List all recovered and configured realms:
    - `realm list`
- *Verification:* Run `id linux_admin@govlab.local`. You should see the user info from Windows!
<img width="1110" height="151" alt="ID_administration" src="https://github.com/user-attachments/assets/306fe115-af16-4737-930c-d487f6e09134" />

#### Step 4: Check user password
- `getent passwd linux_admin@govlab.local`
<img width="948" height="82" alt="getent_passwd" src="https://github.com/user-attachments/assets/3912e847-9b91-470b-8af9-4c8fea675677" />

#### Step 5: Configure Access Control (Sudoers) and ssh into users account
<img width="1111" height="529" alt="linuxadmins_group_sudoers" src="https://github.com/user-attachments/assets/8d0b29c2-03c4-4b67-88da-e17ab21e2edc" />

- SSH into `linux_admin@govlab.local`
  - `ssh linux_admin@govlab.local@localhost`
- Don't let everyone in. Only let the LinuxAdmins group in.
- Edit sssd.conf:
  - `vim /etc/sssd/sssd.conf`
  - Add: `simple_allow_groups = linuxadmins`
- Grant Sudo rights:
  - `vim /etc/sudoers.d/linuxadmins`
  - `%LinuxAdmins@govlab.local ALL=(ALL) ALL`
 
### **Phase 3: Cross-Platform Ansible Automation**
**Step 1: Configure Windows for Ansible**

<img width="1137" height="91" alt="powershell_ansible_connection" src="https://github.com/user-attachments/assets/04c84ed7-12c4-4413-a996-5a16ae21bc97" />

- On the **Windows DC**, open PowerShell as Administrator.
- Run this script to enable WinRM (allows Ansible to connect with Windows Server):
    ```powershell
    $url = "https://raw.githubusercontent.com/ansible/ansible-documentation/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
    $file = "$env:TEMP\ConfigureRemotingForAnsible.ps1"
    (New-Object System.Net.WebClient).DownloadFile($url, $file)
    Set-ExecutionPolicy Bypass -Scope Process -Force
    powershell.exe -ExecutionPolicy Bypass -File $file -Verbos
    ```

**Step 2: Configure Linux Control Node**
- On your Ansible Control Node:
  - `pip install pywinrm`
- Download the Windows modules from Ansible Galaxy:
  - `ansible-galaxy collection install ansible.windows`
    
**Step 3: Update Inventory**
- Create a separate directory for your Ansible projects
  - `mkdir ansible_project`
  - `cd ansible_project` 
- Create a new hosts.ini (or add to your existing one):
  - `vim hosts.ini`  

    ```bash
    [windows]
    172.168.1.10
    
    [windows:vars]
    ansible_user=Administrator
    ansible_password=<YOUR_WINDOWS_PASSWORD>
    ansible_connection=winrm
    ansible_winrm_server_cert_validation=ignore
    ansible_winrm_transport=basic
    ansible_port=5986
    ```
  
- Test the connection by running the Ansible "ping" module specifically for Windows (`win_ping`) to verify everything is talking.
  
  <img width="716" height="96" alt="ping_ansible" src="https://github.com/user-attachments/assets/7274c795-7f96-4372-8b72-a42f427149f7" />
  
  - `ansible -i hosts.ini windows -m win_ping`
    - `-i` tells Ansible specifically which inventory file to use

**Step 4: The "Patch Tuesday" Playbook**
Create patch_tuesday.yml:

```yaml
---
# PLAY 1: Patch Windows Infrastructure
- name: Patch Windows Infrastructure
  hosts: windows
  gather_facts: yes
  tasks:
    - name: Install Security and Critical updates
      ansible.windows.win_updates:
        category_names:
          - SecurityUpdates
          - CriticalUpdates
        state: installed
        reboot: true
```

Run playbook: 
- `ansible-playbook patch_tuesday.yml -i hosts.ini`

Control node:

<img width="865" height="551" alt="ansible_updates" src="https://github.com/user-attachments/assets/3439b8ae-dee6-4e79-9a06-cad2db7d76f7" />

Windows Server:

<img width="984" height="729" alt="windows_updates" src="https://github.com/user-attachments/assets/89c4bf27-1a09-40f6-a636-2d34c1648026" />
