# ------------mosdns开启---------------

防止 DNS 泄漏

# 远程 DNS 服务器

8.8.8.8, 1.1.1.1, 9.9.9.9，选其它DNS在dae的routing下定义

# ----------------dae-----------------------

global {

    # 基础路由与接口配置
    
    tproxy_port: 12345
    lan_interface: "br-lan"
    wan_interface: "auto"

    # 核心分流与解析策略
    
    dial_mode: "domain+"
    
    # 性能与兼容性优化（由 dae 自动处理内核与防火墙）
    
    auto_config_kernel_parameter: true
    auto_config_firewall_rule: true # 强烈建议改为 true
}

# DNS 智能分流解析配置

dns {

    upstream {
    
        mosdns: 'udp://127.0.0.1:5335'
        
    }
}

# 代理节点与订阅源配置

node {

    # 手动填写静态单个节点
    
    node_static_a: 'vless://'
    node_static_b: 'vless://'
}

# 策略组配置

group {

    proxy {
    
        policy: min_moving_avg
        
    }
    
}

routing {

    # 无论哪个设备发起请求，MosDNS 自身的解析流程必须优先保障，防止死循环
    
    pname(mosdns) && dip(8.8.8.8, 1.1.1.1, 9.9.9.9) -> proxy
    
    pname(mosdns) && dip(geoip:cn) -> must_direct
    
    pname(mosdns) && dip(geoip:private) -> must_direct

    pname(mosdns) && domain(geosite:cn) -> must_direct

    # ==================== 2. 内网基础流量放行 ====================
    
    # 所有人访问局域网内部（如路由器后台、NAS 等）全部直连
    
    dip(geoip:private) -> must_direct
    
    # 广告相关域名及 IP 全局拦截
    
    # domain(geosite:category-ads-all) -> block

    # ==================== 3. 核心分流：指定 IP 允许出海 ====================
    
    # 【核心规则】只有来自 192.168.2.224 的流量，在访问海外域名或海外 IP 时，才走proxy
    
    sip(192.168.2.224) && !dip(geoip:cn) && !dip(geoip:private) && !domain(geosite:cn)-> proxy

    # 192.168.2.224 访问国内时，依然保持直连（保证国内速度）
    
    sip(192.168.2.224) -> must_direct

    # ==================== 4. 兜底策略 ====================
    
    # 除了上面针对 .12 的规则外，其他所有设备、所有未命中的剩余流量全部强制直连
    
    fallback: must_direct
}
