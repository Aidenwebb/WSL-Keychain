# WSL-Keychain
Work around WSL not saving SSH passphrase using Windows Credential Manager and KeyChain

## Set Up Windows side

### Install Credential Manager

In an Administrative Powershell session under Windows

```powershell
Install-Module -Name CredentialManager -AllowClobber -Force -Verbose -Scope AllUsers
```

### Set SSH Passphrase in Credential Manager

In Powershell running as your user account under Windows

```powershell
$sshKeyphrase = ConvertTo-SecureString -AsPlainText -Force -String "my-secret-key"
New-StoredCredential -Target "ssh-passphrase-id_ed25519" -SecurePassword $sshKeyphrase -Persist LocalMachine
```

### Create Windows-side script

We will run this script with Task manager on Logon to initialise Keychain in WSL.

```powershell
New-Item -Path $env:USERPROFILE -ItemType Directory -Name ".config\wsl-keychain"
notepad .\.config\wsl-keychain\invoke-wsl-keychain.ps1
```

Paste in the following and update variables to reflect your own configuration

```powershell
# Change these three variables to suit use case
$wslUsername = "username"
$wslDistribution = "Ubuntu"
$sshPassphraseTarget = "ssh-passphrase-id_ed25519" 

$credentials = Get-StoredCredential -Target $sshPassphraseTarget
$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($credentials.Password)
$passphrase = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)
C:\Windows\System32\wsl.exe -u $wslUsername -d $wslDistribution /home/$wslUsername/.config/wsl-keychain/keychain.sh `'$passphrase`'
```

### Set up Task Manager to run the script

From an Administrative command prompt:

```powershell
$stAction = New-ScheduledTaskAction -Execute 'Powershell.exe' -Argument '-WindowStyle Hidden -file C:\Users\Aiden\.config\wsl-keychain\invoke-wsl-keychain.ps1'

$stTrigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME

Register-ScheduledTask -Action $stAction -Trigger $stTrigger -TaskName "WSL-Keychain" -Description "Pass SSH Passphrase from Credential Manager in to WSL Keychain"
```

## Set up Linux side

### Install Keychain in WSL

Open up a WSL terminal 

```bash
sudo apt update && sudo apt install keychain
# Wait for installation to complete
```

### Load Keychain on startup

Add the following to `~/.bash_profile` or `~/.profile` depending on distribution

```bash
# Auto start keychain
eval $(/usr/bin/keychain --eval --quiet id_ed25519)
```

### Create WSL side Keychain script

```bash
mkdir -p ~/.config/wsl-keychain
touch ~/.config/wsl-keychain/keychain.sh
chmod +x ~/.config/wsl-keychain/keychain.sh
nano ~/.config/wsl-keychain/keychain.sh
```

Copy in the following and adjust to suit your needs

```bash
#!/bin/bash
SSH_ASKPASS_SCRIPT=/tmp/ssh-askpass-script
cat > ${SSH_ASKPASS_SCRIPT} <<EOL
#!/bin/bash
echo "$1"
EOL
chmod u+x ${SSH_ASKPASS_SCRIPT}
export DISPLAY="0"
export SSH_ASKPASS=${SSH_ASKPASS_SCRIPT}
/usr/bin/keychain --clear id_ed25519
rm ${SSH_ASKPASS_SCRIPT}
```

