# Network Debugging Commands

Command-by-command playbooks for connectivity, DNS, TCP, protocol-specific, performance and security debugging, plus worked scenarios.

> Merged from 2024–2025 Meta Production Engineer prep notes. Overlapping copies were deduplicated; the newer, denser notes in this folder take precedence where they disagree.

# Network Debugging Guide

## Basic Connectivity Issues

### First Response Commands
1. Check if network interface is up:
```bash
# Check interface status
ip link show
# or traditional way
ifconfig -a

# Check specific interface (e.g., eth0)
ip link show eth0

# Enable interface if down
sudo ip link set eth0 up
```

2. Verify IP configuration:
```bash
# Check IP address assignment
ip addr show
# Check routing
ip route show
# Check DNS resolution
cat /etc/resolv.conf
```

3. Test basic connectivity:
```bash
# Test loopback
ping -c 4 127.0.0.1

# Test local network
ping -c 4 192.168.1.1

# Test internet connectivity
ping -c 4 8.8.8.8

# Test DNS resolution
ping -c 4 google.com
```

### Network Interface Statistics
```bash
# Check interface statistics
ip -s link show eth0

# Monitor interface traffic
sar -n DEV 1
# or
netstat -i

# Check for errors/drops
ethtool -S eth0
```

## DNS Troubleshooting

### DNS Resolution
1. Test DNS resolution:
```bash
# Basic DNS lookup
dig google.com

# Trace DNS resolution path
dig +trace google.com

# Check specific DNS server
dig @8.8.8.8 google.com

# Reverse DNS lookup
dig -x 8.8.8.8
```

2. DNS Configuration:
```bash
# Check local DNS configuration
cat /etc/resolv.conf

# Test with different resolvers
nslookup google.com 8.8.8.8

# Check DNS response times
dig +stats google.com
```

### Advanced DNS Debugging
```bash
# Check DNS cache (if running local resolver)
systemd-resolve --statistics

# Flush DNS cache
systemd-resolve --flush-caches

# Test DNS over specific protocol
dig +tcp google.com  # Force TCP
```

## TCP/IP Analysis

### Connection Status
1. View all connections:
```bash
# List all connections
ss -tunapl

# Check established connections
netstat -tunapl | grep ESTABLISHED

# Monitor real-time connections
watch -n 1 'netstat -tunapl'
```

2. Port Analysis:
```bash
# Check listening ports
sudo lsof -i -P -n | grep LISTEN

# Check specific port
sudo lsof -i:80

# Monitor port connections
tcpdump -i any port 80
```

### TCP Performance
```bash
# Check TCP parameters
sysctl -a | grep net.ipv4.tcp

# Monitor TCP metrics
netstat -s | grep -i tcp

# Check connection latency
mtr google.com
```

## Protocol-Specific Debugging

### HTTP/HTTPS
1. Test HTTP endpoints:
```bash
# Basic HTTP request
curl -v http://example.com

# Test with headers
curl -I http://example.com

# Test HTTPS with cert info
curl -v --insecure https://example.com
```

2. Monitor HTTP traffic:
```bash
# Capture HTTP traffic
tcpdump -i any -A 'tcp port 80'

# Monitor with headers
sudo tcpflow -i any -c port 80
```

### DHCP
```bash
# Check DHCP lease
journalctl -u dhclient

# Monitor DHCP messages
tcpdump -i any port 67 or port 68 -n

# Release/renew DHCP lease
sudo dhclient -r eth0
sudo dhclient eth0
```

## Performance Analysis

### Network Performance
1. Bandwidth Testing:
```bash
# Install iperf3
sudo apt install iperf3

# Run iperf server
iperf3 -s

# Run iperf client
iperf3 -c server_ip
```

2. Latency Analysis:
```bash
# Check network latency
ping -c 10 target_ip | awk '/^rtt/ {print $4}'

# Monitor network latency over time
mtr -n target_ip

# Check routing path
traceroute -n target_ip
```

### System Resource Impact
```bash
# Monitor network-related CPU usage
mpstat -P ALL 1

# Check network buffer usage
netstat -m

# Monitor network device interrupts
watch -n1 'cat /proc/interrupts'
```

