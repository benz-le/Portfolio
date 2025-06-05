# Offline Password Audit
This is a guide to perform an offline of Active Directory NTLM password hashes against an updated Have I Been Pwned Offline NTLM password list. Using DSInternals (PowerShell Module) decrypts the `ntds.dit` (containing all AD hashes) with registry `SYSTEM` into a Hashcat format txt file. The PowerShell script `CompareADHashes.ps1` produces a CSV of users and the frequency of password hashes found on HIBP, representing the compromise of users' passwords. Noted, I was able to create a Windows Server 2016 as a domain controller in VMware to replicate a test environment and extract the two required files `ntds.dit` and `SYSTEM`. I added users with set passwords (some unique, some compromised) for test cases. The script ran for about 3 hours, results vary based on RAM and CPU specs.

## Step 1: Use ntdsutil to get the ntds.dit and SYSTEM hive
On the domain controller, open command prompt and run the following:
```
C:\>ntdsutil

ntdsutil: activate instance ntds
ntdsutil: ifm
ifm: create full c:\temp\audit
ifm: quit
ntdsutil: quit
```

## Step 2: Download the HIBP hash list using PwnedPasswordsDownloader
Follow instructions to download recent HIBP Offline NTLM password list:
[PwnedPasswordsDownloader](https://github.com/HaveIBeenPwned/PwnedPasswordsDownloader)\
This password audit requires NTLM hashes and may take time to download (requires no interruptions).
> Download all NTLM hashes to a single txt file called pwnedpasswords_ntlm.txt
`haveibeenpwned-downloader.exe -n pwnedpasswords_ntlm`



## Step 3: Convert hashes in ntds.dit file to Hashcat formatting
> [!TIP]
> Change current directory into workspace directory like `cd C:\temp\audit`

Make sure you have DSInternals installed from [here](https://github.com/MichaelGrafnetter/DSInternals?tab=readme-ov-file#downloads) or if you a running Powershell 5 `Install-Module -Name DSInternals -Force`.\
Open PowerShell as administrator and run the following (as of DSInternals PowerShell Module 5.3):
```
$key = Get-BootKey -SystemHivePath "C:\temp\audit\registry\SYSTEM"
Get-ADDBAccount -All -DBPath "C:\temp\audit\Active Directory\ntds.dit" -BootKey $key -View HashcatNT | Out-File C:\temp\hashes.txt -Encoding ASCII
```

## Step 4: Compare AD Hashes to HIBP password list[^1] 
Download CompareADHashes.ps1 from [here](https://github.com/Nova281/OfflinePasswordAudit/blob/main/CompareADHashes.ps1).
Open PowerShell as administrator and run the following:
```
Import-Module C:\temp\audit\CompareADHashes.ps1
Compare-ADHashes -ADHashes "C:\temp\audit\hashes.txt" -HashDictionary "C:\temp\audit\pwnedpasswords_ntlm.txt" | Export-Csv "C:\temp\audit\output.csv" -NoTypeInformation
```

## Step 5: Remove Critical Files
Regardless of the results, it's critical to securely delete both the `ADHashes.txt`, `NTDS.dit`, and `SYSTEM` files immediately after you're done. These files contain sensitive information, and if they fall into the wrong hands, they could lead to serious security or legal consequences.\
Using Sysinternals' [SDelete](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete) and run the following:
```
.\sdelete.exe -p 7 -r -s <DIRECTORY OR FILE>
```

[^1]: I modified the comparison script for time complexity (execution speed) from this [article](https://www.pickysysadmin.ca/2019/09/13/how-to-perform-an-offline-audit-of-your-active-directory-ntlm-hashes/).


