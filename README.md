# Local Less Bracket Highlight Fix

本地 VS Code 注入语法扩展，用于修复 Less 中**嵌套括号**导致后续属性失去语法高亮的问题。

- 扩展 ID：`local-dev.less-bracket-fix`
- 当前版本：`1.0.2`
- 仓库：https://github.com/mote-ice/vscode-less-bracket-highlight-fix
- 适用文件：`.less` 文件、Vue 单文件组件中 `<style lang="less">` 的内容
- 纯 TextMate 语法扩展：没有运行时 JS，不需要激活事件

## 问题

VS Code 内置 Less 语法中，`polygon()`、`inset()`、`circle()` 等 shape function 的参数规则只包含长度/百分比，不包含嵌套函数。例如：

```less
.wrapper {
  clip-path: polygon(0 0,
      100% 0,
      100% calc(100% - 4px),
      calc(100% - 4px) 100%,
      4px 100%,
      0 calc(100% - 4px));
  font-size: 13px;
}
```

内置语法会把 `calc(...)` 的第一个 `)` 当作 `polygon` 的结束，导致 `font-size` 等后续属性脱离 `meta.property-list.less`，失去属性名、数字、单位、括号配对等高亮。CSS 文件由内置 CSS 语法正确处理，不受影响。

Vue 单文件组件中的 `<style lang="less">` 使用 Volar 的 `text.html.vue` 根语法并嵌入 `source.css.less`，存在同样问题。

## 实现方式

采用 TextMate **注入语法**（injection grammar），不替换内置 Less 语法：

1. `package.json` 中的 `contributes.grammars`：
   - `injectTo: ["source.css.less", "text.html.vue"]`
   - 前者覆盖独立 `.less` 文件，后者覆盖 Vue 文件里嵌入的 Less 块
2. `syntaxes/less-fix.tmLanguage.json`：
   - `injectionSelector`：`L:source.css.less -comment -string, L:meta.property-value.less -comment -string`
   - `#balanced-css-function` 通过自引用递归解析任意层嵌套函数
   - 函数参数内部的匹配顺序：
     1. `#balanced-css-function`（优先识别嵌套函数）
     2. `source.css.less#property-values`（复用内置的数字、单位、颜色、变量等规则）
     3. 连续空白
     4. 单字符兜底

其中兜底匹配不能跨过 `calc` 这类嵌套函数名。原实现使用 `[^()]+`，会从行首一路把 `calc` 吞掉，导致递归规则永远无法命中——这是 1.0.0 看起来“加载了但没用”的根本原因。

## 安装

源码目录：

```text
~/.vscode/custom_extensions/less-bracket-fix
```

不要放回 `~/.vscode/extensions`：VS Code 会把未登记的手动目录当作 obsolete，并在安装时清理。

打包并安装：

```bash
cd ~/.vscode/custom_extensions/less-bracket-fix
npx @vscode/vsce package -o /tmp/less-bracket-fix.vsix
code --install-extension /tmp/less-bracket-fix.vsix --force
```

然后在 VS Code 中重载窗口：

```text
Cmd+Shift+P → Developer: Reload Window
```

或完全退出 VS Code 后重新打开。

## 验证

确认安装版本：

```bash
code --list-extensions --show-versions | grep local-dev
# local-dev.less-bracket-fix@1.0.2
```

打开 `__test__/test.less` 或 `__test__/test.vue`，用 `Developer: Inspect Editor Tokens and Scopes` 检查：

| 位置 | 期望作用域 |
| --- | --- |
| `.less` 中 `font-size` | `source.css.less > meta.property-list.less > support.type.property-name.less` |
| `.vue` 中 `font-size` | `text.html.vue > source.css.less > meta.property-list.less > support.type.property-name.less` |
| 嵌套 `calc` 的函数名 | 两层 `meta.function.misc.css` |
| `13` | `constant.numeric.less` |
| `px` | `keyword.other.unit.less` |

`__test__` 目录内容：

| 文件 | 用途 |
| --- | --- |
| `test.css` | CSS 正常行为的参考基准 |
| `test.less` | 独立 Less 文件 |
| `test.vue` | Vue 单文件组件中嵌入的 Less |

## 项目结构

```text
less-bracket-fix/
├─ package.json
├─ .vscodeignore
├─ syntaxes/
│  └─ less-fix.tmLanguage.json
├─ __test__/
│  ├─ test.css
│  ├─ test.less
│  └─ test.vue
└─ README.md
```

- `__test__` 仅用于本地验证，已被 `.vscodeignore` 排除，不会打进 VSIX
- 扩展没有 `main` 字段，也没有 `activationEvents`

## 更新流程

修改 `syntaxes/less-fix.tmLanguage.json` 或 `package.json` 后：

1. 递增 `package.json` 的 `version`
2. 重新打包并 `--force` 安装
3. 重载 VS Code 窗口

```bash
npx @vscode/vsce package -o /tmp/less-bracket-fix.vsix
code --install-extension /tmp/less-bracket-fix.vsix --force
```

## 已知限制

- 仅处理 Less（含 Vue 中的 Less 块），不处理 SCSS、Sass 或 CSS
- 通用函数匹配会把 `.mixin(...)` 这类 Less mixin 选择器也套上 `meta.function.misc.css`，只影响作用域检查，不影响实际显示
- 函数名统一命名为 `support.function.misc.css`，不区分 `calc`、`url` 等具体函数
- 参数内部复用了内置 Less 语法的 `source.css.less#property-values`；如果其他扩展替换了内置 Less 语法，数字/单位等作用域可能降级，但括号递归仍然有效
- 字符串和注释中的括号不会参与解析（`injectionSelector` 已排除 `string` 和 `comment`）

## 许可证

MIT License，版权归 mote-ice 所有。详见 [LICENSE](LICENSE)。

## 版本记录

- **1.0.2** — 添加 README、LICENSE 和 repository 元数据
- **1.0.1** — 修复参数贪婪匹配导致递归失效；增加 Vue `text.html.vue` 注入目标
- **1.0.0** — 初始版本