## Security Analysis

### Firewall Checking
1. Check firewall rules:
```bash
# List iptables rules
sudo iptables -L -v -n

# Check specific chain
sudo iptables -L INPUT -v -n

# Monitor firewall logs
sudo tail -f /var/log/iptables.log
```

2. Connection Tracking:
```bash
# View connection tracking
cat /proc/net/nf_conntrack

# Check conntrack statistics
cat /proc/sys/net/netfilter/nf_conntrack_count
```

### Security Scanning
```bash
# Port scanning
nmap -sS -p- localhost

# Service version detection
nmap -sV target_ip

# Check for open ports
netstat -tulpn
```

## Additional Useful Tools

### Network Monitoring
```bash
# Continuous network monitoring
iftop -i eth0

# Packet analysis
tcpdump -i any -w capture.pcap

# Analyze captured packets
wireshark capture.pcap
```

### System Logging
```bash
# Check system logs
journalctl -u networking.service

# Monitor kernel network messages
dmesg | grep -i eth

# Check authentication logs
tail -f /var/log/auth.log
```

## Common Issues and Solutions

1. **High Latency**
```bash
# Check network congestion
iperf3 -c server_ip
# Monitor queue/buffer
tc -s qdisc show dev eth0
# Check for packet drops
netstat -i | grep -i drop
```

2. **Packet Loss**
```bash
# Monitor packet loss
ping -f -c 1000 target_ip | grep loss
# Check interface errors
ethtool -S eth0 | grep -i error
# Monitor dropped packets
dropwatch -l kas
```

3. **DNS Issues**
```bash
# Test DNS servers
for server in $(cat /etc/resolv.conf | grep ^nameserver | cut -d' ' -f2); do
    echo "Testing $server"
    dig @$server google.com
done

# Check DNS response times
dig +stats google.com
```

## Debug Example Scenarios

1. **Web Service Not Accessible**
```bash
# Check service status
systemctl status nginx

# Check port binding
ss -tulpn | grep :80

# Check logs
tail -f /var/log/nginx/error.log

# Test local connection
curl -v localhost:80

# Check firewall
sudo iptables -L | grep 80
```

2. **Slow Network Performance**
```bash
# Check interface speed
ethtool eth0

# Monitor bandwidth usage
iftop -i eth0

# Check for bottlenecks
nethogs eth0

# Monitor TCP retransmissions
netstat -s | grep retransmit
```

3. **Intermittent Connectivity**
```bash
# Monitor connection stability
mtr -n target_ip

# Check for interface errors
ethtool -S eth0

# Monitor link status
watch -n 1 'ip link show eth0'

# Check system logs
journalctl -f
```

### Practical Interview Questions Related to Debugging:

1. "How would you troubleshoot a web service that's accessible locally but not remotely?"

Steps:
1. Verify local service status:
```bash
# Check if service is running and listening
netstat -tuln | grep <port>
ss -tuln | grep <port>
```

2. Check firewall rules:
```bash
# Check iptables rules
iptables -L -n
# Check if port is blocked
sudo ufw status
```

3. Verify network connectivity:
- Test remote connectivity using telnet/curl from another machine
- Check if the service is bound to localhost only vs all interfaces (0.0.0.0)
- Review service configuration for binding addresses

4. DNS resolution:
```bash
# Test DNS resolution
nslookup <service_domain>
dig <service_domain>
```

5. Check system logs:
```bash
# Review service logs
journalctl -u <service_name>
# Check system logs
tail -f /var/log/syslog
```

2. "What steps would you take to diagnose high latency issues between two services?"

1. Baseline measurements:
```bash
# Measure current latency
ping <destination>
traceroute <destination>
```

2. Network path analysis:
```bash
# Check network path
mtr <destination>
# Look for packet loss or high latency hops
```

3. Monitor system resources:
```bash
# Check system load
vmstat 1
# Monitor network interfaces
sar -n DEV 1
```

4. Application-level monitoring:
- Review application logs for slow queries/requests
- Use application performance monitoring tools
- Check for connection pooling issues

5. Use specialized tools:
```bash
# Network monitoring
iptraf
# Packet analysis
tcpdump -i any host <destination>
```

