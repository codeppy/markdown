## 1、RHCE练习环境 

**RHCE三合一包括124,134,294三套环境**

>实验环境物理机的用户名和密码分别是 kiosk:redhat;root:Asimov
>
>虚拟机servera~d,utility,basion,workstation的用户名和密码分别是	student:student;root:redhat
>
>1、初次使用三合一环境的时候，使用***rht-vmctl reset classroom***命令先重置classroom虚拟机
>
>2、然后使用***rht-vmview view classroom***	#访问classroom虚拟机的控制台，确保classroom虚拟机是否已经开启
>
>3、最后使用***rht-vmctl fullreset all***	#将所有其他虚拟机完全重置
>
>virsh list --all	#查看虚拟机运行状态
>
>rht-vmctl status vm-name	#查看虚拟机的状态
>
>rht-vmctl start vm-name	#启动某个虚拟机
>
>rht-vmctl stop vm-name	#关闭某个虚拟机
>
>rht-vmctl reset vm-name	#恢复某个虚拟机的快照
>
>rht-vmview view vm-name	#打开某个虚拟机的控制台（此操作只能在图形界面使用）
>
>rht-vmctl fullreset vm-name #完全恢复某个虚拟机，如果reset不能恢复那么就用fullreset

## 2、使用官方环境做实验

```bash
1、登录workstation
ssh workstation
2、运行加载lab的脚本，例如
lab start users-superuser
3、进行实验练习
4、如果是综合实验，可以使用lab grade开头的命令进行评分，评完分之后使用lab finish开头的命令来结束实验，这种lab finish开头的命令 一般是来清理实验环境的，这样如果想要重复练习，可以在使用lab finish命令之后再使用lab start重新加载实验环境进行练习，小实验没有lab grade,只有lab start和lab finish
```

## 3、三合一环境的切换

```bash
1、切换环境要在外边的虚拟机上操作（foundation0），命令如下
rht-setcourse rh134
2、查看当前环境
cat /etc/rht
3、检查classroom的状态，确保classroom处于running的状态，如果classroom不是running状态，那么就需要使用vmctl reset classroom对classroom虚拟机进行重置
4、检查所有虚拟机的状态，确保都是running状态
rht-vmctl status all
5、从124切换到134
rht-setcourse rh134
6、再次查看所有虚拟机的状态
rht-vmctl status all
```

