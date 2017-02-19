## 2017-02-19 - Changing owncloud from .deb to manual install

On a server I maintain, Owncloud is installed from the Debian repository.
However, the way the .deb package behaves does not fit properly with the rest of the services running on the server.
For example, the Apache config file has to be adjusted almost every update.
Also, you need to maintain the repositories separately in apt if new releases come in, and, on Debian, updates are comparably late.
Because of this, I want to change to a manual install that is not maintained by the package manager.

However, after some discussion with friends, I came to the conclusion that not only an upgrade and the change to a manual installation is in order, but also a change to Nextcloud.
The reasons for the change are:

* Faster development
* More dedicated to openness
* Compatible to the Owncloud client
* Maintainers who appear to take upgrade security more serious
* Features like ActiveSync and Outlook integration

I am aware that not all of these are hard facts.
Still, AS is one of the most important arguments to me because it enables my users to sync with the application without needing to resort to \*DAV adapters.

[The process to change to Nextcloud appears to be straight-forward.](https://nextcloud.com/migration/)
First I need to upgrade to ownCloud 8.2, then switch over to Nextcloud 9.0, then upgrade Nextcloud itself.
Because changing ownCloud to a manual install before switching over to Nextcloud, I will upgrade it using the Debian repositories, and install Nextcloud manually.
Before every step, I take a backup, and after every step, I ensure everything is working before proceeding.
This way, I have the possibility to salvage everything I did up to the point where it fails, and the day will not be wasted.

### 1. Make a backup

To be on the safe side, I backup the ownCloud folder, the database, and, in addition, my apache2 configuration files for ownCloud.

### 2. Upgrade to the latest ownCloud 8.1

Using apt, I intend to upgrade to ownCloud 8.1.9-12.1, which is the current version in the currently enabled Debian repository.
The involved packages are `owncloud`, `owncloud-config-apache`, and `owncloud-server`.
I was pondering on no longer using `owncloud-config-apache` and configure it myself, but that would be a waste of time at this point.
A problem I have is that I am still having the old opensuse repositories configured.
They need to go first, as I am getting warnings all over that the keys I had imported are no longer relevant or trusted.

#### 2.1 Adjust apt repository to that of ownCloud

I get the latest key and repository information from [ownCloud's download page](https://download.owncloud.org/download/repositories/stable/owncloud/) and configure my system accordingly.
I use this opportunity to move the ownCloud repository information out of my current main sources.list file into a separate one.
I also finally disable the Debian Wheezy sources I somehow forgot to remove.
Sloppy me.

Afterwards, I am really unhappy with the result, as apt now offers only the latest release of ownCloud, which is 9.1.4.
To get there, it is recommended to upgrade to 8.2 and 9.0 first, which I now cannot do using the apt package manager.
So I revert to the repository on opensuse.org using [opensuse's ownCloud 8.1 release channel page](http://software.opensuse.org/download/package?project=isv:ownCloud:community:8.1&package=owncloud).
While it seems pointless to download the GPG key for the repository via http, I still do it because it makes a man-in-the-middle at least more difficult.
Now apt shows me the correct packages to update and I can continue.

#### 2.2 Installing the upgrade to 8.1.12

After changing repository, I am offered version 8.1.12, which is more recent than I was offered before.
Yay, the effort of changing repository shows first advantages aside from the disappeared messages regarding untrusted repositories.

#### 2.3 Upgrade aftermath

The upgrade leaves me with a message to run `occ` manually and informs me that ownCloud is in maintenance mode for now.
Checking the page, sure enough, it is.
Checking in the ownCloud directory using `occ status`, I am informed some apps require an upgrade.
I run `occ upgrade`, check again with `occ status`, and then check my apps with `occ app:list`.
I enable calendar, contacts and documents again using the `occ` tool and disable the maintenance mode with `occ maintenance:mode --off`.
After checking ownClouds functionality, I am satisfied.
All the data is there, all the functionality is there, it appears to have gone well.

### 3. Upgrade to the latest ownCloud 8.2

Same as before: Add the proper repository, backup, upgrade, check functionality and data.
I add the [ownCloud 8.2 repository](https://download.owncloud.org/download/repositories/8.2/owncloud/) and its key and update apt's repositories.
The ownCloud package shows now 8.2.10 as most recent version and offers the upgrade.
I backup all files like I did the last time, and start the upgrade via apt.
Afterwards, while ownCloud is in maintenance mode, I check the status with the command line tool and perform `occ upgrade`.
After enabling the apps again and disabling maintenance mode, I check all functionality and make another backup, just in case.


To be continued.
