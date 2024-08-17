# Ansible-scripts

## Ubuntu Linux Installation

### Configuring Linux Inventory

Create an inventory file on disk i.e. linventory.txt and paste the followig into it. Ensure your credentials are correct.

    [linux]
    localhost

    [linux:vars]
    ansible_user=$YOUR_NAME
    ansible_ssh_pass=$YOUR_PASSWORD
    ansible_connection=ssh

### Install Packages

    sudo apt update
    sudo apt upgrade -y
    sudo apt install ansible -y
    sudo apt install sshpass -y

Install open SSH server i.e. <https://ubuntu.com/server/docs/openssh-server>

    sudo apt install openssh-client
    sudo apt install openssh-server

### Login via localhost

    ssh $YOUR_NAME@localhost

### Run the script

Run the script on your control node to start the process:

    ansible-playbook -i linventory.txt linux_playbook.yml --become --become-method=sudo --become-user=root --ask-become-pass

## Windows 11 Installation

### Configure Windows inventory

Create an inventory file on disk i.e. winventory.txt and paste the followig into it. Ensure your credentials are correct.

    [windows]
    YOUR_HOST_NAME_OR_IP

    [windows:vars]
    ansible_user=$YOUR_NAME
    ansible_ssh_pass=$YOUR_PASSWORD
    ansible_connection=winrm
    ansible_winrm_server_cert_validation=ignore
    ansible_winrm_transport=credssp

### Installing Certificate

#### Generate or Obtain a Valid SSL Certificate

You can use a self-signed certificate or obtain one from a trusted Certificate Authority (CA). For a self-signed certificate, you can use PowerShell:

    $cert = New-SelfSignedCertificate -DnsName "YOUR_HOST_NAME_OR_IP" -CertStoreLocation "Cert:\LocalMachine\My"

#### Verify the Certificate

Ensure the certificate is installed in the local machine’s certificate store and meets the following criteria:
The CN (Common Name) matches the hostname or ip.
The certificate is valid (not expired or revoked).
The certificate is appropriate for Server Authentication.

#### Install the Certificate

Open the Microsoft Management Console (MMC):

    Win + R -> mmc.exe

and add the Certificates snap-in for the local computer.

Verify the certificate is under **Certificates (Local Computer) > Personal > Certificates**.

### Configure WinRM to Use the Certificate

#### Use the following PowerShell commands to configure WinRM to use the SSL certificate

Get the certificate thumbprint:

    $thumbprint = (Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object { $_.Subject -match "CN=YOUR_HOST_NAME_OR_IP" }).Thumbprint

#### Open the Firewall for WinRM HTTPS Traffic

Ensure the firewall allows traffic on the WinRM HTTPS port (default is 5986):

    New-NetFirewallRule -Name "WinRM HTTPS" -DisplayName "WinRM HTTPS" -Protocol TCP -LocalPort 5986 -Action Allow

### Installing HTTPS Listeners

#### Modify Group Policy

You need to change the Group Policy setting that controls the auto-configuration of WinRM listeners.

##### Open Group Policy Editor

Press Win + R, type gpedit.msc, and press Enter.

##### Navigate to the Policy

Go to **Computer Configuration -> Administrative Templates -> Windows Components -> Windows Remote Management (WinRM) -> WinRM Service**.

##### Modify the Policy

Find the policy named **Allow remote server management through WinRM** (this might be the renamed version of **Allow Auto Configuration of listeners on WinRm service**).
Set this policy to **Not Configured**.

##### Apply and Close

Apply the changes and close the Group Policy Editor.

After modifying the Group Policy, you should be able to remove the listener.

##### Open PowerShell as Administrator

Right-click on the Start menu and select **Windows PowerShell (Admin)**.

#### Install the Listener

Use the following command to remove the listener:

    winrm delete winrm/config/Listener?Address=*+Transport=HTTPS

After removing the existing listener, recreate it with the correct SSL certificate:

    $thumbprint = (Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object { $_.Subject -match "CN=your_hostname" }).Thumbprint

Then create a new WinRM listener:

    winrm create winrm/config/Listener?Address=*+Transport=HTTPS @{Hostname="YOUR_HOST_NAME_OR_IP"; CertificateThumbprint="$thumbprint"}

Confirm that the new listener has been created successfully:

    winrm enumerate winrm/config/listener

#### Check Firewall Rules

Ensure that the firewall allows traffic on the WinRM HTTPS port (default is 5986)

    New-NetFirewallRule -Name "WinRM HTTPS" -DisplayName "WinRM HTTPS" -Protocol TCP -LocalPort 5986 -Action Allow

#### Start the WinRM Service

Start the WinRM service to apply the changes:

    Start-Service WinRM

### Install CredSSP

#### Ensure requests-credssp is installed on a control node

    pip install requests-credssp

#### Ensure that the package is installed correctly by listing the installed packages

    pip list | grep requests-credssp    

#### Ensure CredSSP is Enabled on the Server

You need to enable CredSSP on the server. Run the following PowerShell command on the server:

    Enable-WSManCredSSP -Role Server

#### Enable CredSSP on the Client

On the client machine, run the following PowerShell command to enable CredSSP:

    Enable-WSManCredSSP -Role Client -DelegateComputer "server_hostname"

#### Check Group Policy Settings

Ensure that the Group Policy settings allow CredSSP. On both the client and server, open the Group Policy Editor (gpedit.msc) and navigate to:
**Computer Configuration -> Administrative Templates -> System -> Credentials Delegation**

Enable the policy **Allow delegating fresh credentials with NTLM-only server authentication** and add the server’s hostname to the list of allowed servers.

#### Update the Encryption Oracle Remediation Policy

On both the client and server, update the Encryption Oracle Remediation policy to allow CredSSP. In the Group Policy Editor, navigate to:
**Computer Configuration -> Administrative Templates -> System -> Credentials Delegation -> Encryption Oracle Remediation**

Set the policy to **Enabled** and the Protection Level to **Vulnerable**. This is a temporary measure to ensure connectivity. After confirming the connection, you should set it back to **Mitigated** or **Force Updated Clients** for better security.

#### Verify Windows Updates

Ensure that both the client and server have the latest Windows updates installed. CredSSP issues can often be resolved by applying the latest security patches123.

#### Restart the WinRM Service

Restart the WinRM service on both the client and server to apply the changes:

    Restart-Service WinRM

### Creating vault password

Create or edit your vault.yml file:

    ansible-vault edit vault.yml

Add the ansible_become_pass variable with your password:

    ansible_become_pass: "YOUR_BECOME_PASSWORD"

Save and close the file. Ansible Vault will encrypt the file using the password you provided.

### Runing the script

Run the script on your control node to start the process:

    ansible-playbook -i winventory.txt windows_playbook.yml --become-method=runas --become-user=Administrator --ask-become-pass
