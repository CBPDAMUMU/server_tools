cat > /root/server_check.sh << 'EOFSCRIPT'
#!/bin/bash
#===============================================================================
# 服务器一键检测与优化脚本 v2.2 (CentOS 7/8 兼容版)
# 支持: CentOS 7/8, Ubuntu 18.04+, Debian 10+
#===============================================================================

set -eo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'

ISSUES=()
OPT_DESC=()

# 检测操作系统
detect_os() {
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS=$ID
        VER=$VERSION_ID
    else
        echo -e "${RED}无法检测操作系统${NC}"
        exit 1
    fi
}

# 基础网络检测
check_network_basic() {
    echo -e "${BLUE}[检测] 基础网络连通性...${NC}"
    if ping -c 3 -W 1 google.com &>/dev/null; then
        echo -e "${GREEN}   ✓ 外网连通正常${NC}"
    else
        ISSUES+=("NET")
        OPT_DESC+=("无法 ping 通 google.com，请检查网络")
        echo -e "${RED}   ✗ 外网不通${NC}"
    fi
}

# UDP阻断检测（含NTP超时）
check_udp_block() {
    echo -e "${BLUE}[检测] UDP 阻断风险...${NC}"
    local found=0
    if systemctl is-active systemd-timesyncd &>/dev/null; then
        if journalctl -u systemd-timesyncd --since "1 hour ago" 2>/dev/null | grep -qi "timed out"; then
            ISSUES+=("NTP")
            OPT_DESC+=("NTP 服务持续超时，存在 UDP 阻断风险")
            echo -e "${RED}   ✗ NTP 服务持续超时${NC}"
            found=1
        fi
    fi
    # 检查 DNS 超时日志
    if [ -f /var/log/syslog ] || [ -f /var/log/messages ]; then
        LOGFILE="/var/log/syslog"
        [ -f /var/log/messages ] && LOGFILE="/var/log/messages"
        if grep -i "dns.*timeout\|unreachable" $LOGFILE 2>/dev/null | tail -5 | grep -q .; then
            ISSUES+=("DNS_TIMEOUT")
            OPT_DESC+=("系统日志中发现 DNS 超时记录")
            echo -e "${RED}   ✗ 发现 DNS 超时记录${NC}"
            found=1
        fi
    fi
    if [ $found -eq 0 ]; then
        echo -e "${GREEN}   ✓ 未发现 UDP 阻断迹象${NC}"
    fi
}

# DNS 配置检测（兼容 CentOS）
check_dns_config() {
    echo -e "${BLUE}[检测] DNS 配置...${NC}"
    DNS_SERVERS=$(grep -E "^nameserver" /etc/resolv.conf 2>/dev/null | awk '{print $2}' | tr '\n' ' ')
    echo -e "   当前 DNS 指向: ${YELLOW}${DNS_SERVERS}${NC}"
    # 检测是否已启用 DoT / systemd-resolved
    if command -v resolvectl &>/dev/null; then
        if resolvectl status 2>/dev/null | grep -q "+DNSOverTLS"; then
            echo -e "${GREEN}   ✓ DNS over TLS 已启用${NC}"
            return
        fi
    fi
    # 对于没有resolvectl的系统，检查是否指向本地缓存大概率表示已优化
    if [[ "$DNS_SERVERS" == "127.0.0.53 "* ]] || [[ "$DNS_SERVERS" == "127.0.0.1 "* ]]; then
        echo -e "${GREEN}   ✓ DNS 已指向本地缓存，可能已优化${NC}"
    else
        ISSUES+=("DNS")
        OPT_DESC+=("未启用 DNS over TLS，使用传统 UDP DNS")
        echo -e "${RED}   ✗ 未启用 DNS over TLS${NC}"
    fi
}

# BBR 检测
check_bbr() {
    echo -e "${BLUE}[检测] TCP 拥塞控制...${NC}"
    CC=$(sysctl -n net.ipv4.tcp_congestion_control 2>/dev/null || echo "unknown")
    echo -e "   当前算法: ${YELLOW}${CC}${NC}"
    if [[ "$CC" == "bbr" ]]; then
        echo -e "${GREEN}   ✓ BBR 已启用${NC}"
    else
        ISSUES+=("BBR")
        OPT_DESC+=("未启用 BBR 加速")
        echo -e "${RED}   ✗ 未启用 BBR${NC}"
    fi
}

