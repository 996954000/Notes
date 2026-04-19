# ObjectARX 核心数据结构笔记

## 一、总体类比

学 ARX 最重要的第一步是建立这个对应关系。DWG 文件不只是一个图形文件，它在内存中是一个完整的数据库对象，所有的图层、线型、块、实体都挂在这个数据库下面。


| CAD 概念    | ARX 对象                     | 类比理解            |
| --------- | -------------------------- | --------------- |
| 一个 DWG 文件 | `AcDbDatabase`             | 整个游戏世界的数据容器     |
| 符号表       | `AcDbSymbolTable` 各子类      | 各类资源的注册管理器      |
| 块定义       | `AcDbBlockTableRecord`     | Unity 预制体       |
| 块参照       | `AcDbBlockReference`       | Unity 场景中实例化的对象 |
| 模型空间      | 特殊的 `AcDbBlockTableRecord` | 游戏的主场景          |
| 实体（线、圆弧…） | `AcDbEntity` 各子类           | 场景中的具体对象        |


---

## 二、数据库结构：两大部分

一个 `AcDbDatabase` 在内存中包含**两大部分**，初学时容易只关注符号表而忽略命名对象字典：

```
AcDbDatabase
├── 符号表（Symbol Tables）
│   └── 管理"有名字的、全局共享的资源"
│       如：图层、线型、文字样式、块定义
│
└── 命名对象字典（Named Objects Dictionary）
    └── 管理"更复杂的结构化数据"
        如：布局、对象组、材质、插件自定义数据
```

**两者的本质区别：**

- 符号表是 CAD 规范定死的九张表，结构固定，每张表管理一类资源
- 命名对象字典是一个可以自由嵌套的键值对树，CAD 自己和第三方插件都可以往里放数据，结构灵活

---

## 三、九张符号表

每张符号表管理一类具名资源。"表"里面存的是"记录"，记录就是具体的配置项。

操作模式是**统一**的：先从数据库拿到表 → 再从表里拿记录 → 再操作记录。学会一张就会用其他的。


| 符号表类                 | 存放内容              | 记录类                        |
| -------------------- | ----------------- | -------------------------- |
| `AcDbBlockTable`     | 块定义（含模型空间、图纸空间）   | `AcDbBlockTableRecord`     |
| `AcDbLayerTable`     | 图层配置（颜色、线型、可见性等）  | `AcDbLayerTableRecord`     |
| `AcDbTextStyleTable` | 文字样式（字体、字高、宽度比例等） | `AcDbTextStyleTableRecord` |
| `AcDbLinetypeTable`  | 线型定义（实线、虚线、点划线等）  | `AcDbLinetypeTableRecord`  |
| `AcDbDimStyleTable`  | 标注样式（箭头、文字、精度等）   | `AcDbDimStyleTableRecord`  |
| `AcDbViewTable`      | 命名视图（保存的视角配置）     | `AcDbViewTableRecord`      |
| `AcDbViewportTable`  | 视口配置              | `AcDbViewportTableRecord`  |
| `AcDbUCSTable`       | 用户坐标系             | `AcDbUCSTableRecord`       |
| `AcDbRegAppTable`    | 注册应用程序（给扩展数据打标签用） | `AcDbRegAppTableRecord`    |


> **注意：** 线型是 `AcDbLinetypeTable`（控制线的形状），文字样式是 `AcDbTextStyleTable`（控制文字字体和外观），是两张不同的表，初学容易混淆。

---

## 四、块表——最核心的符号表

块表的地位特殊，因为**所有实体都必须挂在某个块表记录下面**，没有游离在数据库外的实体。块表是实体的最终归宿。

### 4.1 系统预设的两条记录

打开任何一个 DWG，块表里至少有两条系统预设记录：

```
AcDbBlockTable
├── *Model_Space   → 模型空间（主要绘图区域）
└── *Paper_Space   → 默认图纸空间（布局空间）
    *Paper_Space0  → 第二个布局（有多个布局时）
    *Paper_Space1  → 第三个布局
    ...
```

**模型空间的本质：** 它就是一条普通的 `AcDbBlockTableRecord`，只不过名字被 CAD 规范约定为 `*Model_Space`，CAD 的绘图区域默认显示这条记录里的所有实体。往这条记录里 `appendAcDbEntity()` 追加实体，实体就出现在屏幕上。

代码中获取模型空间的方式：

