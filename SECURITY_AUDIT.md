# 仓库安全审计报告

审计日期：2026-08-01

审计对象：当前 Git 工作树（初始基线提交 `d3db8184`）

项目标识：`release_info.json` 声明版本 `v0.99.1`、上游提交 `7ac1f450`

## 结论摘要

在可读取的 C#、GDScript、Godot 配置、场景/资源描述和依赖清单中，**未发现明确的恶意后门、凭据窃取、持久化植入或经过混淆的下载执行器**。审计时 `git fsck --full` 通过，工作树在修改前没有已跟踪改动、没有未跟踪文件，也没有符号链接或设置了可执行位的已跟踪脚本。

但是，不能据此证明该副本与 Mega Crit 官方发行物逐字节一致：仓库仅有一个无签名初始提交、没有配置远程仓库，也没有官方哈希/签名可用于建立可信根。尤其是仓库内含大量预编译 GDExtension、FMOD、Sentry、Spine 和 Crashpad 二进制文件，纯静态源码检查无法证明这些二进制没有被篡改。

本次确认并修复了一处编辑器插件中的命令注入风险：`addons/dev_tools/dev_tools.gd` 原先把可配置的 `dotnet` 路径及缓存路径拼接到 `cmd /c` 或 `bash -c`，路径中的 shell 元字符可能执行额外命令；现已改为直接调用 `dotnet` 并以参数数组传参。

## 范围与方法

审计覆盖 18,989 个 Git 跟踪文件，并执行了以下检查：

1. Git 对象完整性、工作树状态、历史、远程、符号链接和可执行位检查。
2. 枚举脚本、程序集、原生库、可执行程序、PCK 和归档等高风险文件类型。
3. 检查 Godot 自动加载项、编辑器插件、主场景和 GDExtension 映射。
4. 全仓搜索进程创建、shell、动态程序集加载、反射、原生互操作、网络、文件写入、环境变量、编码/混淆及常见秘密模式。
5. 人工复核命中的编辑器命令执行、Mod 加载、遥测、Sentry、反馈上传和外部 URL。
6. 检查 NuGet 锁文件与项目构建设置。

### 审计限制

- 环境没有 `dotnet`、Godot、ClamAV、YARA 和 `file`，因此未能编译项目、运行测试、进行恶意软件特征扫描或系统化识别所有二进制格式。
- 未执行仓库中的任何原生库或游戏代码，避免在可信度尚未建立时扩大风险。
- 没有官方发布包、官方源码仓库、代码签名证书或软件物料清单（SBOM）作为对照，无法判断初始提交之前的供应链篡改。
- 静态搜索可能存在漏报；反射、PCK、原生代码以及运行时下载的 Steam Workshop Mod 均会显著增加动态行为范围。

## 主要发现

### A-01：缺少可验证的官方来源基线（高风险，未修复）

当前 Git 历史只有单个 `Initial commit`，没有远程地址，也没有签名标签。`release_info.json` 中的提交号只是普通文本，不能证明文件来自该提交。攻击者若在导入仓库之前同时替换源码、二进制和该元数据，Git 完整性检查仍会通过。

**建议：** 从官方签名渠道重新取得相同版本；记录安装包/可执行文件及所有原生库的 SHA-256；验证 Steam depot manifest、平台代码签名和发布者证书；将结果作为只读基线保存。未完成此步骤前，不应把“Git 干净”解释成“官方且未被篡改”。

### A-02：大量不透明原生二进制会在编辑器或游戏启动时加载（高风险，需验证）

`project.godot` 启用了 FMOD 编辑器插件并自动加载 Sentry 与 FMOD。`addons/fmod/fmod.gdextension`、`addons/sentry/sentry.gdextension` 和 `bin/spine_godot_extension.gdextension` 会按平台加载 DLL、SO、framework、dylib 或 WASM；Sentry 还携带 Crashpad handler 可执行程序。这些组件具有当前用户权限，若被替换可绕过所有 C#/GDScript 层面的审计。

发现数个以 `~` 开头的二进制副本。FMOD 和 Spine 的 `~` 文件与对应非 `~` 文件 SHA-256 相同；Sentry 的 `~libsentry.windows.debug.x86_64.dll` 与非 `~` 文件不同，且没有配置引用该 `~` 文件。它可能是导出/恢复工具遗留版本，但在取得供应商哈希前不能确认。

**建议：** 在来源验证前隔离整个仓库，不要用 Godot 打开；逐一对照 FMOD、Sentry、Spine 官方发行包和签名；删除或隔离未被配置引用的 `~` 副本；在 CI 中生成并校验原生文件哈希清单。

### A-03：Mod 机制按设计执行任意第三方代码（高风险，设计行为）

