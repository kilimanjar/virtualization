# Guide: How to Enable Huge Pages to improve VFIO KVM Performance in Fedora 25
August 20, 2017, [http://vfiogaming.blogspot.cz/2017/08/guide-how-to-enable-huge-pages-to.html](http://vfiogaming.blogspot.cz/2017/08/guide-how-to-enable-huge-pages-to.html)

## Why enable Huge Pages and disable Transparent (THP) ?
In the Linux kernel, the standard size for a block of addressable memory is four kilobytes (4k). This is called a page. Every system has a finite number of addressable pages, based on how much physical memory the system has installed (read more in *).

When applications like databases or Virtual Machines need to allocate large contiguous amounts of memory, Kernel developers found an easy way to implement it by creating an automatic way that called Transparent Huge Pages (THP) (more in **).

Unfortunately this automatic procedure has some side effects unresolved by the time like unwanted cpu usage you can see in these two pictures below.
     
The first picture shows the qemu user that running the VM-Guest in the idle state ( the VM  is not running any computation tasks inside VM) . Before i change from THP to HugePages, The cpu usage was never under 50+ %  the idle state and now it is around 20%  . More than 50% idle cpu usage gain !

The second picture is coming from the second link (*) ,where you can see in a different situation on a Hadoop Framework with a oracle database how much lesser can be the average cpu usage if you disable THP and manual add the Huge-Pages.

Also, Alex Williamson suggests in his famous VFIO blog to enable hugepages to help improve VM efficiency !

## Facts about Huge-Pages
 
After reading many articles from Red Hat (***), Fedora (***), Debian (***) , i end up to a procedure that it is clean, fast and safe for the Fedora 25 as Host and Windows as Vm-Guest that i am using to play Games .

Some facts about Pagesizes.  You can not allocate any size you want . Debian link (***)  explains what sizes of Huge pages you can choose between them . For Fedora 25 and for our Gaming VM-GUEST the best solution is 8 GB of RAM allocated for each one ( not less than 8 GB RAM for a windows VM , otherwise you will have problems in games ). if you do not have at least 16 GB of Ram in your Linux ... try find please.

Also . the default pagesize for Fedora 25 and Fedora 26 is 2mb and do not increase it to 1GB please because you will have render problems in your VM-Guest.

For the typical VM-GUEST for Games we need to allocate 1024 x 8 =  8192 MB and if we divide the 8192MB=8388608KB with 2048KB (it is the 2mb default size for fedora and we will keep it) the result is 4096. we keep this number because we will use it when we assign HOW MANY HUGE-PAGES we will allocate to our VM-GUEST ( 2nd link in Extra Links at the bottom explains the calculations )

So, for our VM-GUEST wuth the 8GB of allocated memory we need to persistent allocate :

4096 (number of pages) x 2048KB (the desired and default Page Size) = 8388608KB  aka 8192 MB of Memory = 8GB of dedicated RAM for our VM-GUEST

## HOW TO ASSIGN 8GB HUGE-PAGES TO  KVM
### SETUP GUIDE

1. Check  a) the state of THP  b) the size of each Huge-Page and  c) Huge-Page total   with this command :  
   ```
   cat /proc/meminfo | grep Huge
   ```
   if you see something like that in the picture below .... you don't need to continue and you already have Huge-Pages enabled to your Host and the THP ( the AnonHugePages in the picture ) are disabled , but i am sure this is not our situation here(  because THP are enabled by default in Fedora 25 and Fedora 26) , but our final destination :)

2. So, make sure you have no VM or Database running or any App that is using your THP ( same command as before) ) and add the desired memory (8388608 KiB)  by editing this file ( all commands from now need root or sudo permissions) :
   ```
   sudo nano /etc/security/limits.conf  
   ```
   and  alter the  values inside this file if  they are different to be like or comment them and add the bellow lines :
   ```
   # 8GB for our KVM-GUEST
   soft memlock 8388608
   hard memlock 8388608
   ```
3. Edit the file  /etc/sysctl.conf :
   ```
   sudo nano /etc/sysctl.conf
   ```
   and  assign how many Huge-Pages we want for our KVM-GUEST  ( 4096 )
   ```
   vm.nr_hugepages = 4096
   vm.hugetlb_shm_group = 36
   ```
4. Now we need to create a boot entry for the HugeTLBpage Filesystem . during boot, host uses it for our VM-GUEST. so create a directory on our / (root or sudo permissions) :
   ```
   sudo mkdir -p /hugepages
   ```
   edit the fstab :
   ```
   sudo nano /etc/fstab
   ```
   and add inside the fstab file :
   ```
   hugetlbfs    /hugepages    hugetlbfs    defaults    0 0
   ```
5. Now we need to edit our kvm to inform our kvm (our vm-guest without [] . you can find the name easy from virt-manager ) and the editing is similar with VI . search on google how to use VI .it is the same  :
   ```
   sudo virsh edit [VM-GUEST NAME]
   ```
   as Alex suggests (3 in Extra Links we don't need the cpu pinning for Huge-Pages) and ADD the text below Before the  <os> tag :
   ```
   <memoryBacking>
     <hugepages/>
   </memoryBacking>
   ```
   this is how my xml part looks like after the addition:

6. Finally step is to create a mechanism to force disable the Transparent HugePages during boot. I found that the best method is by creating a system service. there is an  article for centos (****) that describes very well how to do it . So , from [blacksaildivision.com](https://blacksaildivision.com/how-to-disable-transparent-huge-pages-on-centos)
   a) Create following file:
   ```
   sudo vi /etc/systemd/system/disable-thp.service
   ```
   b) and paste there following content:
   ```
   [Unit]
   Description=Disable Transparent Huge Pages (THP)
   
   [Service]
   Type=simple
   ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"
   
   [Install]
   WantedBy=multi-user.target
   ```
   c) Save the file and reload SystemD daemon:
   ```
   sudo systemctl daemon-reload
   ```
   d) Than you can start the script and enable it on boot level:
   ```
   sudo systemctl start disable-thp
   sudo systemctl enable disable-thp
   ```
7. Restart the Host and check how the host is using the hugetlb filesystem and HugePages and if you see that there are 0KB the AnonHugePages and  4096 the HugetotalPages like the first image ... Congrats you done it ! From now on no more unwanted cpu usage from THP  !
   
   P.S. i hope this guide work as it is in Fedora 26 , but i haven't test it yet . if you do it before my upgrade , give me feedback please

### References
(*)

[Everything about Virtualization](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/index.html)

[Hadoop Framework with Oracle Database: Bad Performance with THP](http://www.ghostar.org/2015/02/transparent-huge-pages-on-hadoop-makes-me-sad/)


(**)

[Kernel Doc for HugeTLBpage](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)

[Kernel Doc for THP](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)

[Fedora Hugepage Guide](https://docs.fedoraproject.org/en-US/Fedora/18/html/Virtualization_Administration_Guide/ch08s02.html)



(***)

[Red Hat Memory Tuning  Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/main-memory.html)

[Old Fedora HugePages Guide](https://fedoraproject.org/wiki/Features/KVM_Huge_Page_Backed_Memory)

[Debian Guide for Huge Pages](https://wiki.debian.org/Hugepages#Enabling_HugeTlbPage)


        
(****)

[Disable THP on Centos with system Service](https://blacksaildivision.com/how-to-disable-transparent-huge-pages-on-centos)
