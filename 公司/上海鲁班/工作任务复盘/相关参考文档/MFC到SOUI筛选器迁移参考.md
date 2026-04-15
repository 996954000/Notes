# MFC 筛选器迁移至 SOUI — 技术参考文档

> **文档性质**：基于当前仓库源码梳理的迁移说明与索引，供后续维护、评审与二次迁移对照使用。  
> **基线日期**：2026-04-13（以当时 `arc` 仓库内容为准；若代码变更，请以实际文件为准复核本节中的路径与行为描述）。

---

## 1. 背景与目标

### 1.1 业务目标

将工具条/菜单侧打开的 **构件筛选器** 对话框，从 **MFC + BCG 控件体系** 迁移到 **SOUI**，在尽量保持 **数据接口与筛选结果语义不变** 的前提下，替换 UI 实现方式。

### 1.2 技术目标

- 使用 SOUI 的布局 XML 描述界面，通过 `SHostDialog` 承载对话框生命周期。
- 树形列表采用 **`STreeView` + `STreeAdapterBase` 派生 Adapter** 的模式，以支持 **三态勾选**、展开折叠与行模板定制。
- 数据仍通过既有的 **`ILbSelectionSetViewModel`** 拉取与回写，避免在迁移阶段重写业务规则。

---

## 2. 涉及模块与文件索引（仓库内真实路径）

| 角色 | 路径 |
|------|------|
| 弹窗入口（当前使用 SOUI） | `code/UI/LbMainUI/Frame/MenuPanel/LbCommonUtils.cpp` → `CLbCommonUtils::CreateFilterDialog()` |
| SOUI 对话框 | `code/UI/LbMainUI/Frame/MenuPanel/LbFiltersSouiDialog.h` / `.cpp` |
| 树 Adapter（三态与树数据） | `code/UI/LbMainUI/Common/LbFiltersTreeViewAdapter.h` / `.cpp` |
| SOUI 布局 | `code/UI/LbMainUI/uires/xml/dlg_filters.xml` |
| 布局注册 | `code/UI/LbMainUI/uires/uires.idx`（条目 `xml_dlg_filters` → `xml\dlg_filters.xml`） |
| 数据接口（ViewModel） | `code/FrameWork/LbMainFrame/SelectionSet/ILbSelectionSetViewModel.h` |
| 构件树节点类型 | `code/CAD/LbCadManager/SelectManager/LbSelectComponentNode.h`（及同目录实现） |
| MFC 旧实现（仍保留在工程中） | `code/UI/LbMainUI/Frame/MenuPanel/LbFiltersDialog.h` / `.cpp` |
| MFC 基类对话框（树控件创建等） | `code/UI/LbMainUI/Frame/MenuPanel/LbSelectDialog.h` / `.cpp` |
| 同类 Adapter 参考（楼层选择等） | `code/UI/LbMainUI/Common/LbFloorslTreeViewAdapter.h` / `.cpp` |

---

## 3. 架构与数据流（准确描述）

### 3.1 分层说明

本功能与工程中 **`ILbSelectionSetViewModel`** 的命名一致：接口注释为「选择集 **viewmodel**」。迁移工作 **未改变** 该接口的职责划分：

- **数据获取**：`GetCurSelectdComponentsData(...)` — 输出填充到 `CLbSelectComponentNode` 为根的树形数据。
- **结果回写**：`SetPickFirst(selectedComps)` — 将用户在树中的勾选结果写回选择逻辑。

SOUI 侧新增的 `CLbFiltersTreeViewAdapter` 负责 **UI 树状态**（展开、三态、行模板绑定），与 **业务节点** `CLbSelectComponentNode*` 并行存在：Adapter 内部维护 `STreeAdapterBase<FiltersTreeItemData>` 的树结构，并在确认时根据勾选 GUID 从原始 `m_data` 收集叶子节点。

### 3.2 打开对话框与 Model 注入

当前入口代码（SOUI 路径已启用，MFC 路径以注释形式保留）位于 `LbCommonUtils::CreateFilterDialog()`：

- 构造：`CLbFiltersSouiDialog dlg(m_pModel);`
- `m_pModel` 类型为 `std::shared_ptr<ILbSelectionSetViewModel>`，在对话框构造时保存到成员，供 `InitData()` / `ClickdOk()` 使用。

### 3.3 初始化时序（SOUI）

下列顺序与 `CLbFiltersSouiDialog::OnInitDialog` / `InitData` 及 Adapter 代码一致：

