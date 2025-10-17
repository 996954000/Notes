# UI 
简化一个具有 FGUI 预加载 包懒加载的功能
- ~~FGUI 所有包 bytes 文件的预加载~~
- ~~提供同步 package 加载函数，查找相应的 bytes 文件并载入包~~

包的加载有了，需要接PMVC到Cmpt的Load里去，给包名，路径之类的即可，即懒加载
包的自动卸载还没有

问题是这个PMVC怎么整起来

怎么全局控制UI的加载

Mediator与viewComponent的关联
生命周期管理