# OpenWRT Lucy HTTPS OpenSSL Guide on Windows
Assistance for establishing a ssl connection on openwrt lucy using openssl and windows 10. This is my way on how to create and use ssl certificates. No guarantee this will work for you, I'm not an expert either! I won't give any support, so if you run into trouble, take your time to figure out what has to be done.

## Prerequisite
* Windows 10 (I'm using Pro Build 1909) client
* Router with OpenWRT (I'm using OpenWrt 18.06.2 r7676-cddd7b4c77 / LuCI openwrt-18.06 branch (git-19.051.55698-76cf653) on TP-Link TL-WR1043N/ND v1
* libopenssl, luci-ssl (packages installed via opkg install) and uhttpd 
* PuTTY
* OpenSSL

## Getting started

__Router preparations__:

Make sure the required packages and software is installed (on your windows machine as well as on your router). If you didn't have already, define a hostname in your Lucy web interface (System > System Properties > General Settings > Hostname) and choose a local domain name for your server (Network > DHCP and DNS > Server Settings > General Settings > Local server + Local domain) (make sure your domain suffix matches your local server domain specification) (see images)

You also need to edit your /etc/config/uhttpd file and restart uhttpd afterwards to enable HTTPS. I have uploaded my uhttpd config file for comparison. Here you can also set the path to your server.key and server.crt (e.g. /etc/uhttpd.key and /etc/uhttpd.crt) which will be used to complete the chain.

In addition I created a separate directory in /www/ called "ssl" (/www/ssl) to put the important certificates into for verification reasons. You can choose another directory, but then you have to edit the config files (see "Certificate Config Files" section).

__Client preparations__:

After installing OpenSSL on your windows machine, I highly recommend you to tweak your windows explorer context menu by adding the "open command prompt window here as administrator" (manual and registry file can be found here https://www.tenforums.com/tutorials/59686-open-command-window-here-administrator-add-windows-10-a.html). This is necessary because you need to operate in the openssl installation directory and openssl has the bad habit of getting a hiccup in the current command prompt window after issuing some commands so that you have to re-open the prompt in order to get the job done.

__Generate Certificates__:

Here comes the fun part. Took me countless hours to figure this out and ofc I don't know if there is a simpler way, but for now it works.

In order to complete the ssl chain we need a root authority certificate, an intermediate authority certificate and a server certificate for the router.

First you need to create some directories in your OpenSSL bin directory (e.g. C:\Program Files\OpenSSL-Win64\bin):

- certreqs
- certs
- crl
- im
  - im/certreqs
  - im/certs
  - im/crl
  - im/newcerts
  - im/private
- newcerts
- private

Then you have to create some necessary files (you can also find them in my repo in "certificatefiles") (notice the extension is important):

* random.rnd (empty file)
* rootCA.crlnum (only write "1000" without quotes in it and save)
* rootCA.index (empty file)
* rootCA.serial (empty file)

Put them into the OpenSSL bin main directory (e.g. C:\Program Files\OpenSSL-Win64\bin).

Next create the following files for the intermediate certificate authority:
* intermediateCA.crlnum (only write "1000" without quotes in it and save)
* intermediateCA.index (empty file)
* intermediateCA.serial (empty file)

Put them into the OpenSSL bin/im directory (e.g. C:\Program Files\OpenSSL-Win64\bin\im)

Now that the basics are done, we can proceed with the certificate generation.

### Certificate Config Files

We have three config files which we need to generate the certificate. These files are very important and they can also be found in this repo.

I recommend you to take a look and alter them to fit your local ip and domain config. For instance you have to provide a Subject Alternative Name that matches your routers hostname + domain suffix you set earlier.

In my case my routers ip is 192.168.1.1 and the hostname I've choosen for explanatory reasons here is http://mylocalrouter.localdomain

So the three config files are:
1. rootCA.cnf
2. intermediateCA.cnf
3. 192.168.1.1.cnf

You have to copy them into your OpenSSL bin directory (e.g. C:\Program Files\OpenSSL-Win64\bin) if you didn't change the paths in the config files.

### Certificate Generation

In your openssl bin directory (e.g. C:\Program Files\OpenSSL-Win64\bin) open your windows command prompt as admin and issue the following commands. During this you'll need to provide a password to protect the keys, you can choose whatever password you want.

#### Generate Random File Content
```
openssl rand -out random.rnd -base64
```

#### Create a RootCA Certificate Signing Request (CSR) and the key
```
openssl req -new -config rootCA.cnf -out rootCA.csr.pem
```

#### Verify RootCA CSR
```
openssl req -verify -in rootCA.csr.pem -noout -text -reqopt no_version,no_pubkey,no_sigdump -nameopt multiline
```

#### Create RootCA serial
```
openssl rand -hex 16 >> rootCA.serial
```

Next you can change the *startdate* and *enddate* as you wish. I always renew my certificates every year so I've set the time appropriately in the format YearMonthDayHourMinuteSecond (YYYYmmddHHmmss). Notice the *extensions* argument which refers to the section defined in the rootCA.cnf file.

#### Create and Self sign RootCA
```
openssl ca -selfsign -config rootCA.cnf -in rootCA.csr.pem -out rootCA.pem -extensions rootCA_ext -startdate 20200430120000Z -enddate 20210429235959Z
```

(at this step you should open a new command prompt)

#### Show RootCA
```
openssl x509 -in rootCA.pem -noout -text -certopt no_version,no_pubkey,no_sigdump -nameopt multiline
```

#### Verify RootCA
```
openssl verify -verbose -CAfile rootCA.pem rootCA.pem
```

#### Create the RootCA certification revocation list (CRL)
```
openssl ca -gencrl -config rootCA.cnf -out crl/rootCA.crl
```

(new command prompt window)

#### Create IntermediateCA CSR and the key
```
openssl req -new -config intermediateCA.cnf -out intermediateCA.csr.pem
```

#### Verify IntermediateCA CSR
```
openssl req -verify -in intermediateCA.csr.pem -noout -text -reqopt no_version,no_pubkey,no_sigdump -nameopt multiline
```

#### Create IntermediateCA serial
```
openssl rand -hex 16 >> rootCA.serial
```

(Notice: alter the date as you like. I always choose the same date range for the IntermediateCA and the RootCA)
(new command prompt window)

#### Create IntermediateCA and sign it with the RootCA
```
openssl ca -in intermediateCA.csr.pem -config rootCA.cnf -out certs/intermediateCA.pem -extensions v3_intermediateCA  -startdate 20200430120000Z -enddate 20210429235959Z
```

#### Show IntermediateCA
```
openssl x509 -in certs/intermediateCA.pem -noout -text -certopt no_version,no_pubkey,no_sigdump -nameopt multiline
```

#### Verify IntermediateCA
```
openssl verify -verbose -CAfile rootCA.pem certs/intermediateCA.pem
```

#### Create the Intermediate CA CRL
```
openssl ca -gencrl -config intermediateCA.cnf -out crl/intermediateCA.crl
```

Now we can start generating the certificate for our router/server running at 192.168.1.1 under http://mylocalrouter.localdomain

#### Create server certificate CSR and the key
```
openssl req -new -config 192.168.1.1.cnf -extensions req_extensions -reqexts req_extensions -out 192.168.1.1.csr.pem
```

#### Create server certificate backup key (necessary)
```
openssl genrsa -out private/192.168.1.1.backup.key.pem 3072
```

#### Create intermediateCA serial
```
openssl rand -hex 16 >> im/intermediateCA.serial
```

#### Create server certificate and sign it with the intermediateCA
```
openssl ca -in 192.168.1.1.csr.pem -config intermediateCA.cnf -out certs/192.168.1.1.pem -extensions server_ext
```

And we are done. The last step is to install the certificates properly to get HTTPS working.

### Certificate Installation

This is a bit tricky, because we need to combine certificates, upload them to our router and add them to the trusted stores.

I advise you to copy the needed files first from your OpenSSL bin directory to a new directory so you won't loose track of the important ones.

What you need is:

(listed in bin main directory)
rootCA.pem

(listed in bin/certs directory)
192.168.1.1.pem
intermediateCA.pem

(listed in bin/crl)
intermediateCA.crl
rootCA.crl

(listed in bin/private)
192.168.1.1.key.pem

In your new directory (where you copied the above listed files to) open a new command prompt and issue the commands in this order to build the ssl certificate chain:
```
type intermediateCA.pem rootCA.pem > cabundle.pem
```
and
```
type 192.168.1.1.pem cabundle.pem > uhttpd.pem
```
("type" command only applies to ms windows)

Now rename your 

"uhttpd.pem" to "uhttpd.crt" and 
"192.168.1.1.key.pem" to "uhttpd.key" 

(or choose another name if you altered the names in your uhttpd config file earlier).

And upload them into your routers "/etc/" directory (or another directory if you changed the path in your uhttpd config file).

As I mentioned before I created a "ssl" subdir in my routers /www/ folder. Here you can upload both .crl files (intermediateCA.crl and rootCA.crl) as well as the rootCA.pem + intermediateCA.pem. Notice that these files are also referenced in the config files. Change the location path if your config files are different or if you need to.

