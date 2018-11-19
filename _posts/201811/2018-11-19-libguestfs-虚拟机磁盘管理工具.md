# libguestfs 虚拟机磁盘管理

## libguestfs-tools虚拟机磁盘管理工具：

官网：http://libguestfs.org/

这是一个非常强大的虚拟机磁盘管理工具，该工具包内包含的工具有virt-cat、virt-df、virt-ls、virt-copy/tar-in、virt-copy/tar-out、virt-edit、guestfish、guestmount等工具，具体用法也可以参看官网。该工具可以在不启动KVM guest主机的情况下，直接查看guest主机内的文内容，也可以直接向img镜像中写入文件和复制文件到外面的物理机，当然其也可以像mount一样，支持挂载操作。