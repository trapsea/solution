第一个问题，在把dubbo从2.7.8升到3.2.9和nacos从1.4.2升到2.3.0之后，测试环境一切正常，但在升级当天晚上仍然遇到了问题。

当时看来比较严重的有服务升级后版本号却没有提升，版本号是服务实现灰度调用的最基本依赖，如果升级后服务的版本号不变，基本上没法使用灰度升级了，问题其实很快就定位了，因为在关闭服务后注册中心里仍然存在



第二个问题，某些服务修改应用名后会出现调用No Provider的异常，这个问题在dubbo2.x是不会出现的，原因在于dubbo3已经改成了实例级别的注册，调用方在调用远程的dubbo接口时其实是从nacos里拉取信息，每一个接口都会通过配置文件的形式注册到注册中心，文件内容就是该接口所在的项目应用名，比如A接口的所在应用是P1，现在改为P2，那么A接口文件内容会变成P1,P2，实际上P1已经不存在了，所以会报No Provider，那知道事情是怎么回事，解决方案也很简单，也不在这写了

