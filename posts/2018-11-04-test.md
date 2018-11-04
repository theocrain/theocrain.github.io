---
layout: post
title: Munki + MicroMDM = MunkiMDM?
categories: macOS MicroMDM Munki
---

One of things that I have been thinking about lately is the bare minimum required of an MDM. I have been fairly successful in putting most configuration tasks into Munki, but there is still the issue of UAMDM required profiles. I would hate to stand up a completely independent system with user groups and departments just to install a few profiles (and yes, maybe a lot more than this in the future). I have been thinking about MicroMDM for over a year now and I think I have finally found a way to make it doable by relying on our current manifest groups and logic in Munki. 

# Controlling MicroMDM with Munki

1. Import a UAMDM required profile into Munki. (this will fail, that's ok!)
2. Have the Munki/client machine call the MicroMDM API to install/uninstall the profile.
3. Use Munki to manage which machines need the profile.

The last point is great as we can target a specific group of machines that's already been defined in our infrastructure. Also, for something like the TCC profile, we can make sure that it's only installed on 10.14 machines.

The preinstall script would look something like this:
```sh
#!/bin/sh
udid=$(ioreg -d2 -c IOPlatformExpertDevice \
| awk -F\" '/IOPlatformUUID/{print $(NF-1)}')

export MICROMDM_ENV_PATH="/var/root/env"

/usr/local/micromdm/api/install_profile $udid \
"/Library/Managed Installs/Cache/org.domain.kextpolicy-1.0.mobileconfig"
```
The uninstall script looks like this:
```sh
#!/bin/sh
udid=$(ioreg -d2 -c IOPlatformExpertDevice \
| awk -F\" '/IOPlatformUUID/{print $(NF-1)}')

export MICROMDM_ENV_PATH="/var/root/env"

/usr/local/micromdm/api/remove_profile $udid "org.domain.kextpolicy"
```
## Prereqs
* Have the machine enrolled in MicroMDM.

* For a quick deployment of the MicroMDM API, I simply created a package and deployed the commands to `/usr/local/micromdm/api`, jq to `/usr/local/bin` and the env file that includes the connection details for the API to `/var/root/env`.

>[details on the API setup](https://github.com/micromdm/micromdm/tree/master/tools/api#setup)

## Issues
I'm still trying to grasp the timing of the installation and what that means for munki throwing errors. In my limited tests, the profile doesn't seem to trigger the install when a user is not actively on the console. I'll be doing further troubleshooting and will update when more info is available.

I've also added a install_check_script (yes, lots of other ways to do this) to make sure it only installs if it acutally needs it. 

#### Small Update
I believe the main issue with the install timing is due to the fact that jq was not in the path when running the preinstall_script. Solved it by adding `PATH=$PATH:/usr/local/bin` to the `env` file.

[Complete pkginfo file](https://gist.github.com/joncrain/a307d6ca5de4668d950e656080a75d1f)


## Conclusion
If this becomes viable, I will be looking for other ways to control MicroMDM with Munki or even MunkiReport. If this seems useful to someone else, I'd love to chat about it and learn how to do it better. I'm around most days @joncrain on [MacAdmins Slack](http://macadmins.org/).