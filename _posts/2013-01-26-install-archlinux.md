---
layout: post
title: Archlinux安装小记
disqus: y
tags: [Archlinux, linux, 安装]
---

最早是在<a title="DON'T PANIC" href="http://syco.mad4a.me/" target="_blank">杭神</a>那里听到Archlinux，刚好这几天ubuntu崩掉了，而且比起其他发行版，Arch要轻量的多，所以准备换到Archlinux玩玩看。

Archliinux的官方wiki很全，基本上安装中的各种问题都可以从wiki上面找到，这篇文章仅作一个小记录，具体参考还是请看<a title="Arch wiki" href="https://wiki.archlinux.org/" target="_blank">官方wiki</a>。

自己的本本是Acer5750g，现在本子上还有一个win7，准备做成双系统，最早本来是想用无线网来安装arch的，键入命令：

{% highlight sh %}
# lspci -vnn | grep 14e4
{% endhighlight %}

结果悲剧的提示：

{% highlight sh %}
Network controller [0280]: Broadcom Corporation BCM43227 802.11b/g/n [14e4:4358]
{% endhighlight %}

查表后发现自己的无线网卡不支持arch的那个开源驱动...于是转战有线

插入网线后，键入

{% highlight sh %}
# dhcpcd
{% endhighlight %}

然后测试一下是不是连接上了

{% highlight sh %}
# ping 8.8.8.8
{% endhighlight %}

载入键盘布局

{% highlight sh %}
# loadkeys us
{% endhighlight %}

查看现有的分区

{% highlight sh %}
# fdisk -l
{% endhighlight %}

用mskr来格式化，然后挂载一下

{% highlight sh %}
# mkfs.ext4 /dev/sda9
# mount /dev/sda9 /mnt
{% endhighlight %}

编辑镜像列表(vi /etc/pacman.d/mirrorlist)，把最适合的镜像站放在最前，如果是电信的话，选择163就好。

安装Arch

{% highlight sh %}
# pacstrap /mnt base base-devel
{% endhighlight %}

自己是Bios，简单地安装grub

{% highlight sh %}
# pacstrap /mnt grub-bios
{% endhighlight %}

产生fstab

{% highlight sh %}
# genfstab -p /mnt >> /mnt/etc/fstab
{% endhighlight %}

然后以root权限进入系统并操作

{% highlight sh %}
# arch-chroot /mnt
# vi /etc/hostname //自己写入主机名称
# ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime //调整时区设置
{% endhighlight %}

对grub安装以及写引导

{% highlight sh %}
# grub-mkconfig -o /boot/grub/grub.cfg
# grub-install /dev/sda
{% endhighlight %}

最后用passwd来修改密码。

之后原来的win7进不去了，那光盘修复一下引导，然后在grub2中键入命令就可以启动win7。

{% highlight sh %}
search --set -f /bootmgr
chainloader +1
boot
{% endhighlight %}

之后再修改下grub.cfg增加win7启动菜单。

下来就是配桌面，建议再安装一下vim或者gvim，否则编辑文本会很难受，我选择的是KDE，具体就是先安装Xorg和驱动，然后装kde，最后xstart进入桌面，具体来说在我安装的时候最大问题就是选择显卡驱动，我的显卡是N卡GT　540M，安装闭源驱动悲剧了，最后选择的驱动是xf86-video-nouveau nouveau-dri，A卡和其他卡见wiki。

最后感觉Arch的wiki真心不错...然后发现自己的英文略坑了...

安装时如果wiki看不懂也可以参考一下这个帖子(<a title="Archlinux安装全过程" href="http://tieba.baidu.com/p/1746514728" target="_blank">传送门</a>)

