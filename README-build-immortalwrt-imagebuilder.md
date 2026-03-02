# ImmortalWRT ImageBuilder (x86_64 EFI) 构建说明

## 这个 workflow 做什么
这个 workflow 用于在 GitHub Actions 上，基于 **ImmortalWRT 官方 ImageBuilder** 自动构建自定义固件，不走全源码 `make world`。

目标镜像固定为：
- 架构：`x86_64`
- 规格：`generic-squashfs-combined-efi`

该镜像可用于 **PVE 虚拟机** 场景。

## 关键实现点
- 使用 `ubuntu-latest` runner。
- 自动下载指定版本的 ImmortalWRT ImageBuilder（主 URL 失败时自动重试备用镜像）。
- ImageBuilder 命中哪个镜像源，就自动把默认包仓库地址切到同一镜像源，避免下载到 tar 包但后续包索引仍走不可达主站。
- 先接入第三方 feed（默认 Nikki），再执行 `make image`。
- Nikki 公钥会通过 `opkg-key add` 导入，确保 `Packages.sig` 验签可用。
- 在构建前自动补齐本地 `packages/Packages.gz` 占位索引，避免部分 release 在 CI 中触发 `package_index` 失败。
- 打包自定义软件包并上传 artifacts。
- 上传前会校验是否存在 `*generic-squashfs-combined-efi.img.gz`，若不存在直接失败。

## 关键可修改变量
在文件 `.github/workflows/build-immortalwrt-imagebuilder.yml` 的 `env` 中修改：

- `IMMORTALWRT_VERSION`
  - ImmortalWRT release 版本。
- `TARGET` / `SUBTARGET` / `PROFILE`
  - 当前默认分别是 `x86` / `64` / `generic`。
- `IMAGEBUILDER_URL`
  - 为空时会按版本自动拼接官方下载地址。
  - 若你有自定义镜像源，可填完整 URL。
- `IMAGEBUILDER_FALLBACK_BASE_URLS`
  - 仅当 `IMAGEBUILDER_URL` 为空时生效。
  - 表示官方下载失败后依次重试的镜像源列表（空格分隔）。
- `BASE_PACKAGES`
  - 官方/默认 feed 包列表（可继续追加）。
- `EXTRA_PACKAGES`
  - 额外包列表（可继续追加）。
- `NIKKI_FEED_NAME`
  - 默认 `nikki`。
- `NIKKI_FEED_URL`
  - 默认 `https://nikkinikki.pages.dev`。
- `OPKG_CHECK_SIGNATURE`
  - 默认 `0`（更适合第三方 feed 的 CI 构建稳定性）。
  - 设为 `1` 可启用签名校验（需要可用的签名 key 与 `usign` 环境）。
- `ARTIFACT_RETENTION_DAYS`
  - artifact 保留天数。

## 如何手动触发 GitHub Actions
1. 把本仓库推送到 GitHub。
2. 进入 GitHub 仓库的 `Actions` 页面。
3. 选择 `Build ImmortalWRT ImageBuilder x86_64 EFI`。
4. 点击 `Run workflow` 手动触发。

## 构建产物是什么
上传的 artifact 名称默认是：
- `immortalwrt-x86_64-efi`

包含至少以下镜像文件：
- `*generic-squashfs-combined-efi.img.gz`

并在存在时附带：
- `sha256sums`

## 后续如何追加软件包
- 常规包：直接追加到 `BASE_PACKAGES` 或 `EXTRA_PACKAGES`。
- 建议把“官方常规包”和“第三方 feed 包”分开维护，方便排错。

## 后续如何替换或新增第三方 feed
当前 workflow 已把 Nikki 作为模板流程：
1. 追加 feed 配置到 `repositories.conf`（必要时同步到 `repositories`）。
2. 下载该 feed 的签名 key 到 ImageBuilder 的 `keys/`。
3. 刷新包索引并确认目标包可见。
4. 再执行 `make image`。

你新增其他第三方 feed 时，按同样步骤加一个独立配置段即可。

## 为什么 nikki / luci-app-nikki 不能只写进包列表
必须先说明这个区别：
- `luci-app-wechatpush` 可先按普通包处理（在兼容 release 下通常来自默认可访问包源）。
- `nikki` / `luci-app-nikki` 属于第三方 feed 包。

如果不先接入第三方 feed，ImageBuilder 只看到默认源，`make image` 时会报找不到包。

## 版本兼容性说明（重要）
若某个 ImmortalWRT release 与 Nikki feed 不兼容，常见处理方式：
1. 切换到兼容的 ImmortalWRT release。
2. 调整第三方 feed（地址/分支/包版本）。
3. 改用完整源码集成方案。

本次方案仍然是：**ImageBuilder + 第三方 feed**。
