---
name: openscad-ai-modeling
description: 换一台已装 OpenSCAD 的电脑时，按这份手册接好 openscad-mcp、装上游的 openscad-design skill、隔离项目目录，并用一个冒烟测试确认能测量、导出、渲染，然后才进入 AI 建模。触发词：配置 openscad-mcp、安装 openscad skill、OpenSCAD 换电脑、OpenSCAD 项目目录、给模型写说明文档。
---

# OpenSCAD AI 建模：换机落地手册

前提只有两个：Claude Code 已装，OpenSCAD 已装（2021.01 或更新）。OpenSCAD 怎么装不在这里讲。

终点不是“配置写好了”，而是“冒烟测试通过”：能测尺寸、能导出 STL、能渲染图片。三样都过，才算进入 AI 建模环节。

---

## 1. 新电脑：按顺序搭起来

### 1.1 找到 OpenSCAD 的控制台程序

控制台程序指能在终端里输出文字的那个版本。MCP 靠这些文字读尺寸和报错，GUI 版不行。

| 系统 | 怎么找 | 用哪个 |
|---|---|---|
| Windows | `where.exe openscad.com`；找不到再查 `C:\Program Files\OpenSCAD\`、`D:\OpenSCAD*\` | `openscad.com` |
| macOS | `/Applications/OpenSCAD.app/Contents/MacOS/OpenSCAD` | 这个可执行文件 |
| Linux | `command -v openscad` | 输出里的路径 |

Windows 上 `openscad.com` 和 `openscad.exe` 通常在同一目录。`.com` 能把结果打印到终端，`.exe` 是 GUI 版，MCP 读不到它说的话。

记下完整路径，下一步要用。

### 1.2 装 uv

uv 是启动 MCP 的小工具，顺便负责下载 `openscad-mcp` 和它需要的 Python。机器上不用另外装 Python；uv 找不到合适版本时会自己下载。

Windows：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

macOS / Linux：

```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

装完重开终端，确认能运行：

```powershell
uv --version
uvx --version
```

国内网络先设 PyPI 镜像，否则第一次下载可能一直卡住：

```powershell
$env:UV_DEFAULT_INDEX = "https://mirrors.aliyun.com/pypi/simple"
```

这条只对当前终端有效；下一步会把它写进 MCP 自己的环境里。第一次启动 MCP 需要联网下载 `openscad-mcp`，下载过一次后会走本机缓存。

### 1.3 写 `.mcp.json`

放在工作区根目录，和 `projects/` 同级。已有这个文件时只加 `openscad` 这一项，不要覆盖别的服务器。

```json
{
  "mcpServers": {
    "openscad": {
      "command": "uvx",
      "args": ["openscad-mcp@0.6.1"],
      "env": {
        "OPENSCAD_PATH": "C:\\Program Files\\OpenSCAD\\openscad.com",
        "MCP_ALLOWED_PATHS": "D:\\AI+Openscad",
        "UV_DEFAULT_INDEX": "https://mirrors.aliyun.com/pypi/simple"
      }
    }
  }
}
```

JSON 不能写注释；多一个逗号或少一个引号，整个文件都会失效。

各字段：

- `args` 固定 `0.6.1`，这是本文验证过的版本。以后想升级就改成 `openscad-mcp@latest`。
- `OPENSCAD_PATH` 填 1.1 找到的路径，Windows 的反斜杠要写两个。OpenSCAD 已在 PATH 里时可以删掉这一行。
- `MCP_ALLOWED_PATHS` 是 MCP 唯一能读写的地方，填工作区根目录，不要填整个磁盘。多个目录用 `;` 分隔（Windows）或 `:` 分隔（macOS / Linux）。不填等于不限制。
- `UV_DEFAULT_INDEX` 只在国内网络需要；能直连 PyPI 就删掉。
- 如果 Claude Code 找不到 `uvx`，把 `command` 改成完整路径，例如 `C:\Users\<用户名>\.local\bin\uvx.exe`。

**不要同时装上游插件和写这份 `.mcp.json`。** 插件本身也带一个 `openscad` 服务器，两份同名配置会让排查变得很难。本项目统一用 `.mcp.json`，skill 用下一节的手工目录方式。

工作区换路径时，先改 `OPENSCAD_PATH` 和 `MCP_ALLOWED_PATHS`，再重启 Claude Code；否则 MCP 会拒绝新路径下的文件。

### 1.4 放两个 skill

本 skill 解压到下面任一位置：