1. **`InitData()`**  
   - 按静态成员 `m_type`、`m_floor` 组装 `std::vector<LbCompCode> codes`（当 `m_type == 1` 时加入当前激活构件类型）。  
   - 调用 `m_pModel->GetCurSelectdComponentsData(m_selComNodeData, codes, m_floor)` 填充 `m_selComNodeData`。

2. **`FindChildByID2<STreeView>(R.id.treeview_filters)`** 取得树控件；可选地通过 `m_treeParam` 批量 `SetAttribute`（为后续扩展预留）。

3. **创建 Adapter**：`new CLbFiltersTreeViewAdapter(&m_selComNodeData)`。  
4. **`RebuildFiltersTree()`**：清空 Adapter 内部树，插入根节点「工程」及子树，并 `notifyBranchChanged`。  
5. **`InitSelectNodes(m_guids)`**：按 GUID 集合恢复勾选（见下文 **已知缺口**）。  
6. **`SetAdapter(m_pAdapter)`** 将 Adapter 交给 `STreeView`。

7. **确定**：`ClickdOk()` → `m_pAdapter->GetSelectNodes(selectedComps)` → `m_pModel->SetPickFirst(selectedComps)` → `OnOK()`。

### 3.4 与 MFC 旧版的对应关系

| 环节 | MFC（`CLbFiltersDialog` / `CLbSelectDialog`） | SOUI（当前实现） |
|------|-----------------------------------------------|------------------|
| 对话框基类 | `CBCGPDialog` / `CLbSelectDialog` | `SOUI::SHostDialog` |
| 树控件 | `CBCGPTreeCtrlEx`，样式含 `TVS_CHECKBOXES` | `SOUI::STreeView` + 行模板内 `SCheck3State` |
| 树数据填充 | `InsertItem` + `SetItemData` 挂 `CLbSelectComponentNode*` | Adapter `InsertItem` + `FiltersTreeItemData` |
| 勾选遍历 | `TraverseTreeAndGetValues` 依赖 `GetCheck` | `GetNodesStatus` 收集选中叶子 GUID，再 `CollectSelectedNodesFromData` 从 `m_data` 取节点副本 |
| 命令/消息 | `ON_BN_CLICKED`、`BEGIN_MESSAGE_MAP` | `EVENT_MAP_BEGIN`（如 `R.id.btn_ok`）、`BEGIN_MSG_MAP_EX` |
| AutoCAD 焦点 | `WM_ACAD_KEEPFOCUS` | 同样映射到成员处理（保持与宿主环境兼容） |

### 3.5 关键代码片段（可直接对照源码）

#### A. 对话框入口与 Model 传入（SOUI 当前路径）

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbCommonUtils.cpp
void CLbCommonUtils::CreateFilterDialog()
{
	if (m_pModel)
	{
		/* ... MFC 旧路径已注释 ... */
		CLbFiltersSouiDialog dlg(m_pModel);
		// ... parentHwnd 获取逻辑 ...
		dlg.DoModal();
	}
}
```

#### B. SOUI 对话框构造：布局名 + 选择通知类型保护

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbFiltersSouiDialog.cpp
CLbFiltersSouiDialog::CLbFiltersSouiDialog(std::shared_ptr<ILbSelectionSetViewModel> pModel)
	:SHostDialog(L"LAYOUT:xml_dlg_filters")
{
	m_eOldNotifyType = LbEditorReactorMng::Instance()->GetSelectionNotifyType();
	if (LbEditorReactorMng::Instance()->IsInCommand())
	{
		if (m_eOldNotifyType != eNotNotify)
		{
			LbEditorReactorMng::Instance()->SetSelectionNotifyType(eNotNotify);
		}
	}
	m_pModel = pModel;
}
```

#### C. 初始化主链路：取数据 → 建树 → 恢复选中 → 绑定 Adapter

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbFiltersSouiDialog.cpp
BOOL CLbFiltersSouiDialog::OnInitDialog(HWND wndFocus, LPARAM lInitParam)
{
	InitData();

	m_pTreeTempl = FindChildByID2<SOUI::STreeView>(SOUI::R.id.treeview_filters);
	if (m_pTreeTempl)
	{
		for (const auto& var : m_treeParam)
		{
			m_pTreeTempl->SetAttribute(var.first.c_str(), var.second.c_str(), true);
		}
		m_pAdapter = new CLbFiltersTreeViewAdapter(&m_selComNodeData);
		m_pAdapter->RebuildFiltersTree();
		m_pAdapter->InitSelectNodes(m_guids);
		m_pTreeTempl->SetAdapter(m_pAdapter);
	}
	return FALSE;
}
```

#### D. 数据获取：沿用 ViewModel 接口，不改业务来源

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbFiltersSouiDialog.cpp
void CLbFiltersSouiDialog::InitData()
{
	std::vector<LbCompCode> codes;
	if (m_type == 1)
	{
		LbCompCode code = GetCurrentSelectCompCode();
		if (code != LBCT_Unknown)
		{
			codes.emplace_back(GetCurrentSelectCompCode());
		}
	}
	m_pModel->GetCurSelectdComponentsData(m_selComNodeData, codes, m_floor);
}
```

