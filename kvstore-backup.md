# About:
- First written Q2 2024 by James Wintermute (jameswintermute@protonmail.ch)
- Used effectively on multiple Search Head clusters within client environments
- Please credit me accordingly if you use this script.

# Assumptions and Dependencies
- Splunk Enterprise =>9.x
- On-Premise only
- Linux OS only

# Procedure
- Process for backing up the KV store ahead of significant pieces of work
- This is an automated scripting process
- Also see: [Splunk KVstore endpoint](https://docs.splunk.com/Documentation/Splunk/latest/RESTREF/RESTkvstore#kvstore.2Fbackup.2Fcreate])

# Issue the KVstore search via CLI
- This outputs the list to a file called 'kvlist-raw.txt'
<pre>
sudo -H -u splunk /opt/splunk/bin/splunk search '| rest /servicesNS/-/-/data/transforms/lookups splunk_server=local | search type=kvstore  | fields eai:appName, title, collection, id  | rename eaiappName as app  | search NOT app IN (Splunk_Security_Essentials, python_upgrade_readiness_app)  | eval list =(app+","+collection)  | fields list' > ~/kvlist-raw.txt
</pre>

# Create a Netrc password file
<pre>
vi ~/.netrc
 machine localhost
 login &lt;username&gt;
 password &lt;password&gt;

 # Save and exit
 chmod 400 ~/.netrc
</pre>

# Clean the list:
<pre>
 cat kvlist-raw.txt | egrep '\w+,\w+' | sort | uniq > kvlist-clean.txt
 rm kvlist-raw.txt
</pre>

 # Create a file called 'kvstorebackup'
 <pre>
 touch kvstorebackup 0700
 vi kvstorebackup

 #!/bin/bash
 # Version 1.2 James Wintermute, 2024

 filename='Splunk-x.x.x-upgrade-'$(date +%F)
 splunkhome='/opt/splunk/bin'
 total=$(cat kvlist-clean.txt | wc -l) 

 echo "# Splunk KVstore backup script, there are a $total of KV's to backup:" > backup

 while IFS="," read -r value1 value2 remainder
 do
     echo "curl --retry 20 --retry-delay 30 --retry-max-time 600 -n -k -sS -X POST HTTPS://localhost:8089/services/kvstore/backup/create -d archiveName=$filename-$value1-$value2&appName=$value1&collectionName=$value2'" >> backup
     echo 'sudo -H -u splunk '$splunkhome'/splunk show kvstore-status | grep backupRestoreStatus | grep backupRestoreStatus | cut -d ":" -f 2 | cut -d " " -f 2' >> backup
     echo 'sleep 5s' >> backup
 done

 chmod 0700 backup
 </pre>

  # Generate the backup file:
  <pre>
  ./kvstorebackup < kvlist-clean.txt
  </pre>
  
  # Check the backup file:
  <pre>
  cat backup
  </pre>
  
# Check the KVstore status:
<pre>
 sudo -H -u splunk /opt/splunk/bin/splunk show kvstore-status | grep backupRestoreStatus
</pre>

# execute the backup script
./backup

# Check that the backups are in place:
<pre>
 sudo -iu splunk
 ll /opt/splunk/var/lib/splunk/kvstorebackup
</pre>

