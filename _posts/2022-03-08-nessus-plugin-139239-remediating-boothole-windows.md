---
layout: post
title:  'Remediating Nessus Plugin ID 139239 "Windows Security Feature Bypass in Secure Boot (BootHole)"'
categories: 
excerpt: "How do you remediate BootHole as identified by Nessus Plugin ID 139239 in Windows systems? Here we will be discussing this vulnerability and how to properly remediate it from your Windows hosts."
image: 
permalink: 
---

There are a series of vulnerabilities that are detected on a Windows host by [Nessus Plugin ID 139239](https://www.tenable.com/plugins/nessus/139239) identified as BootHole. This post discusses that vulnerability and how to effectively remediate it against Windows hosts. Some guidance suggests installing [Windows Update KB4535680: Security update for Secure Boot DBX](https://support.microsoft.com/en-us/topic/kb4535680-security-update-for-secure-boot-dbx-january-12-2021-f08c6b00-a850-e595-6147-d0c32ead81e2), but that update is only available for Windows 10 up to version 1909.

If you, like many I would hope, run a currently supported version of Windows 10 there is no (that I could find at least) Windows Update that will apply the required security updates to take care of this BootHole vulnerability. Below we will be taking a look at what options are available to install the required patches into the systems firmware to take care of this vulnerability. 

# Remediating via DBX Updates
This vulnerability is remediated by installing some DBX updates into the UEFI firmware of the affected system, so that Secure Boot knows what boot loaders it should or should not allow to be installed. 

UEFI has published a series of updates to the DBX database that deny a Secure Boot system from installing any of the vulnerable GRUB bootloaders. The hashes of these bootloaders are added a revocation list that Secure Boot checks to make sure the bootloader is not revoked. Once these DBX Updates are installed into the firmware of a system the vulnerable GRUB bootloaders can no longer be installed, thus remediating this vulnerability.

## Windows Update (Win10 versions 20H1+)
If you happen to be running any of the below versions of Windows, then you can install KB4535680 which allegedly updates the Secure Boot with these new DBX updates. 

KB4535680 Applies To:
- Windows Server 2012 x64-bit
- Windows Server 2012 R2 x64-bit
- Windows 8.1 x64-bit
- Windows Server 2016 x64-bit
- Windows Server 2019 x64-bit
- Windows 10, version 1607 x64-bit
- Windows 10, version 1803 x64-bit
- Windows 10, version 1809 x64-bit
- Windows 10, version 1909 x64-bit 
> These were taken directly from [Windows Update KB4535680: Security update for Secure Boot DBX](https://support.microsoft.com/en-us/topic/kb4535680-security-update-for-secure-boot-dbx-january-12-2021-f08c6b00-a850-e595-6147-d0c32ead81e2)

Installing the above KB should remediate the vulnerability as found by Nessus. I have not been able to use this KB, however, since I only have newer versions of Windows 10 that were being identified by Nessus as vulnerable to BootHole. I am also not sure if this KB includes the absolute latest updates from the UEFI Forum. 

## Manual Windows DBX Updates
Microsoft published an article titled [Microsoft guidance for applying Secure Boot DBX update](https://support.microsoft.com/en-us/topic/microsoft-guidance-for-applying-secure-boot-dbx-update-e3b9e4cb-a330-b3ba-a602-15083965d9ca) that provides the manual steps required to install the DBX updates. The steps are summarized below.

### Obtain the DBX Updates
Download the relevant (x64, x86, ARM) updates directly from [Unified Extensible Firmware Interface Forum](https://uefi.org/revocationlistfile). You will also need the older updates, which can be found in the linked archive containing [Prior Versions of DBX files](https://uefi.org/revocationlistfile/archive) as these updates are _not_ cumulative. You will be downloading a single `.bin` binary file for each round of updates.

### Split the DBX files
To apply the updates we need to split the `.bin` files that were downloaded into two parts, a `.p7` signature file and another `.bin` binary file. This can be done via PowerShell. 

You will need to download the `SplitDbxContent.ps1` script which was authored by Microsoft. You can get it from [the PowerShell Gallery](https://aka.ms/DbxSplitScript). Follow the instructions on the Gallery to obtain that file.

Once you have the `SplitDbxContent.ps1` script, run the following PowerShell to create the signature and contents files.
```powershell
.\SplitDbxContent.ps1 <path to downloaded dbxupdate.bin>
```
> 
You must run this from `powershell.exe` built into Windows, if you try to run this from `pwsh.exe` for PowerShell version 7 this script will fail due to changes in the commands in later versions of PowerShell.

You should receive feedback showing that two new files were created, a `signature.p7` and a `content.bin`
![](/images/posts/2022-03-08-nessus-plugin-139239-remediating-boothole-windows/screenshot-splitdbxcontent.png)

> If you have multiple dbxupdates that will need to be done, save them each into a folder named after their update date, and run the `SplitDbxContent.ps1` from that folder. Otherwise the above script will overwrite the signature and content files each time you re-run it. 

### Install DBX Update via PowerShell
Now that you have one or more pairs of signature and content files, you can use PowerShell to install the updates.

> The DBX updates provided by UEFI Forum are _not_ cumulative. This means that you _must_ install each update individually in order to be fully protected from BootHole.

Launch an Administrative PowerShell and enter the following to apply the updates.
```powershell
Set-SecureBootUefi -Name dbx -ContentFilePath "content.bin" -SignedFilePath "signature.p7" -Time "2010-03-06T19:17:21Z" -AppendWrite
```

> Make sure  you use the `-Time "2010-03-06T19:17:21Z"` as-is! Do not change this value otherwise it will not apply. Use this same timestamp for each and every update file.

You should receive output showing the bytes as written.
![](/images/posts/2022-03-08-nessus-plugin-139239-remediating-boothole-windows/screenshot-setdbx.png)

> Microsoft advises to restart the PC before confirming the update applied, however in my experience you can check for the existance of these updates immediately after applying them. Though they may require a restart to actually take affect.

### Confirm Updates Applied
You can confirm the updates were properly applied by using [these PowerShell scripts](https://gist.github.com/out0xb2/f8e0bae94214889a89ac67fceb37f8c0). Download both the `Check-Dbx.ps1` and `Get-UEFIDatabaseSignatures.ps1` scripts. Run them against your `content.bin` files that were installed to confirm they exist in the UEFI firmware. Repeat for each update file that was applied.
```powershell
Get-UEFIDatabaseSignatures.ps1 "content.bin"
```
You should get feedback that the update was applied.
![](/images/posts/2022-03-08-nessus-plugin-139239-remediating-boothole-windows/screenshot-checkdbx.png)

## All-in-One PowerShell Update Script
I had a need to remediate BootHole on dozens of PC's, both local and remote, and since these are running a current version of Windows 10 the KB update was not an option. I followed Microsoft's guidance and created a PowerShell script that will apply the applicable updates to a PC remotely, and without interaction.

There are some other PowerShell scripts available on GitHub that perform this task, but one of my requirements was that the script be completely self-contained. On the systems I was pushing this update script to I didn't have a reliable way to grab the dbx update files from a file share. As such, I split them ahead of time and converted them into base64 to embed them directly into the script. The script then saves these files to disk temporarily right before applying them, then deletes them afterwards. 

If you wish to use this script in a similar fashion, with the update files embedded within, grab [the latest](https://uefi.org/revocationlistfile) and [the archived](https://uefi.org/revocationlistfile/archive) update files and follow the instructions above to split the `.bin` into the `signature.p7` and `content.bin` files. You can then follow the instructions below to Base64 encode these files and paste them into the script.

[Fix-BootHole.ps1 script on GitHub](https://gist.github.com/rufflabs/dccd1008f966e16d99b94a30a6bb1f9f)

### Update Script Variables
If you don't want to embed the update files directly into the script, you can optionally load them from a file share. To do so, follow the instructions previously in this article to download the dbx updates and split the `.bin` files into their respective `signature.p7` and `content.bin` files. 

Store these files on a file share accessible to the system you will be running this script on. Next, edit the script on line 19 to set `$EmbeddedFiles` to `$false` and update the file paths on lines 24 through 59 to the signature and content files. 
![](/images/posts/2022-03-08-nessus-plugin-139239-remediating-boothole-windows/screenshot-updatevariables.png)

> If you intend to embed the files, there is no need to modify the `$EmbeddedFiles` or `$DbxUpdateFiles` variables.

### Converting DBX Updates to Base64 Encoded Strings
If you want to embed the files into the script so that it is a self-contained script, follow the steps previously in this article to download the dbx updates and to split the `.bin` files into their respective `signature.p7` and `content.bin` files.

With your individual signature and content files, you need to convert them into Base64 and then copy/paste the Base64 data into the appropriate variables in the script. 

Run the following PowerShell code against each of the `signature.p7` and `content.bin` files to generate a rather large string of Base64 encoded text. Piping these to `clip` will copy the contents directly into your clipboard so that you can paste them.

```powershell
[System.Convert]::ToBase64String((Get-Content -Path ".\signature.p7" -Encoding byte)) | clip
[System.Convert]::ToBase64String((Get-Content -Path ".\content.p7" -Encoding byte)) | clip
```

Paste each of the signature and content Base64 code into their respective variables in the script. These are under the `$DbxUpdateFiles` variable. You only need to add in the architectures that you plan to deploy to, you can leave the others empty if you won't need them.

![](/images/posts/2022-03-08-nessus-plugin-139239-remediating-boothole-windows/screenshot-dbxupdatefiles.png)

### Running the Script
You can run `Fix-BootHole.ps1` after either setting the file share locations, or pasting in the embedded base64 versions of the files. 

```powershell
.\Fix-BootHole.ps1
```

### Fix-BootHole.ps1 Script
[For the latest script, please see the GitHub](https://gist.github.com/rufflabs/dccd1008f966e16d99b94a30a6bb1f9f)
```powershell
<#
.SYNOPSIS
Applies UEFI dbx updates to fix BootHole vulnerability.

.DESCRIPTION
Applies the UEFI dbxupdates to fix the BootHole vulnerability. Prior to running,
edit this script to choose between Embedded files or link to a file share.
See https://www.rufflabs.com/post/nessus-plugin-139239-remediating-boothole-windows/

.EXAMPLE
Run the script after making required modifications as mentioned in the descriptions.
PS C:\>.\Fix-BootHole.ps1

#>

# Set this to false if you want to load the .p7 and .bin files from a file share
# instead of having them embedded within this script as base64 encoded data.
#$EmbeddedFiles = $False
$EmbeddedFiles = $True

# If using a file share instead of embedding files, update these file paths to point
# to the .p7 and .bin files for each respective update. You can only specify those that
# will be used (such as, if not using arm, ignore the arm ones.)
$DbxUpdatePaths = @{
    "x64" = @{
        "April 2021" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\april2021_x64\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\april2021_x64\content.bin"
        }
        "October 2020" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\october2020_x64\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\october2020_x64\content.bin"
        }
        "July 2020" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\july2020_x64\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\july2020_x64\content.bin"
        }
    }
    "x86" = @{
        "April 2021" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\april2021_x86\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\april2021_x86\content.bin"
        }
        "July 2020" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\july2020_x86\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\july2020_x86\content.bin"
        }
    }
    "arm" = @{
        "April 2021" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\april2021_arm64\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\april2021_arm64\content.bin"
        }
        "July 2020" = @{
            "signature" = "\\sampleserver\sampleshare\dbxupdates\july2020_arm64\signature.p7"
            "content" = "\\sampleserver\sampleshare\dbxupdates\july2020_arm64\content.bin"
        }
    }
}

# Paste in Base64 encoded files below, both signature.p7 and content.bin
# for each update and architecture as needed. Any left blank will not be applied.
$DbxUpdateFiles =  @{
    "x64" = @{
        "April 2021" = @{
            "signature" = ""
            "content" = ""
        }
        "October 2020" = @{
            "signature" = ""
            "content" = ""
        }
        "July 2020" = @{
            "signature" = ""
            "content" = ""
        }
    }
    "x86" = @{
        "April 2021" = @{
            "signature" = ""
            "content" = ""
        }
        "July 2020" = @{
            "signature" = ""
            "content" = ""
        }
    }
    "arm" = @{
        "April 2021" = @{
            "signature" = ""
            "content" = ""
        }
        "July 2020" = @{
            "signature" = ""
            "content" = ""
        }
    }
}

# CA Check adapted from https://github.com/synackcyber/BootHole_Fix/blob/main/boothole.ps1
Write-Output "Checking for matching UEFI CA"
$CAMatch = [System.Text.Encoding]::ASCII.GetString((Get-SecureBootUEFI db).bytes) -match 'Microsoft Corporation UEFI CA 2011'
if(-not $CAMatch){
    Write-Output "Microsoft Corporation UEFI CA 2011 was not found. Updates are not required on this system."
    exit
}
Write-Output "Certificate found, continuing."

# Check architecture
$ArchitectureOptions = @{
    0 = "x86"
    1 = "MIPS"
    2 = "Alpha"
    3 = "PowerPC"
    5 = "ARM"
    6 = "IA64"
    9 = "x64"
}

$Architecture = $ArchitectureOptions[[int](Get-WmiObject -Class Win32_Processor).Architecture]
$SupportedArchitectures = @("x86", "x64", "arm")

if($SupportedArchitectures -notcontains $Architecture) {
    Write-Output "This script does not support $($Architecture) based PC's, and no dbx updates are available for this architecture."
    exit
}

function Update-DbxVariable {
    param(
        $SignaturePath,
        $ContentPath
    )
    Set-SecureBootUefi -Name dbx -SignedFilePath $SignaturePath -ContentFilePath $ContentPath -Time "2010-03-06T19:17:21Z" -AppendWrite
}

Write-Output "Updating UEFI dbx variable for $($Architecture) architecture."

$UpdateFiles = @{}

if($EmbeddedFiles) {
    # Use the embedded files in this script

    Write-Output "Using embedded update files, creating files on disk."
    try {
    
        $DbxUpdateFiles.$Architecture.GetEnumerator() | ForEach-Object {
            $Release = $_.Name
            $Content = $_.Value.Content
            $Signature = $_.Value.Signature
            $TempSigFile = "$($env:temp)\$($Release)_$($Architecture)_signature.p7"
            $TempContentFile = "$($env:temp)\$($Release)_$($Architecture)_content.bin"

            # Only copy data if signature and content are not null. 
            if($null -ne $Signature -and $null -ne $Content) {
                $UpdateFiles.Add($Release, @{"Signature" = $TempSigFile; "Content" = $TempContentFile})
                
                Set-Content -Path $TempSigFile -Value ([System.Convert]::FromBase64String($Signature)) -Encoding Byte -ErrorAction Stop
                Write-Output "Created $($TempSigFile)"
                
                Set-Content -Path $TempContentFile -Value ([System.Convert]::FromBase64String($Content)) -Encoding Byte -ErrorAction Stop
                Write-Output "Created $($TempContentFile)"
            }
        }
    } catch {
        Write-Output "Error creating temporary files. Check permissions to $($env:temp)."
        exit
    }
}else{
    # Use the files from a file share location

    Write-Output "Using update files from file share."
    
    $DbxUpdatePaths.$Architecture.GetEnumerator() | ForEach-Object {
        $Release = $_.Name
        $ContentPath = $_.Value.Content
        $SignaturePath = $_.Value.Signature
        
        # Only add to the queue if both signature and content files are accessible
        if((Test-Path -Path $ContentPath) -and (Test-Path -Path $SignaturePath)) {
            Write-Output "`nPreparing: $($Release)`nSignature: $($SignaturePath)`nContent: $($ContentPath)"
            $UpdateFiles.Add($Release, @{"Signature" = $SignaturePath; "Content" = $ContentPath})
        }
    }
}

$UpdateFiles.GetEnumerator() | ForEach-Object {
    Write-Output "Applying $($_.Name) update..."
    Update-DbxVariable -SignaturePath $_.Value.Signature -ContentPath $_.Value.Content
}   

if($EmbeddedFiles) {
    $UpdateFiles.Values.Values | ForEach-Object {
        # Delete file from disk
        Write-Output "Deleting on disk $($_)"
        Remove-Item -Path $_ -ErrorAction SilentlyContinue
    }
}
```

# References
- [Windows Security Feature Bypass in Secure Boot (BootHole) - Tenable®](https://www.tenable.com/plugins/nessus/139239)
- [ADV200011 - Security Update Guide - Microsoft - Microsoft Guidance for Addressing Security Feature Bypass in GRUB](https://msrc.microsoft.com/update-guide/en-US/vulnerability/ADV200011)
- [KB4535680: Security update for Secure Boot DBX: January 12, 2021](https://support.microsoft.com/en-us/topic/kb4535680-security-update-for-secure-boot-dbx-january-12-2021-f08c6b00-a850-e595-6147-d0c32ead81e2)
- [Microsoft guidance for applying Secure Boot DBX update](https://support.microsoft.com/en-us/topic/microsoft-guidance-for-applying-secure-boot-dbx-update-e3b9e4cb-a330-b3ba-a602-15083965d9ca)
- [UEFI Revocation List File - Unified Extensible Firmware Interface Forum](https://uefi.org/revocationlistfile)
- [ARCHIVE - Prior Versions of DBX files - Unified Extensible Firmware Interface Forum](https://uefi.org/revocationlistfile/archive)