- 全局：`%USERPROFILE%\.claude\skills\openscad-ai-modeling\`
- 项目级：`<工作区>\.claude\skills\openscad-ai-modeling\`

上游建模 skill 在 <https://github.com/RobertCoop/openscad-mcp/tree/main/skills/openscad-design>。把它整个文件夹放到同一个 `skills\` 目录下的 `openscad-design\`。前两种拿法任选：

- 打开上面的链接，下载仓库 ZIP（<https://github.com/RobertCoop/openscad-mcp/archive/refs/heads/main.zip>），解压后复制 `skills/openscad-design`。
- 装了 git：`git clone --depth 1 https://github.com/RobertCoop/openscad-mcp.git`，再复制 `skills/openscad-design`。
- 用上游插件装（见下面的注意事项）；只在你决定不用 `.mcp.json` 时选这条。

不想手工点网页时，用 PowerShell 一次做完（装到全局技能目录）：

```powershell
$zip = Join-Path $env:TEMP "openscad-mcp-main.zip"
$dst = Join-Path $env:USERPROFILE ".claude\skills"
New-Item -ItemType Directory -Force $dst | Out-Null
Invoke-WebRequest "https://github.com/RobertCoop/openscad-mcp/archive/refs/heads/main.zip" -OutFile $zip
Expand-Archive $zip -DestinationPath $env:TEMP -Force
Copy-Item "$env:TEMP\openscad-mcp-main\skills\openscad-design" $dst -Recurse -Force
```

装到项目级就把最后一行目标改成 `<工作区>\.claude\skills\`。

放好后**重启 Claude Code**，新 skill 才会被索引。新电脑不能联网时，把旧电脑 `%USERPROFILE%\.claude\skills\openscad-design\` 整个文件夹一起拷过去，效果一样。

**上游插件的注意事项。** 下面两条命令会同时装 skill 和 MCP 服务器：

```text
/plugin marketplace add robertcoop/openscad-mcp
/plugin install openscad-mcp@openscad-mcp
```

插件里的服务器没有设 `OPENSCAD_PATH` 和 `MCP_ALLOWED_PATHS`。如果你选这条路，就**不要**再写 1.3 的 `.mcp.json`，而是把这两个变量设成机器级环境变量（Windows 用 `setx`，macOS / Linux 写进 `~/.zshrc` 或 `~/.bashrc`），再启动 Claude Code；否则可能找不到 OpenSCAD，也不会限制可读写的目录。

### 1.5 重启并批准

1. 重启 Claude Code。
2. 输入 `/mcp`，批准名为 `openscad` 的服务器。项目级 `.mcp.json` 第一次使用一定需要这一步。
3. 调用 `check_openscad`。返回版本号，说明 MCP 和 OpenSCAD 接上了。
4. 让 Claude 列出当前可用的 skill，应同时看到 `openscad-design` 和 `openscad-ai-modeling`。

### 1.6 冒烟测试：证明真的能建模

冒烟测试就是拿一个最小模型，把测量、导出、渲染各跑一遍，确认整条链路是通的；任何一步不过，后面的建模都会卡。

新建 `projects/_smoke/build/` 目录，再新建 `projects/_smoke/smoke.scad`：

```openscad
// Smoke test: 20 mm cube with a 3 mm through hole.
difference() {
    cube([20, 20, 20], center = true);
    cylinder(h = 22, d = 3, center = true, $fn = 64);
}
```

按顺序做下面六步，任何一步不过就先别开始建模：

| 步骤 | 调用 | 通过标准 |
|---|---|---|
| 1 | `validate(mode="syntax")` | `errors` 为空 |
| 2 | `measure(mode="model")` | 外形 20 × 20 × 20 mm，体积约 7858.9 mm³，封闭，实体数 1 |
| 3 | `validate(mode="geometry")` | `is_watertight: true`、`is_manifold: true`；`mesh_health.manifold: null` 只表示没单独跑这一项，不能拿它当通过依据 |
| 4 | `export_model(output_format="stl", output_path="projects/_smoke/build/smoke.stl")` | 正常结束，文件存在 |
| 5 | `measure(mode="model", mesh="projects/_smoke/build/smoke.stl")` | 数字与第 2 步一致 |
| 6 | `render(mode="views", views=["isometric"])` | 返回一张斜着看的立体图（等轴测图） |

表里的相对路径都从工作区根目录算。第 2 步的体积允许 ±1 mm³ 误差，来自圆孔的多边形近似。第 5 步是关键：它同时证明导出路径可用、`MCP_ALLOWED_PATHS` 覆盖到了项目目录。

六步都过，环境就绪，可以进入第 3 节。

---

## 2. 日常建模：一个项目一个目录

所有源文件、导出件、渲染图、说明文档都放在项目自己的目录里：

```text
<工作区>/
  .mcp.json
  projects/
    small_hardware_box/
      small_hardware_box.scad   模型源文件
      README.html               中文说明，引用 renders/ 里的图
      build/                    导出的 STL，一个零件一个文件
      renders/                  渲染图 PNG
