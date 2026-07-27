# Peer Configurations

Place individual peer configurations here, one file per peer.

## Naming convention

```
/etc/bird/peers/<peer_name>.conf
```

## Example dual-stack peer

```bird
protocol bgp example from dnpeers {
    neighbor 172.23.x.x as 424242xxxx;
}

protocol bgp example_v6 from dnpeers {
    neighbor fdxx:xxxx:xxxx::x as 424242xxxx;
}
```

## Example IPv4-only peer

```bird
protocol bgp example_v4 from dnpeers_v4 {
    neighbor 172.23.x.x as 424242xxxx;
}
```

## Example IPv6-only peer

```bird
protocol bgp example_v6 from dnpeers_v6 {
    neighbor fdxx:xxxx:xxxx::x as 424242xxxx;
}
```

## Community values

Adjust the `(latency, bandwidth, crypto)` tuple in `dnpeers` template according to actual link quality:

| Latency | Bandwidth | Crypto |
|---------|-----------|--------|
| 1: <2.7ms | 21: >=0.1M | 31: plain |
| 2: <7.3ms | 22: >=1M | 32: unsafe |
| 3: <20ms | 23: >=10M | 33: safe |
| 4: <55ms | 24: >=100M | 34: safe+PFS |
| 5: <148ms | 25: >=1G | |

Default in template: `(3, 24, 33)`
