# Debugging Nagios
This repo contains materials used during my session at the Nagios World Conference 2015.
https://conference.nagios.com/schedule/#jportnoy

## Capture command output
The (capture_output.pl) [capture_output.pl] script is used as a wrapper that runs the actual command, stores the STDOUT and STDERR outputs to a log file and then passes the output and RC to Nagios.

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