```cpp
// 方式一：通过数据库直接获取 ObjectId
AcDbObjectId modelSpaceId = acdbCurDwg()->modelSpaceId();

// 方式二：通过块表按名字查找
AcDbBlockTable *pBlockTable;
acdbCurDwg()->getBlockTable(pBlockTable, AcDb::kForRead);
AcDbBlockTableRecord *pModelSpace;
pBlockTable->getAt(ACDB_MODEL_SPACE, pModelSpace, AcDb::kForWrite);
```

### 4.2 用户自定义的块

用户用 `BLOCK` 命令，或通过代码创建 `AcDbBlockTableRecord` 并添加到块表，就形成了一个**块定义**。块定义本身不显示在屏幕上，只是一个模板。

---

## 五、块定义与块参照（预制体与实例的关系）

这是 ARX 里最重要的设计之一，和 Unity 预制体体系完全对应：

```
块定义（AcDbBlockTableRecord）
  ├── 描述"这个块长什么样"
  ├── 内部包含若干实体（如线、弧、文字等）
  ├── 存放在块表中
  └── 本身不直接显示在屏幕上

块参照（AcDbBlockReference）
  ├── 引用某个块定义（存储其 ObjectId）
  ├── 指定插入点、缩放比例、旋转角度
  ├── 本身是一个实体，放进模型空间后才显示
  └── 多个块参照可以引用同一个块定义（一对多）
```

**关键特性：** 修改块定义，所有引用它的块参照都会同步更新——本质上它们都只是在引用同一份数据，自身不存储图形内容。

**与模型空间的关系：** 模型空间 `*Model_Space` 本身也是一条块表记录，往里面加一个 `AcDbBlockReference`，该块参照就显示在屏幕上；往里面直接加 `AcDbLine`，线就直接显示在屏幕上。两者都是往这个"容器"里装东西。

---

## 六、对象标识：ObjectId 与 Handle

ARX 里的对象不能长期持有裸指针，因为数据库有自己的内存管理机制，对象可能随时被换入换出。正确的做法是存**标识符**，需要时再用标识符换取指针。

ARX 提供两种标识符，用途不同：


| 标识符            | 类型              | 有效期             | 用途         |
| -------------- | --------------- | --------------- | ---------- |
| `AcDbObjectId` | 运行时 ID          | 仅当前会话有效，重启失效    | 代码中传递和引用对象 |
| `AcDbHandle`   | 持久化 ID（8字节十六进制） | 保存在 DWG 中，跨会话不变 | 跨会话追踪同一个对象 |


### 6.1 为什么不直接用裸指针

```
❌ 错误：长期保存 AcDbObject* 指针
   → 指针随时可能失效（对象被页出内存）

✅ 正确：保存 AcDbObjectId
   → 需要时用 ObjectId 打开对象，用完 close()
```

### 6.2 两者互转

```cpp
// ObjectId → Handle
AcDbHandle handle;
pObj->getAcDbHandle(handle);

// Handle → ObjectId（需要知道属于哪个数据库）
AcDbObjectId objId;
acdbCurDwg()->getAcDbObjectId(objId, false, handle);
```

### 6.3 ObjectId 的典型使用场景

```cpp
// 存储一个图层的 ObjectId，后续赋给实体
AcDbObjectId layerId;
pLayerTable->getAt(_T("MyLayer"), layerId);

// 创建实体后，通过 ObjectId 指定图层
pLine->setLayerId(layerId);

// 块参照内部存的也是块定义的 ObjectId
AcDbObjectId blockDefId = pBlockRef->blockTableRecord();
```

> 整个 ARX 代码中大量出现的 `AcDbObjectId` 传参，本质都是在传递"对某个数据库对象的引用"，而不是直接传指针。

---

## 七、对象所有权链

ARX 中所有对象都有且只有一个**所有者（Owner）**，形成严格的树状结构。

所有权决定两件事：

1. 对象保存在 DWG 文件里的哪个位置
2. 删除父对象时，子对象是否级联删除

```
AcDbDatabase
  └── AcDbBlockTable（所有者是数据库）
        └── AcDbBlockTableRecord（所有者是块表）
              ├── AcDbLine（所有者是这条记录）
              ├── AcDbCircle（所有者是这条记录）
              └── AcDbBlockReference（所有者是这条记录）
                    └── → 通过 ObjectId 引用另一个 BlockTableRecord（引用≠拥有）
```