# UDP连接跟踪检测
check_conntrack() {
    echo -e "${BLUE}[检测] UDP 连接跟踪...${NC}"
    if lsmod | grep -q nf_conntrack; then
        TIMEOUT=$(cat /proc/sys/net/netfilter/nf_conntrack_udp_timeout 2>/dev/null || echo "30")
        echo -e "   当前 UDP 超时: ${YELLOW}${TIMEOUT}秒${NC}"
        if [[ "$TIMEOUT" -gt 10 ]]; then
            ISSUES+=("CONNTRACK")
            OPT_DESC+=("UDP 连接跟踪超时过长 (${TIMEOUT}秒)")
            echo -e "${RED}   ✗ 超时时间过长${NC}"
        else
            echo -e "${GREEN}   ✓ 超时时间合理${NC}"
        fi
    else
        echo -e "${GREEN}   ✓ conntrack 未加载，无风险${NC}"
    fi
}

# NTP 服务状态检测
check_ntp_service() {
    echo -e "${BLUE}[检测] NTP 服务状态...${NC}"
    if systemctl is-active systemd-timesyncd &>/dev/null; then
        echo -e "   systemd-timesyncd: ${YELLOW}运行中${NC}"
        if journalctl -u systemd-timesyncd --since "1 hour ago" 2>/dev/null | grep -qi "timed out"; then
            echo -e "${RED}   ✗ NTP 服务持续超时${NC}"
        else
            echo -e "${GREEN}   ✓ NTP 服务正常${NC}"
        fi
    elif systemctl is-active chronyd &>/dev/null; then
        echo -e "   chronyd: ${YELLOW}运行中${NC}"
    else
        echo -e "${GREEN}   ✓ NTP 服务未运行${NC}"
    fi
}

# 内核版本检测
check_kernel() {
    echo -e "${BLUE}[检测] 内核版本...${NC}"
    KERNEL=$(uname -r)
    echo -e "   当前内核: ${YELLOW}${KERNEL}${NC}"
    if [[ "$OS" =~ centos|rhel ]] && [[ "$VER" == "7"* ]] && [[ "$KERNEL" == "3.10"* ]]; then
        ISSUES+=("KERNEL")
        OPT_DESC+=("CentOS 7 内核过旧 (3.10)，不支持原生 BBR")
        echo -e "${RED}   ✗ 内核过旧${NC}"
    else
        echo -e "${GREEN}   ✓ 内核版本正常${NC}"
    fi
}

# ========== 优化函数 ==========
fix_dns_dot() {
    if command -v resolvectl &>/dev/null && systemctl is-active systemd-resolved &>/dev/null; then
        echo -e "${YELLOW}正在配置 DNS over TLS...${NC}"
        mkdir -p /etc/systemd/resolved.conf.d
        cat > /etc/systemd/resolved.conf.d/dns-over-tls.conf << 'EOF'
[Resolve]
DNS=223.5.5.5#dns.alidns.com 119.29.29.29#dns.pub
DNSOverTLS=yes
DNSSEC=no
Cache=yes
EOF
        systemctl restart systemd-resolved
        ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
        sleep 2
        echo -e "${GREEN}✓ DNS over TLS 已启用${NC}"
    else
        echo -e "${YELLOW}当前系统不支持直接配置 systemd-resolved DoT。${NC}"
        echo -e "${CYAN}建议手动安装 stubby 或 cloudflared 实现加密 DNS。${NC}"
        echo -e "${CYAN}或者保持现有 DNS 配置，并确保已优化 UDP 连接跟踪。${NC}"
    fi
}

fix_disable_ntp() {
    echo -e "${YELLOW}正在停止并禁用 NTP 服务...${NC}"
    systemctl stop systemd-timesyncd 2>/dev/null || true
    systemctl disable systemd-timesyncd 2>/dev/null || true
    echo -e "${GREEN}✓ NTP 服务已禁用${NC}"
}

fix_bbr() {
    echo -e "${YELLOW}正在启用 BBR...${NC}"
    if sysctl net.ipv4.tcp_available_congestion_control 2>/dev/null | grep -q bbr; then
        grep -q "net.core.default_qdisc=fq" /etc/sysctl.conf || echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
        grep -q "net.ipv4.tcp_congestion_control=bbr" /etc/sysctl.conf || echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
        sysctl -p &>/dev/null
        echo -e "${GREEN}✓ BBR 已启用${NC}"
    else
        echo -e "${RED}当前内核不支持 BBR，请先升级内核${NC}"
    fi
}

