# Debugging Nagios
This repo contains materials used during my session at the Nagios World Conference 2015.
https://conference.nagios.com/schedule/#jportnoy

## Trouble scenarios



### Nagios web interface fails to load
Upon requesting the Nagios web interface, one gets 'Internal Server Error' [HTTP 500]
- Find the Nagios Apache configuration using Apache's CLI API:
```
apachectl -t -DDUMP_VHOSTS
```
- If no custom log was defined, find default error log file.

When using Apache 2.4 and above, this command is available:
```
# apachectl -t -DDUMP_RUN_CFG | grep ErrorLog:
```
Sample output:
```
Main ErrorLog: "/var/log/apache2/error.log"
```
When working with older versions, try to find the value used while Apache was compiled with:
```
# apachectl -V | grep DEFAULT_ERRORLOG
```
Note that unlike -DDUMP_RUN_CFG this shows the compilation defaults and not the runtime values so, it may be overridden in the config files.

- Check log for errors and hopefully, correct them:)

### Trying to reschedule test executation fails
Upon committing, one gets:
```Error: Could not stat() command file '/var/lib/nagios3/rw/nagios.cmd'!```
- Check which user and group Apache is running under.
When using Apache 2.4 and above, this command is available:
```
# apachectl -t -DDUMP_RUN_CFG|grep "^User\|^Group"
```
For older Apache versions, try finding the user and group with:
```
# ps -eo user,group,cmd|grep "apache\|httpd"
```
- Add the Apache user to the nagios group so that Apache can write to it
```
# usermod -a -G nagios $APACHE_USER
```

* Another nice way to handle this issue is to use Apache's ITK MPM instead of Prefork or Worker. ITK allows you to run each Apache vhost under a different user, in which case, you could just run Nagios' vhost as the user 'nagios'.*


### No mail alerts are recieved
- Make sure notifications are enabled for the service
- Check the Nagios log to see which command is used to alert, a sample entry would look like:
```
[1443546666] SERVICE NOTIFICATION: root;host;Service Name;CRITICAL;notify-service-by-email;(Return code of 127 is out of bounds - plugin may be missing)
```
- Find the mail notification definition in the Nagios config
A sample entry:
```
define command{
        command_name    notify-service-by-email
        command_line    /usr/bin/printf "%b" "***** Nagios *****\n\nNotification Type: $NOTIFICATIONTYPE$\n\nService: $SERVICEDESC$\nHost: $HOSTALIAS$\nAddress: $HOSTADDRESS$\nState: $SERVICESTATE$\n\nDate/Time: $LONGDATETIME$\n\nAdditional Info:\n\n$SERVICEOUTPUT$\n" | /usr/bin/mail -s "** $NOTIFICATIONTYPE$ Service Alert: $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$ **" $CONTACTEMAIL$
        }
```
In this example, the notification is sent using:
```
/usr/bin/mail -s "** $NOTIFICATIONTYPE$ Service Alert: $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$ **" $CONTACTEMAIL$
```
- The mail utility expects some MTA to be listening so it can send the emails through it, since no 'special' arguments were passed to the mail utility, it would be try to talk to the MTA over port 25 using TCP, check that you actually have a listener and see which program is listening:
```
# netstat -plnt|grep 25
```
- Try running the command manually from the shell as the nagios user and check the MTA log for errors
In our example, the log reveils:
```
Sep 29 18:26:37 jessex postfix/smtp[14645]: ECB8B120273: to=<nagios@jessex.kaltura.com>, relay=email-smtp.us-east-1.amazonaws.com[54.243.69.182]:587, delay=0.53, delays=0.02/0/0.47/0.05, dsn=5.0.0, status=bounced (host email-smtp.us-east-1.amazonaws.com[54.243.69.182] said: 501 Invalid MAIL FROM address provided (in reply to MAIL FROM command))
```
This is because our mail service expects the email to be sent from a specific mail address and rejects mail sent by nagios@jessex.kaltura.com.

We will correct this by adding a mapping in Postfix, which is what we use as an MTA, so that emails originally sent by nagios@jessex.kaltura.com will acutally be sent using the user our service expects.