> **重点区分"引用"和"拥有"：** 块参照通过 ObjectId **引用**块定义，但不**拥有**它。块定义的所有者是块表。删除块参照，块定义不会被删除；删除块定义，块参照变成悬空引用。

其他符号表遵循同样结构：

```
AcDbDatabase
  ├── AcDbLayerTable
  │     └── AcDbLayerTableRecord（"0"图层、自定义图层...）
  ├── AcDbTextStyleTable
  │     └── AcDbTextStyleTableRecord（"Standard"样式...）
  └── ...
```

---

## 八、实体的通用属性

所有 `AcDbEntity` 子类都继承了一组公共属性。这些属性**不在实体自身存储最终值**，而是存储一个指向符号表记录的引用（图层名或 ObjectId），实际显示时从符号表记录里读取。


| 属性  | 方法                                 | 对应符号表                 |
| --- | ---------------------------------- | --------------------- |
| 图层  | `layer()` / `setLayer()`           | `AcDbLayerTable`      |
| 线型  | `linetype()` / `setLinetype()`     | `AcDbLinetypeTable`   |
| 颜色  | `color()` / `setColor()`           | 无（直接存 `AcCmColor` 对象） |
| 线宽  | `lineWeight()` / `setLineWeight()` | 无（直接存枚举值）             |


### 8.1 ByLayer / ByBlock / 显式颜色

颜色有三种设置方式，这是 CAD 特有的继承机制：

```
ByLayer（默认）：实体颜色 = 所在图层的颜色
                 → 改图层颜色，该图层所有实体颜色跟着变

ByBlock：        实体颜色 = 包含它的块参照的颜色
                 → 用于制作"跟随块"颜色的块定义

显式颜色：       实体有自己固定的颜色，不跟随图层
```

这解释了为什么 CAD 里改一个图层颜色，所有在该图层上的实体颜色都跟着变——实体存的只是"我属于哪个图层"，颜色是从图层记录动态读取的。

### 8.2 代码示例

```cpp
// 读取属性
AcString layerName = pEnt->layer();
AcCmColor color = pEnt->color();

// 设置属性（实体必须以 kForWrite 打开）
pEnt->setLayer(_T("MyLayer"));

// 颜色设置为 ByLayer
AcCmColor color;
color.setColorMethod(AcCmEntityColor::kByLayer);
pEnt->setColor(color);
```

---

## 九、AcGe 几何库

`AcGe` 是 ARX 提供的纯数学库，用于几何运算。它与数据库完全无关，没有 open/close，对象生命周期就是普通 C++ 对象。


| 类              | 用途       | 常见用法                         |
| -------------- | -------- | ---------------------------- |
| `AcGePoint3d`  | 三维点      | `AcGePoint3d pt(x, y, z)`    |
| `AcGeVector3d` | 三维向量     | `AcGeVector3d v(dx, dy, dz)` |
| `AcGePoint2d`  | 二维点      | `AcGePoint2d pt(x, y)`       |
| `AcGeVector2d` | 二维向量     | —                            |
| `AcGeMatrix3d` | 4×4 变换矩阵 | 平移、旋转、缩放、镜像                  |
| `AcGeLine3d`   | 无限直线     | 几何求交等运算                      |
| `AcGePlane`    | 平面       | 实体法向量所在平面                    |


### 9.1 常用操作

```cpp
// 点与向量运算
AcGePoint3d pt1(0, 0, 0), pt2(10, 0, 0);
AcGeVector3d vec = pt2 - pt1;          // 向量
AcGePoint3d mid = pt1 + vec * 0.5;    // 中点

// 变换矩阵
AcGeMatrix3d mat;
mat.setToTranslation(AcGeVector3d(5, 5, 0));  // 平移矩阵
mat.setToRotation(M_PI / 4, AcGeVector3d::kZAxis, AcGePoint3d::kOrigin); // 旋转矩阵

// 应用变换到实体（实体需以 kForWrite 打开）
pEnt->transformBy(mat);
```

### 9.2 内存管理

```cpp
// 栈上分配：自动释放，无需手动管理
AcGePoint3d pt(1, 2, 3);

// 堆上分配：需要 delete
AcGePoint3d *pPt = new AcGePoint3d(1, 2, 3);
delete pPt;   // 普通 C++ 对象，直接 delete
```

