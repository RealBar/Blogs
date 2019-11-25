#golang基础（九）docker部署golang应用
### 缘起
公司使用jenkins+go plugin+pipeline+自定义Shared Libraries的方式进行应用部署，项目相关部署配置通写在项目源码中，pipeline script非常简短，
直接配置下环境变量指向配置文件，然后就调用自定义方法进行构建。

这样带来的好处就是构建层变得很薄，开发人员可以聚焦与源代码中的配置。但是缺点也很明显，就是配置文件的所有配置项由自定义的Shared Library固定了，
不够灵活。

目前已知的问题就是，当前构建golang应用的jenkins插件所配置的golang版本过低，我希望升级，怎么办呢？如果联系jenkins的运维直接修改golang插件配置，
可能会影响其他应用。

### 方案
