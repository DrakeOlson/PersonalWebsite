---
title: "Simple Problems, Simple Answers - Part 1 - Windows Subscription Activation"
date: 2022-06-26T16:48:36-05:00
draft: false
toc: false
images:
tags:
  - Windows 10
  - Activation
  - Key
  - Subscription
---

Activation. It is one of my most least favorite things to talk about because it seems to be a beast of the unknown. Back in the day it used to be a holographic sticker on the side (or bottom) of PCs. You would need to then use that to activate via the telephone or via this new fancy thing called the internet! Both of which are still available for activation, as long as you have a key.

# What About Now?

Well everything seems to be the same, but you don't get a code on the side or bottom of your brand new PC. OEMs have now programmed it into the motherboard where you need to use a command to pull that information out. 

```PowerShell
(Get-CimInstance -Query 'Select * from SoftwareLicensingService').OA3xOriginalProductKey
```

Now, Windows is typically smart about this especially if the device is brand new. In my experience when dealing with systems that have to be wiped and reloaded, Windows doesn't do this fast enough. You may be asking, "Why does this matter if it eventually pulls it automatically?". Well I am impacient, plus a lot of what I do day to day revolves around subscription based activation on systems. 

# What is Subscription Based Activation

Subscription Based Activation does have requirements. You can't just assign a license to a user and call it good. Assuming you have Azure AD Connect setup and working you have the base requirements to assign a license. Once you assign a license that is or contains Windows 10/11 Enterprise/Education Addon (E3,E5,A3,A5), that is just one step in the process. For them to actually activate you have to have Window Pro (if enterprise) or Windows Pro Education (if education) currently activated. These are typically the licenses that are embedded in the motherboard and bought with the PC. Once activated from the motherboard key, the subscription activation will come into place and get you to where you need to be. 

Going back to one of my original points, what happens if it doesn't automatically activate from the motherboard key? Well there is a couple different ways that come to mind, manual, semi-manual, and automatic. 

## 1. Manual Method

By opening up Settings, then Update and Security, finally Activation. You get a brief status of your activation state. The best way I found to help out with this is go and click the *Troubleshoot*. This may not be available anymore due to some security concerns, but if you get your product key from using the above command you can click *Change Product Key* and enter that baby in. As long as the key is valid, you should activate just nicely. 

*Note: This method assumes your devices has already been previously set up.*

## 2. Semi-Manual Method

Now to me that is too many clicks to do on more than one machine. So I have a few lines that you can use with a script to run on the device.

```PowerShell
Start-Transcript -Path "C:\Windows\Logs\Software\ProductKeyActivation.log"
$productKey = (Get-CimInstance -Query 'Select * from SoftwareLicensingService').OA3xOriginalProductKey
if([string]::IsNullOrEmpty($productKey){
  Write-Host "The product key is not programmed into the motherboard."
})
else{
  Start-Process -FilePath "C:\Windows\System32\changepk.exe" -ArgumentList "/Product Key $productKey"
}
Stop-Transcript
```

Try that guy out with whatever method you want that can push PowerShell scripts. Keep in mind, administrative rights will be needed. 

## Automated Method

So what about devices that get wiped and reloaded? We can take that script from above and modify the image we are putting down on devices. I won't go into how this is done, because everyone has different software to accomplish this. The basics of this is to take this script and put it into certain folder path prior to the first boot of the device. During OOBE, Windows looks for a script store in **C:\Windows\Setup\Scripts** called **SetupComplete.cmd**. It will just execute whatever is in this script if it is found.

So we can create the folder structure by mounting the image and putting our script and **SetupComplete.cmd** in there.

```Batch
dism /mount-image /imagefile:Install.wim /index:1 /mountdir:C:\mount
```
*Note: The above command assumes you have the ADK installed on your machine, your current directory contains ***Install.wim*** and also that ***C:\mount*** is blank a blank folder and exists. Also, your ***index*** that you are applying may not be 1.*

We can then navigate to **C:\mount\Windows\Setup\Scripts** (or create it if not already), put create a file called **SetupComplete.cmd** as well as our PowerShell script above in its own **.ps1** file.

In **SetupComplete.cmd**, below is what should be in that file so that our PowerShell script get ran.

```Batch
powershell.exe -ExecutionPolicy Bypass -File "%~dp0\Active-Windows.ps1"
```
*Note: You may have to change the line above to the correct file name of your script*

Finally, make sure that you close out of all process that would have **C:\mount** open. This includes all terminals, file explorers, etc. Then we can unmount the image and save it.

```Batch
dism /unmount-image /mountdir:C:\mount /commit
```

Now you can apply that image that we modified any way you want and devices will pull that key and activate. Thus when the user signs in, Subscription Based Activation should work.

# Resources: Troubleshooting

There are a lot of resources out there for troubleshooting these types of problems. The main one that I will link out is a [blog post by Rudy Ooms](https://call4cloud.nl/2022/05/night-at-the-windows-store-api-service-secret-of-the-subscription-activation/).

The other would be coming from Microsoft which has a lot of [information](https://docs.microsoft.com/en-us/windows/deployment/windows-10-subscription-activation) as well including some scenarios and other requirements that I didn't mention.

# Resources: DISM and Windows ADK

To do some of the lines above, you need to have the ADK installed. [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) is the homepage for downloading the ADK (plus some other useful utilities).

# Resources: SetupComplete.cmd

The SetupComplete script is kind of a hidden gem. [Here](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/add-a-custom-script-to-windows-setup?view=windows-11) is some more information on what the process is during Windows Setup.