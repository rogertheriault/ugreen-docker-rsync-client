# ugreen-docker-rsync-client
A simple rsync client for using rsync via cron from a UGREEN docker container to backup stuff

My NAS is the UGREEN NASync DXP6800 Pro, things may be different on your model.

## Background / UGREEN SYNC and Backup
The steps below are assuming you have a UGREEN NAS and want to back up files from there to some other NAS, say Synology. The example backs up one share "audiobooks" to the remote NAS every night at 3am. Rsync only copies over the changes. You will have the same file tree and timestamps at the destination.

If you need versioned backup, try the UGREEN "sync and backup" app. This project (well, the compose file is basically it) was created to handle a use case not (yet?) handled by that app.

### rsync
UGREEN's sync app doesn't seem to support basic rsync. This attempts to solve it, without major modifications inside the UGREEN filesystem, by adding a docker project through the UGREEN Docker app.

It uses `rsync`, via a public Docker image based on the lightweight Linux Alpine, and `cron`, to enable your NAS to use rsync to some other NAS and copy changes.

You can customize the cron schedule, and the sources and destinations, by adding more `CRON_TASK_` entries to the [compose.yaml](https://github.com/rogertheriault/ugreen-docker-rsync-client/blob/main/compose.yaml) file, and you can mount multiple volumes locally, and communicate with more than one other system.

And, beyond the scope here, you could run the same image as an rsync server container by checking the references and modifying the compose file.

### disclaimer
I'm assuming some unix/linux knowledge here, and I may be a bit overly descriptive for power users, but if this seems to be overly difficult or complex I apologize... please reach out to UGREEN support and encourage them to make this sort of functionality available in an app.

### Sources and reference
- [European Environment Agency's simple rsync container based on Alpine](https://github.com/eea/eea.docker.rsync)
- [Docker Hub image source for the above](https://hub.docker.com/r/eeacms/rsync)
- [rsync manual page](https://linux.die.net/man/1/rsync) or type "man rsync" 
- [Wikipedia cron reference](https://en.wikipedia.org/wiki/Cron)

# Steps

## Prepare a docker project on your UGREEN nas
1. Install the Docker app, from "Apps", if you don't already have it installed
2. Open Docker, select the Project tab, click Create,
3. Name the project (in my case, I called it rsync; keep it short, it's going to be the name of a folder on your filesystem)
4. Choose a storage location. If you have an NVME (M.2) volume, use that!
5. Paste or upload the contents of the compose.yaml file here into the "compose configuration" editor (and modify as needed, e.g. the IP address of your destination, the volumes you want to backup, and the backup destination)
6. BUT IF you don't know those yet, we'll get to that below... however, in this case, the share "audiobooks" in "Volume 1" has a path `/volume1/audiobooks` and I exposed it inside the container as read-only in `/source/audiobooks`
7. Comment out the CRON env var for now, there's more stuff to set up
8. Uncheck "run immediately after creation"

### tips
- do not put quotes after the = in the cron entries, it will wind up in the cronfile

## Set up your destination (e.g. Synology)
1. If you do not have one, create a backups shared folder
2. Create a new user called rsync, with a very strong password, give it admin privileges, and a home directory (so it can use ssh)
3. Edit all the access rights to DENY it access to anything except the backups share, the homes folder, and to only the rsync and sftp services
4. Enable ssh in the Terminal control panel (change the port if you want, just make sure you update the compose file)
5. From your workstation, ssh in to the Synology as rsync, if all is good you will land in that user's home directory. If not, review the above steps.
6. Create a .ssh folder in the rsync user's home directory
7. `chmod 700 .ssh` to prevent other users from accessing it
8. change directory into .ssh and create `authorized_keys` file, then `chmod 600 authorized_keys`
9. Over on UGREEN, in your docker project, "deploy" the project.
10. Still in the UGREEN, click the Logs tab, and you should see a public key that was generated (and persisted) by the docker project. copy it
11. In the Synology ssh session, paste it into that `authorized_keys` file. Leave out the lines with `=======`. Make sure you didn't insert any line breaks or spaces, resize the window to verify, and save.
12. While you're in the Synology, look in the filesystem for the backups folder, and make sure your rsync user can see and change files in it
13. Log out of your ssh session, and go to the Synology Terminal control panel, and restart ssh services by disabling, apply, re-enabling, apply

## Back to the UGREEN (some of this is optional)
1. Turn on ssh, and set the expiration according to your needs
2. Make sure your admin user has a home directory, if it doesn't, give it one
3. Ssh in to the UGREEN as that admin user
4. To save time now and in the future, `sudo usermod -aG docker yourusername` to add your user to the docker group
5. exit and ssh back in
6. run `docker ps` and if your project is still running it'll show up
7. take a look inside the running container with `docker exec -it rsync-client sh` which will give you a shell in the Alpine container.
8. If you already mapped a folder in the volumes section of the compose file, it should be under `/source`, e.g. `ls -la /source`
9. If you specified a cron command, or multiple, `crontab -l` should list them
10. **Run a test rsync command** here to your synology, to verify and also to save the destination host key

Just copy the command from the CRON line without the cron scheduling, e.g. `rsync -avz -e "ssh -p 22" /source rsync@10.10.99.42::backups` if you already mounted say a test share to `/source`. It should copy everything and log it. Run it again, it shouldn't copy anything. If you're not ready to actually copy stuff, substitute something else for `/source` for now.

The reason for doing this (I may have mentioned this was complicated) is you'll be prompted, inside the container, to add the key fingerprint of the host you are connecting to to the .ssh/known_hosts file. If you do not do this, wonderful security will prevent your connection when the cron task runs!

So when prompted, answer yes. Assuming you got the IP correct.
```
/ # rsync -avz -e 'ssh -p 22' /source/ rsync@10.10.99.42::backups
The authenticity of host '10.10.99.42 (10.10.99.42)' can't be established.
ED25519 key fingerprint is SHA256:BdZ+/d/randomcharsgohere-T5RuvWTQHObO5zXU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.99.42' (ED25519) to the list of known hosts.
```
This will be accessible on the NAS in that `ssh-key` folder in a file named `known_hosts`

11. If you're not quite ready, you can still list the rsync "modules" available (see the man page) with `rsync 10.10.99.42::` (use your Synology's IP, and add two colons)
12. End the shell session in the container with `exit`, to drop back into your UGREEN.
13. Look for the Docker directory and your project... in my case it was in `/volume2/docker/rsync`
14. **IMPORTANT** another reason we're here... lock down the ssh-key file! `cd /volume2/docker/rsync` and `sudo chmod 700 ssh-key ssh_host_keys` (notice most of these files are owned by root) so other users can't access them
15. To confirm the permissions change didn't cause issues, you can pop back into the container and try an rsync command.
16. Go into the Synology UI and check the contents of your backups shared folder.


# Conclusion
If you modified the compose file, you can redeploy the project. If you need to test, set the cron schedule to every few minutes e.g.  `0,15,30,45 * * * *` and then check the Docker app's logs.

At this point you may need to revisit that compose file and set up a few more volumes to only back up certain shares (or, folders within them)

```
volumes:
  - /volume1/audiobooks:/source/audiobooks:ro
  - /volume1/videos:/source/videos:ro 
```

When all is working, the Docker project log on UGREEN should log the cron output e.g.

```
rsync-client  | sending incremental file list
rsync-client  | ./
rsync-client  | audiobooks/
rsync-client  | audiobooks/#recycle/
...
```

To exclude certain file patterns, or any other custom handling, see the [rsync man page](https://linux.die.net/man/1/rsync)

## Final checklist
- make sure your .ssh files and folders permissions are locked down
- make sure your rsync user on your backup server has limited access to the NAS (as limited as possible)
- check and adjust the cron entries and the volumes as noted above, based on your backup needs
- rsync should be accessing the synology using the docker project's private key to authenticate via ssh, the public keypair was stored over on the synology
- you can now disable ugreen ssh
- you will need to leave ssh access turned on on the Synology or whatever system is your remote destination
- there may be a better way to handle rsync on a Synology without using ssh, supposedly it has a daemon
- if that NAS is not just a backup server, and has other users, please make sure all admins have strong passwords, add firewall rules, disable password use in the ssh daemon and set up keys for your admin users, change the ssh port, whatever... that's beyond the scope of this but it is very important for security
- it's possible the docker logs grow without being archived or truncated (though, I noticed the UI only shows a few recent lines). Keep an eye out for that kind of thing
- when restarting the Docker project for any reason, the public key will be echoed out again. It *should* be the same as before, so ignore the message to put it in authorized_keys unless for some reason it changes (which will probably only happen if you wipe out the docker subdirectory it's stored in)
- and if you change the server IP or add a new destination, please see the notes above regarding `known_hosts`


