# Linux runuser command - Run Shell with Specified User/Group ID
The runuser command is used to run a shell with specified user and group ID. This command changes the user and group id's. When you want to run some commands as some other user, this command can be used to change the user. This command is like su command, but it does not prompt for password. So, only privileged user, i.e. root user can run this command successfully and can change to any user without any need for password.

This command is quite useful when used in shell scripts. This is because it is non-interactive command. Su command cannot be used for shell scripts for that it prompts for a password when run as any other user than root. But in case of runuser command, it just fails and exits with error (for unprivileged user). As runuser command does not run PAM hooks and authentication modules, it has lesser overheads than su.
runuser command

Here is an example of root user executing runuser command:

```
[root@redhat-server /]# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel) context=root:system_r:unconfined_t:SystemLow-SystemHigh
[root@redhat-server /]# runuser jack
[jack@redhat-server /]$ id
uid=501(jack) gid=501(jack) groups=501(jack),504(javaproject) context=root:system_r:unconfined_t:SystemLow-SystemHigh
```
You can check the current user with id command. Now when an unprivileged user tries to execute this command:

```
[jack@redhat-server /]$ runuser jones
runuser: cannot set groups: Operation not permitted
```

With -l or --login option, the new shell can be made a login shell just as in case of su command. runuser - command has same effect. It changes environment variables as well. Variables like PWD and PATH change their values with this option.

```
[root@redhat-server ~]# runuser - jones
[jones@redhat-server ~]$ id
uid=502(jones) gid=502(jones) groups=502(jones),504(javaproject) context=root:system_r:unconfined_t:SystemLow-SystemHigh
[jones@redhat-server ~]$ pwd
/home/jones
```

You can provide your custom shell if you don't want a default shell with -s option:

```
[root@redhat-server ~]# echo $SHELL
/bin/bash
[root@redhat-server ~]# runuser -s /bin/sh jones
sh-3.2$ echo $SHELL
/bin/sh
sh-3.2$ id
uid=502(jones) gid=502(jones) groups=502(jones),504(javaproject) context=root:system_r:unconfined_t:SystemLow-SystemHigh
```
