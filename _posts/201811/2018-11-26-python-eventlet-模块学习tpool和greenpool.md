# python eventlet 模块学习tpool和greenpool

[转载](https://blog.csdn.net/u011816753/article/details/66978404)

当我们需要使用到python的c接口，特别是一些对os的系统调用，官方说明如下：

	The vast majority of the times you’ll want to use threads are to wrap 
	some operation that is not “green”, such as a C library that uses its 
	own OS calls to do socket operations. The tpool module is provided to make these uses simpler.
	
如： 
nova/virt/disk/vfs/guestfs.py：174

    def setup(self, mount=True):
        LOG.debug("Setting up appliance for %(image)s",
                  {'image': self.image})
        try:
            self.handle = tpool.Proxy(
                guestfs.GuestFS(python_return_dict=False,
                                close_on_exit=False))

其中guestfs.GuestFS是一个c接口，作用是对一个image的操作，对image的读写，即对文件系统文件的读写。

	In [9]: from eventlet import tpool

	In [10]:  import thread

	In [11]: def my_func(thread_ident):
		...:     print ("now raw thrad id:",  thread_ident)
		...:

	In [12]: tpool.execute(my_func, thread.get_ident())
	('now raw thrad id:', 140736349889472)

	In [13]: def my_func(thread_ident):
		...:     print ("now raw thrad id:",  thread_ident, thread.get_ident())
		...:
		...:

	In [14]: tpool.execute(my_func, thread.get_ident())
	('now raw thrad id:', 140736349889472, 123145477103616)

	In [15]: tpool.execute(my_func, thread.get_ident())
	('now raw thrad id:', 140736349889472, 123145481310208)

	In [16]: tpool.execute(my_func, thread.get_ident())
	('now raw thrad id:', 140736349889472, 123145485516800)
	
明显看出thread进程的多进程都是同一个id，而tpool的则是不同的进程。
同时举个关于greenlet的例子

	from eventlet import greenpool
	pool = greenpool.GreenPool()
	for result in pool.imap(worker ,open('test.txt', 'r')):
		print result

	worker in thread 140736349889472
	.

	worker in thread 140736349889472
	worker in thread 140736349889472
	worker in thread 140736349889472
	worker in thread 140736349889472
	worker in thread 140736349889472
	worker in thread 140736349889472
	
greenpool得到的是同一个native thread，这样若使用c接口的话，会影响进程的开关以及进程的block。
