# Heterostructure Builder 3.1.0

Heterostructure Builder 是一个用于构建周期性异质结构的桌面程序。程序基于 `pymatgen` 的表面生成工具和 Zur–McGill（ZSL）二维晶格匹配方法，可从常见晶体结构文件生成二层垂直异质结、三层垂直异质结和二组分横向异质结，并输出可直接用于后续结构检查或第一性原理计算的 POSCAR、CIF 和 JSON 报告文件。

程序的目标是自动完成以下几类工作：

- 从体相结构按照指定 Miller 指数生成 slab；
- 枚举可能的表面终止方式；
- 搜索满足给定长度、夹角和面积约束的二维共格超胞；
- 构建二层或三层垂直堆叠结构；
- 构建周期性的左右横向 `A|B` 结构；
- 记录晶格失配、终止面、超胞变换矩阵、面积和警告信息。

> **重要说明**
>
> ZSL 匹配解决的是二维周期晶格的几何共格问题，并不能判断哪一种表面终止、横向平移、原子配位或界面化学状态在能量上最稳定。生成结构后，应使用 VESTA、OVITO、VMD 等工具检查结构，并根据研究目的进行进一步的结构弛豫和能量比较。

---

## 1. 支持的构建模式

程序顶部的 **Mode** 用于选择构建方式。

### Vertical A/B

构建上下堆叠的二层异质结：

```text
B
──────── interface
A
```

A 作为参考层（reference layer），程序寻找 A 和 B 的二维 ZSL 共格超胞，并将 B 的面内晶格匹配到 A 对应的目标超胞。

A/B 之间的距离由 **Gap A/B** 控制。

---

### Vertical A/B/C

构建上下堆叠的三层异质结：

```text
C
──────── B/C interface
B
──────── A/B interface
A
```

程序采用顺序匹配：

1. 首先搜索 A/B 的共格超胞；
2. 保留若干较优的 A/B 候选；
3. 再将 C 与这些 A/B 共格结构的二维周期晶格进行 ZSL 匹配；
4. 若匹配 C 需要进一步扩大 A/B 的超胞，则整体扩大 A/B；
5. 最终 A、B、C 共用同一个二维周期晶格。

因此三层结构不是把两个独立的 A/B 和 B/C 模型直接拼接。

A/B 和 B/C 的距离分别由 **Gap A/B** 和 **Gap B/C** 控制。

---

### Lateral A|B

构建左右相接的横向异质结：

```text
A | B
```

程序首先寻找 A 和 B 的二维共格超胞，然后沿指定的 `a` 或 `b` 晶格方向扩展，并分别保留左侧和右侧区域。

由于使用周期性边界条件，一个周期性 `A|B` 超胞通常包含两个横向界面：

```text
... B | A | B | A ...
```

横向模式中的参考晶格由 **Reference side** 控制。

---

# 2. 输入结构

## Structure file

每一层或每一个区域都需要一个晶体结构文件。

GUI 直接支持：

- CIF：`*.cif`
- VASP：`*.vasp`
- POSCAR：`*.poscar`
- `POSCAR*`
- `CONTCAR*`

程序内部使用 `pymatgen` 读取结构，因此部分其他 `pymatgen Structure.from_file()` 支持的格式也可能可以读取。

对于 CIF，程序读取完整晶胞而不是主动转换成 primitive cell。

---

# 3. Layers 区域

## Layer A / B / C

用于指定参与异质结构建的材料。

不同模式需要的输入为：

| Mode | 使用的层 |
|---|---|
| Vertical A/B | A、B |
| Vertical A/B/C | A、B、C |
| Lateral A\|B | A、B |

在二层和横向模式中，Layer C 会自动禁用。

---

## Miller h k l

指定从体相结构切割 slab 的 Miller 指数。

输入三个整数，例如：

```text
1 0 0
1 1 0
1 1 1
0 0 1
```

不能输入：

```text
0 0 0
```

