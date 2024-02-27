### uninstall Edge Chromium in widows 11
https://answers.microsoft.com/en-us/microsoftedge/forum/all/i-can-uninstall-microsoft-edge-on-windows-11/20d266d8-2b2c-4a33-850c-28225ad2ec63
```
1. Press Windows Key + R and type regedit
2. Go to HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Microsoft Edge
3. Go to NoRemove and change the value to 0
4. After you did that open the control panel, go to programs and features and select uninstall a program. And the Uninstall Button should be BACK!!! and now you can uninstall microsoft edge
```
### mount and edit regedit in wim file
#### mount wim
```
dism /mount-wim /wimfile:C:\WimImages\Win7.wim /index:2 /mountdir:C:\AIKMount
```
#### mount regedit
```
reg load HKLM\WimRegistry c:\AIKMount\windows\system32\config\software
```
#### umount regedit
```
reg unload HKLM\WimRegistry
```
#### umount wim and save changed
```
dism /unmount-wim /mountdir:C:\AIKMount /commit
```
### Some settings are hidden or managed by your organization
```
https://answers.microsoft.com/zh-hans/windows/forum/all/%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/590b90dd-b083-4c42-af93-b2adbe6c3c11
https://superuser.com/questions/1376568/windows-10-your-phone-add-a-phone-disabled
```
##### modify regedit
change to 0
```
\HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\CloudContent\DisableWindowsConsumerFeatures
```
