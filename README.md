# luci-app-shellinabox

面向 OpenWrt / ImmortalWrt 的 LuCI 网页终端，包含 `shellinabox` 后端和 `luci-app-shellinabox` 前端，可作为 `ttyd` / `luci-app-ttyd` 的替代方案。

本项目的重点是 **CGI 模式按需启动，不需要常驻 ShellInABox 后台服务**。相较于 `ttyd`，ShellInABox 的终端响应和大量输出时的性能通常会差一些，适合偶尔登录路由器执行命令、排查问题，希望减少闲置时后台进程的场景；如果更看重高频使用和大量终端输出的流畅度，建议继续使用 `ttyd`。这里没有提供定量性能测试数据。

## 工作方式

- 登录 LuCI 后，打开“系统 → 终端”才会通过受认证和 CSRF 保护的请求启动 `shellinaboxd --cgi`。
- 每次启动的终端实例从配置的端口范围中选择一个端口，默认范围为 `4200–4210`，浏览器直接连接该端口。
- 会话运行期间仍有后端进程；断开后由 ShellInABox 的空闲超时机制回收，并不是关闭页面就立即退出，也不是每个终端请求都重新启动一次进程。
- LuCI 插件的初始化脚本会停止并禁用独立的 `shellinaboxd` 服务；后端启动脚本检测到 `/etc/config/shellinabox` 时也不会启动常驻实例。不要再手动启用独立服务。
- LuCI / uhttpd 等现有管理服务仍需运行，“不常驻”指的是 ShellInABox 后端。

## 手动作为 feed 构建

以下命令在已经准备好编译依赖的 OpenWrt / ImmortalWrt 源码根目录执行，也适用于匹配设备架构和系统版本的 SDK。需要使用支持 LuCI JavaScript 视图和 ucode 控制器的版本，并提前更新、安装标准的 `packages`、`luci` 等依赖 feed。

把本仓库加入 OpenWrt / ImmortalWrt 源码树：

```sh
echo "src-git shellinabox https://github.com/altman08/luci-app-shellinabox.git" >> feeds.conf.default
./scripts/feeds update shellinabox
./scripts/feeds install -f -p shellinabox shellinabox luci-app-shellinabox
```

如果使用 `feeds.conf`，则将第一条命令中的文件名改为 `feeds.conf`。已有同名 feed 时应替换原条目，不要重复追加。安装命令使用 `-f -p shellinabox`，确保使用本仓库的前后端包，避免其他 feed 中的同名包覆盖本项目的 CGI 集成。

选择包：

```sh
make menuconfig
```

先选择目标平台及设备，再选择：

```text
Network -> shellinabox
LuCI -> Applications -> luci-app-shellinabox
```

或者直接写入 `.config`（已有同名配置项时先修改或移除，避免重复）：

```sh
cat >> .config <<'EOF'
CONFIG_PACKAGE_shellinabox=m
CONFIG_PACKAGE_luci-app-shellinabox=m
CONFIG_PACKAGE_luci-i18n-shellinabox-zh-cn=m
EOF
make defconfig
```

`m` 表示生成独立软件包，不内置到固件；如需编入固件，改为 `y`（菜单中的 `<*>`）。中文翻译包为可选项，不需要时可以省略对应配置。

编译：

```sh
make package/shellinabox/compile V=s
make package/luci-app-shellinabox/compile V=s
```

中文翻译包由 LuCI 构建流程作为前端的子包生成，不需要单独的源码编译目标。

生成的安装包通常位于：

```text
bin/packages/<arch>/shellinabox/
```

具体为 `.ipk` 还是 `.apk` 取决于所用源码或 SDK 的包管理格式。如需编译完整固件，将相关包选为 `y` 后执行 `make -j"$(nproc)"`，固件产物位于 `bin/targets/<目标平台>/<子目标>/`。

如果使用本地仓库目录，可将上述 `src-git` 条目替换为：

```text
src-link shellinabox /path/to/luci-app-shellinabox
```

`src-link` 后面必须是本地目录路径，不能填写 Git URL；目录应指向同时包含 `luci-app-shellinabox/` 和 `net/` 的本仓库根目录。配置后执行相同的 feed 更新、安装和编译命令。

## 使用与注意事项

安装前端及后端后，在 LuCI 的“系统 → 终端”中使用；终端的“配置”页可调整启用状态、端口范围、运行用户、工作目录、启动命令、配色和 SSL。配置保存在 `/etc/config/shellinabox`，修改对新启动的终端实例生效。

- 默认启动命令为 `/bin/login`，仍需进行系统账户登录。不要随意改成无认证的 root shell。
- 浏览器必须能够访问路由器的终端端口范围。只转发或代理 LuCI 的 HTTP/HTTPS 端口不够；跨网段、VPN 或反向代理访问时，需要另外考虑终端端口的可达性和访问控制。
- 默认 UCI 配置为 `ssl=0`。如果使用 HTTPS 访问 LuCI，需要先在插件配置中启用 SSL，避免浏览器拦截混合内容。
- 启用 SSL 并使用默认的 `/etc/shellinabox/certificate.pem` 时，会在首次启动 HTTPS 终端时生成路由器自己的自签名证书。浏览器可能需要先单独访问终端端口并信任该证书。
- 仅在可信管理网络使用，不建议将 LuCI 或终端端口直接暴露到公网。LuCI 启动入口的认证不意味着终端端口可以不做网络访问限制。
- 如果原来安装了 `ttyd`，应自行停止并禁用其服务；本项目不会自动关闭 `ttyd`。确认新终端满足需求后，再按需移除旧软件包。