不同材料可以使用不同的 Miller 面。例如 A 可以使用 `(1 1 1)`，B 可以使用 `(0 0 1)`。

Miller 指数决定界面的晶面取向，是影响 ZSL 匹配结果最重要的输入之一。

---

## Thickness

指定每个 slab 的最小厚度。

该数值的单位由 **Thickness in layers** 决定。

### Thickness in layers = 开启

Thickness 按晶面层数/单位面层理解。

默认：

```text
Thickness = 2
```

表示请求一个至少满足相应层数定义的 slab。

这是程序默认方式，适合希望直接控制 slab 层数的普通使用场景。

### Thickness in layers = 关闭

Thickness 按 Å 处理。

例如：

```text
Thickness = 10
```

表示请求约至少 `10 Å` 厚的 slab。

> 实际生成的 slab 厚度可能略大于输入值，因为程序需要保持完整的晶体层和合法的周期结构。

---

# 4. Geometry and ZSL 参数

## Gap A/B (Å)

默认：

```text
2.0 Å
```

控制垂直结构中 A 与 B 两个 slab 之间的初始界面间距。

对于：

- `Vertical A/B`：有效；
- `Vertical A/B/C`：控制 A/B 界面；
- `Lateral A|B`：不作为左右界面的间距参数使用。

较小的 Gap 会让两个 slab 更接近，较大的 Gap 会让它们更分离。

### 普通建议

初始结构通常可以从：

```text
1.5–3.0 Å
```

范围开始测试。

Gap 只是初始几何距离，不等同于弛豫后的平衡界面距离。

---

## Gap B/C (Å)

默认：

```text
2.0 Å
```

仅在 `Vertical A/B/C` 模式中使用，控制 B 与 C 之间的初始界面距离。

在二层和横向模式中该参数不会影响最终结构。

---

## Vacuum (Å)

默认：

```text
15.0 Å
```

控制沿 slab 表面法向加入的总真空层厚度。

程序会把真空分配在结构上下两侧，使 slab 大致位于周期盒子的中间。

### 普通建议

对于常规表面或异质结模型：

```text
12–20 Å
```

通常是合理的起始范围。

最终所需真空厚度仍应根据具体计算方法、偶极矩、功函数、电荷状态以及层间相互作用做收敛测试。

---

## Thickness in layers

默认：

```text
ON
```

决定 **Thickness** 使用“层数”还是 Å。

- 开启：按层数；
- 关闭：按 Å。

如果希望不同材料具有明确的实际几何厚度，可关闭该选项并直接使用 Å。

---

# 5. ZSL 晶格匹配参数

ZSL（Zur–McGill）算法通过寻找两个二维表面晶格的整数超胞组合，获得近似共格的界面周期晶格。

四个最主要的搜索限制是：

- Length tol
- Angle tol
- Max area
- Area ratio tol

这些参数越严格，得到的候选越少；参数越宽松，越容易找到匹配，但最终结构可能具有更大的应变或更大的超胞。

---

## Length tol

程序默认：

```text
0.05
```

即约：

```text
5%
```

这是 ZSL 判断两组候选二维晶格向量长度是否足够接近的相对容差。

可近似理解为检查：

```text
|L2 / L1 - 1|
```

是否小于给定阈值。

例如：

```text
Length tol = 0.03
```

表示大约允许 `3%` 的长度差。

### 参数影响

减小：

```text
0.05 → 0.03 → 0.02
```

会：

- 降低允许的面内应变；
- 提高几何匹配质量；
- 减少候选；
- 可能找不到可接受的共格结构；
- 可能迫使程序寻找更大的超胞。

增大：

```text
0.05 → 0.08
```

会：

- 增加匹配数量；
- 更容易得到小面积超胞；
- 但可能引入较大的面内应变。

### 普通建议

常规初筛：

```text
0.03–0.05
```

如果没有匹配，可逐步放宽到：

