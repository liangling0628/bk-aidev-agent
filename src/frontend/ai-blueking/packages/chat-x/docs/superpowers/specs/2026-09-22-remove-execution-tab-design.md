# 下线侧栏「执行情况」Tab

日期：2026-09-22  
TAPD：1070093903137745043（workspace 70093903）  
短 ID：137745043  
范围：`@blueking/chat-x` + `@blueking/ai-blueking`

## 1. 背景与目标

侧栏默认「执行情况」Tab（`ExecutionSummary`）把对话里的工具调用 / FlowAgent 消息再渲染一遍，并靠搜索高亮、只读 Token、`executionTabVisible` 等特例维持。产品要求单智能体与流程智能体都去掉这个 Tab。

本次目标：

- 删除侧栏「执行情况」Tab 及专属链路，不兼容旧 API。
- 对话流里 FlowAgent 的「执行情况：成功 N / …」标题栏和节点树保留。
- 剩余侧栏收成一套 Tab 协议，不再按 `name === xxx` 分叉渲染内容。

## 2. 明确不做

- 不改对话流 `FlowAgentContent` 标题栏文案、节点树、详情 / 重试 / 跳过（Share 态仍只读）。
- 不删除 `HighlightKeyword`；`DescPanel` / `ToolcallRender` 继续用它渲染文本。
- 不把「文件产物」改成业务自己注册。
- 不保留 `executionTabVisible` 空实现或废弃别名。

## 3. 原设计问题

1. 同一批执行类消息在对话流和侧栏各渲染一遍；侧栏还要 `EXECUTION_PANEL_TOKEN` 隐藏重试 / 跳过。
2. Tab 显隐走独立 Prop，而不是已有的 `CustomTab.visible`。
3. 关键词 provide 只为侧栏搜索服务，却穿透整棵消息树。
4. 文件产物已经复制了 `if (name === 'file-artifact')` 的特例，后续内置 Tab 还会再分叉。

## 4. 目标架构

```
ChatContainer #aside
├── Tab 栏（displayTabs，按 order 升序）
└── 统一内容槽
      <component :is="resolveSideComponent(tab)" v-bind="tab.data.props">
        #locateButton
      </component>
```

组件解析顺序（与现网一致，只是文件产物也走这条路）：

1. `getSideRenderComponent?.(h, tab.data.props)` 若返回 VNode
2. 否则 `tab.data.component`

`#locateButton` 始终传入。文件产物不使用该插槽；节点详情 / 证据继续用。

内容区高度用统一 wrapper（`calc(100% - 40px)`），不再只绑在文件产物上。

## 5. 删除清单

### 5.1 chat-x

| 类别 | 删除项 |
| --- | --- |
| UI | `ExecutionSummary` 组件与测试、侧栏搜索框 |
| Tab API | `executionTabVisible`、`EXECUTION_TAB_NAME`、执行情况默认 Tab 描述、`ExecutionIcon`（仅 Tab 图标） |
| 数据 | `executionGroups`；`MessageGroup.userMessageTitle`（只在 `executionGroups` 里赋值，给摘要当标题） |
| 只读上下文 | `EXECUTION_PANEL_TOKEN` / `useExecutionPanelInject` |
| 文案 | 仅 ExecutionSummary 使用且删除后无引用的 key（如「搜索 关键字」「清空搜索」）。`执行情况`、`在对话中定位`、`暂无数据`、`搜索结果为空` 保留 |

### 5.2 必须保留 / 若工作区已误删则恢复

当前工作区曾误删高亮链路。实现时必须恢复并保持：

- `HighlightKeyword` + 样式
- `useKeywordInject` / keyword provide（`use-common`）
- `HIGHLIGHT_KEYWORD_CLASS_NAME`
- `DescPanel` / `ToolcallRender` 继续把文本交给 `HighlightKeyword`

没有搜索源时 `useKeywordInject()` 为 `undefined`，组件按原文渲染，不新造搜索入口。

### 5.3 对话流只读

`hideResumeActions` 只由 Share 态决定。不再读取「是否在执行情况面板内」。

### 5.4 ai-blueking

删除并停止透传 `executionTabVisible`（`ChatBot`、`AIBlueking`、vue2 包装、`prop-defaults`、types）。  
playground / Skill 中「展开执行情况」改为打开节点详情或文件产物。  
`use-panel-container` 等只为该 Tab 存在的分支一并删除。

## 6. 统一 Tab 协议

`useCustomTab` 的增删、选中、排序、折叠逻辑不改。`defaultTabs` 只保留文件产物。

### 6.1 文件产物描述（稳定引用，不含文件列表）

```ts
{
  name: FILE_ARTIFACT_TAB_NAME, // 'file-artifact'
  label: t('文件产物'),
  order: -1,
  closable: false,
  data: {
    component: FileArtifactPanel,
    loadOnSelect: false,
    icon: ArtifactTabIcon,
  },
}
```

约定：

- `data.loadOnSelect === false`：选中时不调用 `onCustomTabChange`。未写该字段视为 `true`（节点详情 / 证据保持现网拉数）。
- `data.icon`：Tab 标签默认图标。未写则用 `NodeTabIcon`。禁止再写 `name === FILE_ARTIFACT_TAB_NAME` 选图标。
- `CustomTab.icon?: string` 保持不动，本次不复用、不删。