> `AcGe` 对象不是 `AcDbObject` 子类，不涉及数据库，永远用 `delete` 或栈分配，**不要用 `close()`**。

---

## 十、数据库访问模式

### 10.1 三种开关模式

每次从数据库"取出"一个对象时，必须指定访问模式。这是数据库的并发访问控制机制：


| 模式   | 常量                 | 含义                      |
| ---- | ------------------ | ----------------------- |
| 读模式  | `AcDb::kForRead`   | 共享读，多个调用者可同时持有，不能修改对象   |
| 写模式  | `AcDb::kForWrite`  | 独占写，同一时刻只有一个调用者能持有，可修改  |
| 通知模式 | `AcDb::kForNotify` | 配合 Reactor 使用，通常不需要直接使用 |


**规则：**

- 只读操作用 `kForRead`，代价低
- 需要修改对象时用 `kForWrite`，修改完必须 `close()`
- 对一个已经以 `kForWrite` 打开的对象再次 `kForWrite` 打开，会返回 `eWasOpenForWrite` 错误

```cpp
// 只读：不修改内容时用 kForRead
AcDbBlockTable *pBT;
acdbCurDwg()->getBlockTable(pBT, AcDb::kForRead);

// 读写：需要修改时用 kForWrite
AcDbBlockTableRecord *pModelSpace;
pBT->getAt(ACDB_MODEL_SPACE, pModelSpace, AcDb::kForWrite);
pModelSpace->appendAcDbEntity(pLine);   // 修改操作
pModelSpace->close();
```

### 10.2 遍历数据库对象（迭代器模式）

ARX 使用迭代器模式遍历表或块表记录内的对象：

```cpp
// 遍历模型空间内所有实体
AcDbBlockTableRecordIterator *pIter;
pModelSpace->newIterator(pIter);

for (; !pIter->done(); pIter->step()) {
    AcDbEntity *pEnt;
    pIter->getEntity(pEnt, AcDb::kForRead);

    // 处理实体...
    AcRxClass *pClass = pEnt->isA();  // 获取实体类型

    pEnt->close();   // 每个实体用完必须 close
}

delete pIter;   // 迭代器是非数据库 C++ 对象，用 delete 释放
```

```cpp
// 遍历图层表内所有图层
AcDbLayerTable *pLayerTable;
acdbCurDwg()->getLayerTable(pLayerTable, AcDb::kForRead);

AcDbSymbolTableIterator *pIter;
pLayerTable->newIterator(pIter);

for (; !pIter->done(); pIter->next()) {
    AcDbLayerTableRecord *pLayer;
    pIter->getRecord(pLayer, AcDb::kForRead);

    AcString name;
    pLayer->getName(name);

    pLayer->close();
}

delete pIter;
pLayerTable->close();
```

> **迭代器的内存管理：** 迭代器本身（`AcDbBlockTableRecordIterator`*、`AcDbSymbolTableIterator*`）是用 `newIterator()` 创建的非数据库 C++ 对象，用完必须 `delete`。

---

## 十一、数据流：从 DWG 到屏幕的完整路径

### 读取路径（CAD 打开文件时）

```
读取 DWG 文件
  → 反序列化为 AcDbDatabase（内存中）
  → 解析符号表，建立各类资源索引
  → 访问 *Model_Space 这条 AcDbBlockTableRecord
  → 遍历其中所有 AcDbEntity
  → 渲染引擎将实体绘制到屏幕
```

### 写入路径（代码添加实体）

```
创建实体对象（如 new AcDbLine(startPt, endPt)）
  → 打开 *Model_Space 记录（以写模式 kForWrite）
  → 调用 pModelSpace->appendAcDbEntity(pEntity)
  → 实体的所有者变为 *Model_Space 这条记录
  → 数据库标记为"已修改"
  → 实体出现在屏幕上
```

---

## 十二、命名对象字典（初步了解）

符号表解决不了的问题，交给命名对象字典处理。它的本质是一个 `AcDbDictionary` 对象，键是字符串，值是任意 `AcDbObject`，可以嵌套形成树状结构。

获取根字典：

```cpp
AcDbDictionary *pNamedObjDict;
acdbCurDwg()->getNamedObjectsDictionary(pNamedObjDict, AcDb::kForRead);
```

**CAD 自己往里放的内容：**


