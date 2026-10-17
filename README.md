# SpinPick

一个扑克牌风格的随机抽名工具，基于 Tauri 2 + Vue 3。

## 运行

```bash
npm install
npm run tauri dev
```

## 打包

```bash
npm run tauri build
```

## 使用

- 点击牌堆抽一张，再次点击牌堆继续抽。
- 拖动已抽出的牌回左上角牌堆可还原。
- 点击已抽出的牌可平铺查看；超过 10 张时按住左右拖动浏览。
- 从左边缘向右滑动或按 `Esc` 打开名单设置，可编辑名字和权重。

## 配置文件

名单保存在：

```
%APPDATA%\uno.toolkit.spinpick\names.json
```

首次运行会自动创建。
