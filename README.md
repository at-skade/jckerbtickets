Jamf Connect Kerberos Renewal Fix

Due to issues we’ve found with Jamf Connects ‘AutoRenew’ feature for Kerberos Tickets, and the poor levels of support from Jamf Tech Support on this issue, we have had to build our own custom solution.
The problem: Each time Jamf Connect checks-in it generates a new Kerberos Ticket instead of renewing the existing tickets, we instead end up with lots of duplicates. The knock on effect of this is that the ticket associated with AD never gets updated and eventually expires. As a result Identity Awareness on our firewall loses the username associated with the users session and they can lose access to some services.
The solution, still utilising Jamf Connect as this holds the authentication information, meaning we won't need users to sign in again. We can use Jamf Connect to generate the ticket but use a LaunchD task to schedule our own script, which will delete any existing Kerberos Tickets and then use Jamf Connect to create a new ticket.

Script in basic form:

#!/bin/bash
#Destroy existing kerb tickets
kdestroy -A
#Renew Kerberos Ticket Using JamfConnect
#Password Sync check
open jamfconnect://networkcheck
 
As default Jamf Connect “Checks-In” every 15 minutes, each time it does it generates another ticket. The check-in can be disabled but is needed for the purpose of Jamf Connect, which is to ensure the AD, AAD and local password of the Mac remain in sync.

The solution for this was to disable the automatic check-in by setting the <key>NetworkCheck</key> to 0. Then instead our script we would do the check-in using open jamfconnect://networkcheck. This will both create us our Kerberos Ticket and check the passwords are all in sync.

We then created a Launch Daemon to trigger this script every 2 hours, allowing us to remove any existing Kerberos Tickets using kdestroy -A before Jamf Connect Generates a new ticket. The results are consistent tickets and no more duplicates.
LaunchDaemon:
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd%22 >
<plist version="1.0">
<dict>
<key>Label</key>
<string>com.autotrader.renewkerbkey</string>
<key>ProgramArguments</key>
<array>
<string>/usr/local/at/renewkerb.sh</string>
</array>
<key>StartInterval</key>
<integer>7200</integer>
</dict>
</plist>
Manually renewing: In the past when we was using NoMad, we could hit the “Renew Tickets” button in the NoMad menu to get new tickets. This was replaced in Jamf Connect with the “Connect” button.
Unfortunately the “Connect” button has the same issues as above, it doesn’t renew the existing ticket it instead creates a new one.
To resolve this we have added the following keys, so that the “Connect” button triggers our script.

Jamf Connect “Connect” button script trigger

<key>Scripting</key>
<dict>
<key>OnAuthSuccess</key>
<string>/usr/local/at/renewkerb.sh</string>
</dict>


Resource: https://docs.jamf.com/jamf-connect/2.4.1/documentation/Menu_Bar_App_Preferences.html#ID-00007bec
