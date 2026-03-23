# MDDS Decentralized Discovery Benchmark Results

## Test Environment

- **Platform**: Linux 5.15.0-91-generic
- **Test Date**: 2026-03-23
- **Multicast Address**: 239.255.0.1
- **Discovery Port**: 7411
- **Data Port**: 7412

## Test Configuration

Each publisher instance sends 10 data messages. Each subscriber receives messages from all publishers via multicast.

## Results

| Instances (Pub+Sub) | Messages Sent | Messages Received | Publish Time | Notes |
|---------------------|---------------|-------------------|--------------|-------|
| 10 + 10             | 100           | 1,000             | ~2007 ms     | All expected messages received |
| 20 + 20             | 200           | 4,000             | ~2025 ms     | All expected messages received |
| 30 + 30             | 300           | 9,000             | ~2031 ms     | All expected messages received |
| 50 + 50             | 500           | 18,335            | ~2073 ms     | Limited by igmp_max_memberships=20 |

## Observations

### 1. Scalability
- Discovery time scales linearly with number of instances
- Publish operation time remains relatively constant (~2 seconds for all publishers)
- Message delivery scales with N×M (publishers × subscribers)

### 2. Multicast Group Limits
- System limit: `igmp_max_memberships = 20`
- Only 20 instances can join the multicast group simultaneously
- This limits effective subscriber count to 20 for reliable delivery
- Beyond 20 subscribers, some messages are lost

### 3. Performance Characteristics
- **Instance Creation**: ~5-6 ms for 10 instances
- **Discovery**: Automatic via multicast announcements every 5 seconds
- **Data Distribution**: Near real-time delivery via UDP multicast

## Test Code

```cpp
// Benchmark participants
./discovery_benchmark -n <count> -p  // Run as publishers
./discovery_benchmark -n <count> -s  // Run as subscribers
```

## Recommendations

1. **For large deployments**: Use multiple multicast groups to work around the 20-member limit
2. **For critical data**: Implement acknowledgment/retransmission mechanism
3. **For cross-subnet**: Configure router for multicast (TTL > 1)

## System Limits

```
/proc/sys/net/ipv4/igmp_max_memberships = 20
```

To increase temporarily:
```bash
sudo sysctl -w net.ipv4.igmp_max_memberships=100
```