#### E. Adapter 行渲染绑定：三态勾选 + 展开开关

```cpp
// code/UI/LbMainUI/Common/LbFiltersTreeViewAdapter.cpp
void CLbFiltersTreeViewAdapter::getView(SOUI::HTREEITEM loc, SOUI::SWindow* pItem, SOUI::pugi::xml_node xmlTemplate)
{
	if (pItem->GetChildrenCount() == 0)
	{
		pItem->InitFromXml(xmlTemplate);
	}

	auto pii = m_tree.GetItemPt((HSTREEITEM)loc);
	LB_RETURN_VOID_IF(pii == nullptr);

	SOUI::SCheck3State* pCheck = pItem->FindChildByID2<SOUI::SCheck3State>(SOUI::R.id.check_filters);
	pCheck->GetEventSet()->subscribeEvent(SOUI::EVT_CMD, Subscriber(&CLbFiltersTreeViewAdapter::OnCkFilterClick, this));
	pCheck->SetCheck3State(pii->data.sCheck);
	pCheck->SetWindowTextW(pii->data.strName.c_str());

	SOUI::SToggle* pSwitchType = pItem->FindChildByID2<SOUI::SToggle>(SOUI::R.id.tgl_templ_type_switch);
	pSwitchType->GetEventSet()->subscribeEvent(SOUI::EVT_CMD, Subscriber(&CLbFiltersTreeViewAdapter::OnSwitchClick, this));
	pSwitchType->SetVisible(TRUE);
}
```

#### F. 三态核心入口：点击后写状态并整树刷新

```cpp
// code/UI/LbMainUI/Common/LbFiltersTreeViewAdapter.cpp
bool CLbFiltersTreeViewAdapter::OnCkFilterClick(SOUI::EventArgs* pEvt)
{
	SOUI::SCheck3State* pCk = SOUI::sobj_cast<SOUI::SCheck3State>(pEvt->sender);
	SOUI::SItemPanel* pItem = SOUI::sobj_cast<SOUI::SItemPanel>(pCk->GetRoot());
	SOUI::HTREEITEM loc = (SOUI::HTREEITEM)pItem->GetItemIndex();

	bool check{ false };
	if (pCk->GetCheckState() == SOUI::Check3State::C3S_mix || pCk->GetCheckState() == SOUI::Check3State::C3S_normal)
	{
		check = false;
	}
	else
	{
		check = true;
	}
	SetCheckState(loc, check);
	notifyBranchChanged(ITEM_ROOT);
	return true;
}
```

#### G. 确认回写：仅回写选中叶子对应业务节点

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbFiltersSouiDialog.cpp
void CLbFiltersSouiDialog::ClickdOk()
{
	std::vector<CLbSelectComponentNode> selectedComps;
	m_pAdapter->GetSelectNodes(selectedComps);
	m_pModel->SetPickFirst(selectedComps);
	OnOK();
}
```

---

## 4. SOUI 页面与控件约定

### 4.1 布局名与资源

- 对话框构造：`SHostDialog(L"LAYOUT:xml_dlg_filters")`。  
- `uires.idx` 中注册名 **`xml_dlg_filters`**，指向 **`dlg_filters.xml`**。

### 4.2 `dlg_filters.xml` 中与代码绑定的关键节点

- **树**：`<treeview name="treeview_filters" ...>` — 代码中使用 **`R.id.treeview_filters`** 查找（name 参与 SOUI 资源 ID 生成；若新增控件 ID 未生成，需 **重新编译资源** 以更新 `R.id`）。  
- **行模板**：  
  - `SToggle`：`name="tgl_templ_type_switch"` → `R.id.tgl_templ_type_switch`，用于展开/折叠。  
  - `SCheck3State`：`name="check_filters"` → `R.id.check_filters`，用于三态勾选。

对应 XML 片段（仓库原文）：

```xml
<!-- code/UI/LbMainUI/uires/xml/dlg_filters.xml -->
<treeview name="treeview_filters" ...>
	<template ...>
		<window ...>
			<toggle name="tgl_templ_type_switch" ... />
			<check3state name="check_filters" ... />
		</window>
	</template>
