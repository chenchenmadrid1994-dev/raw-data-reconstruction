# raw-data-reconstruction

**从已发表论文表格反推原始数据，并导出为 Excel 文件。**

适用场景：拥有一篇含有均值±标准差、计数/比例等汇总统计的论文，需要还原出逐行患者原始数据，用于教学演示、二次分析或统计练习。

---

## 使用方式

将此 SKILL.md 上传至 Claude，然后发送类似指令：

> 阅读文献，根据表1到表4的内容倒推原始数据，导出为 Excel，观察组放 Sheet1，对照组放 Sheet2。

---

## 核心工作流程

### 第一步：读取文献（docx 格式）

```bash
pandoc /mnt/user-data/uploads/paper.docx -t markdown 2>/dev/null
```

提取所有表格中的以下信息：

| 信息类型 | 示例 |
|----------|------|
| 计量资料（均值±SD） | 年龄 71.56±5.28 岁 |
| 计数资料（例/百分比） | 男/女 30/22 |
| 分组人数 | AKI 组 n=52，非 AKI 组 n=68 |
| 各指标单位 | mmol/L、µmol/L、kg/m²、% 等 |

---

### 第二步：检索各指标的合理范围（必须执行）

在生成任何数据前，**必须先检索**每个指标的临床参考范围，不可凭记忆假设。

检索关键词模板：

```
{指标名} 正常参考值范围 临床
{疾病名} 患者 {指标} 典型范围
```

常见指标参考范围（仅供参考，具体以检索结果为准）：

| 指标 | 典型范围 | 小数位 |
|------|----------|--------|
| 年龄 | 按纳入标准（如 ≥60 岁） | **整数** |
| 糖尿病病程 | 1～30 年 | 1 位 |
| BMI | 16～32 kg/m² | 2 位 |
| 空腹血糖（DKA） | 13.9～33.3 mmol/L | 2 位 |
| HbA1c（糖尿病） | 7～18 % | 2 位 |
| 血肌酐 Scr | 60～120 µmol/L | 2 位 |
| BUN | 3.6～20 mmol/L | 2 位 |
| eGFR | 15～90 mL/min/1.73m² | 2 位 |
| ACAG | 14～35 mmol/L | 2 位 |
| KPS 评分 | 以 10 为阶差，60～100 | **整数** |

> **原则**：年龄、KPS 评分等以整数记录的指标必须生成整数；其余指标保留与原文一致的小数位数。

---

### 第三步：生成数据

#### 整数变量（年龄等）

通过大样本池+迭代抽样，同时优化均值和标准差：

```python
def gen_age_int(n, mean, std, lo, hi):
    best, best_score = None, 9999
    for seed_offset in range(50):
        np.random.seed(seed_offset * 7 + 13)
        arr = np.random.normal(mean, std * 1.15, 20000)
        arr = np.clip(arr, lo, hi)
        arr = np.round(arr).astype(int)
        arr = arr[(arr >= lo) & (arr <= hi)]
        if len(arr) < n:
            continue
        for _ in range(300):
            s = list(np.random.choice(arr, n, replace=False))
            score = (abs(np.mean(s) - mean) * 3
                     + abs(np.std(s, ddof=1) - std) * 1.5)
            if score < best_score:
                best_score, best = score, s[:]
    return best
```

> 乘以 1.15 扩展分布宽度，使截断后的样本更易达到目标标准差。

#### 连续变量（BMI、FBG 等）

```python
def gen_cont(n, mean, std, lo, hi, decimals):
    pool = np.random.normal(mean, std, 100000)
    pool = np.round(pool[(pool >= lo) & (pool <= hi)], decimals)
    pool = list(pool[:8000])
    best, best_score = None, 9999
    for _ in range(60000):
        s = random.sample(pool, n)
        score = (abs(np.mean(s) - mean) * 3
                 + abs(np.std(s, ddof=1) - std) * 0.5)
        if score < best_score:
            best_score, best = score, s[:]
    return best
```

#### 计数/分类变量

