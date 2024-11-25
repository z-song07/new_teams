# Teams Update

Some computers in the organization display the following when opening Teams, and employees cannot access Teams channels unless Teams are updated:
![Teams Needs Update](image.png)

the following are steps to upgrade to new Teams:
## 1. Remove teams machine-wide installer using  <u>scripts and remediations</u> under Windows devices in Intune

Removing machine-wide installer will automatically remove all old Teams versions.

First, detect machine-wide installer.

Detect Script:
```powershell
# Detect Teams Machine-Wide Installer

try{
    if (Get-Package -Name "Teams Machine-Wide Installer")
    {
        Write-Host "FOUND: Teams Machine-Wide Installer found!"
        exit 0
    }
    else
    {
        Write-Host "Teams Machine-Wide Installer not found."    
        exit 1

    }
}
catch{
    Write-Error "Error detecting the Teams Machine-Wide Installer."
    exit 1
}
```
Next, uninstall machine-wide installer

Remediation Script:
```powershell
# Remediation script
# Uninstall Teams machine-wide installer
try{
    Get-Package -Name "Teams Machine-Wide Installer" | Uninstall-Package
    Write-Host "Teams Machine-Wide Installer successfully uninstalled."
    exit 0
}
catch{
    Write-Error "Error uninstalling Teams Machine-Wide Installer: " + $_.Exception.Message
    exit 1
}
```

## 2. Deploy New Teams via intune

The Teams deployment app will be a Windows App (Win32).
The app will have three scripts to detect, install, and uninstall new Teams.

### 1. Detect new teams

Detect Script:
```powershell
$newTeams = Get-AppxPackage MSTeams
if($newTeams) 
{
    Write-Host "DETECTED: New Teams is installed"
    exit 0
}else{
    Write-Host "NO: New Teams is not found"
    exit 1
}
```
### 2. Install new teams

The following files are required to install new teams:

1. teamsbootstrapper.exe

    A lightweight online installer that allows administrators to install the Teams application for all users on a given target machine.

2. MSTeams-x64.msix

    Local Teams MSIX to install new Teams offline, minimize bandwidth.

3. IntuneWinAppUtil.exe

    Use the Microsoft Win32 Content Preparation Tool to convert the application installation files to the .intunewin format, so that it can be added as Windows app (Win32) to Intune
        https://learn.microsoft.com/en-us/mem/intune/apps/apps-win32-app-management

    
4. install script
```powershell
# PSScriptRoot is to get the relative path
$msix = "$($PSScriptRoot)\MSTeams-x64.msix"
$destination_path = "C:\ProgramData\Microsoft\NEW-TEAMS-TEMP"
$exe_path = "$($PSScriptRoot)\teamsbootstrapper.exe"

# create destination directory if not exist
if(!(Test-Path $destination_path)){
    mkdir $destination_path
}

# put the msix file into the destionation directory
Copy-Item -Path $msix -Destination $destination_path -Force

# execute the teamsbootstrapper
# -p is for execution
# -o is to output to the destination MSTEams msix
# -Wait will wait for the process to exit
# -WindowsStyle Hidden is to hide the powershell window from users 
try{
    Start-Process -FilePath $exe_path -ArgumentList "-p", "-o", "$($destination_path)\MSTeams-64.msix" -Wait -WindowStyle Hidden

    Write-Host: "SUCCESS"
    exit 0
}
catch{
    Write-Host: "Failed"
    exit 1
}
```

### 3. uninstall new teams
uninstall script:
```powershell
"$($PSScriptRoot)\teamsbootstrapper.exe -x"
```

### 4. Set up the Win32 App
1. Put all the files into a folder called "New_Teams"

2. Run the IntuneWinAppUtil.exe to convert the installation script into .intunewin format.
    
    -c <setup_folder>
    
    -s <setup_file>	
    
    -o <output_folder>
    
    -q	Quiet mode.

In the command prompt, go to the New_Teams path, run the following command

```shell
IntuneWinAppUtil -c .\ -s install.ps1 -o .\ -q
```

### 5. Upload the Teams Installation Win32 App to Intune

1. Go to Microsoft Intune admin center
2. Select Apps > Windows Apps > Add
3. Select Windows app (Win32) under select app type
4. Select app package file, Add packge file, choose the .intunewin file
5. Set up app information

## References:

MicrosoftHeidi. (2024, June 19). Bulk deploy the new Microsoft Teams desktop client - Microsoft Teams. Microsoft Learn. https://learn.microsoft.com/en-us/microsoftteams/new-teams-bulk-install-client

Erikre. (2024, September 11). Add and assign Win32 apps to Microsoft Intune. Microsoft Learn. https://learn.microsoft.com/en-us/mem/intune/apps/apps-win32-add