3. "How would you investigate a sudden increase in network traffic?"

1. Identify traffic patterns:
```bash
# Monitor network interfaces
iftop -i <interface>
# Check network statistics
netstat -s
```

2. Analyze traffic by type:
```bash
# Capture and analyze packets
tcpdump -i any -w capture.pcap
# Analyze with Wireshark for detailed inspection
```

3. Monitor system metrics:
```bash
# Check system load
top
# Monitor IO
iostat -xz 1
```

4. Process investigation:
```bash
# Find processes using network
lsof -i
# Check per-process network usage
nethogs
```

4. "What tools would you use to monitor real-time network performance?"

Key tools:
1. Basic monitoring:
```bash
# Interface statistics
ifconfig
ip -s link
```

2. Real-time monitoring:
```bash
# Network traffic
iftop
# Bandwidth usage
nethogs
# System activity
dstat --net
```

3. Advanced monitoring:
```bash
# Detailed network statistics
netstat -s
# Interface monitoring
sar -n DEV 1
```

4. Application level:
- Nagios/Prometheus for metrics collection
- Grafana for visualization
- Custom application metrics

5. "How would you debug DNS resolution issues in a production environment?"

Steps:
1. Check DNS configuration:
```bash
# Review DNS settings
cat /etc/resolv.conf
# Test DNS servers
dig @<dns_server> <domain>
```

2. Verify DNS resolution:
```bash
# Test forward/reverse lookups
nslookup <domain>
host <domain>
```

3. Check DNS propagation:
```bash
# Check DNS propagation across servers
dig +trace <domain>
```

4. Monitor DNS performance:
```bash
# DNS response times
dig <domain> +stats
```

6. "Explain how you would identify and resolve network bottlenecks"

Process:
1. Baseline measurement:
```bash
# Network utilization
sar -n DEV 1
# Interface statistics
ethtool -S <interface>
```

2. Identify saturation points:
```bash
# Check drops/errors
netstat -i
# Monitor queues
tc -s qdisc show
```

3. Performance analysis:
```bash
# Bandwidth monitoring
iperf3 -c <server>
# Latency monitoring
ping with various packet sizes
```

7. "What metrics would you monitor to ensure network health?"

Key metrics:
1. Interface metrics:
- Bandwidth utilization
- Packet loss
- Error rates
- Interface drops

2. Connection metrics:
- Connection states
- Retransmission rates
- TCP window sizes
- Connection timeouts

3. System metrics:
- CPU usage
- Memory utilization
- Interrupt handling
- Network buffer usage

4. Application metrics:
- Request latency
- Error rates
- Connection pool status
- Response times

8. "How would you troubleshoot SSL/TLS issues in a web application?"

Steps:
1. Certificate verification:
```bash
# Check certificate validity
openssl x509 -in cert.pem -text
# Test SSL connection
openssl s_client -connect host:port
```

2. Configuration review:
- Check SSL/TLS versions enabled
- Review cipher suites
- Verify certificate chain

3. Connection testing:
```bash
# Test SSL handshake
curl -v https://domain
# Check SSL configuration
nmap --script ssl-enum-ciphers -p 443 domain
```

9. "What steps would you take to diagnose packet loss?"

Approach:
1. Measure packet loss:
```bash
# Continuous ping test
ping -c 100 destination
# MTR for hop-by-hop analysis
mtr destination
```

2. Interface statistics:
```bash
# Check interface errors
ifconfig
# Monitor drops
netstat -i
```

3. Network analysis:
```bash
# Capture packets
tcpdump -i any host destination
# Analyze retransmissions
tcp_probe monitoring
```

10. "How would you identify the root cause of connection timeouts?"

Investigation steps:
1. Connection testing:
```bash
# Test basic connectivity
telnet host port
# Check TCP connection state
netstat -np | grep host
```

2. Resource monitoring:
```bash
# Check system resources
vmstat 1
# Monitor network stack
netstat -s
```

3. Network path analysis:
```bash
# Path analysis
traceroute host
# Connection tracking
conntrack -L
```

4. Application logs:
- Review application error logs
- Check for connection pool exhaustion
- Monitor application metrics
