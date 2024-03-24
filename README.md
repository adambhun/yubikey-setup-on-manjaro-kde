The intention of this document is to spare you the time of figuring out how to use a yubikey with manjaro and aws and to help you avoid my mistakes. This document is intented to be simple; I do not fully understand the technologies Yubico builds upon and I don't need the highest security standards used by the yubikey bio series.
This is a work in progress.

This guide is for Yubikey 5 NFC Firmware 5.4.3 written in May 2023.

# First steps
1. intsall yubikey manager (yubikey personalization tool is still available, but it's deprecated.)
2. Plug in your yubikey and open yubikey manager.
3. Click on interfaces and select the functionality you want.
4. Select `Applications/FIDO2`  click on use default and set your new FIDO2 PIN. At the time of writing this guide, the pin can be 6-48 characters long and can contain alphanumeric characters. You have 8 retries for this code, so 48 characters is enough for a paranoid. Keep in mind that this is the PIN you have to enter when you register your yubikey to a service, so make sure you can memorize it, or keep it in a password manager.
5. Select `Applications/PIV`  click on use default and set your new PIN and PUK codes. You will not need to remember these codes, but setting them is important in case your key gets stolen. You can keep these in a password manager and forget about them.

More detailed guide: https://www.youtube.com/watch?v=rNtzmgt0Al4

# Setting up authentication against manjaro with yubikey
IMPORTANT NOTES
- You should either buy and setup multiple yubikeys for authentication or set yubikey authentication as sufficient not required method of authentication, so that you can still authenticate with your password
- Encrypted disks can be problematic. I have set up disk encryption when installing manjaro and I do not have the possibility of unlocking my disk at strartup with my yubikey, only with my password. That's okay for me. However, if you set up disk encryption after installing manjaro, make sure you really know what you are doing, to prevent yourself from getting locked out.
- before you make any modifications to your files under `/etc/pam.d` open a terminal window and enter the `sudo -s` command, so that you have a way of undoing everything in case something goes awry. If you set an incorrectly configured yubikey as your only method of authentication <span style="color:red;">YOU CAN LOCK YOURSELF OUT!</span>

## Prerequisites of Manjaro
Install dependencies:
`sudo pacman -S autoconf automake libtool pkg-config libfido2 pam-u2f`

Start pcscd service:
```
systemctl status pcscd.service
sudo systemctl enable pcscd.service
sudo systemctl start pcscd.service
```
Note: this was the solution to the 'Unknown error' Yubikey manager threw, when trying to set up my yubikey.

Add yubikeys:
```
mkdir -p ~/.config/Yubico
pamu2fcfg >> ~/.config/Yubico/u2f_keys  # You will need to touch your yubikey at this point. It should flash green.
```

## Manjaro KDE configuration
Add this line 
`auth sufficient pam_u2f.so cue [cue_prompt=Tap your Yubikey]`
to the TOP of the following files:
```
/etc/pam.d/sudo
/etc/pam.d/kde
/etc/pam.d/sddm
/etc/pam.d/login
/etc/pam.d/polkit-1
```

For multiple users search the documentation of arch for "nouserok". I do not need that, so I did not bother with it.

IMPORTANT NOTE: when logging in to manjaro KDE, the system may not prompt you to touch your yubikey. I guess it takes it some time to fully wake up, because if you tap your yubikey a few times, it will let you in anyway.

# AWS
Console setup is straightforward, if you are authorised to set up MFA for yourself, but it can be more complicated in some cases due to having to set up IAM permissions. Here is a guide that can help with that:
https://resources.yubico.com/53ZDUYE6/as/2trqjptbcrgshncr2w2hrn/AWS_setup_instructions_for_Yubico_YubiKeys
If your policies are in place, you can add your yubikey similarly to as you would add a Virtual MFA device, just select Hardware key on the console. You will be prompted for your FIDO2 key.
## AWS CLI
At the time of writing this guide, AWS CLI does not support Universal 2nd factor MFA, but you can work around that by adding your yubikey as a virtual device using the TOTP standard.

So, on the AWS Console when when adding an MFA device, select "Authenticator app". Note: the name of your yubikey as virtual mfa device can be the same as your yubikey as hardware token as some random characters will be appended to the end of the name of the security key.
Click on "Show secret key" and copy the key.

Notes about the command below:
- only SHA1 is supported.
- "\<name for your account\>" is what this account will be identified by your yubikey, so it can be anything.
Register your account to your yubikey with the following command:
`ykman oath accounts add --touch --period 30 --oath-type TOTP --digits 6 --algorithm SHA1 --issuer AWS <name for your account> <secret key>`

Generate the two codes asked by AWS with this command:
`ykman oath accounts code <name for your account>`

Finish adding your yubikey in the console.

That should be it, so now let's test it. 
Get your token in the first, just like before: `ykman oath accounts code <name for your account>`
Don't forget to set the  ARN for your MFA device in the command below to that of your yubikey as "an authenticator app".
`aws sts get-session-token --serial-number arn:aws:iam::<AWS_ACCOUNT_ID:mfa/<name of your yubikey as "an authenticator app"> --token-code`

Refer to this if you run into some policy-related issues or want to know more\:
https://aws.amazon.com/blogs/security/enhance-programmatic-access-for-iam-users-using-yubikey-for-multi-factor-authentication/
 
# Emit a static password when touched
You can set this up with yubikey manager under Applications/OTP.
 
# GPG, SSH
https://developers.yubico.com/PGP/PGP_Walk-Through.html
https://developers.yubico.com/PGP/Importing_keys.html
https://support.yubico.com/hc/en-us/articles/360013790259-Using-Your-YubiKey-with-OpenPGP

# Best sources

https://github.com/drduh/YubiKey-Guide
 