直接按论文中的频数分配，随机打乱顺序：

```python
# 示例：性别
gender = ['男'] * 30 + ['女'] * 22
random.shuffle(gender)

# 示例：分级不良反应
def assign_grade(n, total, grade3_4):
    result = ['无'] * n
    affected = random.sample(range(n), total)
    for i, idx in enumerate(affected):
        result[idx] = 'III-IV级' if i < grade3_4 else 'I-II级'
    return result
```

---

### 第四步：验证（必须执行）

生成后必须逐指标打印验证，与原文目标值对比：

```python
def verify(label, arr, target_mean, target_std):
    m = np.mean(arr)
    s = np.std(arr, ddof=1)
    status = '✓' if abs(m - target_mean) < 0.05 and abs(s - target_std) < 0.05 else '✗'
    print(f"{status} {label}: 均值={m:.2f}(目标{target_mean}), SD={s:.2f}(目标{target_std})")

# 计数变量验证
print(f"性别 男/女: {g.count('男')}/{g.count('女')} (目标 30/22)")
```

**合格标准**：均值误差 < 0.05，SD 误差 < 0.10。

---

### 第五步：导出 Excel

使用 `openpyxl` 生成带格式的 Excel 文件：

- **Sheet1**：第一组（观察组 / AKI 组等）
- **Sheet2**：第二组（对照组 / 非 AKI 组等）
- **Sheet3**：指标说明（字段名、单位、取值范围、依据）

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side
from openpyxl.utils import get_column_letter

wb = Workbook()
# 写入数据、设置样式……
wb.save('/mnt/user-data/outputs/原始数据.xlsx')
```

样式规范：

| 元素 | 规范 |
|------|------|
| 标题行 | 深蓝色背景（`1F4E79`），白色加粗字体 |
| 列标题行 | 中蓝色背景（`2E75B6`），白色加粗字体 |
| 数据行（交替） | 奇数行白色，偶数行浅蓝色（`D6E4F0`） |
| 字体 | Arial，标题 11px，数据 10px |
| 对齐 | 居中，允许自动换行 |
| 边框 | 细实线（`thin`） |
| 冻结窗格 | 冻结标题行（`A3`） |

---

## 输出文件结构示例

```
原始数据.xlsx
├── Sheet1（观察组，n=32）
│   ├── 编号
│   ├── 年龄（岁）        ← 整数
│   ├── 性别
│   ├── 糖尿病病程（年）  ← 1位小数
│   ├── BMI（kg/m²）      ← 2位小数
│   ├── 高血压病史
│   ├── 近期疗效（CR/PR/SD/PD）
│   ├── 各不良反应等级
│   └── KPS评分（治疗前/后）← 整数，以10为阶差
│
├── Sheet2（对照组，n=40）
│   └── （同上）
│
└── Sheet3（指标说明）
    ├── 字段名
    ├── 单位
    ├── 取值范围
    └── 来源依据
```

---

## 常见问题

**Q：为什么整数年龄的 SD 比目标值偏低？**  
A：整数化会损失精度，导致方差偏小。解决方案是在生成分布时乘以 1.1～1.2 的扩展系数，扩大原始池的离散度，再通过迭代抽样找到最优子集。

**Q：迭代次数太多导致超时怎么办？**  
A：将 `n_iter` 降低至 30000～60000，或缩小 pool 大小至 5000～8000。连续变量通常 60000 次迭代、1 分钟内可完成。

**Q：某指标在文献中未给出单位或范围怎么办？**  
A：先检索该指标的标准参考范围，再结合研究人群特征（疾病状态、年龄段）合理收窄边界，并在 Sheet3 注明假设依据。

**Q：表格中存在"总有效率 = (CR+PR)/n"这类派生指标怎么处理？**  
A：直接按频数分配基础类别（CR/PR/SD/PD），派生字段由代码计算，不单独生成，确保逻辑一致。

---

## 依赖

```
pandas
numpy
openpyxl
pandoc（命令行工具，用于读取 docx）
```

---

## License

MIT
