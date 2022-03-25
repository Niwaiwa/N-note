###### tags: `Linux`

# Linux 網路類指令

## 網路類(是否有通)查詢指令

### tcpdump 監控即時流量

```
UDP 4789:
tcpdump -i ens160 udp port 4789 -nn
TCP/UDP 7946
tcpdump -i ens160 port 7946 -nn
```

### nc 測試port是否有通

```
TCP:
nc 202.133.245.177 2377 -vz
UDP:
nc 202.133.245.177 7946 -vzu
```
