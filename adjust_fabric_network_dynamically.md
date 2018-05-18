# 简介

fabric的组织配置信息一般是提前写在configtx.yaml文件中的，通过configtx.yaml生成创世块和通道文件。而系统启动和通道创建是通过创世区块和通道文件进行的，因此要在fabric系统运行时，再调整网络中的组织、channel或者orderer是非常麻烦的的。而fabric提供了`configtxlator`工具来辅助进行操作。

到了fabric 1.1版本，会自带

# 动态增加组织

## 增加新联盟

##不增加新联盟

# 动态添加channel



# 动态增加orderer

orderer支持solo和kafka两种共识模式。solo模式下，orderer都是单个节点，不存在扩容的情况。只有才kafka模式下，才能支持动态增加orderer节点。

> 1.1

# 动态增加peer

