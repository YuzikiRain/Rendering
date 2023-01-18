## earlyz

由硬件实现，如果片元着色器中存在discard、clip或直接修改深度等会导致深度发生变化的操作，那么earlyz会失效。因为使用者的目的就是有可能修改深度，earlyz有可能会导致不执行。

为了让其不失效，可以这么处理alphaTest：

- pass1只做discard操作，并进行或不进行深度写入
- pass2只计算片元颜色

后果是多pass可能无法合批

## prez

一般指的是由使用者手动用额外pass渲染一遍场景写入了深度。
