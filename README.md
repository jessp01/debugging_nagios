# Debugging Nagios
This repo contains materials used during my session at the Nagios World Conference 2015.
https://conference.nagios.com/schedule/#jportnoy

## Capture command output
The [capture_output.pl] (capture_output.pl) script is used as a wrapper that runs the actual command, stores the STDOUT and STDERR outputs to a log file and then passes the output and RC to Nagios.

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

