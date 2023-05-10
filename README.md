# Report LDAP User Accounts

This plugin can match user accounts in Moodle with an LDAP directory in the organisation.
There is a new reports page that takes users from Moodle, checks whether these
exist in the LDAP directory and displays them in a table. Various filters can be used
to select users. The report may also be exported.

Another functionality of the plugin is that it provides a cli script. This has the same
functionality like the reports page, but can also be used to automatically suspend, delete
or set the emailstop flag for users. The idea is that the process can be automated to
update the users in Moodle e.g. at the start of a new semester.

## Motivation
 
The use case for the need of this plugin was that the Moodle authentication is
done via [Sibboleth](https://en.wikipedia.org/wiki/Shibboleth_(software)) (an SSO service
that is not part of the institution). If the user is authenticated via the SSO provider
and the identity service of the institution, the user is redirected to Moodle together
with some information who he is. He then is logged in or even created. LDAP may work here
as the identity provider for Shibboleth but there is no direct connection from/to Moodle.

Whenever a user drops out of the institution the identity provider will not allow to
log him in anymore. However, the account in Moodle is still active and emails may be
send out to the users email address.

Therefore, there is a need to detect such users and disable or delete them from Moodle
once they do not yet exist in the LDAP anymore.

## Installation

Unpack the zip file and copy it's content into a subdirectory of
your Moodle installation below the `report` directory.

Go to the settings page
"Site administration" -> "Plugins" -> "Report" -> "Settings for Moodle Accounts in LDAP Report"
and add the connection data to your LDAP server.

## User identification between Moodle and LDAP

The user match between Moodle and LDAP is done via email solemnly. At the moment there
is no other property considered to be taken as the user identity. The user data
is taken from the user table in Moodle. The matching email field in LDAP can be configured
in the settings of the plugin.

When querying the LDAP directory it's only checked whether there is a match or not.
The email field is the only field that is returned by the LDAP query together with the
`dn` field (the user id). However, the latter is not used by the plugin.

## Settings

At "Site administration" -> "Plugins" -> "Report" -> "Settings for Moodle Accounts in LDAP Report"
you find all settings regarding this plugin.

There is a list of fields with the LDAP prefix that handle the connection to the LDAP
server. You may also provide certificate files for the connection in case necessary.

The setting "Email field in LDAP" may contain a different field name where the email
is stored in the LDAP directory. By default this is `mail`.

The setting "LDAP Query" may contain some additional LDAP query parameters to select
an user. Imagine the use case that the LDAP directory contains person and institution
items, both having an email field. In this setting the query might contain additional
conditions e.g. `(objectClass=person)` to select the person items only.

## Privacy

In general the LDAP report just fetches data from the LDAP server and lists data
in the report that are already in the Moodle database. However, there are two
exceptions.

- If the user selects to download the report as a CSV file, the data gets stored in
  a file that is located in the Moodle data directory. The file remains there until
  it's removed from an external job or manually by the Admin (see below).
- For debug reasons logging can be enabled. This logs the entire communication between
  Moodle and the LDAP server. The logs are stored in the database of Moodle and in
  case of the LDAP response may contain personal information from the LDAP entries that
  are not yet in Moodle and also will not be processed any further. By default, the
  reports page fetches the email field only. The response contains the `mail` field
  and the `dn` field only. On the command line the query may look more complex and
  may contain additional data that would be in this case in the log as well.

## Report page

The report page can be called using the path `/report/ldapaccounts/`. You also get to
that page via "Site administration" -> "Reports" -> "Moodle Accounts in LDAP".

### Filter

In the form the first section "Filter user data" contains filters to select which
users to fetch from the Moodle database that should appear in the report.
If you have more than one authentication method you will see a selection to filter
by authentication method. The selections "Deleted", "Suspended", "Emailstop", and
"LDAP status" work all the same, 0 means option disabled and 1 means option enabled.
By default, any of the options can be set i.e. the filter is not applied to the search.
The Filters firstname and lastname can be used to select users starting with these
letters. No wildcard has to be used.

