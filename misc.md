## 随手记

### Continuous Profiling

development -> CI -> CD -> production -> Continuous Profiling --> feedback developers

https://github.com/grafana/pyroscope

Paper: [Google-Wide Profiling: A Continuous Profiling Infrastructure for Data Centers](https://research.google/pubs/pub36575/)

- **Why**:
	- even minor performance improvements translate into huge cost savings
	- traditional performance **analysis** (profiling) is too complicated
- so, why not profile on live traffic?
	- sample + analyze automatically
	- datas:
		- stack traces
		- heap
		- hardware + kernel events
		- lock contention
- with continuous profiling, we can answer these key questions easily:
	- `profile`: what are the hottest, or the most memory-consuming code regions?
	- `continuous`: how does performance differ across software versions?

TODO: what about profile overhead?

### 函数计算

函数计算可以解决业务逻辑的 where 跟 what 的问题。

然而其中的 what 在传统方案下其实也并不是问题。

至于 where 的问题，以往一般由各家公司自己的基础平台团队搭建与维护。函数计算确实能解决这个问题——代价是与云厂商有很深程度的绑定。

非要说的话，只要你上了云，就多多少少跟云厂商做了绑定。但是以往的绑定大多基于通用的接口（interface，而不是api），比如云服务器可以对应上虚拟机的概念，云数据库也可以通过自建 MySQL 来曲线救国。如果需要像对象存储这种，跟云厂商有了 api 上的绑定，实在不行还可以提前引入一层抽象层/转译层来隔离变化，以后迁移到自己的（开源）方案时直接加一个 driver 就可以，各个业务逻辑都可以不用动。

而上了函数计算之后，连整个代码框架都与云厂商产生了绑定（目前没看到各家云有比较统一的规范）。而框架层面的问题，万一以后要下云，就比 api 层面的要难搞定得多了。

这方面看到现在有相关的 cncf 项目 openfunction 了，保持关注下这个项目的进展。不过如果云厂商都不愿意推的话，感觉也很难有很大的前景。

而对于 when 的问题，目前函数计算主要是通过事件驱动机制来实现的。（也许同样作为事件驱动的 dapr 运行时很适合充当以后下云的开源方案）
