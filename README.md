# Create-New-User-OWCHC
A script to create a new user in OneWorld AD
<#
.Synopsis
    Create new user for OneWorld CHC domain
.DESCRIPTION
    -Create new user for OneWorld CHC domain. Username must be 6 charaters long and not already used.
    -Add user Job name as AD account description.
    -Sets U Drive and Remote Desktop Services Home Folder/Drive
   
    -Adds users to following groups:
        Domain Users - Done by default
        Citrix Users
        
            - Loops for more user creation.
    
.NOTES
    Created:	 2018-10-04
    Updated:     
    Version:	 1.0
    Based on:
    -https://blogs.technet.microsoft.com/heyscriptingguy/2008/10/23/hey-scripting-guy-how-can-i-edit-terminal-server-profiles-for-users-in-active-directory/
    
    Author(s) : 
                K. Chris Bryson & Dustin DeTurk
                
    Disclaimer:
    This script is provided 'AS IS' with no warranties, confers no rights and 
    is not supported by the author.
    
#>


#### Generate email with new user info ###
Function GenerateEmail
{
    $CitrixPassword = 'OneWorld1'
   
if ($Global:Location -eq 'CHDI')
{
    $MailCreds = $null
    $MailCreds = @"
      <tr>
        <th class="tg-ltad">Credential</th>
        <th class="tg-0lax">Value</th>
      </tr>
      <tr>
        <td class="tg-0lax">Full Name</td>
        <td class="tg-0lax">$Global:FirstName $Global:LastName</td>
      </tr>
      <tr>
        <td class="tg-0lax">Username</td>
        <td class="tg-0lax">$Global:UserName</td>
      </tr>
      <tr>
        <td class="tg-0lax">Citrix Password</td>
        <td class="tg-0lax">$CitrixPassword</td>
      </tr>
      <tr>
"@
}
else
{
    $MailCreds = $null
    $MailCreds = @"
      <tr>
        <th class="tg-ltad">Credential</th>
        <th class="tg-0lax">Value</th>
      </tr>
      <tr>
        <td class="tg-0lax">Full Name</td>
        <td class="tg-0lax">$Global:FirstName $Global:LastName</td>
      </tr>
      <tr>
        <td class="tg-0lax">Username</td>
        <td class="tg-0lax">$Global:UserName</td>
      </tr>
      <tr>
        <td class="tg-0lax">Citrix Password</td>
        <td class="tg-0lax">$CitrixPassword</td>
      </tr>
    
"@
}


$Global:MailCreds = $Global:MailCreds + $MailCreds
}


#### Send Email ###
Function SendMail
{
$MailUser = $env:UserName
$MailTo = Get-ADUser -Filter {SamAccountName -eq $MailUser} -Properties EmailAddress | select EmailAddress
$MailTo = $MailTo.emailaddress
$MailFrom = 'it@oneworldomaha.org'
$SMTPServer = 'mail.oneworldomaha.local'
$MailSubject = 'New User Setup'

$HTMLStyle = @"
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;border-color:#999;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:bold;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4;}
.tg .tg-m7ex{font-weight:bold;font-size:15px;text-align:left;vertical-align:top}
.tg .tg-cbs6{font-size:15px;text-align:left;vertical-align:top}
</style>
<table class="tg" style="undefined;table-layout: fixed; width: 400px">
<colgroup>
<col style="width: 200px">
<col style="width: 200px">
</colgroup>
"@

$MailBody = $HTMLStyle + $Global:MailCreds + '</table>'

Send-MailMessage -To $MailTo -from $MailFrom -subject $MailSubject -BodyAsHtml $MailBody –Smtp $SMTPServer

Add-Content $LogPath "Email sent. To: $MailTo From: $MailFrom at $(Get-Date)"
}