```

规则：

- 文件夹名用英文小写加下划线，避免路径和编码问题。
- 每次动手前先确认当前项目目录。不要在工作区根目录或别的项目里写文件。
- 一次只改一个项目。导出的 STL 不要跨项目复用，容易拿错件。
- MCP 的所有文件参数（`scad_file`、`mesh`、`output_path`、`include_paths`）都必须在 `MCP_ALLOWED_PATHS` 里。临时 STL 放在工作区外时，先拷进来再 `measure(mesh=...)`。
- MCP 的相对路径从工作区根目录算，不是从当前项目目录算。统一写 `projects/<项目名>/...`，或者直接用绝对路径；写成 `build/x.stl` 会找错地方。
- `.scad` 只写英文注释。中文 Windows 上 MCP 读文件用系统 GBK 编码，中文注释会让 `check` 和 `scad_eval` 报 `'gbk' codec` 错误。中文说明写进 `README.html`。
- 模型顶部留四个预览参数，渲染时靠它们分辨零件：

  ```openscad
  part        = "all";   // [all, body, lid]
  show_colors = true;    // preview only, not in STL
  explode     = 0;       // lift the lid in preview
  cutaway     = false;   // cut the front half in preview
  ```

- 尺寸只信 `measure`。图片只用来确认朝向、结构和零件位置。
- 导出时把 `part` 设成单个零件，一个零件一个 STL；`all` 只用于装配预览。
- 导出 STL 后，必须对文件本身再测一次，不能只信导出前的模型测量。

---

## 3. 进入建模环节

先读 `openscad-design`，按它的循环做。核心只有一句：**数字判定，图片确认**。

- 写代码前先说清单位、朝向、基准、打印工艺。没说的按 mm、Z 向上、底面中心、FDM 0.4 mm 喷嘴 / 0.2 mm 层高。
- 可能改的尺寸和出现两次以上的数字，全部写成文件顶部的变量。
- 顺序固定：`validate syntax` → `measure model/parts` → `validate geometry` → `check interference/clearance` → `measure printability` → `export` → 复测 STL。
- 一个零件一个 module。两个面正好贴在一起时，让它们重叠 0.01 mm，避免破面。
- 图片一次只看一两张，用来找“朝向错了、特征在反面、零件内外翻转”这类数字看不出的问题。

开始建模时可以用这段提示：

```text
用 openscad-design 的流程建模。先量后看，尺寸以 measure 为准。
项目目录 projects/<项目名>/，完成后按 openscad-ai-modeling 出 README.html。
```

---

## 4. 完成后出一份 HTML 说明

最终交付物是项目目录里的 `README.html`：一个能离线打开的单文件，图片放在 `renders/`，用相对路径引用。不套固定模板，按模型实际功能组织内容。样式直接写在 HTML 里，不依赖外部文件。

### 4.1 先写清这个模型是什么

开头用一两句话说明用途、外形尺寸和关键特征，然后按实际情况取舍：

- 外形尺寸、壁厚、关键特征（孔、槽、卡扣、配合面、内部结构）。
- 多零件时写装配关系和装配步骤；单件跳过。
- 打印摆放方向、打印参数、耗材估算。
- 可调参数表：变量名、当前值、作用。
- 文件清单和验证数字：尺寸、体积、封闭性。
- 已知限制或使用注意。

没有的内容不要硬凑。不要为了套模板编造不存在的分格、盖子或装配步骤。

### 4.2 渲染能说明问题的图

图片放 `renders/`，数量按需要定。常用几类：

| 要说明的事 | 怎么渲染 |
|---|---|
| 整体外观 | `part="all"`，等轴测 |
| 内部结构或配合 | `cutaway=true`，或 `render(mode="section")` |
| 打印摆放 | `part="<零件>"`，从打印方向看 |
| 多零件关系 | 加大 `explode`，或 `render(mode="parts")` |
| 关键尺寸 | `render(mode="views", grounded=true)`，图上有标尺 |

命令行示例（bash，把 `openscad.com` 换成 1.1 找到的完整路径，相机按模型包围盒调整）：

```bash
openscad.com --render --imgsize=1600,1200 --projection=o --viewall --autocenter \
  --colorscheme=Tomorrow --camera=<eye_x,eye_y,eye_z,center_x,center_y,center_z> \
  -D 'part="all"' -o renders/overview.png model.scad