```text
0.06–0.08
```

不建议在没有检查最终应变的情况下盲目使用很大的容差。

---

## Angle tol (relative)

程序默认：

```text
0.05
```

这是**相对夹角容差**，不是：

- `0.05°`
- `0.05 rad`

可近似理解为比较两个二维超胞基矢夹角：

```text
|θ2 / θ1 - 1|
```

是否在容差内。

例如对于约 `60°` 的二维夹角：

```text
Angle tol = 0.05
```

大致对应允许数度量级的角度差，但实际判据是相对值，而不是固定的“几度”。

### 参数影响

较小：

```text
0.01–0.02
```

会严格限制剪切/夹角失配。

较大：

```text
0.05–0.10
```

会增加候选，但可能允许较明显的晶胞剪切差异。

### 普通建议

一般可以从：

```text
0.02–0.05
```

开始。

如果界面几何对角度变化非常敏感，应使用更严格的值。

---

## Max area (Å²)

程序默认：

```text
400.0 Å²
```

限制 ZSL 搜索的二维超晶格最大面积。

这是控制计算规模和搜索时间的重要参数。

较小的 Max area：

- 搜索更快；
- 原子数通常更少；
- 但可能找不到低失配匹配。

较大的 Max area：

- 允许更大的整数超胞；
- 通常更容易获得低失配；
- 但原子数和后续 DFT 成本可能显著增加；
- 三层匹配的搜索量也会快速增加。

### 普通建议

初筛：

```text
200–400 Å²
```

如果没有合适匹配：

```text
500–800 Å²
```

逐步增加通常比一次设置得非常大更合适。

---

## Area ratio tol

程序默认：

```text
0.09
```

即约：

```text
9%
```

控制 ZSL 在生成 film/substrate 超晶格组合时，对两个候选超晶格面积比例的允许差异。

ZSL 会先寻找面积接近的整数超胞组合，再进一步检查长度和夹角。

较小的值：

- 面积匹配更严格；
- 候选减少；
- 搜索更有选择性。

较大的值：

- 允许更多整数超胞组合进入后续检查；
- 更容易找到匹配；
- 搜索量可能增加。

### 普通建议

默认：

```text
0.09
```

适合作为普通起点。

如果匹配数量非常多，可以适当减小；如果完全找不到候选，也可以和 Max area、Length tol 一起逐步调整。

---

## Bidirectional ZSL

默认：

```text
ON
```

该选项控制 ZSL 的**匹配接受判据是否进行双向检查**。

由于相对长度差：

```text
L2 / L1 - 1
```

与反向：

```text
L1 / L2 - 1
```

在数学上并不完全相同，在容差边界附近可能出现一个方向通过、另一个方向不通过的情况。

开启 Bidirectional ZSL 后，匹配判据对 film/substrate 的标签顺序更不敏感。

### 重要

该选项**不会改变实际参考晶格**。

它不意味着：

- 两种材料各承担一半应变；
- 自动对两个结构做平均应变；
- 自动交换最终 film/substrate。

实际构建中：

- `Vertical A/B`：A 是 reference，B 被匹配到 A；
- `Vertical A/B/C`：先以 A 为第一参考建立 A/B，再让 C 与 A/B 共格结构匹配；
- `Lateral A|B`：由 **Reference side** 决定哪一侧为 reference。

### 普通建议

保持：

```text
ON
```

通常更稳健。

如果研究中希望严格按照单一指定方向解释失配容差，可以关闭。

---

# 6. 表面终止参数

## Termination ftol (Å)

程序默认：

```text
0.25 Å
```

该参数用于 slab termination 枚举时判断沿表面法向高度非常接近的原子是否属于同一个 terminating atomic plane。

较小的值会更严格地区分不同高度：

- 可能产生更多 termination；
- 不容易把轻微不同的表面层合并；
- 但可能把轻微 buckling 也当成不同层。

较大的值：

