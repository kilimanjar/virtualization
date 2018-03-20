
How To Use Virt-Manager, Libvirt With Normal User Without Root Privileges and Without Asking Password
by İsmail Baydan · Published 01/03/2017 · Updated 17/06/2017

[https://www.poftut.com/use-virt-manager-libvirt-normal-user-without-root-privileges-without-asking-password/](https://www.poftut.com/use-virt-manager-libvirt-normal-user-without-root-privileges-without-asking-password/)

Virt-manager and libvirt is core tools used for virtualization in Linux ecosystem. As a end user I am using these tools to create and run virtual machines. I am running this tools as normal user without using privigleged user like root But every time I try to run these tool the sudo password is asked me. This become a nightmare after some time. Are there any solution to run these tools without putting password and changing any permission in virtualization side. Yes.
Polkit

PolicyKit or simply Polkit is a component used to controlling system wide privileges in Unix and Linux operating systems. Fedora Linux distribution heavily uses Polkit. We will use Polkit to authenticate our self and start virt-manager without password.
Create Group For Virtualization

To run virtualization services and software we need a group which have right to access related system resources. Most of the operating systems create this group as libvirt . If not create the group with the following command. But keep in mind this needs root privileges.

```	
$ groupadd libvirt
```
Put User To Virtualization Group

Now we need to put our normal or current user to the virtuliazation group. As stated previous step the group name is libvirt but if it is different please change accordingly. In this command we added secondary group named libvirt to the user john

```	
$ sudo usermod -a -G libvirt john
```

Create Polkit Rule

We will create polkit rule by using libvirt group. Create a file like


```	
/etc/polkit-1/rules.d/80-libvirt.rules
```

And put following content to the file. This rule will give libvirt groups users access to the virtualization capabilities without password.
LEARN MORE  How To Get Today's Date?

```
polkit.addRule(function(action, subject) {
 if (action.id == "org.libvirt.unix.manage" && subject.local && subject.active && subject.isInGroup("libvirt")) {
 return polkit.Result.YES;
 }
});
```

```	
polkit.addRule(function(action, subject) {
 if (action.id == "org.libvirt.unix.manage" && subject.local && subject.active && subject.isInGroup("libvirt")) {
 return polkit.Result.YES;
 }
});
```
