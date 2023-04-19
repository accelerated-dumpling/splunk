# Scripts

Scripts shared from jimmyatSplunk \
ref: https://gist.githubusercontent.com/jimmyatSplunk


### thp.sh
```
#!/bin/bash
## Script developed by Jimmy Maple - Splunk Professional Services

## This script was designed to allow for quick deployment of THP, ulimit
## and Splunk user accounts for Splunk infrastructure. It was developed
## using RHEL 7 as the Splunk host. There may be issues using other
## Linux OS for this and should be altered and tested if necessary
## particularly using the find command for the THP changes and any
## additional commands for user and group creation. Confirm the
## commands before proceeding. There is some flexibility when it comes
## to OS checking that could improve the script to be usable on any
## OS so share your configs.
## USE AT YOUR OWN RISK!

## This script was developed after the systemd awareness of Splunk 7.2.3 and has been testing successfully on the following OS:

## - RHEL (6 and 7)
## - CentOS 
## - Suse Linux
## - Ubuntu 18

## Consult with your customers about using systemd or initd for start-up and management. 
## Look over this link in Default Answers for help on the discussion
## https://splservices.atlassian.net/wiki/spaces/DA/pages/326008917/Advisory+Systemd+awareness+in+7.2.2+and+higher+causes+authentication+prompts+when+restarting+at+the+command+line

clear
if [ "$(whoami)" != "root" ]; then
	echo "Script must be run as root..."
	exit
fi

splunk_user() {
	echo "------------------------------------------------------------------"
	read -e -i "splunk" -p "Enter the username that will run Splunk: " RUNUSER
	if id "$RUNUSER" > /dev/null 2>&1; then
		echo "------------------------------------------------------------------"
		echo "Splunk user \"$RUNUSER\" exists. Skipping account creation..."
	else
		echo "------------------------------------------------------------------"
		echo "Splunk user \"$RUNUSER\" does not exist. Creating account..."
		useradd "$RUNUSER"
	fi
	echo "------------------------------------------------------------------"
	read -e -i "/opt" -p "Enter the path of the Splunk installation directory: " INSTALLDIR
	if [ -d "${INSTALLDIR}/splunk" ];then
    echo "------------------------------------------------------------------"
    echo "Splunk installation found..."
		if [ -f /etc/init.d/splunk ] || [ -f '/etc/systemd/system/Splunkd.service' ]; then
			echo "------------------------------------------------------------------"
			BOOT_START_FOUND=1
	    echo "Splunk boot-start configuration found..."
		fi
    CURRENT_RUNTIME_USER=$(ps -ef | grep 'splunkd pid' |head -1 | awk '{ print $1 }')
    if [ "${CURRENT_RUNTIME_USER}" != "${RUNUSER}" ]; then
      echo "------------------------------------------------------------------"
      echo "Splunk is not currently running as '${RUNUSER}'. Current runtime user is ${CURRENT_RUNTIME_USER}."
      echo "This script can change it to the intended runtime user. This will require a restart of the Splunk service."
      echo "If you choose not to change it now, ulimit settings will be implemented as the expected runtime user."
      echo "This should allow you to change the runtime user at the appropriate time."
      read -e -i "y" -p "Do you wish to change it now? [Y/N] " RESTART_PROMPT
      case $RESTART_PROMPT in
        [Yy]* )
          if [ -f /etc/init.d/splunk ]; then
            su - "$CURRENT_RUNTIME_USER" -c "$INSTALLDIR/splunk/bin/splunk stop" > /dev/null 2>&1+
						chown -R "${RUNUSER}." "${INSTALLDIR}/splunk"
            ${INSTALLDIR}/splunk/bin/splunk disable boot-start > /dev/null 2>&1
            ${INSTALLDIR}/splunk/bin/splunk enable boot-start -user "${RUNUSER}" > /dev/null 2>&1
            service splunk start
          elif [ -f '/etc/systemd/system/Splunkd.service' ]; then
            systemctl stop Splunkd
						chown -R "${RUNUSER}." "${INSTALLDIR}/splunk"
            ${INSTALLDIR}/splunk/bin/splunk disable boot-start > /dev/null 2>&1
            ${INSTALLDIR}/splunk/bin/splunk enable boot-start -systemd-managed 1 -user "${RUNUSER}" -group "${RUNUSER}" > /dev/null 2>&1
            systemctl start Splunkd
          else
            su - "$CURRENT_RUNTIME_USER" -c "$INSTALLDIR/splunk/bin/splunk stop" > /dev/null 2>&1
						chown -R "${RUNUSER}." "${INSTALLDIR}/splunk"
            ${INSTALLDIR}/splunk/bin/splunk enable boot-start -user "${RUNUSER}" > /dev/null 2>&1
            service splunk start
          fi
          ;;
        * )
          echo "------------------------------------------------------------------"
          echo "Skipping user change..."
            ;;
      esac
		fi 
	else
		SPLUNK_INSTALLED="n"
		echo "------------------------------------------------------------------"
		echo "Splunk is not currently installed in \"$INSTALLDIR\"."
		echo "The \"boot-start\" actions of this script will not be taken."
		sleep 5
	fi
}

os_detection() {
	MACH=("$(uname -m)")
	ID=("$(cat /etc/*-release | grep ^ID= | sed 's|ID=||' | sed 's|\"||g')")
	DIST=("$(cat /etc/*-release | grep ^PRETTY_NAME= | sed 's|PRETTY_NAME=||' | sed 's|\"||g')")
	echo "------------------------------------------------------------------
Operating System: $DIST $MACH"
}

disable_thp() {
	TUNED_STATUS=$(systemctl status tuned | grep Loaded | grep -cv enabled)
	if [ -n "$(pgrep -x tuned)" ] ; then
		TUNED_DISABLED=0
		if [ -f /etc/tuned/splunknothp/tuned.conf ]; then
			echo '------------------------------------------------------------------
THP has already been disabled...'
		else
			echo '------------------------------------------------------------------
Disabling THP...'
			THPPROFILE=("$(tuned-adm active | sed 's/Current active profile: //')")
			mkdir /etc/tuned/splunknothp > /dev/null 2>&1
			cat <<EOT > /etc/tuned/splunknothp/tuned.conf
[main]
include=$THPPROFILE

[vm]
transparent_hugepages=never
EOT
			tuned-adm profile splunknothp > /dev/null 2>&1
		fi
	elif [ "$TUNED_STATUS" -eq 0 ]; then
		TUNED_DISABLED=1
		echo '------------------------------------------------------------------
tuned is disabled on this host. Redirecting configuration to GRUB boot config and updating GRUB... '
		if [ "$(grep -c transparent_hugepage /etc/default/grub)" -eq 0 ]; then
			sed -i '/GRUB_CMDLINE_LINUX="/s/"$/ transparent_hugepage=never"/' /etc/default/grub
			update-grub > /dev/null 2>&1
		else
			echo '------------------------------------------------------------------
Disabling THP has already been configured in /etc/default/grub...'
		fi
		if [ ! -d "/sys/firmware/efi" ]; then
			cat /boot/grub2/grub.cfg > /boot/grub2/grub_"$(date +%d%m%Y)"_backup.cfg
			grub2-mkconfig -o /boot/grub2/grub.cfg > /dev/null 2>&1
		else
			cat /boot/efi/EFI/${ID}/grub.cfg > /boot/efi/EFI/${ID}/grub_"$(date +%d%m%Y)"_backup.cfg
			grub2-mkconfig -o /boot/efi/EFI/${ID}/grub.cfg > /dev/null 2>&1
		fi
	fi
	echo '------------------------------------------------------------------
Disabling THP directly in THP files...'
	THP=("$(find /sys/kernel/mm/ -name transparent_hugepage -xtype d | tail -n 1)")
	for SETTING in "enabled" "defrag"; do
		if test -f "$THP"/"$SETTING"; then
			echo never > "$THP"/"$SETTING"
		fi
	done
}

alter_ulimits() {
	echo '------------------------------------------------------------------'
	if [ -f /etc/security/limits.d/99-splunk-limits.conf ]; then
		echo "Ulimits have already been set..."
	else
		echo "Setting ulimits in /etc/security/limits.d/99-splunk-limits.conf..."
		cat <<EOT > /etc/security/limits.d/99-splunk-limits.conf
# Recommended ulimits set for Splunk
$RUNUSER hard core 0
$RUNUSER hard maxlogins 10
$RUNUSER soft nofile 65535
$RUNUSER hard nofile 65535
$RUNUSER soft nproc 20480
$RUNUSER hard nproc 20480
$RUNUSER soft fsize unlimited
$RUNUSER hard fsize unlimited
EOT
	fi
}

remove_outdated() {
	echo "------------------------------------------------------------------"
	echo "Removing legacy THP and ulimit configurations..."
	if [ -n "$(cat /etc/security/limits.conf | grep splunk)" ]; then
		sed -i '/Splunk/d' /etc/security/limits.conf
		sed -i '/splunk/d' /etc/security/limits.conf
	fi
	if [ -f "/etc/rc.d/rc.local" ]; then
		if [ -n "$(cat /etc/rc.d/rc.local | grep SPLUNK)" ]; then
			sed -i '/SPLUNK/,+6 d' /etc/rc.d/rc.local
		fi
	fi
}

boot_start() {
	if [ ! -f "$INSTALLDIR/splunk/ftr" ]; then
		SPLUNK_VERSION=("$(grep VERSION "$INSTALLDIR/splunk/etc/splunk.version" | sed 's|VERSION\=||' | sed 's|\.||g')")
		ESCAPED_INSTALLDIR=("$(echo "$INSTALLDIR" | sed 's|\/|\\\/|g')")
		if [ "$SPLUNK_VERSION" -ge 722 ]; then
			echo "------------------------------------------------------------------"
			read -p "Does your customer plan to take advantage of Workload Management or prefer running services with systemd? [Y/N] " WLM
				case $WLM in
					[Yy]* )
						echo "------------------------------------------------------------------"
						echo "Configuring boot-start through systemd..."
						su - "$RUNUSER" -c "$INSTALLDIR/splunk/bin/splunk stop" > /dev/null 2>&1
						"$INSTALLDIR/splunk/bin/splunk" enable boot-start -user "$RUNUSER" > /dev/null 2>&1
						;;
					* ) 
						echo "------------------------------------------------------------------"
						echo "Configuring boot-start through init.d..."
						"$INSTALLDIR/splunk/bin/splunk" enable boot-start -user "$RUNUSER" -systemd-managed 0 > /dev/null 2>&1
						sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" start --no-prompt --answer-yes|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk start --no-prompt --answer-yes\"|" /etc/init.d/splunk
						sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" stop|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk stop\"|" /etc/init.d/splunk
						sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" restart|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk restart\"|" /etc/init.d/splunk
						sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" status|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk status\"|" /etc/init.d/splunk
						systemctl daemon-reload
				esac
		else
				echo "------------------------------------------------------------------"
				echo "Configuring boot-start through init.d..."
				"$INSTALLDIR/splunk/bin/splunk" enable boot-start -user "$RUNUSER" > /dev/null 2>&1
				sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" start --no-prompt --answer-yes|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk start --no-prompt --answer-yes\"|" /etc/init.d/splunk
				sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" stop|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk stop\"|" /etc/init.d/splunk
				sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" restart|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk restart\"|" /etc/init.d/splunk
				sed -i "s|\"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk\" status|su - $RUNUSER -c \"$ESCAPED_INSTALLDIR\/splunk\/bin\/splunk status\"|" /etc/init.d/splunk
				systemctl daemon-reload
		fi
	else
		echo "------------------------------------------------------------------"
		echo "The license for Splunk has not been accepted yet. The script will skip enabling boot-start for $RUNUSER..."
		echo "To enable boot-start after accepting the license, use this command: $INSTALLDIR/splunk/bin/splunk enable boot-start -user $RUNUSER"
		echo "------------------------------------------------------------------"
	fi
}

query_splunk_restart() {
	if [ ! -f "$INSTALLDIR/splunk/ftr" ]; then
		echo '------------------------------------------------------------------'
		read -p "Do you wish to restart Splunk? [Y/N] " ANSWER
		case $ANSWER in
			[Yy]* )
				echo '------------------------------------------------------------------
Restarting Splunk...
------------------------------------------------------------------'
			if [ -f /etc/systemd/system/Splunkd.service ]; then
				systemctl restart Splunkd.service
				echo "Sleeping to allow Splunk to restart..."
				sleep 20
			else
				su - "$RUNUSER" -c "$INSTALLDIR/splunk/bin/splunk restart"
				fi
				echo '------------------------------------------------------------------
Greping the log for ulimit messages...
------------------------------------------------------------------'
				grep ulimit "$INSTALLDIR/splunk/var/log/splunk/splunkd.log" | tail -n 12
				echo '------------------------------------------------------------------
Complete!
Confirm settings for ulimits and THP are correct.
If not, please restart the server and check splunkd.log to
ensure ulimits and THP are configured properly.
------------------------------------------------------------------'
				;;
			[Nn]* )
				echo '------------------------------------------------------------------
Complete!
Please restart the server and check splunkd.log to
ensure ulimits and THP are configured properly.
------------------------------------------------------------------'
				exit
				;;
			* ) 
				;;
		esac
	fi
}

echo "------------------------------------------------------------------"
echo "Greetings, programs!"
echo "This script will configure THP and ulimits"
echo "according to Splunk best practices. This will"
echo "require a restart to ensure THP is disabled."
echo "------------------------------------------------------------------"
echo "   ________________________________           "
echo "  /                                \\         "
echo "  |    All Batbelt. No tights.      |         "
echo "  \\_______________________________/          "
echo "                             ()    \\\\       "
echo "                               O    \\\\  .   "
echo "                                 o  |\\\\/|   "
echo "                                    / \" '\\  "
echo "                                    . .   .   "
echo "                                   /    ) |   "
echo "                                  '  _.'  |   "
echo "                                  '-'/    \\  "
echo "------------------------------------------------------------------"
read -p "Do you wish to proceed? [Y/N] " START
case "$START" in
	[Yy]* )
		splunk_user
		os_detection
		disable_thp
		alter_ulimits
		remove_outdated
		if [ ! -n "$SPLUNK_INSTALLED" ]; then
			boot_start
			query_splunk_restart
		fi
		;;
	[Nn]* ) echo 'Aborting...'
		exit
		;;
	* ) echo 'Please answer yes or no.';;
esac
```


### aliases.sh
```
# Core Splunk Commands
alias splunk='/opt/splunk/bin/splunk'
alias apps='cd /opt/splunk/etc/apps'
alias splunk_log='tail -f /opt/splunk/var/log/splunk/splunkd.log'
alias btool='/opt/splunk/bin/splunk btool'

# Deployment server commands
alias deploy_apps='cd /opt/splunk/etc/deployment-apps'
alias deploy_reload='/opt/splunk/bin/splunk reload deploy-server'

# Indexer cluster commands
alias masterapps='cd /opt/splunk/etc/master-apps'
alias managerapps='cd /opt/splunk/etc/manager-apps'
alias peerapps='cd /opt/splunk/etc/peer-apps'
alias validate_cluster_bundle='/opt/splunk/bin/splunk validate cluster-bundle'
alias apply_cluster_bundle='/opt/splunk/bin/splunk apply cluster-bundle'

# Search head cluster commands
alias shcluster_apps='cd /opt/splunk/etc/shcluster/apps'
alias apply_shcluster_bundle='/opt/splunk/bin/splunk apply shcluster-bundle --answer-yes -target'
alias shcluster_restart='/opt/splunk/bin/splunk splunk rolling-restart shcluster-members'
```
