title: 符号执行
date: 1970-1-1
categories:
- RE
---

# angr

- 首先导入from angr import *
- 将程序加载进来p = Process("filename",auto_load_libs=False)
- 可以理解将程序变成一个特殊的状态,这样默认函数默认入口点,如果在()内加入addr=0xxxxxx,会指定另一个入口点s = p.factory.entry_state()
- 就有一些类似我们的打log时复现代码那一环,设定一个模拟器 smt = sm=p.factory.simulation_manager(s)
- sm就是之前载入文件,设定入口点,建立模拟对象,现在就可以explore来开始符号执行,后面是约束条件r = sm.explore(find=0x41414141,avoid=0x41414141) 
- 跑完以后print r.found[0].posix.dumps(0)

# ./filename flag执行

- 需要一个叫做claripy的模块
- args = [proj.filename, claripy.BVS('arg1', n*8)]  //8为8bit
- state = proj.factory.entry_state(args=args)
- 随后同上过程

# 特殊约束

```python
for byte in arg1.chop(8):
    state.add_constraints(byte != '\x00') 
    state.add_constraints(byte >= ' ') 
    state.add_constraints(byte <= '~') 
```
- 大概就是如上过程