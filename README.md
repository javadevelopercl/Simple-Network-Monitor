# Simple-Network-Monitor
## Overview

This is a basic script I wrote some time ago as an illustration of how to get started writing a network monitor. It is probably a little out of date now but hopefully still runs on all recent flavors of Ubuntu or Debian (and probably other distributions as well). It will require **nmap**, **SQLite** and **Pxssh**. You should have admin privileges on the target network. We'll divide the app into 4 smaller pieces:


 * A monitoring daemon to run on a server on the network (*monitor.py*)
 * A configuration file with a list of hosts and services to monitor
 * A SQLite database for logging results (*monitor.db*)
 * A shell script to view monitoring reports (*report.sh*)


### monitor.py

First we'll create the monitoring daemon and have it run in the backgroind on our server. To keep it simple I'll just report system uptime and free memory on each host. We can always add other reporting functions later. I'll also use Pxssh from the [Pexpect](http://pexpect.readthedocs.io/en/stable/index.html#) library to handle ssh logins.



```python
import sys, os , pxssh, re , commands, sqlite3, time
 host_file = 'hosts.txt'
 error_file = '/tmp/error.log'
 dbase  = "monitor.db"
 dbtable = "monitor"
 
 def initialize_database():
   #if database exists return
   if os.path.isfile(dbase):
     conn = sqlite3.connect(dbase,timeout=20)        
     return conn
   # else create table and trigger to insert timestamps with every entry    
   conn = sqlite3.connect(dbase)
   c = conn.cursor()     
   c.execute("create table monitor ( mydata TEXT, timestamp DATETIME)")
   c.execute("CREATE TRIGGER insert_time AFTER INSERT ON monitor BEGIN UPDATE monitor SET timestamp =    
      DATETIME('NOW') WHERE rowid = new.rowid;  END;")
   c.execute("CREATE TRIGGER del_old_recs BEFORE INSERT ON monitor BEGIN DELETE FROM monitor WHERE timestamp <
      DATETIME('now','-1month'); END;")    
   return conn
 
 def dox(s, sendln, pattern):    
   s.sendline(sendln)
   if s.prompt():        
     return re.search(pattern, s.before).group(1)    
   return 'NA'
 
 def monitor(host, ports, dbase, errfile, datestamp ):
   cursor = dbase.cursor()    
   flags= '-T4 -oG /dev/stdout' 
   nmap_res = commands.getoutput('nmap -p %s %s %s' % (",".join(ports), flags, host))    
   host_res =     
   try:
     cursor.execute("insert into monitor (mydata) values ('%s')" % (nmap_res,) )
     dbase.commit()
     s= pxssh.pxssh()
     if s.login(host, 'admin', 'passwd',login_timeout=6) == True:            
      uptime = dox(s,'uptime', r'(\d{1,2}:\d{2}:?\d{0,2})')
      memfree = dox(s,'cat /proc/meminfo', r'MemFree:\s+(\d+)') 
      ipaddr = commands.getoutput('host %s | egrep -o "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+"' % host)
      host_res = "Ihost: %s  Uptime: %s FreeMem: %s  %s" % (ipaddr, uptime, memfree, datestamp)
      cursor.execute("insert into monitor (mydata) values (\"%s\")" % (host_res,) )
      dbase.commit()                
      s.logout()
      s.close()                    
   except Exception, e:
     errfile.write(" %s: %s %s\n" % (host, unicode(e), datestamp) )             
 
 def main():    
    datestamp = commands.getoutput('date')    
    errfile = open(error_file, 'w')
    dbase_obj = initialize_database()    
    while True:
     for line in open(host_file):
      all = line.strip('\n').split(',')
 
      if os.fork() == 0:    #should use threadpool here                                            
       monitor(all[0], all[1:] ,dbase_obj, errfile, datestamp)                         
       dbase_obj.close()
       errfile.close()
       os._exit(0)
     time.sleep(86400)
 
 main()
```

You can have this run in the background as a daemon. Run the following from the command line or better yet put it as a line in /etc/rc.d/rc.local file to run at startup: ` (path/to/monitor.py </dev/null >/dev/null 2>/dev/null &)`


### The configuration file:

We'll just use a basic text file called *hosts.txt* with comma delimited entries for hosts (hostnames, ip address or address range) and ports (by common name or port number) to monitor, like so:

