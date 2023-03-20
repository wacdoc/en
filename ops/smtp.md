# Build your own SMTP mail sending server

## preamble

SMTP can directly purchase services from cloud vendors, such as:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali cloud email push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

You can also build your own mail server - unlimited sending, low overall cost.

Below, we demonstrate step by step how to build our own mail server.

## Server selection

The self-hosted SMTP server requires a public IP with ports 25, 456, and 587 open.

Commonly used public clouds have blocked these ports by default, and it may be possible to open them by issuing a work order, but it is very troublesome after all.

I recommend buying from a host that has these ports open and supports setting up reverse domain names.

Here, I recommend [Contabo](https://contabo.com) .

Contabo is a hosting provider based in Munich, Germany, founded in 2003 with very competitive prices.

If you choose Euro as the currency of purchase, the price will be cheaper (a server with 8GB memory and 4 CPUs costs about 529 yuan per year, and the initial installation fee is free for one year).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

When placing an order, remark `prefer AMD` , and the server with AMD CPU will have better performance.

In the following, I will take Contabo's VPS as an example to demonstrate how to build your own mail server.

## Ubuntu system configuration

The operating system here is Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

If the server on ssh displays `Welcome to TinyCore 13!` (as shown in the figure below), it means that the system has not been installed yet. Please disconnect ssh and wait for a few minutes to log in again.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

When `Welcome to Ubuntu 22.04.1 LTS` appears, the initialization is complete, and you can continue with the following steps.

### [Optional] Initialize the development environment

This step is optional.

For convenience, I put the installation and system configuration of ubuntu software in [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Run the following command to install with one click.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Chinese users, please use the following command instead, and the language, time zone, etc. will be automatically set.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo enables IPV6

Enable IPV6 so that SMTP can also send emails with IPV6 addresses.

edit `/etc/sysctl.conf`

Modify or add the following lines

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Follow up with [the contabo tutorial: Adding IPv6 connectivity to your server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Edit `/etc/netplan/01-netcfg.yaml` , add a few lines as shown in the figure below (Contabo VPS default configuration file already has these lines, just uncomment them).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Then `netplan apply` to make the modified configuration take effect.

After the configuration is successful, you can use `curl 6.ipw.cn` to view the ipv6 address of your external network.

## Clone the configuration repository ops

```
git clone https://github.com/wactax/ops.soft.git
```

## Generate a free SSL certificate for your domain name

Sending mail requires an SSL certificate for encryption and signing.

We use [acme.sh](https://github.com/acmesh-official/acme.sh) to generate certificates.

acme.sh is an open source automated certificate signing tool,

Enter the configuration warehouse ops.soft, run `./ssl.sh` , and a `conf` folder will be created in the upper directory.

The directory structure is as follows:

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/W2occKn.webp)

Find your DNS provider from [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , edit `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Then run `./ssl.sh 123.com` to generate `123.com` and `*.123.com` certificates for your domain name.

The first run will automatically install [acme.sh](https://github.com/acmesh-official/acme.sh) and add a scheduled task for automatic renewal. You can see `crontab -l` , there is such a line as follows.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

The path for the generated certificate is something like `/mnt/www/.acme.sh/123.com_ecc。`

Certificate renewal will call `conf/reload/123.com.sh` script, edit this script, you can add commands such as `nginx -s reload` to refresh the certificate cache of related applications.

## Build SMTP server with chasquid

[chasquid](https://github.com/albertito/chasquid) is an open source SMTP server written in Go language.

As a substitute for the ancient mail server programs such as Postfix and Sendmail, chasquid is simpler and easier to use, and it is also easier for secondary development.

Run `./chasquid/init.sh 123.com` will be installed automatically with one click (replace 123.com with your sending domain name).

## Configure Email Signature DKIM

DKIM is used to send email signatures to prevent letters from being treated as spam.

After the command runs successfully, you will be prompted to set the DKIM record (as shown below).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Just add a TXT record to your DNS (as shown below).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## View service status & logs

 `systemctl status chasquid` View service status.

The state of normal operation is as shown in the figure below

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` or `journalctl -xeu chasquid` can view the error log.

## Reverse domain name configuration

The reverse domain name is to allow the IP address to be resolved to the corresponding domain name.

Setting a reverse domain name can prevent emails from being identified as spam.

When the mail is received, the receiving server will perform reverse domain name analysis on the IP address of the sending server to confirm whether the sending server has a valid reverse domain name.

If the sending server does not have a reverse domain name or if the reverse domain name does not match the IP address of the sending server, the receiving server may recognize the email as spam or reject it.

Visit [https://my.contabo.com/rdns](https://my.contabo.com/rdns) and configure as shown below

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

After setting the reverse domain name, remember to configure the forward resolution of the domain name ipv4 and ipv6 to the server.

## Edit the hostname of chasquid.conf

Modify `conf/chasquid/chasquid.conf` to the value of the reverse domain name.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Then run `systemctl restart chasquid` to restart the service.

## Backup conf to git repository

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

For example, I back up the conf folder to my own github process as follows

Create a private warehouse first

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Enter the conf directory and submit to the warehouse

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Add sender

run

```
chasquid-util user-add i@wac.tax
```

Can add sender

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Verify that the password is set correctly

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

After adding the user, `chasquid/domains/wac.tax/users` will be updated, remember to submit it to the warehouse.

## DNS add SPF record

SPF ( Sender Policy Framework ) is an email verification technology used to prevent email fraud.

It verifies the identity of a mail sender by checking that the sender's IP address matches the DNS records of the domain name it claims to be, preventing fraudsters from sending bogus emails.

Adding SPF records can prevent emails from being identified as spam as much as possible.

If your domain name server does not support SPF type, just add TXT type record.

For example, the SPF of `wac.tax` is as follows

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF for `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Note that I have `include:_spf.google.com` here, this is because I will configure `i@wac.tax` as the sending address in the Google mailbox later.

## DNS configuration DMARC

DMARC is the abbreviation of (Domain-based Message Authentication, Reporting & Conformance).

It is used to capture SPF bounces (maybe caused by configuration errors, or someone else is pretending to be you to send spam).

Add TXT record `_dmarc` ,

The content is as follows

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

The meaning of each parameter is as follows

### p (Policy)

Indicates how to handle emails that fail SPF (Sender Policy Framework) or DKIM (DomainKeys Identified Mail) verification. The p parameter can be set to one of three values:

* none: No action is taken, only the verification result is fed back to the sender through the email reporting mechanism.
* Quarantine: Put the mail that has not passed the verification into the spam folder, but will not reject the mail directly.
* reject: Directly reject emails that fail verification.

### fo (Failure Options)

Specifies the amount of information returned by the reporting mechanism. It can be set to one of the following values:

* 0: Report validation results for all messages
* 1: Only report messages that fail verification
* d: Only report domain name verification failures
* s: only report SPF verification failures
* l: Only report DKIM verification failures

### rua & ruf

* rua (Reporting URI for Aggregate reports): Email address for receiving aggregated reports
* ruf (Reporting URI for Forensic reports): email address to receive detailed reports

## Add MX records to forward emails to Google Mail

Because I couldn't find a free corporate mailbox that supports universal addresses (Catch-All, can receive any emails sent to this domain name, without restrictions on prefixes), I used chasquid to forward all emails to my Gmail mailbox.

**If you have your own paid business mailbox, please do not modify the MX and skip this step.**

Edit `conf/chasquid/domains/wac.tax/aliases` , set forwarding mailbox

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indicates all emails, `i` is the email address prefix of the sending user created above. To forward mail, each user needs to add a line.

Then add the MX record (I point directly to the address of the reverse domain name here, as shown in the first line in the figure below).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

After the configuration is complete, you can use other email addresses to send emails to `i@wac.tax` and `any123@wac.tax` to see if you can receive emails in Gmail.

If not, check the chasquid log ( `grep chasquid /var/log/syslog` ).

## Send an email to i@wac.tax with Google Mail

After Google Mail received the mail, I naturally hoped to reply with `i@wac.tax` instead of i.wac.tax@gmail.com.

Visit [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) and click "Add another email address".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Then, enter the verification code received by the email that was forwarded to.

Finally, it can be set as the default sender address (along with the option to reply with the same address).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

In this way, we have completed the establishment of the SMTP mail server and at the same time use Google Mail to send and receive emails.

## Send a test email to check whether the configuration is successful

Enter `ops/chasquid`

Run `direnv allow` to install dependencies (direnv has been installed in the previous one-key initialization process and a hook has been added to the shell)

then run

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

The meaning of the parameters is as follows

* user: SMTP username
* pass: SMTP password
* to: recipient

You can send a test email.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

It is recommended to use Gmail to receive test emails to check whether the configurations are successful.

### TLS standard encryption

As shown in the figure below, there is this small lock, which means that the SSL certificate has been successfully enabled.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Then click "Show Original Email"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

As shown in the figure below, the Gmail original mail page displays DKIM, which means that the DKIM configuration is successful.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Check the Received in the header of the original email, and you can see that the sender address is IPV6, which means that IPV6 is also configured successfully.