- 会合并更多高度接近的原子；
- termination 数量减少；
- 搜索更快；
- 但可能减少可枚举的表面终止方式。

### 普通建议

默认：

```text
0.25 Å
```

适合作为兼顾稳定性和搜索量的通用值。

对于高度规整的晶体，可尝试：

```text
0.10–0.20 Å
```

对于存在轻微结构畸变或 buckling 的结构：

```text
0.20–0.30 Å
```

通常更宽容。

如果 termination 本身是研究重点，建议检查不同 ftol 下枚举出的 slab 数量是否稳定。

---

## Filter symmetric slabs

默认：

```text
ON
```

控制是否去除由不同切割位置产生、但经过晶体对称性判断后等价的 slab。

开启后：

- 减少重复 termination；
- 降低 ZSL × termination 的组合数量；
- 加快构建。

关闭后：

- 保留更多 slab/termination；
- 适合需要详细检查不同表面终止方式的情况；
- 可能出现物理上等价或高度相似的重复候选。

### 普通建议

普通自动建模：

```text
ON
```

如果研究重点就是 termination、非对称表面或特殊表面化学，可以关闭后进一步人工筛选。

---

# 7. Candidate limit

程序默认：

```text
24
```

Candidate limit 是 Heterostructure Builder 自己的候选保留数量参数，不是 ZSL 算法本身的容差。

程序会：

1. 完成 ZSL match 枚举；
2. 枚举可用 termination；
3. 尝试构建有效的整数二维超胞；
4. 根据实际匹配质量排序；
5. 只保留排名最好的若干候选。

排序主要优先考虑：

1. 最大长度失配；
2. 相对角度失配；
3. RMS 长度失配；
4. 二维界面面积；
5. 必要时再考虑结构规模。

### 对 Vertical A/B

最终只使用排名第一的候选，因此只要完整枚举过程可以正常完成，Candidate limit 大于 1 通常不会改变最终第一名。

它主要影响内部保留的数据量。

### 对 Lateral A|B

与二层模式类似，最终横向结构使用最佳 A/B 匹配。

### 对 Vertical A/B/C

Candidate limit 非常重要。

程序不是只取最好的一个 A/B 再匹配 C，而是保留多个 A/B 候选，再逐一尝试与 C 建立共同超胞。

因此较大的 Candidate limit 可以避免：

```text
A/B 第一名与 C 很难匹配
```

而：

```text
A/B 第五名或第十名反而能形成更好的 A/B/C 公共超胞
```

这种候选被过早丢弃。

### 普通建议

二层：

```text
10–24
```

三层：

```text
24–50
```

复杂体系或希望更充分搜索时可以进一步增加，但会提高计算时间和内存占用。

---

# 8. Lateral A|B options

以下选项仅在：

```text
Lateral A|B
```

模式下启用。

---

## Split axis

可选：

```text
a
b
```

决定左右区域沿哪个二维晶格方向分割。

### Split axis = a

A/B 区域沿 `a` 方向排列：

```text
← a →

A | B
```

界面线大致平行于 `b`。

### Split axis = b

A/B 区域沿 `b` 方向排列，界面线大致平行于 `a`。

该选项会直接改变横向界面的晶向，因此选择时应结合希望研究的边界方向。

---

## Left units

默认：

```text
1
```

指定左侧区域沿 Split axis 占据多少个匹配单元。

---

## Right units

默认：

```text
1
```

指定右侧区域沿 Split axis 占据多少个匹配单元。

例如：

```text
Left units  = 2
Right units = 3
```

会形成宽度比例约为：

```text
A : B = 2 : 3
```

的横向周期结构。

### 参数影响

增加 Left units / Right units 会：

- 增加对应区域宽度；
- 增大超胞长度；
- 增加原子数；
- 增大两个周期性界面之间的距离。

如果需要减弱周期边界下两个横向界面之间的相互作用，应适当增加区域宽度。

---

## Z alignment