### Create user ###
Function CreateUser
{
$DefaultPassword = 'OneWorld1'
$Path = 'OU=MemberOrgs,OU=HeartlandUsersandGroups' #need to change this to OneWorld
$DomaninDN = (Get-ADDomain).DistinguishedName
$DefaultGroup = 'Domain Users'
$HomeDrive = 'U:'
$HomeDirectory = '\\hn-fsclusterfs\users\' #oenworld stuff as well. 
$RemoteUserGroup = 'RemoteAccess_'
$Global:FirstName = $null
$Global:LastName = $null
$Global:Username = $null
$FirstName = $null
$LastName = $null
$Username = $null
$FirstNameCharCount = 1
$Location = $null
$LongLocation = $null
$Terminate = $null
$LogPath = '\\HN-FSCLUSTERFS\Public\IT\Scripts\UserManagement\New User Toolkit\Log\NewUser.log' #oneworld stuff here. 

#Begin user input - Draw Username popup
do
{
    $Valid = $false

    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    $form = New-Object System.Windows.Forms.Form
    $form.Text = 'Enter First and Last Name'
    $form.Size = New-Object System.Drawing.Size(300,200)
    $form.StartPosition = 'CenterScreen'

    $OKButton = New-Object System.Windows.Forms.Button
    $OKButton.Location = New-Object System.Drawing.Point(75,120)
    $OKButton.Size = New-Object System.Drawing.Size(75,23)
    $OKButton.Text = 'OK'
    $OKButton.DialogResult = [System.Windows.Forms.DialogResult]::OK
    $form.AcceptButton = $OKButton
    $form.Controls.Add($OKButton)

    $CancelButton = New-Object System.Windows.Forms.Button
    $CancelButton.Location = New-Object System.Drawing.Point(150,120)
    $CancelButton.Size = New-Object System.Drawing.Size(75,23)
    $CancelButton.Text = 'Cancel'
    $CancelButton.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
    $form.CancelButton = $CancelButton
    $form.Controls.Add($CancelButton)

    $label = New-Object System.Windows.Forms.Label
    $label.Location = New-Object System.Drawing.Point(10,20)
    $label.Size = New-Object System.Drawing.Size(280,20)
    $label.Text = 'Please enter users FIRST NAME:'
    $form.Controls.Add($label)

    $FirstBox = New-Object System.Windows.Forms.TextBox
    $FirstBox.Location = New-Object System.Drawing.Point(10,40)
    $FirstBox.Size = New-Object System.Drawing.Size(260,20)
    $form.Controls.Add($FirstBox)

    $label = New-Object System.Windows.Forms.Label
    $label.Location = New-Object System.Drawing.Point(10,70)
    $label.Size = New-Object System.Drawing.Size(280,20)
    $label.Text = 'Please enter users LAST NAME:'
    $form.Controls.Add($label)

    $LastBox = New-Object System.Windows.Forms.TextBox
    $LastBox.Location = New-Object System.Drawing.Point(10,90)
    $LastBox.Size = New-Object System.Drawing.Size(260,20)
    $form.Controls.Add($LastBox)

    $form.Topmost = $true

    $form.Add_Shown({$FirstBox.Select()})
    $result = $form.ShowDialog()

    if ($result -eq [System.Windows.Forms.DialogResult]::OK)
    {
        $Global:FirstName = $FirstBox.Text
        $Global:LastName = $LastBox.Text
        if ($Global:FirstName.Length -gt '0' -and $Global:LastName -gt '0' -and $Global:FirstName.Length -lt '20'-and $Global:LastName.Length -lt '20')
        {
            $Valid = $true
        }
        $Global:FirstName
        $Global:LastName
    }
    else
    {
        if ($Global:MailCreds -ne $null)
        {
            SendMail
        }
        Return
    }
}
While (!$Valid)







### Define Username ###
$Global:Username = "$($Global:FirstName.SubString(0,$FirstNameCharCount))$Global:LastName"

#Confirm Username is 6 characters long and not used
if ($Global:Username.Length -lt 6)
{
    do
    {
        ++$FirstNameCharCount
        $Global:Username = "$($Global:FirstName.SubString(0,$FirstNameCharCount))$Global:LastName"
    }
    While ($Global:Username.Length -lt 6 -and $FirstNameCharCount -le $Global:FirstName.Length)
}

#Check if user is already existing
try
{

if (Get-AdUser $Global:Username)
{
    do
    {
        #Get existing user info
        $ActiveUser = Get-ADUser $Global:Username
        $ActiveName = $ActiveUser.Name
        $ActiveStatus = $ActiveUser.Enabled
        $ActiveGroups = Get-ADPrincipalGroupMembership $Global:Username | select name
        $ActiveGroups = $ActiveGroups.name -join ", "

        #If username is existing, confirm the user is not the same.
        $form = New-Object System.Windows.Forms.Form
        $form.Text = 'Is this the same user?'
        $Form.AutoSize = $True
        $Form.AutoSizeMode = "GrowAndShrink"
        $form.StartPosition = 'CenterScreen'

        $OKButton = New-Object System.Windows.Forms.Button
        $OKButton.Location = New-Object System.Drawing.Point(150,160)
        $OKButton.Size = New-Object System.Drawing.Size(75,23)
        $OKButton.Text = 'Yes'
        $OKButton.DialogResult = [System.Windows.Forms.DialogResult]::Yes
        $form.AcceptButton = $OKButton
        $form.Controls.Add($OKButton)

        $CancelButton = New-Object System.Windows.Forms.Button
        $CancelButton.Location = New-Object System.Drawing.Point(225,160)
        $CancelButton.Size = New-Object System.Drawing.Size(75,23)
        $CancelButton.Text = 'No'
        $CancelButton.DialogResult = [System.Windows.Forms.DialogResult]::No
        $form.CancelButton = $CancelButton
        $form.Controls.Add($CancelButton)

        $label = New-Object System.Windows.Forms.Label
        $label.Location = New-Object System.Drawing.Point(10,20)
        $label.Size = New-Object System.Drawing.Size(200,12)
        $label.Text = 'Is this the same user?'
        $form.Controls.Add($label)

        $nameLable = New-Object System.Windows.Forms.Label
        $nameLable.Location = New-Object System.Drawing.Point(40,40)
        $nameLable.Size = New-Object System.Drawing.Size(200,12)
        $nameLable.Text = "Name: $ActiveName"
        $form.Controls.Add($nameLable)

        $EnableLable = New-Object System.Windows.Forms.Label
        $EnableLable.Location = New-Object System.Drawing.Point(40,60)
        $EnableLable.Size = New-Object System.Drawing.Size(200,12)
        $EnableLable.Text = "Enabled: $ActiveStatus"
        $form.Controls.Add($EnableLable)

        $GroupsLable = New-Object System.Windows.Forms.Label
        $GroupsLable.Location = New-Object System.Drawing.Point(40,80)
        $GroupsLable.Size = New-Object System.Drawing.Size(400,80)
        $GroupsLable.Text = "Groups: $ActiveGroups"
        $form.Controls.Add($GroupsLable)

        $result = $form.ShowDialog()

        #Exit if operator confirms user exists
        if ($result -eq [System.Windows.Forms.DialogResult]::Yes)
        {
            Write-Host -ForegroundColor Red "Unable to create user with First:$Global:FirstName Last:$Global:LastName. Please check Active Directoy to re-use existing account."
            Add-Content $LogPath "Unable to create user with First:$Global:FirstName Last:$Global:LastName. Please check Active Directoy to re-use existing account."
            if ($Global:MailCreds -ne $null)
            {
                SendMail
            } 
            Exit
        }
        #Else find another username
        else
        {
            ++$FirstNameCharCount
            $Global:Username = "$($Global:FirstName.SubString(0,$FirstNameCharCount))$Global:LastName"
            #Test if no usable username could be found
            if ( $FirstNameCharCount -gt $Global:FirstName.Length )
            {
                write-host "`n"
                Write-Host -ForegroundColor Red "Unable to use $Global:FirstName $Global:LastName. Please create account manually"
                write-host "`n"
                Add-Content $LogPath "Unable to use $Global:FirstName $Global:LastName. Please create account manually"
                if ($Global:MailCreds -ne $null)
                {
                    SendMail
                }       
                Exit
            }
        }
    }
    while (Get-AdUser $Global:Username)
}

}
Catch {}

$ErrorActionPreference = $EaPrefBefore

#Create user account
$NewUserParams = @{
    'DisplayName' = "$Global:FirstName $Global:LastName"
    'UserPrincipalName' = $Global:Username
    'Name' = "$Global:FirstName $Global:LastName"
    'GivenName' = $Global:FirstName
    'Surname' = $Global:LastName
    'SamAccountName' = $Global:Username
    'AccountPassword' = (ConvertTo-SecureString $DefaultPassword -AsPlainText -Force)
    'Enabled' = $true
    'Path' = "OU=$LongLocation,$Path,$DomaninDN"
    'ChangePasswordAtLogon' = $true
    'HomeDrive' = $HomeDrive
    'HomeDirectory' = $HomeDirectory+$Global:Username
    'Description' = $LongLocation+' Employee'
}

New-ADUser @NewUserParams

if (Get-AdUser $Global:Username)
{
    write-host "`n"
    Write-Host -ForegroundColor Green "$Global:Username added as a new user to $LongLocation"
    write-host "`n"
    Add-Content $LogPath "Fullname:$Global:FirstName $Global:LastName Username:$Global:Username created on $(Get-Date)"
}
else
{
    write-host "`n"
    Write-Host -ForegroundColor Red "Unable to use $Global:Username. Please create account manually"
    write-host "`n"
    return
}

#Set Remote Desktop Profile Path and Drive
$OU = [ADSI]"LDAP://OU=$LongLocation,$Path,$DomaninDN"
$TSuser = $OU.psbase.get_children().find("CN=$Global:FirstName $Global:LastName")
$TSuser.psbase.invokeSet("TerminalServicesHomeDirectory","$HomeDirectory$Global:Username")
$TSuser.psbase.invokeSet("TerminalServicesHomeDrive",$HomeDrive)
$TSuser.setinfo()

#Set proper names for groups
if ($Location -eq 'MT')
{
    $Location = 'NF'
}
elseif ($Location -eq 'AC')
{
    $Location = 'CB'
}
elseif ($Location -eq 'FHN')
{
    $Location = 'Frontera'
}
elseif ($Location -eq 'SEIA')
{
    $Location = 'SE_IA'
    $LongLocation = 'Community Health Center of Southeast Iowa'
}

#Users added to Domain Users by default
Add-Content $LogPath "Username:$Global:Username added to 'Domain Users' group"

#Add user to Practice Group
$AddGroup = $null
try
{
    $AddGroup = Add-ADGroupMember -Identity "$LongLocation" -Members $Global:Username
}
catch
{
    $AddGroup = $_
}

if ($AddGroup -eq $null)
{
    Add-Content $LogPath "Username:$Global:Username added to '$LongLocation' group"   
}
else
{
    Write-Host -ForegroundColor red "An error occured adding $Global:Username to '$LongLocation' group"
    Add-Content $LogPath "An error occured adding $Global:Username to '$LongLocation' group"
    Add-Content $LogPath $AddGroup
}

#Add user to Remote Users group
$AddGroup = $null
try
{
    $AddGroup = Add-ADGroupMember -Identity "$RemoteUserGroup$Location" -Members $Global:Username
}
catch
{
    $AddGroup = $_
}

if ($AddGroup -eq $null)
{
    Add-Content $LogPath "Username:$Global:Username added to '$RemoteUserGroup$Location' group"  
}
else
{
    Write-Host -ForegroundColor red "An error occured adding $Global:Username to '$RemoteUserGroup$Location' group"
    Add-Content $LogPath "An error occured adding $Global:Username to '$RemoteUserGroup$Location' group"
    Add-Content $LogPath $AddGroup
}

#Add user to Citrix Users group
$AddGroup = $null
try
{
    $AddGroup = Add-ADGroupMember -Identity "Citrix Users" -Members $Global:Username
}
catch
{
    $AddGroup = $_
}

if ($AddGroup -eq $null)
{
    Add-Content $LogPath "Username:$Global:Username added to 'Citrix Users' group"
}
else
{
    Write-Host -ForegroundColor red "An error occured adding $Global:Username to 'Citrix Users' group"
    Add-Content $LogPath "An error occured adding $Global:Username to 'Citrix Users' group"
    Add-Content $LogPath $AddGroup
}


#Run GenerateEmail function
$Global:Location = $Location
GenerateEmail

#Run ConfirmExit Function
ConfirmExit

}