| 键名                  | 内容            |
| ------------------- | ------------- |
| `ACAD_LAYOUT`       | 所有布局（图纸空间的配置） |
| `ACAD_GROUP`        | 对象组           |
| `ACAD_MLINESTYLE`   | 多线样式          |
| `ACAD_PLOTSETTINGS` | 打印设置          |


**插件往里放的内容：**

- 扩展数据（XData）：挂在单个实体上的自定义数据
- 扩展词典（Extension Dictionary）：每个 `AcDbObject` 可以挂一个子字典，存储与该对象绑定的插件数据

> 命名对象字典和扩展数据是插件开发中存储自定义数据的两种主要方式，具体用法待后续深入。

---

## 十三、内存管理与对象释放

ARX 里释放对象的方式不统一，**释放方式由对象的来源和归属决定**，不能混用。

### 13.1 三种对象类型与对应释放方式


| 对象类型           | 典型来源                           | 释放方式                           |
| -------------- | ------------------------------ | ------------------------------ |
| 数据库对象（已在 DB 中） | `getBlockTable`、`getAt`、`open` | `close()`                      |
| C++ 堆对象（未入库）   | 自己 `new`、尚未 `append` 进数据库      | `delete`                       |
| C 风格结构体        | `resbuf`、选择集                   | `acutRelRb()` / `acedSSFree()` |


### 13.2 `close()` — 数据库对象用

从数据库"签出"的对象用完后必须归还，`close()` 是归还操作，**不是销毁**，对象依然活在数据库里。

```cpp
AcDbBlockTable *pBT;
acdbCurDwg()->getBlockTable(pBT, AcDb::kForRead);
// ... 使用 pBT
pBT->close();   // 归还，对象仍然存在于数据库中
```

对数据库对象误用 `delete`：数据库内部索引变成悬空指针，导致崩溃或保存出错。

### 13.3 `delete` — 自己 new 出来、尚未入库的对象用

**成功加入数据库后：** 数据库接管所有权，只需 `close()` 自己这侧的持有

```cpp
pModelSpace->appendAcDbEntity(pLine);  // 数据库接管
pLine->close();                        // 不是 delete
```

**加入数据库失败，需要清理：** 对象没进数据库，你仍然拥有它

```cpp
Acad::ErrorStatus es = pModelSpace->appendAcDbEntity(pLine);
if (es != Acad::eOk) {
    delete pLine;   // 必须 delete，否则内存泄漏
}
```

> **判断依据：这个对象有没有进入数据库？** 进了用 `close()`，没进用 `delete`。

### 13.4 `acutRelRb()` — resbuf 链表用

`resbuf` 是 AutoLISP 接口遗留的 C 风格链表结构，不是 C++ 对象，没有析构函数，只能用专用函数释放。常见于读写扩展数据（XData）和接收命令行参数。

```cpp
resbuf *pXdata = pEntity->xData(_T("MY_APP"));
// ... 使用
acutRelRb(pXdata);   // 递归释放整个链表的所有节点
```

> `resbuf` 是链表，`acutRelRb` 会递归释放所有 `next` 节点。若用 `delete` 只释放第一个节点，其余全部泄漏。

### 13.5 释放方式汇总


| 释放方式             | 适用对象                 | 获取方式                        |
| ---------------- | -------------------- | --------------------------- |
| `pObj->close()`  | 从数据库打开的 `AcDbObject` | `getXxx` / `open`           |
| `delete pObj`    | 自己 new 的、未入库的 C++ 对象 | `new`                       |
| `acutRelRb(pRb)` | `resbuf*` 链表         | `xData()`、`acedGetArgs()` 等 |
| `acedSSFree(ss)` | `ads_name`（选择集）      | `acedSSGet()`               |
| `delete pIter`   | 迭代器对象                | `newIterator()`             |
| 事务自动托管           | 事务内打开的所有对象           | `pTrans->getObject()`       |


### 13.6 判断决策树

```
拿到一个指针，该如何释放？

是 resbuf* 类型？
  → 是：acutRelRb()

是 ads_name（选择集）？
  → 是：acedSSFree()

是迭代器（BlockTableRecordIterator 等）？
  → 是：delete

是 AcGe 库对象（Point3d、Vector3d 等）？
  → 是：delete（或栈上分配无需手动释放）

是 AcDbObject 子类？
  └── getXxx / open 调用成功了吗？（返回值 == Acad::eOk）
        ├── 失败：指针为 nullptr，对象从未被签出，什么都不做
        └── 成功：对象已被签出
              ├── 在数据库中（通过 getXxx / open 获取）→ close()
              └── 不在数据库中（自己 new 的，还没 append）→ delete
```

