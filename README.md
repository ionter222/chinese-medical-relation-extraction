# 中文医学实体关系抽取(GPLinker + CMeIE 完整修复版)

> 让 2022 年停更的老项目在现代 Colab 环境上**一键跑通**的现代化改进方案,
> 并提供**医学领域**的完整成功实例。

---

## 一、原项目介绍

本仓库基于以下开源项目改进:

**[taishan1994/pytorch_GlobalPointer_triple_extraction](https://github.com/taishan1994/pytorch_GlobalPointer_triple_extraction)**

- **方法**:苏剑林(苏神)提出的 **GPLinker 联合抽取模型**的 PyTorch 实现
  - 基于 GlobalPointer 统一做**实体识别 + 关系抽取**
  - 一步前向直接输出 `(实体1, 关系, 实体2)` 三元组,无需 NER→RE 两步
  - 参考论文:《GPLinker: 基于GlobalPointer的实体关系联合抽取》(kexue.fm/archives/8888)
- **预训练模型**:`chinese-bert-wwm-ext`(HuggingFace: hfl/chinese-bert-wwm-ext)
- **技术栈**:PyTorch + transformers + BERT 中文预训练

**原项目的问题**:代码写于 2022 年,依赖当时的旧库版本。在 2026 年的
Python 3.12 / transformers 4.5x+ / 新 torch 环境下,**存在多处兼容性问题无法直接运行**,
且项目已停止维护(最后一次更新在 2022 年),大量后来者被这些问题卡住。

---

## 二、本仓库做了什么

### 2.1 修复的 5 个兼容性问题(老项目在新环境的坑)

| # | 问题 | 根因 | 修复方案 |
|---|------|------|---------|
| 1 | `ImportError: cannot import name 'AdamW' from 'transformers'` | transformers 在 4.x 后期将 `AdamW` 移出顶层导出 | 改为 `from torch.optim import AdamW`(与 transformers 版本无关) |
| 2 | `BertTokenizer.from_pretrained('.../vocab.txt')` 报错 | 新 transformers 要求传**目录**而非单个文件 | 改为传 `model_hub/chinese-bert-wwm-ext` 目录 |
| 3 | 首次训练报 `checkpoints/bert/model.pt 不存在` | main.py 默认执行 `test()` 加载历史模型,而 train() 被注释 | 启用 `bertForNer.train()`,跳过首次无模型时的 `test()` |
| 4 | `ModuleNotFoundError: No module named 'tensorboardX'` | 环境缺依赖 | 安装 `tensorboardX` |
| 5 | **CMeIE 数据 subject 丢失 → F1=0** | CMeIE 的 text 是 `主题疾病@正文` 格式,subject 来自 `@` 前缀;粗暴切分会把 subject 丢掉,data_loader 在文本中找不到实体,标签全空 | **保留完整 text 不切分**,确保 subject 命中率 100% |

### 2.2 提供的现代化解决方案

- **固定工作目录**:使用 `/content/gplinker-med` 单一目录,避免嵌套克隆
- **自动修复**:所有修复已内置在 notebook 中,无需手动改代码
- **自动下载数据**:CMeIE 医学数据集从 HuggingFace 直接加载
- **数据质量自检**:生成数据后自动验证 subject 命中率,提前发现数据问题
- **演示文本替换**:将 main.py 里写死的娱乐类演示文本换成医学文本
- **一键训练**:num_tags 自动读取,无需手填

### 2.3 目录结构

```
实体关系抽取/
├── 医学实体关系抽取_CMeIE_完整修复版.ipynb   ← 核心:完整可运行笔记本
└── README.md                                ← 本说明
```

---

## 三、医学领域成功实例(核心笔记本)

**`医学实体关系抽取_CMeIE_完整修复版.ipynb`** 是一个**从头到尾一键运行**的完整方案:

- **数据**:CMeIE 中文医学信息抽取数据集(2.8万句 / 7.5万三元组 / 43种医学关系)
  - 来源:HuggingFace `Aunderline/CMeIE`
  - 关系类型:药物治疗、临床表现、影像学检查、鉴别诊断、病因等
- **模型**:GPLinker + chinese-bert-wwm-ext
- **流程**:环境 → 克隆 → 修复 → 数据 → 模型 → 训练 → 测试(9 个单元格,全自动)

### 实测效果

用医学文本测试,模型成功抽取三元组:

```
输入: 氟西汀是治疗抑郁症的常用药物,禁与单胺氧化酶抑制剂合用
输出: ('抑郁症', '药物治疗', '氟西汀')

输入: 抑郁障碍的核心症状包括情绪低落和兴趣丧失
输出: ('抑郁障碍', '临床表现', '情绪低落')
      ('抑郁障碍', '临床表现', '兴趣丧失')
```

### 使用方法

1. 打开 [Google Colab](https://colab.research.google.com)
2. 上传 `医学实体关系抽取_CMeIE_完整修复版.ipynb`
3. 运行时类型选择 **GPU(T4)**
4. 从头到尾依次运行,全程无需手动修改

---

## 四、扩展到其他领域

本笔记本的流程**通用**——换领域只需修改数据准备单元格:

1. 将你的领域数据整理为同格式:
   ```json
   {"text": "句子", "spo_list": [{"subject": "实体1", "predicate": "关系", "object": "实体2"}]}
   ```
2. 数据生成后自动得到领域关系列表(predicates.json)
3. 模型只能抽取你的数据中定义的关系

> 提示:如果训练数据中某实体不在 text 中出现(如 CMeIE 的 `@` 前缀设计),
> 务必保留完整文本,否则标签会丢失导致 F1=0(见修复 #5)。

---

## 五、开源许可

- 原项目:`MIT License`(Copyright (c) taishan1994)
- 本仓库:保留原项目版权声明的基础上,基于 MIT 许可发布改进内容
- 使用方法参考:苏剑林《GPLinker: 基于GlobalPointer的实体关系联合抽取》
  - https://kexue.fm/archives/8888

## 六、致谢

- [苏剑林](https://kexue.fm)(GPLinker / GlobalPointer 方法原创者)
- [taishan1994](https://github.com/taishan1994)(PyTorch 实现)
- CMeIE 数据集构建团队(郑州大学 NLP 实验室等)

---

**欢迎 star / issue / PR,一起让中文信息抽取的老项目在新环境下重获新生。**
