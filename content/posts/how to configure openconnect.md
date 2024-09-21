---
title: "How to configure openconnect"
slug: how-to-configure-openconnect
tags: [linux]
published: 2021-08-15
---

A few weeks ago, I noticed that youtube has been slow when I was connected to my office's vpn. I listen to music on youtube so it was annoying having it buffer all the time. Although recently it seems to have recovered, it still buffers or plays in low video quality. This led me to learn that the vpn is configured to route all outgoing traffic except the IPs of certain domains like zoom, which motivates me to figure out how I can setup vpn [split tunneling](https://en.wikipedia.org/wiki/Split_tunneling). A quick search in the internal code repos, I found a repo that someone made doing exactly what I wanted. The code uses openconnect and vpn-slice:
`sudo openconnect <vpn url> --user=<username> --passwd-on-stdin --authgroup=<authgroup> -s 'vpn-slice --verbose --dump <space seperated list of internal domains>'`. However it only worked for the vpn whose authentication only required username and password. The vpn that I am using authenticates using username, password, OTP, and client certificate. So it requires more configuration specifically using `--token-mode`, `--token-secret` for OTP, and `-c` for client certificate.

Since openconnect is a rather niche tool (243 [github](https://github.com/openconnect/openconnect) stars and 106 [gitlab](https://gitlab.com/openconnect/openconnect) stars as of writing) and it supports a variety of vpn protocols, there is little documentation on troubleshooting or guides on how to migrate from Cisco AnyConnect to openconnect. Openconnect has its own guide on [how to connect to vpn](https://www.infradead.org/openconnect/connecting.html) but I find it quite lacking in details. So after much digging, I finally got it to work.

Here is my guide on how to configure openconnect. Openconnect was written to support Cisco AnyConnect (`--protocol=anyconnect`) ([link to doc](https://www.infradead.org/openconnect/anyconnect.html)) and has experimental support for these vpn protocols:

- Juniper Network Connect (`--protocol=nc`) [link to doc](https://www.infradead.org/openconnect/juniper.html)
- Junos Pulse aka Pulse Secure (`--protocol=pulse`) [link to doc](https://www.infradead.org/openconnect/pulse.html)
- PAN GlobalProtect (`--protocol=gp`) [link to doc](https://www.infradead.org/openconnect/globalprotect.html)
- F5 Big-IP (`--protocol=f5`) [link to doc](https://www.infradead.org/openconnect/f5.html)
- Fortinet Fortigate (`--protocol=fortinet`) [link to doc](https://www.infradead.org/openconnect/fortinet.html)
- Array Network SSL (`--protocol=array`) [link to doc](https://www.infradead.org/openconnect/array.html)

I am using macOS and Cisco AnyConnect, so this guide might not be as applicable if you are using different vpn protocol, but should show you what you need even if you're on different OS.

## 1. Find out what you need

Try running `openconnect <vpn gateway url>`

If it errors out "Certificate Validation Error", it means the vpn gateway expects user certificate. The certificate should be installed on your computer by the IT team. On Mac, it is located in Keychain. To find which is the user certificate, find your config file. In my case, it is ~/.anyconnect. Please refer to page 36 of the [AnyConnect manual](https://www.cisco.com/c/en/us/td/docs/security/vpn_client/anyconnect/anyconnect46/administration/guide/b_AnyConnect_Administrator_Guide_4-6.pdf) to check your config file location.

If you can authenticate with just your username and password, you can skip to step 3.

## 2. Extracting the certificate file

Read your config file, there should be a line like: `<ClientCertificateThumbprint>09D2AF8DD22201DD8D48E5DCFCAED281FF9422C7</ClientCertificateThumbprint>`

This the certificate SHA1 fingerprint. In my case, the certificate is located in Keychain > System > My Certificates. Right click the certificate > Get Info > scroll to bottom, it should show the matching fingerprint hash.

Right click the certificate > export to mycert.p12 file.

Run `openssl pkcs12 -nodes -in mycert.p12 -out mycert.pem` to [convert to .pem file](https://stackoverflow.com/a/54719547/4858751) with plaintext private key.

Run `openconnect <vpn gateway url> -c mycert.pem` and it should prompt you for username and password.

## 3. [Optional] OTP secrets extraction

If you are prompted with a second password input, this is for your OTP. Here I have automated generating of OTP from the laptop. You can skip this step if you don't use OTP or are fine with OTP as it is. I basically followed what was described in [this post](https://news.ycombinator.com/item?id=20936222).

There are 2 types of OTP - HOTP and TOTP.

- [HOTP](https://en.wikipedia.org/wiki/HMAC-based_one-time_password) is HMAC based OTP which does not have a countdown and is generated from a secret **token** and a **counter**.
- [TOTP](https://en.wikipedia.org/wiki/Time-based_One-Time_Password) is time based OTP which typically has 30 seconds countdown for the generated OTP and is generated from a secret **token**.

Authenticator apps keep these secret value(s) somewhere in your phone that is typically inaccessible by the user. In my case, on Android, they are stored under `/data/data/<app id>` as I found out from the [Android OTP extractor](https://github.com/puddly/android-otp-extractor/blob/master/src/android_otp_extractor/apps.py). If you have a rooted phone, you can try to run that repo.

Otherwise if you cannot or don't want to root your phone like in my case, you can try transferring the token to another rooted device and extract it from there. I use Duo as my authenticator app which can backup the secrets into Google Drive and restore on another device. I don't have another Android device, so I ran Android emulator on my laptop with root access. If you want to do the same, create the emulator with the [system image that does not have Google Play on it](https://stackoverflow.com/a/45668555/4858751). Download the authenticator app APK from [apkpure](https://m.apkpure.com/search) and install it onto the emulator/rooted device. Restore your OTP account on the app but **be warned that doing so will invalidate the OTP account on your current device at least in the case of Duo**, ie the OTP generated by your phone will no longer be valid, only the OTP generated from the emulator/rooted device works. Read [here](https://guide.duo.com/duo-restore) for details on Duo Restore:

> Restoring or reactivating any "Duo-Protected" and "Duo Admin" accounts on the new device deactivates those accounts on the old device.

I am ok with this and I can probably rollback by deleting my app data and restoring it again on my device, invalidating the account on my emulator.

To extract the file from the rooted emulator/device, run:

1. `adb root`
2. `adb pull /data/data/<app id>/ appdata`

Refer to the [Android OTP extractor](https://github.com/puddly/android-otp-extractor/blob/master/src/android_otp_extractor/apps.py) and get your app's secret token file/db. You may need to use sqlite to extract the token for some authenticator apps.

In the case of Duo, the token is in the form of base32 string in a json file at `/data/data/com.duosecurity.duomobile/files/duokit/accounts.json`, and the counter is an integer in the same file.

Save the token and counter into its own files. These will be used as the --token-secrets argument.

Alternatively, you can use [oathtool](https://www.nongnu.org/oath-toolkit/) to generate the OTP:

- For HOTP: `oathtool --hotp -b $token -c $counter`
- For TOTP: `oathtool --totp -b $token`

For HOTP, you can create a `gen-token.sh` script like:

```bash
#!/bin/bash
counter=$(<~/counter)
oathtool --hotp -b $(<~/secret) -c $counter
echo $((\$counter + 1)) > ~/counter
```

## 4. vpnc script

Openconnect only handles the vpn connection. The routing is done with [vpnc-script](https://www.infradead.org/openconnect/vpnc-script.html). The [default script](https://gitlab.com/openconnect/vpnc-scripts/raw/master/vpnc-script) basically configures the routing based on the rules returned by the VPN server, which essentially routes like the VPN route details tab in Cisco AnyConnect statistics window. You may choose to skip this if you are fine with the routing rules returned by the server. I want to route traffic to the VPN only when necessary. For this, I use [vpn-slice](https://github.com/dlenski/vpn-slice) which allows me to choose which domains/IPs/CIDRs will go through the vpn. I configured to route only a handful of internal domains through the vpn.

vpn-slice works by resolving the internal domains with the internal DNS then appending the IP addresses onto /etc/hosts. When openconnect is stopped, it will call the vpn-slice disconnect hook which reverts back the changes on that file.

## 5. [Optional] Add openconnect to sudoers

As openconnect needs to run as root to create a tun network interface, you can [add it to sudoers](https://gist.github.com/moklett/3170636) so that it doesn't prompt you for password every time you run it.

1. Run `sudo visudo`
2. Append this line to the file `%admin ALL=(ALL) NOPASSWD: /usr/local/bin/openconnect`

Alternatively, you can [run openconnect as non root user](https://www.infradead.org/openconnect/nonroot.html) by creating a tun device in advance and pass it with the `-i` argument to openconnect.

## 6. Assembling the script

Create a `vpn-connect.sh` script like:

```bash
#!/bin/bash
counter=$(<~/counter)
echo "$VPN_PWD" | sudo openconnect vpn.example.com --protocol=anyconnect --user=username -c ~/mycert.pem --token-mode=hotp --token-secret=base32:$(<~/secret),$counter -s 'vpn-slice --verbose --dump \
 internal.example.com \
 internal2.example.com'
echo $(($counter + 1)) > ~/counter
```

Just run the script when you need to connect to the vpn. Alternatively, if you have `gen-token.sh` from step 3, you can run it like:

`echo "$VPN_PWD\n`gen-token.sh`" | sudo openconnect vpn.example.com --protocol=anyconnect --user=username --passwd-on-stdin -c ~/mycert.pem -s 'vpn-slice --verbose --dump internal.example.com internal2.example.com'`

## 7. Disconnecting

To disconnect, run `sudo pkill openconnect`. Openconnect handles SIGINT and SIGTERM signals and calls the disconnect hook on the configured vpnc script.
