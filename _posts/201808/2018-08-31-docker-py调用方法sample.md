# dockerpy调用

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
