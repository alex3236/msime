# Agent Note: Fcitx5 主题同步每一拍只比对廉价的键，不再扫描全部主题

Status: implemented

## Problem

#5988：Debian 13 上的 Fcitx5 用户打字、删除都明显卡顿，卸掉大部分 Fcitx5 主题后才顺畅。

有焦点的输入上下文每 250 ms 跑一拍 `FcitxState::refreshProviderSockets()`，每一拍都调 `syncCandidatePanelTheme()`，后者无条件做两件事：

- `FcitxEngine::publishCandidatePanelStatus()` 调两次 classicui 的 `getConfig()`。fcitx5 的 `ClassicUI::getConfig()`（5.0.x 到 master 都一样）每次都扫描 PkgData 下所有 `themes` 目录、逐个打开解析 `theme.conf`，还要在 PATH 里查一次 Plasma 主题工具。开销随已装主题数线性增长，而且跑在处理 `keyEvent` 的同一个事件循环上。
- `FcitxEngine::applyCandidatePanelTheme()` 在和上次结果比较之前就调 `fcitx_candidate_theme()` 生成整份主题，也就是把十来张形状按 1x、2x 栅格化（逐像素 erfc/hypot 阴影加逐位 CRC），在容器里实测每次约 1.5 ms。比较不等时它再调两次 `getConfig()`。用户选了第三方主题、没有经典界面（如 kimpanel）或写主题失败时，函数提前返回，从不记下结果，于是每一拍都重新栅格化、重新扫四遍主题，每秒 16 次全量扫描。

旧逻辑还有一个反方向的缺陷：一旦接管过一次，`candidate_theme_applied_` 只跟配色比较，用户在 fcitx5-configtool 里先选第三方主题、再改回默认，配色没变就不会再接管。

## Decision

拍子照旧每一拍同步主题，变的是每一拍做什么。

- 新增 `read_classicui_theme_selection()`：用 `fcitx::readAsIni(config, "conf/classicui.conf")` 读 classicui 落盘的 `Theme` 与 `DarkTheme`。这个单参数重载在 5.0.20 到 master 都存在，内部都按 PkgConfig 查找，和 classicui 自己 `reloadConfig()` 读配置用的是同一个调用；classicui 的 `setConfig()` 载入后立刻 `safeSaveAsIni` 回这个文件，所以文件就是现值。不用 `StandardPath::Type` 那个重载，master 已把 `StandardPath` 标为 deprecated，`-Werror` 下会编不过。
- `publishCandidatePanelStatus()` 不再调 `getConfig()`，只在当前界面是 classicui 时读这份选择。缺省的处理不变：`Theme` 缺失按空串，`DarkTheme` 缺失按没有；classicui 的缺省值 `default`、`default-dark` 本来也都可替换，所以文件不存在、键缺失时的结论和以前问 `getConfig()` 一样。
- `applyCandidatePanelTheme()` 先算一把廉价的键：`candidate_theme_request(...)` 的 JSON（全局主题、自定义主题、明暗、布局和所画皮肤包的目录条目，共享层的解析是纯函数，颜色完全由它决定）、圆角、是否用户圆角、装饰图的戳、classicui 是否存在，以及上面读到的 `Theme`/`DarkTheme`。键不变直接返回。键变了先记下，再调一次 `getConfig()` 按现值判断能否接管，能接管才调 `write_fcitx_candidate_theme()` 栅格化并写盘，最后经新的 `set_classicui_config(classicui, current, config)` 重载接管，复用刚取到的现值做卸载记录，不再为记录多调一次 `getConfig()`。接管把文件里的选择改成了 `msime`，所以写完按改后的选择重算一次键，下一拍不会把同一份主题再写一遍。
- 比较不再生成主题文本，`candidate_theme_applied_` 换成 `candidate_theme_key_`。不可接管、没有经典界面、写主题失败这三条路径都记下键。
- `resetSessions()` 清空这把键。

一拍的开销于是变成：读一个小 INI、一次约 20 µs 的主题解析调用（容器里的 debug 构建）、一次装饰图 stat。`getConfig()` 和栅格化只在输入或选择变化时各做一次。