默认：

```text
center
```

横向 A|B 中，两侧 slab 的厚度可能不同，因此需要决定沿表面法向如何对齐。

可选：

### center

按 slab 中心对齐。

这是最通用的默认方式。

### bottom

让两侧 slab 的底部大致对齐。

适合希望两种区域共享相近底面高度的模型。

### top

让两侧 slab 的顶部大致对齐。

适合希望两种区域共享相近上表面高度的模型。

Z alignment 只是初始几何对齐方式，并不进行结构弛豫。

---

## Reference side

默认：

```text
left
```

决定横向异质结中哪一侧保留 ZSL 匹配得到的参考面内晶格。

### left

左侧 A 为 reference，右侧 B 被施加必要的面内匹配变形。

### right

右侧 B 为 reference，左侧 A 被施加必要的面内匹配变形。

### 普通建议

如果没有明确的物理基底，可以分别尝试 left 和 right，并比较：

- 实际面内失配；
- 结构原子数；
- 最终应变；
- 后续弛豫能量。

如果一侧代表实验中的基底或更刚性的主体材料，通常可将该侧作为 reference。

---

## Overlap warning (Å)

默认：

```text
0.8 Å
```

横向结构拼接完成后，程序会计算左右区域原子之间的最短周期距离。

如果该距离小于此阈值，程序会在日志和 JSON 报告中给出警告。

### 重要

该参数：

- **不会删除原子**；
- **不会自动移动原子**；
- **不会改变最终结构**。

它只是结构合理性检查阈值。

默认 `0.8 Å` 是一个用于发现明显原子重叠的保守警告值，而不是普适的化学键长判断标准。

生成横向结构后仍应人工检查界面附近的原子距离。

---

# 9. Output 区域

## Directory

指定输出目录。

默认使用程序启动时的当前工作目录。

点击：

```text
Browse
```

可以选择其他目录。

---

## Prefix

默认：

```text
heterostructure
```

用于构造输出文件名。

程序会自动把不适合作为文件名的特殊字符替换为下划线。

例如：

```text
Prefix = test_AB
```

输出：

```text
POSCAR_test_AB
test_AB.cif
test_AB_report.json
```

---

# 10. 输出文件

每次成功构建会生成三个文件。

## POSCAR_<prefix>

VASP POSCAR 格式结构。

例如：

```text
POSCAR_heterostructure
```

可直接作为后续 VASP 建模的基础。

程序输出时会按元素整理 species 顺序，减少不同来源层导致的同种元素重复分组问题。

---

## <prefix>.cif

CIF 格式结构。

适合：

- VESTA；
- Materials Studio；
- OVITO；
- 其他结构查看和转换软件。

---

## <prefix>_report.json

保存本次构建的详细信息。

通常包括：

- 程序版本；
- 构建模式；
- 输入文件；
- Miller 指数；
- slab thickness；
- Gap；
- Vacuum；
- ZSL 参数；
- 匹配失配；
- 相对角度差；
- 实际角度差；
- 二维界面面积；
- termination；
- 超胞变换矩阵；
- reference/strained layer；
- 原子数；
- 最终晶格矩阵；
- 横向结构最短跨界距离；
- 警告信息。

JSON 报告建议与 POSCAR 一起保存，便于以后追溯结构是如何生成的。

---

# 11. 匹配结果中的主要指标

JSON 报告会记录若干用于评价匹配质量的指标。

## max_length_mismatch

两条二维匹配基矢中较大的相对长度失配。

通常：

```text
越小越好
```

这是程序候选排序中最优先考虑的量。

---

## rms_length_mismatch

两条二维基矢长度失配的 RMS 值。

用于反映总体长度匹配程度。

---

## relative_angle_mismatch

二维基矢夹角的相对失配。

注意它不是直接以 degree 表示。

---

## angle_difference_deg

两个二维晶格夹角的实际差值，单位：

```text
degree
```