```

Windows PowerShell 要把字符串参数的引号转义，程序路径换成 1.1 的：

```powershell
& "D:\OpenSCAD (Nightly)\openscad.com" --render --imgsize=1600,1200 --projection=o `
  --viewall --autocenter --colorscheme=Tomorrow --camera=<eye_x,eye_y,eye_z,center_x,center_y,center_z> `
  -D 'part=\"all\"' -o renders\overview.png model.scad
```

剖面图的相机要放在切口保留的一侧，否则只能看到外壳。渲染完先打开图片确认方向，再写文档。

### 4.3 自查

- 单个 HTML 文件，图片用相对路径 `renders/xxx.png`，整个项目文件夹拷走后仍能打开。
- 不引用外部字体、CDN 或网络图片，离线可看。
- 颜色图例（如果有）与渲染图一致。
- 文档里的尺寸、体积来自 `measure` 和 STL 复测，不是看图估的。
- 用无头浏览器截图检查排版和图片是否加载；没有 Edge 就换成 Chrome。这步只是排版自查，可以跳过。

  ```powershell
  & "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --headless=new `
    --disable-gpu --window-size=1200,5200 `
    --screenshot="renders\readme_preview.png" "file:///D:/path/to/README.html"
  ```

---

## 5. 卡住时查这张表

| 现象 | 原因 | 处理 |
|---|---|---|
| `/mcp` 里没有 openscad | 没批准、没重启、JSON 写错，或不在工作区根目录启动 | 查 JSON 逗号；回到工作区根目录重启；`/mcp` 批准 |
| 启动时报 `uvx` 找不到 | uv 没装或不在 PATH | 重开终端；或把 `command` 改成 `uvx.exe` 全路径 |
| 第一次启动一直卡住 | 直连 PyPI 太慢 | 在 `.mcp.json` 的 `env` 里加 `UV_DEFAULT_INDEX` 镜像 |
| `check_openscad` 报错 | `OPENSCAD_PATH` 指向 GUI 版或路径不存在 | 改成 `openscad.com` 的完整路径 |
| `check_openscad` 版本低于 2021.01 | 版本太旧，部分工具不可用 | 升级 OpenSCAD |
| `check` / `scad_eval` 报 `'gbk' codec` | `.scad` 里有中文注释 | 改成英文注释，中文移进 `README.html` |
| MCP 拒绝读写某个文件 | 文件在 `MCP_ALLOWED_PATHS` 之外 | 把文件移进工作区，或把根目录加进这个变量 |
| `/mcp` 里出现两个 openscad | 插件和 `.mcp.json` 同时生效 | 只留一个：删插件，或删 `.mcp.json` 里的 `openscad` |
| 导出结果是空文件 | `difference()` 第一个子对象太小，或 `part` 取值不对 | 先跑 `validate syntax`，再检查 `part` |
| 测量结果是 `UNRESOLVED` | 距离小于网格误差 | 用 `quality="high"` 或更大的 `$fn` 重跑 |
| 改了 include 文件但数字没变 | 命中了缓存 | 先 `clear_cache` 再测 |
| 渲染图两个零件分不清 | 同色且贴合 | 打开 `show_colors`，加大 `explode`，或开 `cutaway` |
| 剖面图只看到外壳 | 相机站在被切掉的一侧 | 把相机移到保留的那一侧 |

---

## 6. 收尾检查

换机完成：

- `check_openscad` 返回版本号。
- 1.6 的六步冒烟测试全部通过，包括导出后复测 STL。
- 技能列表里同时有 `openscad-design` 和 `openscad-ai-modeling`。
- `.mcp.json` 里只有一个 `openscad` 条目。
- `MCP_ALLOWED_PATHS` 覆盖工作区，没有设成整个磁盘。

模型交付：

- 所有文件都在 `projects/<项目名>/` 内。
- `.scad` 里没有非 ASCII 字符。
- 尺寸、体积、封闭性都有 `measure` / `validate` 的数字，不是看图猜的。
- 导出的 STL 已单独复测。
- `README.html` 图片全部显示，颜色图例和渲染图一致。
