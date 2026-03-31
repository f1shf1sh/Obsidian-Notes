
# 计算节点配置
配置相关yaml文件，/etc/xxxx

# 沙箱管理
每一个沙箱启动需要安装三个tipu对应的包
监控：node_exporter服务
日志：fluent.bit服务

# 作业流程
1. user用户驱动签发（驱动给用户使用）
2. 导出结果：可以直接从/mnt/result目录读数据从页面上查看
3. 作业启动：host22有问题，后续流程用host11