`syncVoiceOverlayTheme()` 每一拍也调 `msime_client_resolve_theme`，没有改。Rust 侧的解析不碰文件，请求只有几百字节，同一个容器里测得每次约 23 µs，和上面的问题差了两到三个数量级，再加一层缓存只多一份要维护的状态。`refreshThemeMenu()`、`noteSchemeOptions()` 在输入不变时本来就提前返回，也没有改。

## 写失败的重试

写主题失败（主题目录被普通文件占住、磁盘满、权限不对）时不在每一拍上重试，否则又回到每秒四次栅格化。下列任一情况发生后再试：

- 主题输入变化：改皮肤、改全局主题、系统明暗切换、装饰图被替换。
- classicui 的主题选择变化。
- 用户在设置页点「重启输入法服务」（`ReloadAddonConfig` → `resetSessions`）或按 `Ctrl+Shift+Alt+R`。

## Alternatives considered

- 不再从 250 ms 拍子里调 `syncCandidatePanelTheme()`，只在偏好变化、系统明暗变化时同步：改动最小，热路径上一点开销都不留。否掉的原因是 fcitx5-configtool 改 classicui 主题不会通知插件，用户把第三方主题改回默认后，要等下一次改偏好才恢复接管，这是可见的回退。
- 继续用 `getConfig()`，只把结果按时间节流（比如每 5 秒问一次）：判断仍然准确，但每 5 秒还是一次全量主题扫描，装几十个主题的机器上照样周期性卡一下，而且选择变化最多要晚 5 秒才被注意到。读落盘的 classicui.conf 同样准确，只要几微秒。
- 写失败按退避重试（1 s、2 s、4 s…）：能在临时故障恢复后自己补上，但要多维护一份计时状态，而这类失败多半是持久的（目录被占、只读文件系统），自动重试救不回来。「重启输入法服务」已经是用户手里现成的重试入口。
- 键里放解析出的颜色，而不是解析请求：一样准确，但要把 `CandidateColors` 的每个字段（包括菜单槽位）逐个序列化，以后加字段忘了同步就会漏掉配色变化。请求 JSON 是颜色唯一的输入，结构变了键自动跟着变。

## Consequences

收益：主题输入和 classicui 选择都没变时，每一拍不再扫描主题、不再栅格化，开销与已装主题的数量无关。在 fcitx5-configtool 里把第三方主题改回默认后，下一拍恢复接管，旧逻辑做不到。

代价：

- 每一拍多读一个小 INI 文件。
- 有人手改 classicui.conf 但没让 fcitx5 重载时，文件和现值不一致。这时键会变，`getConfig()` 按现值判断一次，结论仍以现值为准，只是状态页的「第三方主题」提示可能按文件报。
- 写主题失败后，在下一次输入变化之前候选窗保持旧样子。

测试：

- `fcitx5-native-test` 的 `classicuiThemeTicks` 用一个替身经典界面（和真实 classicui 一样在 `setConfig` 里落盘）钉住几种情况：第三方主题连续四拍只调一次 `getConfig()`，不写主题；改回默认后下一拍恢复接管，`Theme`、`DarkTheme` 都指向 `msime`；接管后的拍子不再调 `getConfig()`，主题文件 inode 不变；配色变化时重写；写失败只试一次，输入变化后再写。另有一段走真实的 `refreshProviderSockets()`：没有经典界面时记下的键在后续拍子里不变。
- 把键比较临时注释掉之后，第一条断言失败。
- `tests/core/fcitx5_contract.py` 静态钉住：`publishCandidatePanelStatus` 不出现 `getConfig`；`applyCandidatePanelTheme` 只有一次 `getConfig()`，位于键比较之后，且不调 `fcitx_candidate_theme(`；`resetSessions` 清键；`refreshProviderSockets` 仍同步主题。

尚未验证：

- 没有在装了大量主题的真实 Fcitx5 桌面上量过按键延迟。
- 没有接真实 classicui 跑。build-gate 容器的 fixture 用 `--disable=all`，不加载 classicui。
- 只在 Debian bookworm 的 Fcitx5 5.0.21 上编译过。master 上 `readAsIni` 单参数重载的可用性来自上游源码。