`ModManager` 递归发现本地和 Steam Workshop manifest。在用户同意且 Mod 未禁用后，它可通过 `AssemblyLoadContext.LoadFromAssemblyPath` 加载 DLL、通过 `ProjectSettings.LoadResourcePack` 挂载 PCK，然后调用标注的初始化方法或对程序集执行 Harmony `PatchAll`。这不是沙箱：Mod 能获得与游戏相同的文件、网络和进程权限，并可修改任意托管方法。

已有的用户确认和 `nomods` 参数降低误加载概率，但不能约束已获准 Mod 的能力。Steam Workshop 更新后发现的内容也应视为不受信代码。

**建议：** 审计期间始终使用 `nomods`；对 Mod 包实施发布者签名、内容哈希和明确的更新确认；在受限操作系统账户/容器中运行 Mod；UI 应明确说明“Mod 可执行任意代码”，而不只描述游戏稳定性风险。

### A-04：编辑器插件 shell 命令注入（高风险，已修复）

启用的 `dev_tools` 插件原先读取编辑器设置中的 `dotnet/dotnet_cli_path`，并连同缓存目录插入 shell 命令。文件存在性检查不能排除文件名含引号或 shell 元字符，因此恶意项目设置或特殊路径可以在点击 “Clear NuGet Cache” 时注入命令。

修复后不再调用 `cmd`/`bash`，而是用 `OS.execute(dotnet_path, ["nuget", "locals", "all", "--clear"], ...)` 直接传递参数，也不再进行并非该命令所需的目录切换。

### A-05：存在外发遥测、崩溃与反馈数据（中风险，已披露/有部分同意控制）

项目配置内包含 Sentry DSN，启动时自动加载 Sentry。托管 Sentry 设置 `SendDefaultPii = false`，在正常初始化路径中用 `UploadData` 偏好检查用户同意，并在加载 Mod 时阻止事件；异常报告仍会附带场景、种子、进度、角色和房间等游戏状态。另有 GDExtension Sentry 实现，应独立确认它与托管实现采用一致的同意策略。

运行指标会上传到 `sts2-metric-uploads.herokuapp.com`，包含稳定的玩家 ID、构筑、选择、系统内存、语言与显示设置等。代码会在发行版、非编辑器且 `UploadData` 为真时上传，但 `PrefsSave.UploadData` 默认值为 `true`。反馈功能会向 `feedback.sts2.megacrit.com` 上传用户输入和截图，其地址可被进程环境变量 `STS2_FEEDBACK_URL` 覆盖；能控制进程环境的本地程序可借此把反馈导向其他服务器。

**建议：** 首次运行采用明确的 opt-in，而不是默认开启；在发送前展示字段和目标域名；统一审查两个 Sentry SDK 的同意门；生产构建忽略反馈 URL 环境覆盖，或限制为 HTTPS 且执行主机 allowlist；对日志 breadcrumb 做敏感信息清洗。

### A-06：构建与供应链加固不足（中风险，未修复）

NuGet 依赖有锁文件和 content hash，这是积极控制；但项目允许动态加载和 unsafe 代码，引用两个开发机绝对路径下的 `Steamworks.NET.dll` 与 `0Harmony.dll`，且未提交这些程序集的可信哈希。SDK 设置 `rollForward: latestMajor`，未来机器可能使用与预期不同的主版本进行构建。仓库也没有发现 CI 安全扫描、SBOM 或二进制来源说明。

**建议：** 使用可复现的相对路径/受控包源；固定 SDK 策略；对所有离线 DLL 做哈希和签名验证；CI 增加 `dotnet list package --vulnerable --include-transitive`（或新版等价命令）、秘密扫描、Semgrep/CodeQL、二进制恶意软件扫描与 SBOM 生成。

## 未发现的恶意模式

在可读源码中未发现以下明确迹象：

- 启动时下载并执行任意负载；
- PowerShell、`curl | sh`、计划任务、启动项、注册表 Run 键或 shell profile 持久化；
- 键盘记录、浏览器凭据/钱包搜索或剪贴板窃取；
- 大段 Base64/加密字符串解码后动态执行；
- 硬编码私钥、云访问令牌或通用 API 密钥。

这项结论不覆盖不透明原生二进制、未来下载的 Mod，且不等价于恶意行为不存在。

## 建议的安全运行流程

1. 在网络隔离的临时虚拟机中进行首次打开和构建，不使用含真实 Steam 凭据或个人存档的账户。
2. 先从可信来源恢复官方哈希/签名并验证所有原生文件；无法验证的文件一律隔离。
3. 禁用所有编辑器插件和自动加载扩展后逐个启用；游戏运行使用 `nomods`。
4. 抓取 DNS/HTTPS 连接，确认仅访问报告列出的 Mega Crit、Sentry、Steam 和指标域名。
5. 将工作树与只读哈希基线比较；任何更新都重新进行源码差异审计和二进制验证。
6. 只有在上述步骤通过后，才允许仓库接触开发者密钥、签名证书或生产环境。
