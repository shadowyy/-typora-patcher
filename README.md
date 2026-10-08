# Typora 激活脚本 - 操作说明

## 压缩包内容

解压后包含以下文件：

| 文件/目录 | 说明 |
|---|---|
| `start.bat` | 启动器，双击运行 |
| `typora_crack.js` | 激活主脚本 |
| `node.js/` | Node.js 安装包（`.msi` 推荐安装 + `.zip` 便携版备用） |
| `typora-his/` | 已验证可激活的 Typora 版本（见下方说明） |

## 使用步骤

### 第一步：安装 Node.js

已装过 Node.js 可跳过这一步。

双击 `node.js/` 中的 `node-v24.18.0-x64.msi`，一路"下一步"安装。（没有安装包的话去 https://nodejs.org/zh-cn 下载 LTS 版本）装完验证：

```
Win+R → 输入 cmd → 回车 → node -v
```

显示 `v24.18.0` 即装好。

### 第二步：安装 Typora

`typora-his/` 提供了以下已验证版本：

| 安装包 | 版本 | 状态 |
|---|---|---|
| `typora-setup-x64-1.13.7.exe` | 1.13.7 | ✅ 推荐，长期验证稳定 |
| `typora-setup-x64-1.14.6.exe` | 1.14.6 | ✅ 已验证，同样可用 |
| `typora-setup-x64-1.14.10.exe` | 1.14.10 | ✅ 已验证（Electron 42，需 v1.0.4 及以上脚本） |

> 如之前装过 Typora，建议先卸载再装这里的版本。装完打开一次 Typora 再关闭。
>
> **激活前请先在 Typora 设置里关掉「自动检查更新」**：Typora 的自动更新会用官方安装包覆盖安装目录，已注入的补丁会被还原。

### 第三步：运行激活脚本

双击 `start.bat`，按提示操作：

1. **选择 Typora 安装目录** — 脚本自动从注册表检测，按回车确认
2. **输入机器码** — 打开 Typora → 点「激活」→ 点「输入序列号」按钮 → 选「离线激活」，复制里面的机器码粘贴
3. **输入邮箱** — 随便填个邮箱格式，如 `abc@123.com`

首次运行会自动安装依赖，等几分钟即可。完成后打开 Typora 即为激活状态。

```powershell
1. 进入脚本目录
cd C:\projects\github\-typora-patcher
2. 初始化 npm（如果还没 node_modules）
npm init -y
3. 安装所有依赖
npm install asar chalk@4 readline-sync iconv-lite @electron/fuses
4. 运行脚本
node typora_crack.js
```

## 已激活状态

如果再次运行脚本，会提示"检测到 Typora 已经被激活过"，可选择是否回滚还原为官方原版。

## 回滚还原

运行脚本 → 检测到已激活 → 输入 `Y` → 自动还原所有修改，恢复为官方原版。

## 常见问题

**Q：双击 start.bat 闪退？**
A：右键 start.bat → 编辑，确认路径正确。或直接用命令行运行查看报错。

**Q：提示"Node.js not found"？**
A：用压缩包里的 `node.js/node-v24.18.0-x64.msi` 安装 Node.js，或去 https://nodejs.org/zh-cn 下载。

**Q：激活后过几天又提示"还有 X 天到期"？**
A：当前 Typora 版本与脚本不兼容，或 Typora 自动更新已把安装目录覆盖还原。卸载后用 `typora-his/typora-setup-x64-1.13.7.exe` 重装、关掉自动检查更新，再重新激活。

**Q：运行脚本时控制台打印 `fingerprint: undefined` 或 `⚠️ 机器码中未找到 deviceId/fingerprint 字段`？**
A：机器码没复制全，或该版本机器码字段名与预期不同。重新打开 Typora →「激活」→「输入序列号」→ 选「离线激活」，完整复制机器码（不要带空格和换行）再运行。上方 `machineCode keys:` 那行会告诉你实际有哪些字段。

**Q：想换一台电脑用？**
A：解压压缩包，按上面三步走一遍即可。

## 文件说明

| 文件/目录 | 说明 |
|---|---|
| `start.bat` | 启动器（双击运行） |
| `typora_crack.js` | 激活主脚本 |
| `node.js/` | Node.js 安装包 |
| `typora-his/` | Typora 安装包（已验证版本） |
| `docs/TECHNICAL.md` | 技术文档：脚本实现原理 |
| `docs/COMPATIBILITY.md` | 版本兼容性核查（含 1.14.10 核查详情与实测清单） |
| `docs/CHANGELOG.md` | 更新日志 |
