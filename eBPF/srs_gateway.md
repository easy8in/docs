采用 L7 + L4 实现流量网关

L7 实现 HTTP-API 负载均衡，通过解析客户端请求 SDP 中的 ice-ufrag，在客户端请求，在请求中加入 ice-ufrag 生成的值，通知 srs 使用客户端提供 ice-ufrag （srs version5.0 源码：srs_app_rtc_server.cpp line:547）,生成 stun username = 生成的 ice-ufrag + : + 客户端 sdp 中的 ice-ufrag，验证 stun username 是否唯一，```一定要确保唯一```。轮询、hash 选择 srs 真实的服务器，发送 http-api 请求，请求成功后，在 ebpf-map 中 插入 stun username 作为 key  ，srs 服务器地址 + 端口组为值保存。

L4 实现 TCP/UDP 协议的包转发

1. packet 是不是 stun 协议
2. 如果是 stun 协议，解析 stun 中的 username，通过 username 获取 ebpf-map 中的目标 srs 服务器地址和端口
3. 重写 packet 保重的目标地址和端口为上一步获取到的地址和端口 return XDP_TX;
4. srs 处理后，将 packet 发送到网关，网关重写源地址

支持的模式 full-nat ( kubernetes 必然需要)

推送过程描述

http-api 过程描述

1. 客户端发送请求 srs-gateway 的 /rtc/v1/publish/
2. srs-gateway 解析请求参数中的 stream 值 执行 hash % Real SRS 集合长度选定真实 SRS 服务的 ip 和端口（或者根据检测 SRS 负载情况，选择负载最低的 SRS）
3. 解析请求参数中 sdp 的 ice-ufrag 值，动态生成服务端 ice-ufrag 值，拼接 服务器端 ufrag: 客户端 ufrag 为 stun username 值
4. 扩展请求参数，添加服务器 ufrag，同时发送请求到真实的 SRS 服务
5. 真实的 SRS 服务返回正确结果后，继续下面的步骤；失败，则清理资源返回 SRS 结果
6. 记录 stream 对应的真实 SRS 服务器 ip:port
7. 记录 stun username 对应的真实 SRS 服务器 ip:port，同时写入 ebpf-map
8. 返回 SRS 正确结果

TCP/UDP 过程描述（XDP + EBPF）

收到客户端发给自己的包：

1. 客户 http-api 过程执行成功， PeerConnection setRemote SDP 后，
2. 如果是 STUN 包，xdp 程序解析 STUN 包，获得 stun username 获取 ebpf-map 中真实 SRS 服务 ip 地址和端口
3. 将源 IP+ 源端口号 记录为 uint64 CIP = uint64（源端口 << 48 | 源 IP）
4. 选择本地未占用 IP + port 记录未 INTERNAL-IP
5. 真实 SRS 服务器 ip 地址和端口 记录为 REAL-IP
6. 修改 packet 包，源 IP 修改为 INTERNAL-IP，源端口修改为 INTERNAL-PORT
7. 修改 packet 包，目标 IP 修改为 REAL-IP，目标端口修改为 REAL-PORT
8. 转发 packet 包

流量负载

>[!TOP]
> 合理的述求，不合理的实现

> [!NOTE] Title
> Contents

> [!Qoato]- Title
> Contents
