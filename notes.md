# Ideas

1 Timestamp as part of message text if the message is sent later due to bad internet connection
2 Offline storage on device (bad internet connection)
3 system font: font-family: ui-system, sans-serif;
4 perfom mcp google accessibility check
5 Reorganising CSS?

* other AI suggested optimisations, like compressing the file?

## FTP deployment — learned the hard way

**ECONNRESET on data socket (kasserver + GitHub Actions)**

Root cause: Node.js connects to the FTP server via an IPv4-mapped IPv6 socket (`::ffff:x.x.x.x`). The `basic-ftp` library (used by SamKirkland/FTP-Deploy-Action) reads `socket.remoteFamily === 'IPv6'` and picks EPSV for the data connection. Kasserver drops IPv6 data connections → ECONNRESET during upload.

Fix: disable IPv6 at the kernel level before the deploy step so Node.js is forced to use a pure IPv4 socket:
```yaml
- name: Disable IPv6
  run: |
    sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
    sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

**Does NOT work:** modifying `/etc/gai.conf` — that only changes DNS address preference, not the socket address family.