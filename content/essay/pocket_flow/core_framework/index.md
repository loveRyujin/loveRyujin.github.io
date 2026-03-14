+++
date = '2026-03-10T23:30:00+08:00'
draft = false
title = 'PocketFlow核心框架'
tags = ["杂谈"]
+++

最近在github上发现一个有趣的项目[PocketFlow](https://github.com/The-Pocket/PocketFlow)，一个总代码量在100行左右的 LLM 框架，使用 python 编写，零依赖、简洁的实现。阅读了一下项目代码，非常适合学习，故以此篇文章总结分析一下框架源码。

PocketFlow 的核心基础组件主要有：BaseNode、Node、Flow，然后在此基础之上衍生出BatchNode、BatchFlow、AsyncNode等等。

我理解 PocketFlow 的核心就是两件事：
- **图编排**：将各个 Node 通过边组合起来，边即为各类 action。
- **共享数据**：Node 之间通过 SharedData 共享数据，是一个公用的存储。

## BaseNode
BaseNode 是最基础的父类。定义如下：
```python
class BaseNode:
    def __init__(self): self.params,self.successors={},{}
    def set_params(self,params): self.params=params
    def next(self,node,action="default"):
        if action in self.successors: warnings.warn(f"Overwriting successor for action '{action}'")
        self.successors[action]=node; return node
    def prep(self,shared): pass
    def exec(self,prep_res): pass
    def post(self,shared,prep_res,exec_res): pass
    def _exec(self,prep_res): return self.exec(prep_res)
    def _run(self,shared): p=self.prep(shared); e=self._exec(p); return self.post(shared,p,e)
    def run(self,shared): 
        if self.successors: warnings.warn("Node won't run successors. Use Flow.")  
        return self._run(shared)
    def __rshift__(self,other): return self.next(other)
    def __sub__(self,action):
        if isinstance(action,str): return _ConditionalTransition(self,action)
        raise TypeError("Action must be a string")

class _ConditionalTransition:
    def __init__(self,src,action): self.src,self.action=src,action
    def __rshift__(self,tgt): return self.src.next(tgt,self.action)
```
BaseNode 定义节点的生命周期规范、执行能力和连接不同节点的能力。

生命周期三阶段：
- `prep`：从共享存储 shared 中获取特定数据，进行预处理。
- `exec`：拿到 prep 阶段的结果，执行计算逻辑。
- `post`：结合 prep 和 exec 阶段的结果，进行后置处理，并将特定数据写回共享存储 shared 中。
三个阶段默认都是 pass，需要子类去重写实现。

执行入口：
- `run`：运行节点。
- `_run`：默认顺序调用 prep、_exec、post 三阶段。
- `_exec`：默认调用 exec 阶段。

连接节点能力：
- `__rshift__`：重载 >> 运算符，表示关联另一个 node 作为下一个节点，通过 next 方法，默认action 为 default。比如 a >> b，即a.__rshift__(b)。
- `__sub__`：重载 - 运算符，表示节点之间通过特定 action 关联起来。比如 a - "action" >> b，会先构造 _ConditionalTransition，也就是条件转换，然后调用它的 __rshirt__ 方法，利用 action 将 a、b 关联起来。

## Node
Node 是 BaseNode 的进一步封装，在 BaseNode 的基础上增加了重试和兜底机制。
```python
class Node(BaseNode):
    def __init__(self,max_retries=1,wait=0): super().__init__(); self.max_retries,self.wait=max_retries,wait
    def exec_fallback(self,prep_res,exc): raise exc
    def _exec(self,prep_res):
        for self.cur_retry in range(self.max_retries):
            try: return self.exec(prep_res)
            except Exception as e:
                if self.cur_retry==self.max_retries-1: return self.exec_fallback(prep_res,e)
                if self.wait>0: time.sleep(self.wait)
```
- max_retries：最大重试次数
- wait：等待的时间间隔
- exec_fallback：最大重试次数过后走到的兜底逻辑