# jinkens从节点ssh-slave依赖java8问题解决

jinkens配置的从节点依赖java8的运行环境。
这里借助jinkens自身的机制，jinkens会优先在工作目录下找java并判断版本号。这给我们配置java环境提供了不少便利，可以不改变系统的配置。
原本系统java是什么版本还是什么版本，不会破坏编译环境。

## 普通linux系统

从Oracle官网直接下载jre的二进制包。解压缩后放到
/home/jenkins/jdk

## freebsd

oracle和openjdk都没有相应的二进制可以下载。需要从freebsd的源上考下来
如：https://mirrors.ustc.edu.cn/freebsd-pkg/FreeBSD%3A10%3Aamd64/latest/All/openjdk8-jre-8.181.13.txz 

下载后同样解压缩成/home/jenkins/jdk

## 验证


	/home/vms/jenkins/jdk/bin/java - version

执行上面命令看到正确返回java版本为1.8就可以了。
