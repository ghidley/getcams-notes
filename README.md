# getcams-notes
Notes on getcams usage and containerization

Started 2018-09-18

Server requirements (has been tested and operates under CentOS 7):

Need routing to access # 172.16.0.0/16         HPWREN-local private network

Need /etc/hosts file with camera base names  and IP addresses

Need user and group hpwren:
	ghidley@c1 ~$ grep hpwren /etc/passwd /etc/group*
	/etc/passwd:hpwren:x:30001:30001:HPWREN role account:/home/hpwren:/bin/bash
	/etc/group:hpwren:x:30001:hwb,ghidley,geoff,davis,jmeyer,lirichards,abrust

Need the following software packages:
	perl (with , Log::Log4perl and File::Copy qw(copy) libs)
	python, python-swiftclient
	swift, s3cmd
	ppmlabel, ppmarith, convert, pnmscale (from ImageMagick/netpbm ports)

Storage areas used by getcams system:
	ARCHDIR=/Data/archive   (for long term archival)
	CDIR=$(ARCHDIR)/incoming/cameras   (for last fetched images)
	DATADIR=/Data
	INCOMING=$(ARCHDIR)/incoming/cameras/tmp
	RUNDIR=~hpwren/bin/getcams
	LOCALDIR=/Data-local/scratch
	SYSLOCAL=/var/local/hpwren
	LOCKDIR=$(SYSLOCAL)/lock
	LOGDIR=$(SYSLOCAL)/log

Options for starting getcams system (from Readme file):
	Testing start (-I) -- one of the following:
	    sudo -u hpwren ~hpwren/bin/getcams/run_cameras -I
	    make start
	    systemctl start getcams
	
	Testing restart (-R) -- one of the following:
	    sudo -u hpwren ~hpwren/bin/getcams/run_cameras -R
	    make restart
	    systemctl restart getcams
	
	Testing stop (-X) -- one of the following:
	    sudo -u hpwren ~hpwren/bin/getcams/run_cameras -X
	    make stop
	    systemctl stop getcams

Overview of getcams system:

getcams system consists of the following executables and control files:
	run_cameras: daemon that starts and maintains running system, reading cam_params control file and starting getcams-*.pl as needed
	getcams-*.pl: camera drivers, one each for Iqueye (getcams-iqeye.pl) , Mobotix (getcams-mobo.pl) and Axis cameras (getcams-axis.pl)
	cam_params: camera control file for indicating camera fetch parameters
	cam_access: camera login and password information (not in git repository)


(from run_cameras):
 Will be managed by systemctl start/stop/restart when in production mode:
       Start=/home/hpwren/bin/run_cameras -I
       ExecStop=/home/hpwren/bin/run_cameras -X

 Will be invoked at boot once "systemctl enable getcams" is run, from:
       /etc/systemd/system/multi-user.target.wants/getcams.service
 Other options include -d (parent debug) and -D (parent and child debug) or a "make test"
 Service control options added: -X to stop, -R to restart, -I to start (initialy at boot)
 Can also control running process by adding KILLALL or HALT as special camname keyword to cam_params
    HALT removes the run_cameras lockfile and halts the run_cameras process (leaves getcams* running)
    KILLALL terminates all camera fetch processes (getcams*), removes all lockfiles and then halts the run_cameras process
    NOTE: must remember to comment out the above HALT or KILLALL before starting run_cameras again!

 run_cameras:
 Starts fetching images from all enabled cameras
 Reads camera parameters from cam_params:
    NAME:PROGRAM:TYPE:STARTUP_DELAY:LABEL:RUN_ONCE:CAPTURES/MINUTE
    hpwren-iqeye7:getcams-iqeye.pl:c:2:"Cal Fire Ramona AAB, http\://hpwren.ucsd.edu c1":1:1
    smer-tcs3-mobo:getcams-mobo.pl:c:3:"SDSU SMER TCS3, http\://hpwren.ucsd.edu c1":1:1
    KILLALL: Special case to terminate all running camera fetch process and run_cameras (this program)
    HALT: Special case to terminate run_cameras script (this program)

 Locks self using $lockpath/RUNCAM_PID
 Locks children using $lockpath/$getcams-???.lock
 Logs debug info to stdout
 Logs to $logpath/runcamlog: KILLALLs, HALTs, and getcam invocations
 Logs to /var/log/messages (via logger, syslog) upon signal trap or exit:
       getcams_???[pid] was terminated: -- restart with  \"run_cameras -I\" if needed"

 Overall logic:
 while true
   while read cam_params (e.g. for each camera entry ...)
     if first time through loop or if cam_params changed since last reading
       Kill any running fetches (getcams-???'s), remove their lockfiles ... and fall through to [re]Exec below
     else (in monitoring mode)
       if cam fetch process running, continue (skip) to next cam fi, ... otherwise
       Remove any leftover lockfiles and fall through to [re]exec fetch (getcams-???'s)
     fi
     [Re-]exec camera fetch
   done while read cam_params
 done

 Above while loops also manage special case cam keywords of HALT and KILLALL