**getXxx 失败时为何什么都不做：**

```cpp
AcDbBlockTable *pBlkTable = nullptr;
Acad::ErrorStatus es = acdbCurDwg()->getBlockTable(pBlkTable, AcDb::kForWrite);

if (es != Acad::eOk) {
    // pBlkTable 此时是 nullptr，对象从未被签出
    // close(nullptr)  → 崩溃
    // delete nullptr  → 语义错误（它不是你 new 的）
    // 正确做法：直接处理错误，无需任何释放
    return;
}

// 走到这里 pBlkTable 才有效
pBlkTable->close();
```

这个规律适用于所有 `getXxx` 操作（`getLayerTable`、`getAt`、`open` 等）：**调用失败 = 从未签出 = 无需释放**。

### 13.7 事务模式下的特殊情况

事务内通过 `getObject()` 打开的对象**既不需要 close() 也不需要 delete**，事务结束时统一处理：

```cpp
AcTransaction *pTrans = actrTransactionManager->startTransaction();
AcDbObject *pObj;
pTrans->getObject(pObj, objId, AcDb::kForWrite);
// 直接使用，不要手动 close，不要 delete
actrTransactionManager->endTransaction();  // 自动处理所有已打开的对象
```

这是事务机制的重要价值之一——把分散的 `close()` 调用统一托管，避免遗漏导致的资源泄漏。

---

## 十四、整体结构总览

```
AcDbDatabase（一个 DWG 文件）
│
├── 符号表（九张固定的表）
│   ├── AcDbBlockTable ← 最核心，所有实体的归宿
│   │   ├── *Model_Space（模型空间，装实体的主容器）
│   │   ├── *Paper_Space（图纸空间）
│   │   └── 自定义块定义1、2...（每条记录内含实体）
│   ├── AcDbLayerTable
│   │   └── AcDbLayerTableRecord（"0"图层、自定义图层...）
│   ├── AcDbTextStyleTable
│   ├── AcDbLinetypeTable
│   ├── AcDbDimStyleTable
│   ├── AcDbViewTable
│   ├── AcDbViewportTable
│   ├── AcDbUCSTable
│   └── AcDbRegAppTable
│
└── 命名对象字典（AcDbDictionary，可自由嵌套）
    ├── ACAD_LAYOUT（布局管理）
    ├── ACAD_GROUP（对象组）
    └── 插件自定义键值...

每个对象（AcDbObject）
  ├── 有唯一的 ObjectId（运行时标识）
  ├── 有唯一的 Handle（持久化标识，存入 DWG）
  ├── 有唯一的所有者（Owner）
  └── 可附加扩展数据（XData）或扩展词典
```

---

## 十五、用户交互

### 15.1 交互层次总览

```
┌─────────────────────────────────────────┐
│  Jig（拖拽追踪）                          │
│  AcEdJig — 鼠标移动时实时更新实体位置      │
├─────────────────────────────────────────┤
│  选择集（Selection Set）                  │
│  acedSSGet — 让用户框选多个实体           │
├─────────────────────────────────────────┤
│  单体选取                                 │
│  acedEntSel — 让用户点选一个实体          │
├─────────────────────────────────────────┤
│  基础输入                                 │
│  acedGetPoint / acedGetInt / ...        │
└─────────────────────────────────────────┘
```

### 15.2 基础输入（acedGetXxx 系列）

| 函数 | 作用 | 结果写入 |
|---|---|---|
| `acedGetPoint()` | 让用户点击一个点 | `ads_point`（即 `double[3]`） |
| `acedGetInt()` | 让用户输入整数 | `int*` |
| `acedGetReal()` | 让用户输入实数 | `ads_real*`（即 `double*`） |
| `acedGetDist()` | 让用户指定距离 | `ads_real*` |
| `acedGetAngle()` | 让用户指定角度（弧度） | `ads_real*` |
| `acedGetString()` | 让用户输入字符串 | `char*` |
| `acedGetKword()` | 让用户从关键字中选一个 | `char*` |

**返回值是状态码，不是结果：**