打开侧栏仍是：点文件卡片 → `openPreview` → `addCustomTab(FILE_ARTIFACT_TAB)`。

### 6.2 文件产物数据走 inject

产物列表随消息和待发送附件变化，禁止写入 `tab.data.props`（会反复改 `tabs`）。

扩展已有 `ARTIFACT_PREVIEW_TOKEN`：

- 现有：`activeArtifactId`、`setActiveArtifactId`、`openPreview`、`resolveArtifactUrls`、`canResolveArtifactUrl`
- 新增：`artifacts: ComputedRef<SessionArtifact[]>`

`ChatContainer` 把已算好的 `sessionArtifacts` 传入 `useArtifactPreviewProvider`。  
`FileArtifactPanel` 只从 inject 读列表和命中项，不再接收 `artifacts` / `activeId` props。  
测试与文档站示例通过 Provider 注入，不走裸挂载。

### 6.3 选中回调

```
onTabChange(tab):
  if (tab.data.loadOnSelect === false) return
  挂 loading props → await onCustomTabChange(tab) → 回写 data
```

用 `loadOnSelect` 判断，不用组件引用相等，也不用 name 字符串。

## 7. 效率

- 去掉执行类消息的第二次整树渲染，以及每条消息变更时的 `executionGroups` 过滤。
- 文件列表走 inject，Tab 描述保持稳定引用。
- 高亮链路保留；无 keyword 时是空注入上的原文渲染，不扫 DOM 做匹配。

## 8. 异常与空态

- 无可见 Tab → 侧栏折叠（现网 `useCustomTab`）。
- 文件产物为空 → 面板自身空态，Tab 仍在且不可关闭。
- 选中 Tab 被移除或 `visible: false` → 切到 `displayTabs[0]`。
- 自定义 Tab 既无 `data.component` 也无 `getSideRenderComponent` 返回值 → 内容槽空白（与现网一致）。

## 9. 测试

### chat-x

- 删除 `execution-summary.spec`。
- `chat-container.spec`：去掉执行情况 Tab / `executionTabVisible` / 面板只读；补默认仅文件产物、统一内容槽渲染 `tab.data.component`、`loadOnSelect: false` 不触发 `onCustomTabChange`、图标走 `data.icon`。
- `use-custom-tab.spec`：`defaultTabs` 仅文件产物，未手动切换时选中 order 首位。
- `use-message-group.spec`：不再断言 `executionGroups` / `userMessageTitle`。
- `flow-agent-content`：保留标题栏「执行情况」与 Share 隐藏重试 / 跳过。
- `desc-panel` / `toolcall-render`：保留 HighlightKeyword 用例；不依赖执行情况搜索源。
- `file-artifact-panel.spec`：包 Provider 注入产物；覆盖空列表与有数据列表。

### ai-blueking

- 删除 `executionTabVisible` 相关用例与默认值断言。
- 侧栏 / panel-container 测试改为文件产物或自定义 Tab。

实现后跑 `pnpm test:ui`（须用户同意后再执行）。不擅自跑 lint。

## 10. 文档与 Skill

实现后按 `chat-x-update-docs` / `ai-blueking-docs-update` 同步，不另开需求：

- 删除 `execution-summary` 文档页及目录入口。
- 更新 `chat-container`、`custom-side-tab`、`custom-side-content`、`use-message-group`、`use-flow-node-actions`：默认 Tab 只有文件产物。
- 更新 `activity-message`、`flow-agent-content`：删除「侧栏执行情况面板只读」；保留对话流标题栏说明。
- 更新小鲸 Skill：去掉 `executionTabVisible` 与 `ExecutionSummary` 导出。

## 11. 实现约束

- 新分支从已验证的 `upstream/develop`（或任务指定 base）拉出，命名 `feat/remove-execution-tab`。不要写在 `feat/flow-agent-terminated-state` 上。
- 当前工作区已有部分删除，可作参考，不能整单提交：必须按本文恢复 `HighlightKeyword`，并做统一内容槽，而不是停在「只删 Tab、文件产物仍 name 分叉」。
- 切换分支前须用户确认脏区策略（保留 / 另存 / 丢弃），禁止静默 stash。
- 提交信息按仓库约定：`feat: <摘要> --story=137745043`。公开 PR 不写完整 TAPD 标题或内网链接。

## 12. 验收

- 单智能体、流程智能体侧栏都没有「执行情况」Tab。
- 对话流 FlowAgent 标题栏仍显示「执行情况」统计和节点树；失败节点在非 Share 下仍有重试 / 跳过 / 详情。
- 文件产物、节点详情、有效证据仍可用；选文件卡片仍打开文件产物 Tab。
- 包入口和类型里找不到 `executionTabVisible`、`ExecutionSummary`、`EXECUTION_TAB_NAME`。
- `HighlightKeyword` 仍存在，DescPanel / ToolcallRender 仍使用它。
- chat-x 与 ai-blueking 文档 / Skill 与代码一致。
