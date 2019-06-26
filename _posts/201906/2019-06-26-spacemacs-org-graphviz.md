# spacemacs org graphviz

## spacemacs

spacemacs原始配置文件有bug，需要先改动一下graphviz的packages.el。重新load下.spacemacs
这样org mode就能调用dot代码了。

插入如下代码，按C-c C-c
```
#+BEGIN_SRC dot :file ./test.png :cmdline -Kdot -Tpng
digraph time {

    rankdir = "LR";
    node[shape = "point", width = 0, height = 0];
    edge[arrowhead = "none", style = "dashed"];

    {
        rank = "same"
        edge[style = "filled"];
        plugin[shape = "plaintext"];
        plugin -> step00  step01  step02 -> step03 -> step04 -> step05 -> step06 ->step024;
    }
    
    {
        rank="same";
        edge[style="solided"];
        services[shape="plaintext"];
        services   step10 -> step11;
    }
    {
        rank="same";
        edge[style="solided"];

        work_task[shape="plaintext"];
        work_task step21 -> step22 -> step23 -> step24 -> step25 -> step26 -> 
        step27 -> step28 -> step29 -> step210 -> step211 -> step212 ->  
        step213 -> step214 -> step215 -> step216 -> step217 -> step218 -> 
        step219 -> step220 -> step221 -> step222 -> step223 -> step224;
    }
    {
        rank="same";
        edge[style="filled"];
        sub_task[shape="plaintext"];
        sub_task   step33 -> step34  step35 -> step36 step37 -> step38 step39 -> 
        step310 step311 -> step312 step313 ->step314 step315 -> step316 step317 -> step318 step319 -> step320;
    }

    step10 -> step11 [label="check parameter"];
    step21 -> step22 [label="check domain exist"];
    step22 -> step23 [label="check glance image exist"];
    step00 -> step10 [label="send deploy xml", arrowhead="normal"];
    step11 -> step21 [label="create work_task", arrowhead="normal"];
    step11 -> step02 [label="return work_task id", arrowhead="normal"];
    step23 -> step33 [label="create sub_task:download glance image", arrowhead="normal"];
    step34 -> step24 [label="download glance image finished", arrowhead="normal"];
    step25 -> step35 [label="create sub_task:deploy vm", arrowhead="normal"]
    step36 -> step26 [label="deploy finished", arrowhead="normal"]
    step27 -> step37 [label="create sub_task:addDevice", arrowhead="normal"]
    step38 -> step28 [label="addDevice finished", arrowhead="normal"]
    step26 -> step27 [label="check addDevice list"]
    step28 -> step29 [label="check device list"]
    step29 -> step39 [label="create sub_task:device", arrowhead="normal"]
    step310 -> step210 [label="device finished", arrowhead="normal"]
    step210 -> step211 [label="check attachLun list"]
    step211 -> step311 [label="create sub_task:attachLun", arrowhead="normal"]
    step312 -> step212 [label="attackLun finished", arrowhead="normal"]
    step212 -> step213 [label="check imageDevice list", arrowhead="normal"]
    step213 -> step313 [label="create sub_task:imageDevice", arrowhead="normal"]
    step314 -> step214 [label="imageDevice finished", arrowhead="normal"]
    step214 -> step215 [label="check configDrive"]
    step215 -> step315 [label="create sub_task:configDrive", arrowhead="normal"]
    step316 -> step216 [label="configDrive finished", arrowhead="normal"]
    step216 -> step217 [label="check qos"]
    step217 -> step317 [label="create sub_task:qos", arrowhead="normal"]
    step318 -> step218 [label="qos finished", arrowhead="normal"]
    step219 -> step319 [label="create sub_task:vm start", arrowhead="normal"]
    step320 -> step220 [label="vm start finished", arrowhead="normal"]
    step220 -> step221 [label="need to set protectedStatus", arrowhead="normal"]
    step221 -> step222 [label="set protectedStatus"]
    step222 -> step223 [label="check protectedStatus"]
    step224 -> step024 [label="vm init finished", arrowhead="normal"]


}
#+END_SRC
```

![1.1](https://github.com/dong1224/dong1224.github.io/blob/master/_posts/201906/1.1.png?raw=true)