| 状态码 | 含义 |
|---|---|
| `RTNORM` | 正常输入 |
| `RTCAN` | 用户按 Esc 取消 |
| `RTNONE` | 用户直接回车（未输入） |

**`acedInitGet()`** 必须在 `acedGetXxx` 之前调用，用于限制输入行为（禁止负数、禁止零、禁止回车跳过等）。

```cpp
ads_point pt;
acedInitGet(RSG_NOZERO | RSG_NONEG, NULL);   // 不允许零点和负值
int rc = acedGetPoint(NULL, _T("\n指定点: "), pt);
if (rc != RTNORM) return;   // 用户取消或异常，直接返回
```

### 15.3 单体选取

```
acedEntSel  → 用户点选一个实体 → 直接返回该实体的 ads_name
acedSSGet   → 用户框选多个实体 → 返回选择集的 ads_name（容器句柄）
                                       ↓
                                  acedSSName(ss, i) 按索引取每个实体的 ads_name
```

两者后续操作完全一样：拿到实体 `ads_name` → 转 `ObjectId` → `open` 操作。

```cpp
ads_name ename;
ads_point pickPt;
int rc = acedEntSel(_T("\n选择实体: "), ename, pickPt);
if (rc != RTNORM) return;

AcDbObjectId objId;
acdbGetObjectId(objId, ename);    // ads_name → ObjectId

AcDbEntity *pEnt;
acdbOpenObject(pEnt, objId, AcDb::kForRead);
// ... 操作实体
pEnt->close();
```

`acedNEntSel()` 是 `acedEntSel` 的增强版，支持选到块参照内部的嵌套实体，同时会返回将嵌套实体坐标变换到世界坐标系的变换矩阵。

### 15.4 选择集

选择集是 CAD 内部维护的**实体名列表**，你持有的 `ads_name ss` 只是这个列表的句柄，不是实体本身。选择集相当于多选版的 `acedEntSel`。

**创建选择集：**

```cpp
ads_name ss;

// 完全交互：让用户自己框选，支持所有方式
acedSSGet(NULL, NULL, NULL, NULL, ss);

// 代码指定范围：窗选
ads_point pt1 = {0, 0, 0}, pt2 = {100, 100, 0};
acedSSGet(_T("W"), pt1, pt2, NULL, ss);

// 交叉选（穿越窗口边界也选中）
acedSSGet(_T("C"), pt1, pt2, NULL, ss);

// 带过滤器：只选特定类型或图层的实体
resbuf *pFilter = acutBuildList(RTDXF0, _T("LINE"), RTNONE);
acedSSGet(_T("W"), pt1, pt2, pFilter, ss);
acutRelRb(pFilter);   // 过滤器传进去后立即释放，与选择集生命周期无关
```

**读取选择集内容：**

```cpp
long len = 0;
acedSSLength(ss, &len);          // 获取实体数量

for (long i = 0; i < len; i++) {
    ads_name ename;
    acedSSName(ss, i, ename);    // 按索引取实体的 ads_name

    AcDbObjectId objId;
    acdbGetObjectId(objId, ename);

    AcDbEntity *pEnt;
    acdbOpenObject(pEnt, objId, AcDb::kForRead);
    // ... 处理实体
    pEnt->close();
}

acedSSFree(ss);   // 选择集用完必须释放，不是 delete
```

**选择集的完整生命周期：**

```
acedSSGet()     → 创建选择集，拿到句柄 ss
acedSSLength()  → 获取实体数量
acedSSName()    → 按索引逐个取实体 ads_name
acdbGetObjectId → 转为 ObjectId
open / close    → 操作实体
acedSSFree()    → 释放选择集
```

### 15.5 ads_ 前缀的来历

`ads_` 前缀来自 AutoCAD Development System（ADS），ARX 的 C 语言前身。ARX 出现后没有完全替换 ADS，而是将其包含进来继续使用，所以两套 API 混用至今：

| ADS 类型 | 实质 | 说明 |
|---|---|---|
| `ads_name` | `long[2]` | 实体或选择集的句柄 |
| `ads_point` | `double[3]` | 三维点坐标数组 |
| `ads_real` | `double` | 实数 |

这类函数参数都是 C 风格（数组、指针），不是 C++ 对象，不涉及 open/close。

### 15.6 resbuf 的三个使用场景

`resbuf` 是 ADS 遗留的 C 风格通用链表节点，因为 C 语言没有泛型，所以用联合体 + 类型码来装任意类型的值。**它是数据载体，不是容器**，不参与存储选择集实体列表。

