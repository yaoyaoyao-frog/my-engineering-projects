# CadBeamLoad — AutoCAD 墙体荷载自动计算插件

一个运行在 AutoCAD 2024 中的 .NET 插件，把**建筑图里的墙体**自动换算成**线荷载**，并写回 **YJK 结构模型（.ydb）**。

```
建筑图墙线(AutoCAD) ──拾取/配对──> 墙中线+墙厚 ──匹配梁板──> 线荷载 ──写回──> YJK .ydb
```

## 主要功能

- **CAD 墙体识别**：按图层批量拾取直线/多段线，双线墙平行线配对 → 计算墙中线与厚度；自动合并被门窗洞口打断的共线墙段。
- **门窗洞口处理**：从块参照（含动态块）提取门窗宽高，按洞口面积计算折减系数 η。
- **YJK 模型读取**：解析 `.ydb` 中的楼层、梁、柱、板（截面、偏心、板厚、板多边形）。
- **墙梁空间匹配**：墙中点投影到梁中心线，按梁宽与容差判定归属，并自动扣除梁端柱截面占宽。
- **荷载计算**：
  - 有梁墙：净高 = 层高 − 梁高；
  - 无梁墙（板上墙）：洞口共线合并 → 射线法板匹配 → 跨板分段，净高 = 层高 − 板厚；
  - 线荷载 = 净高 × (容重×墙厚 + 抹灰 + 面层) × 洞口折减系数；相近荷载按阈值聚类归并。
- **结果写回与导出**：梁线荷载、板线荷载写入 YJK 模型并生成新的 `.ydb`；结果表格可查看逐段计算过程。
- **辅助工具**：梁高标注识别（从 CAD 标注文字读取梁截面）、YJK 模型在 AutoCAD 模型空间中 2D 预览、Ribbon 功能区、现代化参数配置对话框。



## 环境要求

- Windows + .NET Framework 4.8 / C# 7.3，x64
- AutoCAD 2024（引用 `acmgd`、`acdbmgd`、`accoremgd`、`AdWindows`、`AdUiMgd`，路径在 `.csproj` 中按本机安装位置配置）
- YJK SDK：`ClrYJKAPI.dll`、`YAPIData.dll`、`YJKAPI.dll`、`Newtonsoft.Json.dll`，默认从 `..\csharp-yjks-8.0\不依赖YJK案例\dll\` 引用并在编译后复制到输出目录

## 编译与使用

1. 用 Visual Studio 以 **x64 Debug** 编译 `CadBeamLoad\CadBeamLoad.csproj`，产物为 `CadBeamLoad\bin\x64\Debug\CadBeamLoad.dll`。
2. 在 AutoCAD 2024 命令行执行 `NETLOAD`，选择上述 DLL。
3. 常用命令：

| 命令 | 作用 |
|------|------|
| `WallLoadCommand` | 打开墙体荷载计算主窗口 |
| `SHOWRIBBON` / `HIDERIBBON` | 显示 / 隐藏「结构荷载工具」Ribbon |
| `YdbInfoCommand` | 查看 YJK 模型信息 |
| `LoadCheckCommand` | 荷载校核 |
| `LoadHelpCommand` | 帮助 |