</treeview>
```

### 4.3 Adapter：`getView` 职责

`CLbFiltersTreeViewAdapter::getView` 在每一行首次创建时 `InitFromXml(xmlTemplate)`，随后：

- 绑定 `EVT_CMD` 到 `OnCkFilterClick`、`OnSwitchClick`。  
- 根据 `FiltersTreeItemData` 设置勾选态、标题；**无子节点**时隐藏展开开关，并对叶子 **关闭三态**（`Enable3State(FALSE)`），与「组节点」区分。

---

## 5. 三态勾选与楼层 Adapter 的关系

三态联动逻辑（`SetCheckState`、`SetChildrenState`、`CheckChildrenState`、`CheckState` 等）与 **`CLbFloorslTreeViewAdapter`** 中的同类实现 **同构**，属于项目内可复用的 UI 状态机模式：在 **Adapter 内部树** 上维护 `Check3State`，点击后 `notifyBranchChanged(ITEM_ROOT)` 刷新展示。

**说明**：这是 **UI 层模式复用**，不是业务层复制；业务数据仍只通过 `CLbSelectComponentNode` 与 ViewModel 交互。

楼层 Adapter 中可对照到同类方法族（命名一致或同构）：`InitSelectNodes`、`GetNodesStatus`、`SetChildrenState`、`CheckChildrenState`、`CheckState`、`SetCheckState`，可作为迁移其它树控件时的参考模板。

---

## 6.1 MFC 旧实现关键片段（用于行为对照）

#### A. MFC 旧版树填充 + 名称排序

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbFiltersDialog.cpp
void CLbFiltersDialog::PopulateTreeRecursively(HTREEITEM& hRoot, const std::shared_ptr<CLbSelectComponentNode>& node)
{
	HTREEITEM tempFloor = m_wndTree.InsertItem(node->sName.c_str(), hRoot);
	m_wndTree.SetCheck(tempFloor, TRUE);
	m_wndTree.SetItemData(tempFloor, (DWORD_PTR)node.get());
	if (node->childData.empty()) return;

	if (node->childData.size() > 0)
	{
		std::sort(node->childData.begin(), node->childData.end(), compareByNodeName<std::shared_ptr<CLbSelectComponentNode>>);
	}
	for (auto& var : node->childData)
	{
		PopulateTreeRecursively(tempFloor, var);
	}
}
```

#### B. MFC 旧版确认回写

```cpp
// code/UI/LbMainUI/Frame/MenuPanel/LbFiltersDialog.cpp
void CLbFiltersDialog::ClickdOk()
{
	m_floor = m_checkFloor.GetCheck();
	m_type = m_checkType.GetCheck();
	std::vector<CLbSelectComponentNode> selectedComps;
	HTREEITEM hRootItem = m_wndTree.GetRootItem();
	TraverseTreeAndGetValues(hRootItem, selectedComps);
	m_pModel->SetPickFirst(selectedComps);
	OnOK();
}
```

---

## 7. MFC 与 SOUI 迁移时的注意点（经验条）

1. **先找入口**：谁构造对话框、是否传入 `ILbSelectionSetViewModel`、模态/非模态、父窗口句柄（当前 `CreateFilterDialog` 中取工具条/Acad 框架 HWND 的逻辑与 MFC 时代意图一致，但 **SOUI `DoModal` 调用处未传入 parent** 的细节若需与旧版完全一致，可单独评审）。  
2. **先搭 XML 再挂逻辑**：布局与间距对齐后，再接入 Adapter 与事件，便于分工与回归。  
3. **消息与事件分离**：MFC `ON_*` 与 SOUI `EVENT_*` / `MSG_*` 不要混在同一套心智模型里，按文件分区维护。  
4. **BCG 专用消息**：如 `BCGM_GRID_ROW_CHECKBOX_CLICK` 仅在 MFC 树/网格路径使用；SOUI 路径以 **控件 `EVT_CMD`** 为主。  
5. **统一风格**：与同仓库其他 SOUI 对话框一致使用 `SDpiHandler`、`R.id`、资源索引注册方式，降低后续维护成本。

---

## 8. 当前实现相对 MFC 版的差异与缺口（基于源码的事实）

以下条目均可在对应文件中核对，用于测试用例与后续迭代：

1. **子节点排序**  
   - MFC：`CLbFiltersDialog::PopulateTreeRecursively` 中对 `childData` 有 **`std::sort`**（按名称）。  
   - SOUI：`CLbFiltersTreeViewAdapter::PopulateTreeRecursively` **未见同等排序**。  
   - **影响**：若依赖字典序展示，SOUI 版顺序可能与旧版不一致，需在验收中明确是否接受或补排序。