```
bob.mynetwork.com,22,23,80
testserver.testing.com,22,telnet,25,80
192.168.1.1,smtp,22,http
192.168.1.0/24,431,http,ssh
```
### monitor.db
And now for the SQLite database. Ours will have a very simple structure- just a simple table with 2 fields to store the text results from each session with a timestamp. I also added 2 insert [triggers](https://www.google.com/#q=database+trigger), one to add the timestamp with each insert and another to delete records older than a certain time period to keep the database manageable (very inefficient way to do this). The `initialize_database()` function in *monitor.py* above creates the database.

### Searching and reporting: report.sh
Finally we'd like to read the output in the databases. For now I'm going to keep it simple with some basic text tables accessible from the bash shell. I'd like to view the results by portnumber/service and also view all the results since a certain time period. One neat trick we can use is to enable us pipe the unix date command like this,

```
date --date="3 days ago"| report telnet http ssh //alpha.mydomain.com//
```
for example, to view the statuses of ports 23,2, and 80 on the host machine called alpha. The **date** command allows finer grained results for e.g,
`date -s="01/01/2010 12:15:00"`.
This may not be a fancy gooey/gui way to view results but it enables us to write fancy command line scripts to, for example, collect all the results between 2 time periods or batches of time periods. Or to write scripts to display it in whatever fancy format you want. The following Bash script mostly just makes translates **date** commands into an appropriate format for SQLite:

```bash
#!/usr/bin/env bash
  dbase="monitor.db"
  function convert_to_sql_date(){
    mnth=${DATE[1]}
    case $mnth in
     "Jan") mm="01" ;;
     "Feb") mm="02" ;;
     "Mar") mm="03" ;;
     "Apr") mm="04" ;;
     "May") mm="05" ;;
     "Jun") mm="06" ;;
     "Jul") mm="07" ;;
     "Aug") mm="08" ;;
     "Sep") mm="09" ;;
     "Oct") mm="10" ;;
     "Nov") mm="11" ;;
     "Dec") mm="12" ;;
    esac
    yr=${DATE[5]}    
    dd=${DATE[2]}; 
    if [ $dd -le 10 ]; then dd="0${dd}" ; fi    
    hms=${DATE[3]}
    echo "${yr}-${mm}-${dd} ${hms}"
  }

  function do_results(){
    uptime=`echo "$RAWDATA" | awk '/Ihost.*'"$HOST"'/ {print $4}'`
    freemem=`echo "$RAWDATA" | awk '/Ihost.*'"$HOST"'/ {print $6}'`
    uptime="${uptime:--}"  
    freemem="${freemem:--}"
    printf "%-20s  %-15s  %-10s  " $HOST $uptime $freemem
    for port in ${args[@]}; do
      if [ $port -eq $port 2>/dev/null ];then
        open="${port}/open"
        close="${port}/close"
      else
       open="/open/[a-zA-Z]*//$port"
       close="/open/[a-zA-Z]*//$port"
      fi
      line=`echo "$RAWDATA" | grep -v "^[[:space:]]*#" | grep "$HOST"`
      echo $line | egrep -o "$open" >/dev/null
      if [ $? -eq 0 ];then
        printf "%-8s  " "YES"
      else
        echo $line | egrep "$port" >/dev/null
        if [ $? -eq 0 ]; then 
          printf "%-8s  " "NO"
        else
          printf "%-8s  " "NA"
        fi
      fi
    done
    printf "\n" 
  }
  #Check for cmd line args
  if [ ! $1 ];then
    echo "Usage: $0 port1 [port2] [port3] ..."
    echo "     : date [options] | $0 port1 [port2] [port3] ..."
    exit 0
  fi
  #Check if date command was piped in
  if ! [ -t 0 ]; then
    dt=`cat /dev/stdin`
    DATE=( $dt )
    sqldate="$(convert_to_sql_date)"
    RAWDATA=$(sqlite3 $dbase "select * from monitor where timestamp >= datetime('$sqldate');")
  else
    RAWDATA="$(sqlite3 $dbase 'select * from monitor;')"        
  fi
  printf "%-20s  %-15s  %-10s  " "HostIP" "Uptime" "Free Mem"
  for port in $*;do
    printf "%-8s  "  $port
  done
  printf "\n"
  args=( $* )
  hosts=`echo "$RAWDATA" |awk '/Host:/ {print $2}'`
  for HOST in $hosts; do
    do_results
  done

```

Check out the full source files for more detail.