First, we need to set:
```
smtp_generic_maps = hash:/etc/postfix/generic
```
In our Postfix main config file - /etc/postfix/main.cf. Then we add:
```
nagios@jessex.kaltura.com good@email.addr
```
to ```/etc/postfix/generic``` and use postmap to create a proper Postfix lookup table out of this file:
```
# postmap /etc/postfix/generic
```
Next, we need to restart Postfix:
```
# service restart postfix
```
*Note that the last few commands are specific to Postfix but have parallel in other MTAs. Another alternative would have been to change the notify-service-by-email command and set the sender there but this is more global, which is usually more desirable.***


### Host check fails cause ICMP is blocked
In many cases, this is out of your control, in such cases, the easiest thing to do is to set an alternative command for checking that the host is alive.

This can easily be done by adding this directive to the host definition:
```
check_command alternative_check_command
```
This will override the default which will typically be to use the check_ping core plugin.


## Capture command output
The [capture_output.pl] (capture_output.pl) script is used as a wrapper that runs the actual command, stores the STDOUT and STDERR outputs to a log file and then passes the output and RC to Nagios.

If the original command's return code is bigger than 3 [UNKNOWN], 3 will be returned and the original return code will appear as part of the output.


### Plugin fails with '(Return code of 127 is out of bounds - plugin may be missing)'
Nagios can only handle 0,1,2,3 as return codes, anything else will result in the output above.

To debug this, lets use the [capture_output.pl] (capture_output.pl) introduced in a previous slide.
For this example, the following command is failing:
```
define command{
        command_name check_ssl_cert_bad
        command_line   /usr/lib/nagios/plugins/check_special_http -H '$HOSTADDRESS$' -I '$HOSTADDRESS$' -C10
}
```

We will now revise the command_line to use the capture wrapper, like so:
define command{
        command_name check_ssl_cert_bad
        command_line    /usr/lib/nagios/plugins/capture_plugin.pl /usr/lib/nagios/plugins/check_special_http -H '$HOSTADDRESS$' -I '$HOSTADDRESS$' -C10
}

This will help us in two ways:
- Nagios will now show the following output instead of '(Return code of 127 is out of bounds - plugin may be missing)':
```
Original RC: 127, /usr/lib/nagios/plugins/check_special_http: error while loading shared libraries: libssl.so.0.9.8: cannot open shared object file: No such file or directory 
```
- The captured-plugins.log will have an entry with the full command so we can try to run it in the shell and debug:
```
 2015-8-29 16:10:56 ------ debugging
cmd=[/usr/lib/nagios/plugins/check_special_http '-H' 'kino.kaltura.com' '-I' 'kino.kaltura.com' '-C10']
output=[/usr/lib/nagios/plugins/check_special_http: error while loading shared libraries: libssl.so.0.9.8: cannot open shared object file: No such file or directory
]
retcode=127

```

### check_mysql plugin fails with Can't connect to MySQL server on 'mysql.host' (111) 
- Use [capture_output.pl] (capture_output.pl) to log the command output to a file
- Try running the command from the shell
- On the MySQL server, check what IP/network the daemon is binded with, and what port is the listener on:
```
# netstat -plntu|grep mysql
```
In our example, the output is:
```
tcp        0      0 127.0.0.1:3306              0.0.0.0:\*                   LISTEN      6326/mysqld 
```
Meaning all connections from outside will be blocked.

This needs to be correct by changing the value for bind-address in the MySQL config file.

- After correcting that, use telnet/nc to see if the traffic is blocked
- Check mysql.user table to make sure the username and host Nagios uses is allowed with:
```
mysql> select user,host from mysql.user;
mysql> show grants for 'nagios_user'@'host';
```
In our example, the output is:
```
+--------+-----------+
| user   | host      |
+--------+-----------+
| nagios | 127.0.0.1 |
+--------+-----------+

+---------------------------------------------------------------------------------------------------------------+
| Grants for nagios@127.0.0.1                                                                                   |
+---------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'nagios'@'127.0.0.1' IDENTIFIED BY PASSWORD '*664299055A186321D3041F9E59595E52BC96CCA1' |
| GRANT SELECT ON `kaltura`.* TO 'nagios'@'127.0.0.1'                                                           |
+---------------------------------------------------------------------------------------------------------------+

```
Meaning the nagios user can only connect from localhost and perform select operations on one DB called 'kaltura'.
This needs to be corrected so that the Nagios host is allowed to connect, we can accomplish this with:
```
mysql> GRANT SELECT ON kaltura.* TO 'nagios'@'nagios_host' identified by 'S0ME_PassWD';
mysql> FLUSH PRIVILEGES;
```