2. **「当前楼层 / 当前类型」勾选 UI**  
   - MFC：`CLbSelectDialog` 中 `m_checkFloor`、`m_checkType` 等控件在 `InitData` 中参与逻辑（虽 `CLbFiltersDialog::InitData` 对部分控件 `ShowWindow(FALSE)`，但 `ClickdOk` 仍会写回 `m_floor`/`m_type`）。  
   - SOUI：`dlg_filters.xml` **未包含** 对应勾选控件；`m_floor`/`m_type` 仍为静态成员，但 **缺少与旧版等价的 UI 驱动更新路径**（除非其他入口修改这两个静态量）。  
   - **影响**：与「仅当前层 / 仅当前类型」相关的交互若在旧版依赖界面，需在 SOUI 版单独补齐或确认已废弃。

3. **`InitSelectNodes(m_guids)`**  
   - `LbFiltersSouiDialog` 持有 `m_guids`，但 **当前 cpp 中未见赋值接口**（对比 `LbFloorSelectDlg::SetSelectFloorsGuid` 等模式）。  
   - **结果**：默认以 **空集合** 调用 `InitSelectNodes`，即 **不会** 按外部 GUID 预选。  
   - **后续**：若需「打开即预选上次选择」，需增加 setter 或在 `CreateFilterDialog` 注入数据。

4. **Adapter 中 `UpdateAllItemFilteredState()`**  
   - `RebuildFiltersTree` 末尾调用该方法（来自基类能力）。具体过滤语义需结合 `STreeAdapterBase` 与项目内重写核对；若业务上无「过滤」需求，应确认是否为复制楼层模块时的 **冗余调用**（仅性能/无副作用则需文档注明）。

5. **头文件中遗留的 MFC 风格声明**  
   - `LbFiltersSouiDialog.h` 仍声明 `PopulateTreeRecursively(HTREEITEM& ...)`、`TraverseTreeAndGetValues` 等，**对应 .cpp 中未实现**（属历史合并/迁移残留）。  
   - **建议**：后续清理头文件，避免读者误以为 SOUI 路径仍使用 `HTREEITEM`。

6. **静态代码审阅备注（非功能说明）**  
   - `LbFiltersTreeViewAdapter::OnCkFilterClick` 中存在 **`SASSERT(pToggle)`** 与当前上下文的 `pCk` **不一致** 的笔误嫌疑；若开启断言可能影响调试体验，建议在独立提交中修正（本文档不展开改法）。

---

## 9. 建议的回归测试清单

- [ ] 打开筛选器：数据与旧版相比，**树结构完整**（根「工程」+ 各级构件）。  
- [ ] 全选/全不选/半选：父节点三态与 **子节点联动** 正确。  
- [ ] 展开/折叠：`SToggle` 与 `ExpandItem(TVC_TOGGLE)` 行为正常，刷新后状态不错乱。  
- [ ] 确定：`SetPickFirst` 收到的 `selectedComps` 与勾选叶子一致（含 **仅叶子** 收集逻辑）。  
- [ ] 命令进行中打开对话框：`LbEditorReactorMng` 对 `SelectionNotifyType` 的 **临时抑制与析构恢复** 与旧版一致（构造/析构对称）。  
- [ ] **排序**：若产品要求与 MFC 一致，对比同一工程下节点 **显示顺序**。  
- [ ] **m_type / m_floor**：若功能仍依赖二者，验证 SOUI 版是否在无 UI 情况下仍满足业务假设。

---

## 10. 推荐迁移流程（可复用到其他对话框）

1. 定位 **调用栈与 Model 传递**（谁 `new` / 栈上对话框、谁持有 `shared_ptr`）。  
2. 编写 **SOUI XML** 与 `uires.idx` 注册，做到视觉与交互占位一致。  
3. 将 **数据读写** 固定在既有 ViewModel / Service 接口上，UI 只适配数据形状。  
4. 复杂列表优先查仓库内 **已有 Adapter 范例**（如楼层树），再实现本对话框 Adapter。  
5. 事件与消息逐条 **对照 MFC 行为**，最后做 **去重、删无用类、补测试**。  

---

## 11. 版本与维护

- 若删除或完全下线 `CLbFiltersDialog`，应全局检索 `CLbFiltersDialog`、`IDD_FILTER_DLG` 等引用后再移除，并更新本文档「涉及文件」表。  
- 若 `ILbSelectionSetViewModel` 签名变更，需同步更新第 3 节数据流描述与所有调用处。

---

**文档结束。**