**结构：**

```cpp
struct resbuf {
    resbuf *rbnext;    // 下一个节点
    short   restype;   // 类型码，决定读联合体的哪个字段
    union {
        double  rreal;       // RTREAL
        double  rpoint[3];   // RT3DPOINT
        short   rint;        // RTSHORT
        char   *rstring;     // RTSTR
        long    rlname[2];   // RTENAME（实体名）
        long    rlong;       // RTLONG
    } resval;
};
```

**核心规则：restype 决定读哪个字段，两者必须匹配，否则读出来是垃圾数据。**

| restype | 对应字段 | 含义 |
|---|---|---|
| `RTSTR` | `resval.rstring` | 字符串 |
| `RTREAL` | `resval.rreal` | 实数 |
| `RTSHORT` | `resval.rint` | 短整型 |
| `RTLONG` | `resval.rlong` | 长整型 |
| `RT3DPOINT` | `resval.rpoint` | 三维点 |
| `RTENAME` | `resval.rlname` | 实体名（ads_name） |
| `RTNONE` | — | 链表终止标记 |

**三个出现场景（类型码含义不同，注意区分）：**

| 场景 | 类型码 | 方向 |
|---|---|---|
| 选择集过滤器 | DXF 组码（0=实体类型，8=图层…） | 你构造后传入 `acedSSGet` |
| 扩展数据 XData | DXF 组码（1001=应用名，1000=字符串…） | `pEnt->xData()` 返回给你 |
| 命令行 / LISP 参数 | RT* 系列（RTSTR、RTREAL…） | `acedGetArgs()` 返回给你 |

**构造推荐用 `acutBuildList`，不要手动 new 节点：**

```cpp
// 选择集过滤器：只选 LINE 类型、在 MyLayer 图层上的实体
resbuf *pFilter = acutBuildList(
    RTDXF0, _T("LINE"),
    8,      _T("MyLayer"),
    RTNONE
);
acedSSGet(_T("W"), pt1, pt2, pFilter, ss);
acutRelRb(pFilter);

// 扩展数据
resbuf *pXdata = acutBuildList(
    1001, _T("MY_APP"),
    1000, _T("hello"),
    1070, 42,
    RTNONE
);
```

**遍历读取：**

```cpp
for (resbuf *pRb = pHead; pRb != nullptr; pRb = pRb->rbnext) {
    switch (pRb->restype) {
        case RTSTR:  /* 用 pRb->resval.rstring */ break;
        case RTREAL: /* 用 pRb->resval.rreal   */ break;
        case RTSHORT:/* 用 pRb->resval.rint    */ break;
    }
}
```

### 15.7 Jig（鼠标拖拽追踪）

`AcEdJig` 是实现"创建实体时跟随鼠标预览"效果的机制。需要继承该类并实现两个方法：

| 方法 | 调用时机 | 作用 |
|---|---|---|
| `sampler()` | 鼠标每次移动时 | 采样当前鼠标位置，存入成员变量 |
| `update()` | 采样后自动调用 | 根据新位置更新实体几何，返回是否需要重绘 |

基本流程：
```
继承 AcEdJig
  → 在 sampler() 中用 acquirePoint() 采样鼠标位置
  → 在 update() 中修改实体坐标
  → 调用 drag() 启动拖拽循环
  → 用户点击后 drag() 返回，实体位置确定
  → 将实体 append 进模型空间
```

---

## 十六、目前已知边界 & 待深入的方向

| 方向 | 当前状态 | 待学内容 |
|---|---|---|
| 数据库结构 | 基本掌握 | 命名对象字典的读写操作 |
| 对象标识 | 了解概念 | Handle 的跨会话追踪用法 |
| 实体操作 | 会基础添加 | 实体属性修改 |
| 几何运算 | 了解类型 | AcGeMatrix3d 变换的实际应用 |
| 图纸空间 | 未接触 | Paper Space、布局、视口的关系 |
| 事务机制 | 知道写法，原理待理解 | 为什么要用事务、读写模式的意义 |
| 扩展数据 | 浅度接触 | XData 和扩展词典的完整读写流程 |
| 用户交互 | 基本掌握 | Jig 的完整实现 |
| 插件行为 | 未接触 | 命令注册、Reactor 反应器 |


