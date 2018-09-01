# robotframe测试docker

## need to read
[docker-py docs](https://docker-py.readthedocs.io/en/stable/index.html)
[robotframe docs](http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html)
[docker api](https://docs.docker.com/engine/api/v1.24/)

## basic use docker-py
<pre><code>import docker

class dockerPy:

    client = 0

    def __init__(self):
        self.client = docker.DockerClient(base_url='unix:///var/run/docker.sock')

    def docker_version(self):
        for component,version in self.client.version().iteritems():
            return component,version

    def docker_images(self):
        return self.client.images.list()

    def docker_run_portinfo(self,image,containername,portinfo):
        self.client.containers.run(image,name=containername,ports=portinfo,stdin_open=True,detach=True)</code></pre>

		
## 远程调用docker 
在/usr/lib/systemd/system/docker.service，配置远程访问。主要是在[Service]这个部分，加上下面两个参数
<pre><code># vim /usr/lib/systemd/system/docker.service
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock</code><pre>

<pre><code>import docker

class dockerPy:

    client = 0

    def __init__(self):
        self.client = docker.DockerClient(base_url='tcp://192.168.100.70:2375')
</code></pre>