### check_disk plugin is failing with DISK CRITICAL - /run/user/1001/gvfs is not accessible: Permission denied 
- Check the capture_plugin.log to find the full command Nagios executed
In our example, the command is:
```
/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -e
```
The check_disk plugin is part of the Nagios core plugins and is written in C, to get a better understanding of what it does, we can use the strace utility.
From the man page:
```
strace - trace system calls and signals
```
The strace output goes into STDERR, we will therefore want to run it like so:
```
strace /usr/lib/nagios/plugins/check_disk -w 20% -c 10% -e 2>&1 |vim -
```
This command will redirect all STDERR output to STDOUT and then open VIM, telling it to display the contents of what it read from STDIN.

This is useful cause VIM provides nice syntax highlighting for the strace output.
The relevant error will be:
```
stat("/run/user/1001/gvfs", 0x7f03977f0090) = -1 EACCES (Permission denied)
```
From which we can understand that the check_disk plugin calls stat on each file and fails on /run/user/1001/gvfs.

If we then try to call stat on this file manually from the shell, we'll find out that, even as root, one gets permission denied.
It is therefore not surprising that our nagios gets it as well.

Looking into what GVFS is, it is understandable why it should be this way:
* gvfs is a userspace virtual filesystem where mounts run as separate processes which you talk to via D-Bus. It also contains a gio
 module that seamlessly adds gvfs support to all applications using the gio API. It also supports exposing the gvfs mounts to non-gio
 applications using fuse. *

Luckily, for us, the check_disk plugin can skip this file and others like it if you pass along the following as args:
```
 -A -i '.gvfs'
```


### SSL Certificate check shows wrong certificate
The check we are running is:
```
/usr/lib/nagios/plugins/check_http -H '$HOSTADDRESS$' -I '$HOSTADDRESS$' -C10
```
It returns with RC 0 [OK] but prints out:
OK - Certificate '*.mediaspace.kaltura.com' will expire on 07/22/2017 22:59. 
However, when looking at the certificate from a browser or using curl, a different certificate is returned:
```
$ curl https://kino.kaltura.com -I -v

* Server certificate:
*        subject: OU=Domain Control Validated; CN=*.kaltura.com
*        start date: 2015-08-27 12:46:41 GMT
*        expire date: 2018-08-27 12:46:41 GMT
*        subjectAltName: kino.kaltura.com matched
*        issuer: C=US; ST=Arizona; L=Scottsdale; O=GoDaddy.com, Inc.; OU=http://certs.godaddy.com/repository/; CN=Go Daddy Secure Certificate Authority - G2
```

- First, lets try to figure out how the check_http plugin performs its check. Since the plugin is a precompiled binary [written in C], lets try:
```
$ strace /usr/lib/nagios/plugins/check_http -H kino.kaltura.com -p 443 -S -C10
```
Looking at the output, we can see the plugin uses the OpenSSL library to do its work:
```
open("/lib/x86_64-linux-gnu/libssl.so.1.0.0", O_RDONLY|O_CLOEXEC) = 3
```
- Therefore, lets try to use the openssl CLI client to run a similar check:
```
$ openssl s_client -connect kino.kaltura.com:443
```
Indeed, the result is similar to what the Nagios plugin gave us:
```
subject=/OU=Domain Control Validated/CN=*.mediaspace.kaltura.com
issuer=/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./OU=http://certs.godaddy.com/repository//CN=Go Daddy Secure Certificate Authority - G2
```

Reason? SNI support is not the default for OpenSSL.

What is SNI?
*Server Name Indication (SNI) is an extension to the TLS computer networking protocol[1] by which a client indicates which hostname it is attempting to connect to at the start of the handshaking process.*

This means that, when using a web server that hosts multiple domains, such as say, Apache with several VHosts, if the client does not indicate which domain it wishes to speak to, a wrong certificate may be used.
We will therefore change our command to use SNI, like so:
```
/usr/lib/nagios/plugins/check_http -H kino.kaltura.com -p 443 -S -C10 --sni
```
which will return the desired result.

