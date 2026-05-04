# NX12 FBM 加工规则数据库（可直接落地）

> 目标：让 NX12 CAM 在 **Feature Based Machining (FBM)** 下，实现“识别特征→自动选刀→自动生成工序→自动参数化”。

---

## 1. 推荐数据库结构

建议先用 **Excel/CSV** 维护，再导入 Teamcenter Manufacturing Resource Library（MRL）或映射到 CAM 模板。

核心由 4 张表组成：

1. `feature_rule`：特征规则主表（核心）
2. `tool_lib_map`：刀具映射表
3. `cut_param_lib`：切削参数库
4. `op_template_map`：工序模板映射表

---

## 2. 表1：feature_rule（特征规则主表）

每一行代表一条“命中条件 + 执行策略”。

| 字段 | 类型 | 示例 | 说明 |
|---|---|---|---|
| rule_id | string | FR-POCKET-001 | 规则唯一ID |
| enabled | bool | true | 是否启用 |
| priority | int | 100 | 优先级（数字越大越先） |
| feature_type | enum | POCKET / HOLE / SLOT / FACE / BOSS | NX识别特征类型 |
| subtype | enum | CLOSED_POCKET / THROUGH_HOLE | 子类型 |
| material | string | P20 / 45# / AL6061 | 材料 |
| machine_group | string | VMC-3AXIS-10000RPM | 机床组 |
| size_min | float | 10 | 特征最小尺寸（mm） |
| size_max | float | 80 | 特征最大尺寸（mm） |
| depth_min | float | 0 | 最小深度 |
| depth_max | float | 30 | 最大深度 |
| tol_grade | enum | ROUGH / SEMI / FINISH | 精度级别 |
| surface_ra_max | float | 3.2 | 粗糙度要求 |
| op_chain | string | OPR_FACING>OPR_CAVITY_MILL>OPR_FLOOR_FINISH | 工序链 |
| tool_strategy | string | ENDMILL_FLAT_AUTO | 刀具策略标签 |
| param_strategy | string | P20_ROUGH_STD | 参数策略标签 |
| avoid_collision_level | enum | LOW/MID/HIGH | 避让等级 |
| coolant_mode | enum | OFF/FLOOD/MQL | 冷却策略 |
| remark | string | 模腔开粗通用规则 | 备注 |

---

## 3. 表2：tool_lib_map（刀具映射表）

用于把“策略标签”映射到具体刀具筛选条件。

| 字段 | 类型 | 示例 | 说明 |
|---|---|---|---|
| tool_strategy | string | ENDMILL_FLAT_AUTO | 与 `feature_rule` 对应 |
| tool_type | enum | FLAT_END_MILL / BALL_END_MILL / DRILL | 刀具类型 |
| dia_formula | string | min(0.8*feature_width, 16) | 推荐刀径公式 |
| flute_len_min_formula | string | depth+2 | 刃长要求 |
| holder_type | string | HSK63A | 刀柄要求 |
| overhang_max | float | 70 | 最大伸出 |
| brand_prefer | string | Sandvik,OSG | 可选品牌优先 |
| alt_tool_strategy | string | ENDMILL_FLAT_SMALL | 候补策略 |

---

## 4. 表3：cut_param_lib（切削参数库）

把材料 + 工艺阶段 + 刀具类型映射为切削参数。

| 字段 | 类型 | 示例 |
|---|---|---|
| param_strategy | string | P20_ROUGH_STD |
| material | string | P20 |
| stage | enum | ROUGH/SEMI/FINISH |
| tool_type | enum | FLAT_END_MILL |
| vc_m_per_min | float | 140 |
| fz_mm_per_tooth | float | 0.06 |
| ap_mm | float | 1.5 |
| ae_ratio | float | 0.35 |
| rpm_cap | int | 9000 |
| feed_cap | int | 3200 |
| step_over_finish | float | 0.2 |
| stock_leave_wall | float | 0.2 |
| stock_leave_floor | float | 0.15 |

---

## 5. 表4：op_template_map（工序模板映射）

映射到 NX CAM 已配置好的 Operation 子类型模板。

| 字段 | 类型 | 示例 | 说明 |
|---|---|---|---|
| op_code | string | OPR_CAVITY_MILL | 工序代号 |
| nx_op_type | string | Cavity_Mill | NX Operation 类型 |
| template_name | string | TPL_CAVITY_ROUGH_V1 | 模板名 |
| geometry_method | string | MCS_AUTO | 几何指派策略 |
| drive_method | string | FEATURE_BOUNDARY | 驱动边界策略 |
| default_cut_pattern | string | FOLLOW_PART | 默认刀路 |

---

## 6. 可直接复制的 CSV 示例

### 6.1 feature_rule.csv

