版本分支：ceph_dev

##ceph数据安全特性
- ceph uadk crypto插件，支持rgw对象网关数据加解密算法AES-256-CBC开销卸载
- 国密算法支持。支持rgw对象网关数据加密采用SM4-CBC算法，后期也将SM4-CBC算法开销卸载至鲲鹏硬件加速器。

##ceph数据压缩特性
- ceph zlib isa-l在aarch64平台上的优化
- 支持ceph uadk zlib卸载至鲲鹏硬件加速器