这个值更适合人工快速理解角度差有多大。

---

## match_area

匹配后的二维共格超胞面积，单位：

```text
Å²
```

在失配接近时，较小面积通常意味着较少原子和较低后续计算成本。

---

# 12. 按钮

## Build structure

开始构建。

构建在独立工作线程中执行，GUI 不需要等待计算完成后才能刷新。

运行日志会显示：

- ZSL match 数量；
- termination 数量；
- 保留候选数量；
- 三层匹配进度；
- 被拒绝的无效候选；
- 原子数；
- 输出路径；
- 警告；
- 错误堆栈。

---

## Open output folder

打开当前设置的输出目录。

---

## Clear log

清空 GUI 日志窗口。

不会删除已经生成的结构文件。

---

# 13. 默认参数

当前程序默认参数如下：

| 参数 | 默认值 |
|---|---:|
| Mode | Vertical A/B |
| Miller A/B/C | 1 0 0 |
| Thickness A/B/C | 2 |
| Thickness in layers | ON |
| Gap A/B | 2.0 Å |
| Gap B/C | 2.0 Å |
| Vacuum | 15.0 Å |
| Length tol | 0.05 |
| Angle tol | 0.05 |
| Max area | 400 Å² |
| Area ratio tol | 0.09 |
| Termination ftol | 0.25 Å |
| Candidate limit | 24 |
| Bidirectional ZSL | ON |
| Filter symmetric slabs | ON |
| Split axis | a |
| Left units | 1 |
| Right units | 1 |
| Z alignment | center |
| Reference side | left |
| Overlap warning | 0.8 Å |
| Prefix | heterostructure |

这些默认值面向普通的初始结构搜索，目标是在：

- 搜索成功率；
- 晶格失配；
- 超胞大小；
- 计算时间；

之间取得相对平衡。

它们不是针对某一种材料设计的固定最佳参数。

---

# 14. 推荐的普通工作流

对于一个新的异质结体系，可以按照下面的顺序使用。

### Step 1：准备体相结构

确认：

- 晶格常数正确；
- 元素和占位正确；
- CIF/POSCAR 没有明显结构问题。

---

### Step 2：确定晶面

为每个材料输入希望暴露的 Miller 面。

晶面的选择通常比单纯调整 ZSL tolerance 更重要。

---

### Step 3：先使用默认 ZSL 参数

推荐从：

```text
Length tol     = 0.05
Angle tol      = 0.05
Max area       = 400 Å²
Area ratio tol = 0.09
```

开始。

---

### Step 4：如果没有匹配

优先尝试：

1. 增大 Max area；
2. 适当放宽 Length tol；
3. 适当放宽 Angle tol；
4. 检查 Miller 指数；
5. 检查输入晶胞是否正确。

例如可尝试：

```text
Max area = 600 Å²
```

然后再逐步调整其他容差。

---

### Step 5：如果匹配结构太大

可以尝试：

- 减小 Max area；
- 适当放宽 Length tol；
- 换一个更容易共格的晶面组合。

这是“低应变”和“小超胞”之间的典型权衡。

---

### Step 6：检查生成结构

至少检查：

- 界面是否位于预期位置；
- 是否有原子重叠；
- 真空方向是否正确；
- slab 厚度是否合理；
- 横向周期边界是否正确；
- 原子数是否可接受；
- termination 是否符合预期。

---

### Step 7：检查 JSON

特别关注：

```text
max_length_mismatch
angle_difference_deg
match_area
terminations
supercell_transforms
warnings
```

---

### Step 8：再进入 DFT

生成结构应视为：

```text
initial geometry
```

而不是最终平衡结构。

正式计算前通常需要根据体系重新设置：

- Selective Dynamics；
- 固定层；
- 磁矩；
- 电荷；
- 真空；
- k 点；
- ENCUT；
- 偶极修正；
- vdW；
- 结构弛豫策略。

这些内容不由本程序自动决定。
