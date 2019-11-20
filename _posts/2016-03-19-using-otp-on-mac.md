---
title: "Using One-Time-Password on Mac"
categories:
  - general 
tags:
  - One Time Password
  - OTP
excerpt: "Finding it hard to auto-generate OTP on Mac ?"
modified: 2016-03-19
comments: true
share: true
---

I felt challenged when my geeky/talented colleagues discussed how to automate generation of an OTP ( **O**ne **T**ime **P**assword ) on Mac. After an hour of effort I could make it work on my Macbook Pro with freely available and built-in utilities. 

Before I list out the steps for the setup, a huge **DISCLAIMER**!

> This method relies on an insecure storage of the token/key/secret. Understand the risks involved and use it at your own risk. If you know or find a better/easier way to do the same, send me a link to the documentation and I will happily update the post and recommend the link as the first place to go.

### Prerequisites

* Whether it is your workplace or any site, you should be given or should have means to generate a __Software Token__. You have to be careful of the kind (TOTP or HOTP) token you generate, this setup is using __TOTP__. The URL under the **QR** code of the form shown gives that away: `otpauth://hotp/BLAH?secret=TOPSECRET&counter=0`
* Install [Homebrew](http://brew.sh/)

_For colleagues at Red Hat :_ When generating the token, choose __Advanced Options__ on the right and select __Enroll TOPT Token__. The secret is embedded in the URL under the **QR** code). 

### Setup 

#### Step 1 : Install OATH Toolkit<sup>1</sup>

Run the following command to install the OATH toolkit

```sh 
brew install oath-toolkit 
```

#### Step 2 : Script the OTP generation

Build two files in your home directory as shown below

```sh
# Use the OTP token provided to you in the following command
echo "[OTP Token]" > ~/.totp_key
```

Copy/Paste the following script as __`~/.totp_script`__

```sh 
key=`cat ~/.totp_key`
code=`oathtool --totp -b -d 6 $key`
echo -n $code | pbcopy
```

If you were doing the same on **Fedora**, following would be the script:

```sh
key=`cat ~/.totp_key`
code=`oathtool --totp -b -d 6 $key`
echo -n $code | xclip -selection clipboard
#xdotool key ctrl+v
sleep 0.4; xdotool type "$(xclip -selection clipboard -o)" 
```

#### Step 3 : Script an Automator service

1. Open __Automator__ application on your Mac 
2. Choose __Service__ as the document you want to work with
3. In the search window on the left, type `Run Shell Script` and double click on it. Paste `. ~/.totp_script` in the window presented
4. Now type `Run AppleScript` in the search window, double-click on the result and paste the following piece of code into the window presented
5. Save the service as **TOTP**

```applescript
on run
    
    tell application "System Events" to tell (1st process whose frontmost is true) to keystroke "v" using command down
    
end run
```

If the steps were correctly followed, below is how your Automater service would look like
[![](/images/totp-automator.png)](/images/totp-automator.png)

#### Step 4 : Assign a keyboard shortcut to the service

1. Open  __System Preferences → Keyboard → Shortcuts__
2. Choose **Services** on the left hand side and navigate to **TOTP** service on the right side
3. Click/Highlight the service and click on `add shortcut`
4. Provide a unique key combination as shortcut. The one I chose was __Shift + Command + 1__

Here is how my shortcut looks like
[![](/images/totp-shortcut.png)](/images/totp-shortcut.png) 

#### Step 5 : Enjoy the shortcut!

Click into any text box of any window and invoke the shortcut. It will paste the 6 digit OTP just as how Yubikey does. 

_For colleagues at Red Hat_: Make sure you resync the token before you could start using it with the pin.

### References:

1. [https://github.com/daveio/shell-otp](https://github.com/daveio/shell-otp)