### Confirm Exit ###
function ConfirmExit
{
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing
$result = $null

$form = New-Object System.Windows.Forms.Form
$form.Text = 'Create another user?'
$form.Size = New-Object System.Drawing.Size(300,200)
$form.StartPosition = 'CenterScreen'

$OKButton = New-Object System.Windows.Forms.Button
$OKButton.Location = New-Object System.Drawing.Point(75,120)
$OKButton.Size = New-Object System.Drawing.Size(75,23)
$OKButton.Text = 'OK'
$OKButton.DialogResult = [System.Windows.Forms.DialogResult]::OK
$form.AcceptButton = $OKButton
$form.Controls.Add($OKButton)

$CancelButton = New-Object System.Windows.Forms.Button
$CancelButton.Location = New-Object System.Drawing.Point(150,120)
$CancelButton.Size = New-Object System.Drawing.Size(75,23)
$CancelButton.Text = 'Cancel'
$CancelButton.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
$form.CancelButton = $CancelButton
$form.Controls.Add($CancelButton)

$label = New-Object System.Windows.Forms.Label
$label.Location = New-Object System.Drawing.Point(10,20)
$label.Size = New-Object System.Drawing.Size(280,20)
$label.Text = 'Would you like to create another user?'
$form.Controls.Add($label)

$result = $form.ShowDialog()

if ($result -eq [System.Windows.Forms.DialogResult]::OK)
{
    CreateUser
}
else
{
    SendMail
    Return
}
}


### Start Function ###

CreateUser