fix_conntrack() {
    echo -e "${YELLOW}正在优化 UDP 连接跟踪超时...${NC}"
    echo 10 > /proc/sys/net/netfilter/nf_conntrack_udp_timeout 2>/dev/null || true
    echo 10 > /proc/sys/net/netfilter/nf_conntrack_udp_timeout_stream 2>/dev/null || true
    grep -q "nf_conntrack_udp_timeout" /etc/sysctl.conf || echo "net.netfilter.nf_conntrack_udp_timeout=10" >> /etc/sysctl.conf
    grep -q "nf_conntrack_udp_timeout_stream" /etc/sysctl.conf || echo "net.netfilter.nf_conntrack_udp_timeout_stream=10" >> /etc/sysctl.conf
    sysctl -p &>/dev/null
    echo -e "${GREEN}✓ UDP 连接跟踪超时已缩短至 10 秒${NC}"
}

fix_kernel_centos7() {
    echo -e "${YELLOW}正在升级 CentOS 7 内核到主线版本...${NC}"
    echo -e "${YELLOW}注意: 升级后需重启生效${NC}"
    read -p "确认继续? (y/N): " confirm
    if [[ "$confirm" != "y" ]]; then
        echo "已取消"
        return
    fi
    rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    rpm -Uvh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
    yum --enablerepo=elrepo-kernel install kernel-ml -y
    grub2-set-default 0
    echo -e "${GREEN}✓ 内核升级完成，请执行 'reboot' 重启后生效${NC}"
}

# ========== 交互菜单 ==========
show_summary() {
    echo -e "\n${BLUE}==================== 检测结果汇总 ====================${NC}"
    if [ ${#ISSUES[@]} -eq 0 ]; then
        echo -e "${GREEN}🎉 恭喜！服务器状态优秀，未发现需优化项。${NC}"
        exit 0
    else
        echo -e "${YELLOW}发现以下可优化项目:${NC}"
        for i in "${!OPT_DESC[@]}"; do
            echo -e "  ${CYAN}$((i+1)))${NC} ${OPT_DESC[$i]}"
        done
        echo -e "\n${BLUE}=====================================================${NC}"
        echo -e "输入数字选择要执行的优化 (多个用空格分隔，如: 1 3)，输入 0 退出"
        read -p "请选择: " choices
        
        for choice in $choices; do
            if [[ "$choice" == "0" ]]; then
                echo "退出脚本"
                exit 0
            fi
            idx=$((choice-1))
            if [[ $idx -ge 0 ]] && [[ $idx -lt ${#ISSUES[@]} ]]; then
                local key="${ISSUES[$idx]}"
                case "$key" in
                    "NTP")
                        fix_disable_ntp
                        ;;
                    "DNS_TIMEOUT"|"DNS")
                        fix_dns_dot
                        ;;
                    "BBR")
                        fix_bbr
                        ;;
                    "CONNTRACK")
                        fix_conntrack
                        ;;
                    "KERNEL")
                        if [[ "$OS" =~ centos|rhel ]] && [[ "$VER" == "7"* ]]; then
                            fix_kernel_centos7
                        else
                            echo -e "${RED}当前系统无需升级内核${NC}"
                        fi
                        ;;
                    "NET")
                        echo -e "${RED}网络不通，请先检查网络配置${NC}"
                        ;;
                    *)
                        echo -e "${RED}未找到对应的优化函数${NC}"
                        ;;
                esac
            else
                echo -e "${RED}无效选项: $choice${NC}"
            fi
        done
        
        # 优化后重新检测相关项
        echo -e "\n${BLUE}正在重新检测以确认优化效果...${NC}"
        sleep 1
        ISSUES=()
        OPT_DESC=()
        check_udp_block
        check_dns_config
        check_bbr
        check_conntrack
        check_ntp_service
        echo -e "\n${BLUE}==================== 优化后状态 ====================${NC}"
        if [ ${#ISSUES[@]} -eq 0 ]; then
            echo -e "${GREEN}🎉 所有优化项已成功处理！服务器状态优秀。${NC}"
        else
            echo -e "${YELLOW}以下项目仍需关注:${NC}"
            for i in "${!OPT_DESC[@]}"; do
                echo -e "  ${CYAN}$((i+1)))${NC} ${OPT_DESC[$i]}"
            done
        fi
    fi
}

# ========== 主程序 ==========
main() {
    # 不执行 clear 避免无输出
    echo -e "${GREEN}======================================${NC}"
    echo -e "${GREEN}   服务器一键检测与优化脚本 v2.2    ${NC}"
    echo -e "${GREEN}======================================${NC}"
    detect_os
    echo -e "${BLUE}操作系统: ${YELLOW}$OS $VER${NC}\n"
    
    check_network_basic
    check_udp_block
    check_dns_config
    check_bbr
    check_conntrack
    check_ntp_service
    check_kernel
    
    show_summary
}

main
EOFSCRIPT

chmod +x /root/server_check.sh
