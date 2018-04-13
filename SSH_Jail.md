# SSH_Jail.md

Table of Contents
=================




## Backgroud 

One of my Azure customer wants to perform ssh jail. I suggested him to follow with https://allanfeid.com/content/creating-chroot-jail-ssh-access but it seems it doesn't work so well. 

In this case, I tested in my lab enviornment and provide the solution below 

## Work Flow 

### Create account 

command line (copy& paste)

```bash
groupadd greysin
adduser -g greysin greysin
passwd greysin 
```

samples

```bash
# groupadd greysin
# adduser -g greysin greysin
Creating mailbox file: File exists
# passwd greysin
Changing password for user greysin.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

### Create Jail folder 

command line (copy& paste)

```bash
mkdir -p /var/jail/{dev,etc,lib,user,bin}
mkdir -p /var/jail/usr/bin
chown root.root /var/jail/
mknod -m 666 /var/jail/dev/null c 1 3
cd /var/jail/etc
cp /etc/ld.so.cache .
cp /etc/ld.so.conf .
cp /etc/nsswitch.conf .
cp /etc/hosts . 
cp -pr /home/greysin/ /var/jail/home/
echo 'PATH=/bin:/sbin:$PATH' >> /var/jail/home/greysin/.bashrc
```

The echo command is used to create the PATH environment for jailed user 

### Create Chroot binary 

Here I wrote a simple script to archieve this. Copy the shell script to your VM and run it 

```bash
#!/bin/bash
# Author Ying

target=/var/jail
CMDclear(){
    if which $1 &> /dev/null;then
        CMDpath=`which --skip-alias $CMD`
        echo $CMDpath
    else
        return 5
    fi
}
  
CMDcp(){
    cmdDir=`dirname $1`
    [ -d ${target}${cmdDir} ] || mkdir -p ${target}${cmdDir}
    [ -f ${target}$1 ] || cp $1 ${target}${cmdDir}
}
  
LIBcp(){
    for lib in `ldd $1 |grep -o "/[^[:space:]]\{1,\}"`;do
        libDir=`dirname $lib`
        [ -d ${target}${libDir} ] || mkdir -p  ${target}${libDir}
        [ -f ${target}${lib} ] || cp $lib ${target}${libDir}
    done
}
       
while true;do
read -p "Enter a Command[Enter q|quit to quit]: " CMD
    if [ $CMD == 'quit' -o $CMD == 'q' ];then
        echo quiting
        exit 0
    fi
CMDclear $CMD
    if [ $? = 5 ];then
        echo "the command doesn't exsit"
        continue
    fi
CMDcp $CMDpath
LIBcp $CMDpath
done
```

Give it executation right and run it 

```bash
chmod +x M_chroot.sh
./M_chroot.sh
```

Samples 

```bash
# chmod +x ./M_chroot.sh
[root@yingrhel74 ~]# ./M_chroot.sh
Enter a Command[Enter q|quit to quit]: ls
/bin/ls
Enter a Command[Enter q|quit to quit]: pwd
/bin/pwd
Enter a Command[Enter q|quit to quit]: cd
/bin/cd
Enter a Command[Enter q|quit to quit]: touch
/bin/touch
Enter a Command[Enter q|quit to quit]: mkdir
/bin/mkdir
Enter a Command[Enter q|quit to quit]: rm
/bin/rm
Enter a Command[Enter q|quit to quit]: cp
/bin/cp
Enter a Command[Enter q|quit to quit]: cat
/bin/cat
Enter a Command[Enter q|quit to quit]: vi
/bin/vi
Enter a Command[Enter q|quit to quit]: bash
/bin/bash
Enter a Command[Enter q|quit to quit]: quit
quiting
```

### Modify sshd configuration to add jail option 

Modify /etc/ssh/sshd_config

```bash
Match group greysin
        ChrootDirectory /var/jail
        X11Forwarding no
        AllowTcpForwarding no
```

And be sure that password authentication was enabled 

```bash
PasswordAuthentication yes
```

Restart sshd/ssh service (centos/ubuntu)

```bash
systemctl restart sshd 
```

### Try to login 

```bash
$ ssh greysin@52.163.124.52
greysin@52.163.124.52's password:
-bash-4.2$ pwd
/home/greysin
-bash-4.2$ ls -lrat
total 12
-rw-r--r--. 1 1004 1004 193 Oct 17  2016 .bash_profile
-rw-r--r--. 1 1004 1004  18 Oct 17  2016 .bash_logout
drwx------. 2 1004 1004  62 Apr 13 05:51 .
drwx------. 3 1004 1004  77 Apr 13 05:53 ..
-rw-r--r--. 1 1004 1004 253 Apr 13 05:53 .bashrc
-bash-4.2$ cd /
-bash-4.2$ ls -lrat
total 0
drwxr-xr-x.  2    0    0   6 Apr 13 05:52 user
drwxr-xr-x.  2    0    0   6 Apr 13 05:52 lib
drwxr-xr-x.  3    0    0  17 Apr 13 05:52 usr
drwxr-xr-x.  2    0    0  18 Apr 13 05:52 dev
drwxr-xr-x.  2    0    0  77 Apr 13 05:52 etc
drwx------.  3 1004 1004  77 Apr 13 05:53 home
drwxr-xr-x. 10    0    0  98 Apr 13 05:56 ..
drwxr-xr-x. 10    0    0  98 Apr 13 05:56 .
drwxr-xr-x.  2    0    0 214 Apr 13 05:57 lib64
drwxr-xr-x.  2    0    0 116 Apr 13 05:57 bin
```