```csv
rule_id,enabled,priority,feature_type,subtype,material,machine_group,size_min,size_max,depth_min,depth_max,tol_grade,surface_ra_max,op_chain,tool_strategy,param_strategy,avoid_collision_level,coolant_mode,remark
FR-POCKET-001,true,100,POCKET,CLOSED_POCKET,P20,VMC-3AXIS-10000RPM,10,80,0,30,ROUGH,6.3,OPR_FACING>OPR_CAVITY_MILL,ENDMILL_FLAT_AUTO,P20_ROUGH_STD,MID,FLOOD,模腔开粗
FR-POCKET-002,true,90,POCKET,CLOSED_POCKET,P20,VMC-3AXIS-10000RPM,10,80,0,30,FINISH,1.6,OPR_REST_MILL>OPR_FLOOR_FINISH>OPR_WALL_FINISH,ENDMILL_BALL_AUTO,P20_FINISH_STD,HIGH,FLOOD,模腔精加工
FR-HOLE-001,true,110,HOLE,THROUGH_HOLE,45#,VMC-3AXIS-10000RPM,3,20,5,60,SEMI,3.2,OPR_CENTER_DRILL>OPR_DRILL>OPR_CHAMFER,DRILL_STD,STEEL_DRILL_STD,LOW,FLOOD,通孔标准钻削
```

### 6.2 tool_lib_map.csv

```csv
tool_strategy,tool_type,dia_formula,flute_len_min_formula,holder_type,overhang_max,brand_prefer,alt_tool_strategy
ENDMILL_FLAT_AUTO,FLAT_END_MILL,"min(0.8*feature_width,16)","depth+2",HSK63A,70,"Sandvik,OSG",ENDMILL_FLAT_SMALL
ENDMILL_BALL_AUTO,BALL_END_MILL,"min(0.25*feature_radius,12)","depth+3",HSK63A,75,"Mitsubishi,OSG",ENDMILL_BALL_SMALL
DRILL_STD,DRILL,"feature_diameter","depth+5",HSK63A,90,"Guhring,OSG",DRILL_PECK
```

### 6.3 cut_param_lib.csv

```csv
param_strategy,material,stage,tool_type,vc_m_per_min,fz_mm_per_tooth,ap_mm,ae_ratio,rpm_cap,feed_cap,step_over_finish,stock_leave_wall,stock_leave_floor
P20_ROUGH_STD,P20,ROUGH,FLAT_END_MILL,140,0.06,1.5,0.35,9000,3200,0.00,0.20,0.15
P20_FINISH_STD,P20,FINISH,BALL_END_MILL,180,0.03,0.4,0.12,10000,2500,0.20,0.00,0.00
STEEL_DRILL_STD,45#,SEMI,DRILL,90,0.00,0.0,0.00,7000,1200,0.00,0.00,0.00
GENERIC_ROUGH_SAFE,GENERIC,ROUGH,FLAT_END_MILL,100,0.04,0.8,0.20,6000,1800,0.00,0.30,0.20
```

### 6.4 op_template_map.csv

```csv
op_code,nx_op_type,template_name,geometry_method,drive_method,default_cut_pattern
OPR_FACING,Face_Milling,TPL_FACE_STD_V1,MCS_AUTO,PART_TOP,ONE_WAY
OPR_CAVITY_MILL,Cavity_Mill,TPL_CAVITY_ROUGH_V1,MCS_AUTO,FEATURE_BOUNDARY,FOLLOW_PART
OPR_REST_MILL,Rest_Milling,TPL_REST_STD_V1,MCS_AUTO,IPW_REST,CONSTANT_Z
OPR_FLOOR_FINISH,Floor_Finish,TPL_FLOOR_FIN_V1,MCS_AUTO,FEATURE_FLOOR,SPIRAL
OPR_WALL_FINISH,Wall_Finish,TPL_WALL_FIN_V1,MCS_AUTO,FEATURE_WALL,ZIG
OPR_CENTER_DRILL,Spot_Drilling,TPL_SPOT_STD_V1,HOLE_AXIS,HOLE_POS,POINT_TO_POINT
OPR_DRILL,Drilling,TPL_DRILL_STD_V1,HOLE_AXIS,HOLE_POS,POINT_TO_POINT
OPR_CHAMFER,Chamfer_Milling,TPL_CHAMFER_STD_V1,HOLE_EDGE,HOLE_EDGE,CONTOUR
```

---

## 7. NX12 落地步骤（建议）

1. **先标准化刀具库**：刀具命名、柄长、刃长、材料分组一致。  
2. **固化 Operation Template**：先手工调好 10~20 个模板并冻结参数。  
3. **导入规则表并验证命中**：从 POCKET、HOLE 两类特征先跑通。  
4. **增加优先级与兜底规则**：避免无规则命中导致断流。  
5. **统计命中率 KPI**：目标 >85% 特征自动出工序。  

---

## 8. 推荐的“兜底规则”

- `feature_type=POCKET, size_max=999, depth_max=999, priority=10`
- `tool_strategy=ENDMILL_FLAT_AUTO`
- `param_strategy=GENERIC_ROUGH_SAFE`（需在 `cut_param_lib.csv` 中存在同名策略）
- `op_chain=OPR_CAVITY_MILL`

这样可确保未知特征也能自动生成基础刀路，再由程序员二次优化。

---

## 9. 你可以直接这样开始（最小可用版本）

- 先用上面 4 个 CSV 建一个 `fbm_rules_v1` 文件夹。  
- 先只覆盖 3 类特征：`POCKET`、`HOLE`、`FACE`。  
- 每周复盘一次“自动命中失败”的特征并补规则。  

如果你愿意，我下一步可以继续给你：
1) 一版 **“按你机床和材料定制”的规则初稿**；
2) 一版 **NX12 FBM 命名规范（刀具/模板/规则ID）**；
3) 一版 **可导入 Teamcenter MRL 的字段映射表**。
