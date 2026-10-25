# sing-box-udpadv

基于 [sing-box](https://github.com/SagerNet/sing-box) **v1.12.8**（commit `573c6179ab4f`）的补丁版，给 `socks` / `mixed` 入站加了一个选项：`udp_advertise_address`。

## 解决的问题

在 1:1 NAT 的云服务器上（例如阿里云 EIP 的 NAT 模式），公网 IP 不在网卡上，sing-box 只能监听私网 IP。
客户端发起 SOCKS5 `UDP ASSOCIATE` 时，sing-box 回复的中继地址（BND.ADDR）是**私网 IP**，
严格按协议实现的客户端会把 UDP 发到这个私网地址，结果 UDP 不通（TCP 不受影响）。

设置 `udp_advertise_address` 后，回复里改为报告这个公网 IP；中继 socket 仍然绑定在原来的私网地址上。
不设置时行为与官方 sing-box 完全一致。

```json
{
  "type": "socks",
  "tag": "in-1",
  "listen": "10.0.0.5",
  "listen_port": 10001,
  "users": [{ "username": "user", "password": "pass" }],
  "udp_advertise_address": "203.0.113.5"
}
```

## 下载

二进制就在仓库根目录，可直接下载：

| 文件 | 说明 | 直链 |
|---|---|---|
| `sing-box-linux-amd64` | Linux x86_64，静态编译（CGO 关闭） | https://raw.githubusercontent.com/drvcode-code/sing-box-udpadv/main/sing-box-linux-amd64 |
| `sing-box-linux-arm64` | Linux ARM64，静态编译（CGO 关闭） | https://raw.githubusercontent.com/drvcode-code/sing-box-udpadv/main/sing-box-linux-arm64 |
| `SHA256SUMS` | 校验值 | |

```bash
curl -fsSL -o /usr/local/bin/sing-box https://raw.githubusercontent.com/drvcode-code/sing-box-udpadv/main/sing-box-linux-amd64
chmod +x /usr/local/bin/sing-box
```

编译标签与官方发布版一致：`with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale`，
版本号显示为 `1.12.8-udpadv`。

## 源码与编译

完整改动见 [`udp-advertise-address.patch`](udp-advertise-address.patch)（4 个文件：
`option/simple.go`、`protocol/socks/inbound.go`、`protocol/mixed/inbound.go`、新增 `common/listener/listener_advertise.go`）。

```bash
git clone https://github.com/SagerNet/sing-box.git && cd sing-box
git checkout v1.12.8
git apply ../udp-advertise-address.patch
# Go 1.25.1
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath \
  -tags "with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale" \
  -ldflags "-X github.com/sagernet/sing-box/constant.Version=1.12.8-udpadv -s -w -buildid=" \
  -o sing-box-linux-amd64 ./cmd/sing-box
```

## 许可证

与上游 sing-box 相同：GPL-3.0-or-later，见 [LICENSE](LICENSE)。