The email field may use a wildcard. Here you can use a concrete email or e.g. select
all emails from a certain domain only. Wildcard character is the asterisk.

While all filters work directly with the data from the user table in Moodle the
"LDAP status" field is used on the results after the user items have been queried at
the LDAP directory. If the filter is set, only the users are displayed that match the
criteria (i.e. exist in the LDAP or not).

### Display user data

The field "columns" can contain any field name that exists in the user table. If
the `id` field is not selected, it will be put automatically as the first place. The
"LDAP status" field cannot be selected here and will be inserted right after
the `id` field. Also, at the end the fields "User details", "Delete", and "Disable
notifications" are added automatically. These contain links to the users profile
page, to easily be able to delete, suspend or disable notifications for a user.
The `password` and the `secret` fields cannot be displayed. Custom defined user
fields are also not considered.

Each field name must be written on a separate line in the text area.

### CSV Download

The report can also be downloaded as a csv file, in case the checkbox is set in the
form. A download link appears below the report table. The files are stored in a
subdirectory `report_ldapaccounts` of the Moodle data directory. Old files are not
deleted at the moment, therefore, a cronjob should be installed to do this.

### Permalink

Below the user table a permalink is displayed. This link reflects the current query.
Used in another time it queries the data by the same criteria. This link can be
bookmarked so that the user doesn't need to remember each setting in the reports page
form. When calling the link, the form is displayed and preset with the search terms
already.

## CLI

The cli script resides in `report/ldapaccounts/cli/ldapaccounts.php`. Administrators
may run it from the command line occasionally or put it into a cronjob to run it
periodically.

The output is always a csv formatted string witten to standard out. You may write
that into a file using the io redirection arguments.

For a list of commands and possible arguments use
the `--help` switch to display it.

### Mode

There are two modes of the script. The first mode works the same as the report page.
You select user data. The resulting records are print to standard out.

The other option is to use the `--action` parameter to define an action with users
that do not exist in LDAP. These can be `delete`, `suspend`, or `emailstop`.
To suspend user accounts that are not in LDAP you would use `--action=suspend` to set
the `suspended` property to 1 in the user table.

Also, when using the `--action` parameter the output are the data sets that have been
modified (actually they will be modified within an update command before the next
chunk of user records is fetched). If a user record should be modified, but may have
that flag set already that is defined by the action, then this data set is not printed
out.

### Filters

Filters are more flexible than in the reports page. You can basically use any
column that exists in the user table to define some criterion and filter data with
this condition. Each filter criterion is used with an and conjunction. If a filter
is set on the `firstname` and on the `lastname` both criteria must match so that the
user record is selected.

If you use any wildcard on strings the asterisk must be set. Also if you use > or <
then these operators must be prepended on the value.

Filters must be written as json. To filter all users whose surname starts with W
and that are not yet suspended and that have an email in the example.org institution
would use the following filter (including the argument switch):

```
-f='{"lastname":"w*","suspended":0,"email":"*example.org"}'
```

### Other features

In case you use a different ldapmail field or add a query part to the ldap query
these options can be submitted by the arguments `--ldapmail` and `--ldapquery`.
These two values can also be defined in the settings and be used always for all
queries in the cli script and the reports page.

To have no output witten to standard out, the switch `--silent` might be used. This
apparently makes sense only if you also use the `--action` switch.

## Limitations and possible future features

Originally the plugin was tailored with a lot of hard coding facts for our personal
use. We tried to make it more generic, so that it can be used by other institutions
as well. At the moment the functionality of the plugin meets our requirements.

Possible future changes could be:

- More flexibility in matching users in Moodle and LDAP (the username field could be
  used for that, too).
- Logging of requests and results in the communication with the LDAP server.
- Use output options of Moodle and not the hard coded csv export.
- Fetch and display other values from the LDAP server, not only the email field to
  match the users.
- Use some ajax so that the queries may work in the background and the table is
  displayed after the data has been populated. For our institution we can query
  more that 30k records within less than a minute, therefore we use the direct way
  to display the data.
- A command that deletes old CSV files that have been generated before a certain time.