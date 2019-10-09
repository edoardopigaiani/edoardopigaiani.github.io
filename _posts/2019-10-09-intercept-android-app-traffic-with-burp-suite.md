---
layout: post
title:  "Intercept Android app traffic with Burp suite"
date:   2019-10-09 22:25:11 +0200
categories: Android
---
## Introduction and clarifications

If the app we are going to analyze uses Certificate pinning you shouldn’t be referring to this guide.
[Certificate pinning](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning) is the process of comparing the server’s TLS certificate against a saved copy of that certificate, app developers are often encouraged to bake in a copy of the server’s certificate and make use of certificate pinning because it increases the complexity of MITM attacks. There are two ways of bypassing it: the first one is to decompile the .apk, patch the smali code and recompile it; the second one is to install the Burp CA as system-level CA on the device. I’m going to cover the second one, since last [Paged out!](https://pagedout.institute/) issue  explained how to decompile .apk files to inspect them.


## Prerequisites
   - Burp suite, openssl, adb
   - Rooted [Android 7+](https://android-developers.googleblog.com/2016/07/changes-to-trusted-certificate.html) device
   - Wireless network shared between the two devices (the one running Burp suite and the Android one)

## Procedure

### Install the Burp certificate as system-level CA
   - Export the Burp CA
        Start Burp suite, navigate to Proxy > Options > Import/export Ca certificate
* Convert the CA using openssl, since Android wants it in .pem format and to have the filename equal to the subject_hash_old value appended with .0
```
$ openssl x509 -inform DER -in cacert.der -out cacert.pem
$ openssl x509 -inform PEM -subject_hash_old -in cacert.pem | head -1  
$ mv cacert.pem <hash>.0
```

- Mount ```/system``` as writable, then copy the certificate to the device
```
$ adb root  
$ adb remount  
$ adb push <cert>.0 /sdcard/  
```

- Spawn a shell, move the certificate where it belongs and ```chmod``` to 644
```
$ adb shell
$ mv /sdcard/<cert>.0 /system/etc/security/cacerts/  
$ chmod 644 /system/etc/security/cacerts/<cert>.0 
```

- Reboot the device, browsing to Settings → Security → Trusted credentials should show “Portswigger CA” as system certificate.


### Configure the proxy server on Burp suite
   - Start Burp suite, navigate to Proxy → Options → Proxy listeners → Add and add a new proxy binded to an unused port and to all the interfaces.


### Configure the proxy server on Android
   * Long press the name of the wireless network you want to modify the proxy for (the one you will share between the two devices), then navigate to Modify network → Advanced options → Proxy → Manual
   * Use the IP of the machine running Burp as Proxy address, and set the same port used on Burp proxy in order to properly route the traffic.


### Intercept the traffic
   * Reconnect to the wireless network on your Android device, you should start seeing traffic flowing on Burp’s Intercept tab.