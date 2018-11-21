# libguestfs 虚拟机磁盘管理

## libguestfs-tools虚拟机磁盘管理工具：

[官网](http://libguestfs.org/)

这是一个非常强大的虚拟机磁盘管理工具，该工具包内包含的工具有virt-cat、virt-df、virt-ls、virt-copy/tar-in、virt-copy/tar-out、virt-edit、guestfish、guestmount等工具，具体用法也可以参看官网。该工具可以在不启动KVM guest主机的情况下，直接查看guest主机内的文内容，也可以直接向img镜像中写入文件和复制文件到外面的物理机，当然其也可以像mount一样，支持挂载操作。

## libguestfs实现原理

libguestfs主要有三个部分：guestfsd，guestfs-lib，guestfish

其中，guestfsd是一个demon，libguestfs是一个lib，guestfish是一个命令行的工具。

 

guestfsd是一个daemon，但是他不是运行在host上的daemon，它运行在guest上，libguestfs首先febootstrap和febootstrap-supermin-helper两个工具。

将host中的kernel，用得到的一些modules，配置文件和一些工具的rpm package重新组合到一起，接着在后台启动一个qemu进程读取这个由febootstrap工具链生成的image。在用qemu启动的一个通信的协议，他可以通过socket接受子host端guestfs-lib写道socket的数据。guestfsd通过分析接受到的数据，进而执行相应的do_*操作，do_*操作实际上市对guest端普通命令的一些封装，如果想实现一个new api，只要在guestfsd里用相应的do_*对普通命令进行封装即可，但是一定要将这个普通命令程序通过febootstrap大宝到qemu启动读取的image中。

guestfs-lib 是一个库，他实现了一些libguestfs的库函数-guestfs_*.这些库函数想socket发送相应的数据，数据就会被guest段的guestfsd接收到，进而分析所要执行的操作。

guestfish是对guestfs-lib接口函数的一些应用，guestfish的命令哦都市通过调用guestfs-lib的库函数来实现的。

因此在使用libguestfs的时候，可以使用guestfish这样的命令行工具，也可以直接在程序（包括C,java以及libguestfs支持的一些script language）中调用guestfs-lib实现的库函数。