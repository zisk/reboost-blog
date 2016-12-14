Title: Unziping Files With Powershell v5
Date: 2015-12-17 21:34
tags: powershell

One of the best new features in Windows 10 is the upgrade to Powershell 5. Along with a ton of useful new cmdlets (like [Invoke-WebRequest](https://technet.microsoft.com/en-us/library/hh849901.aspx)), you can now expand zip files natively. The [Archive Module](https://technet.microsoft.com/en-us/library/dn818910.aspx) actually includes both [compression](https://technet.microsoft.com/en-us/library/dn841358.aspx) and [expansion](https://technet.microsoft.com/en-us/library/dn841359.aspx) cmdlets, but I found the unzipping tool the be the most useful for quick one liners.

In its most basic form, simply call `Expand-Archive` and the path to the archive to expand to the current directory. However, things get interesting when you string commands together. For example, this will recursively unzip every archive in a directory tree to its current folder and delete the original archive:

```language-powershell
Get-ChildItem -Recurse -Filter *.zip $path | \
%{ Expand-Archive -Path $_.FullName -DestinationPath $_.DirectoryName; Remove-Item $_.FullName }
```
