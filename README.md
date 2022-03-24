# Imprivata_User_Maintenace
Auto-remove inactive users from Imprivata (PowerShell)

This script will automatically purge users from Imprivata (Active Directory Security Group) based on a duration _that you set_ of inactivity (Last Log On date) or number of days a user remains unenrolled.  Once setup, this script will completely automate this user maintenace as well as send email reports of the removed users, license count etc, when the script is executed.

**Prerequisites:**
1. User account for Imprivata maintenance in Active Directory
2. Network share with permissions, granting the Imprivata maintenance user R/W access. (Example:  \\\MyServer\Imprivata\Reports\Inactive\Exports )
3. Server or Workstation that can be used for running scheduled tasks and has the ActiveDirectoy PowerShell module installed.
    Go here to get the module:   https://blogs.technet.microsoft.com/ashleymcglone/2016/02/26/install-the-active-directory-powershell-module-on-windows-10/
4. Imprivata **MUST** be set to syncronize users based on an Active Directory Security Group (Example: "imprivata_users"  (CN=imprivata_users,OU=Groups,DC=ourcompany,DC=local))</br>
_If your environment is set to Synchronize based on OU, this script will not work!_
   
**SETUP**

**IMPRIVATA CONSOLE - Auto-Report Creation**</br>
1. Log on to your Imprivata admin console
2. Goto Reports > Add New Report > User Details
3. Specify a name for the report (Example: Inactivity Maintenance)
4. Filter Users by "state" / "enabled"
5. Click "Save and Export >>"
6. On the right-hand side, there is a box labled "File Server Configuration"</br>
 -Protocol = Network Share</br>
 -Server = server name (no double backslashes and no FQDN.  Just the server's name [example:  MyServer])</br>
 -Username = Imprivata maintenance user in AD</br>
 -Password = Password for the maintenance user</br>
 -Domain = Your domain name (Example: ourcompany.local)</br>
7. On the same page, set the frequency to daily or weekly and select a time (example:  Daily  @  12:00 AM)
8. Set the save location, which should be a share on the server that you named in the config box.
  Location should be in the following format:  \Imprivata\Reports\Inactive\Exports
9. Select the Imprivata Site that you want to run the report against.
10. Click save.</br>
You should be able to manually run the report and see the .CSV file that gets placed in your network share.  _If you do not, you most likely have a permissions error or a syntax error_</br>
If you see the file, you are ready for the next step.

**IMPRIVATA CONSOLE - Auto-Sync setup**</br>
1. Goto Users > Synchronize
2. Select a domain and click next
3. Scroll down to the bootom where the headding "Automate the synchronization process" is located.
4. Check "Automate Synchronization?"
5. As a safeguard, I like to check "Do not Synchronize if Users are to be Deleted?" and set "Only if the number of users to be deleted exceeds" to 20 or 30.
6. Select "Every day" from the drop-down menu and select a time that is approximately 1 hour after the above report is scheduled to run (Example: 1:00 AM)
7. Click Save.
 
**ACTIVE DIRECTORY**</br>
1. Create a security group that will contain users which will receive the report emails from the script.
2. Example: SCRIPTSendMail_Imprivata  (CN=SCRIPTSendMail_Imprivata,OU=Groups,DC=ourcompany,DC=local)
3. Add your admins (recipients) to this group.
4. If you have any Generic AD accounts that use Imprivata, make the AD "description" field "Generic Account" or something uniform.
 
**POWERSHELL SCRIPT**</br>
1. Copy the PowerShell script to your share (Example: \\\MyServer\Imprivata\Reports\Inactive)</br>
_DO NOT place it in the "Exports" sub-directory, as that directory gets parsed by the script_
2. Edit lines 2 - 14 of the script to suit your environment.</br>
```
$ImprivataLicences = 1000 # This is the total number of Imprivata Licenses that you have.
$InactivityTime = 42 # Amount of DAYS since last logon. Any account LAST LOGON DATE greater than this number will be removed.
$UnenrolledDaysLimit = 42 # Amount of DAYS that the account remains UNENROLLED greater than this number will be removed.
$AdSecurityGroup = "imprivata_users" # ActiveDirectory Security group in which the Imprivata users are assigned.
$SSOPolicyName = "SSO Users" # Name of the Imprivata SSO Policy in which this script will enforce.  This is good if you are also using Confirm ID and have a seperate policy that you do not with to be included in this script.
$ExcludedUsers = "impmaintacct,headhoncho" # This is a list of users that you want to Exclude from being removed. These are generally managers or service accounts.
$EmailFromAddress = "Imprivata.Maintenance@ourcompany.org" # This is the FROM address that will appear in the email.
$EmailGroup = "SCRIPTSendMail_Imprivata" # This is the AD Security group to wich the members will be sent the report.
$EmailSubject = "Imprivata User Maintenance - REMOVED ACCOUNTS" # This is the email SUBJECT.
$EmailSMTPServer = "ourcompany-org.mail.protection.outlook.com"  # This is the SMTP server for the email function.
$ScriptDir = "\\MyServer\Imprivata\Reports\Inactive" # This is the root directory in which the script resides.
$ImpCSVDir = "$ScriptDir\Exports" # This is the directory to which Imprivata exports the CSV reports. THIS IS CONFIGURED IN IMPRIVATA.
$GenericAcctDesc = "Generic Account" # Line text from AD description field that indicates a Generic account
```
These are all the changes you need to make.</br>
If you feel comfortable changing the HTML email portion at the bottom, do so to suit your needs.

**SCHEDULED TASK CONFIGURATION**</br>
1. Schedule a task on a Windows Server or Workstation with PowerShell.
(NOTE: This machine needs the ActiveDirectory PowerShell module, as noted in prerequisite #3, above)
2. Run the task as a user that has R/W permissions on the network share direcories and "Domain Admin" permissions.
3. Based on the previously configured Imprivata report, The CSV file was being created at 12:00 AM
4. Create a Scheduled task that runs daily (or weekly, depening of your Imprivata report). Schedule it for 12:30AM
5. The action should be:  C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
6. The command line argument should be:  -File \\\MyServer\Imprivata\Reports\Inactive\Imprivata_User_Maintenance.ps1

You are now done!</br>
You can test the script by manually running the scheduled task.  You should receive an email (if you are in the recipient security group).

**TROUBLESHOOTING**</br>
If you are not recieving an email after manually running the scheduled task or if the task indicates a failure, you can open the script in the PowerShell ISE and run it from there.
That will give you any error codes that may arise.

**COMMON ERRORS**</br>
SMTP relay server misconfiguration.</br>
Network share permissions.</br>
Shecduled Task Run-As user is not a "Domain Admin".</br>
The "ActiveDirectory" PowerShell module is not installed on the task server.

**THINGS TO NOTE**</br>
The "Logs" directory that gets created is used for multiple items, one of which is a .txt logfile.  This file gets written to with the name of the Imprivata .CSV exported file, so that the script does not run the same file twice.  This is a fail-safe to prevent an undeleted CSV from being parsed again.</br>
The script does a good job at cleaning up after itself. The "Exports" directory should only contain the fresh CSV until it is parsed, after-which it will be deleted.</br>
**After 08/19/2019**</br>
You will notice a new Directory after applying the update, "UnenrolledCheck"</br>
This is for handling unenrolled users.  There will be a file that resides in this directory. Leave it alone, as it keeps track of your unenrolled users and the amount of time they have remained unenrolled.</br>
_As of right now, the unenrolled user maintenance is based on running this script once per-day, as it increments the days of your unenrolled users.  Hindsight being 20/20, I will most likely change this to a date/time stamp, to accomodate users that run it more than once per day or less than once per day. For now, just know that the "days unenrolled" are incremented by 1, each time it is ran_.</br>
**After 02/20/2020**</br>
You will notice a new variable "$SSOPolicyName"</br>
This is for limiting the script to enforcing the maintenance on a single User Policy.  This comes in handy if you have implemented and have created a separate policy for Confirm ID.  This will skip your Confirm ID policy, so that you are not removing users that have enrolled for ConfirmID.
