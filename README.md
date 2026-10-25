# commondev

主管式多仓库开发流水线：登记项目、开工程对话、为每条线拉起内置 pi 并注册一对通道，投递任务给 pi 并在同一会话内续投。

HologramAgent 插件，随 SDK `0.1.0` 的示例插件拆分而来。

## 构建

```bash
hmapdev build
```

产物 `dist/commondev_bundle.hmap` 经 `plugin_install` 安装。

## 自动更新

本仓库地址登记在 `plg.json` 的 `repo` 字段，安装后随产物写入
`plugin.json` 的 `repository` 字段 —— 内核的 `plugin_update` 靠它
从本仓库拉最新源码并重编安装。
