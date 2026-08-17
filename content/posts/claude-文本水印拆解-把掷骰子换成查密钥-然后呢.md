---
title: "Claude 文本水印拆解：把掷骰子换成查密钥，然后呢？"
date: 2026-08-18T00:00:00+08:00
draft: false
---

如果你把一段 Claude 生成的文字丢进检测器，它可能回你一句“99% 概率由 AI 生成”。可你逐字检查过——没有隐藏字符、没有多余空格、token 数量和语感都和人类写的没两样。水印到底藏在哪里？

答案有点反直觉：它根本不在文本“里”，而在生成文本的那个随机数发生器里。2026 年 8 月 14 日，Anthropic 首次公开技术细节，官方定性是“a version of the SynthID-Text approach”——一种采样级（sampling-time）的密钥化统计水印。本文把它拆开，讲清楚原理、天然弱点，以及它为什么更接近“合规工具”而非“可靠溯源”。

## 一、原理：不藏字，只换“骰子”

大模型逐 token 生成文本。在“低风险选词”处——比如写天气时选 overcast 还是 grey——原本由随机数发生器掷骰子定夺。启用 Claude 水印后，这个随机源被替换成“密钥（key）+ 前文几个词”算出来的伪随机数。Anthropic 的工程师说得直白：模型本身并不知道自己被水印了，用户的 prompt 也无法关闭它，API 层面没有 opt-out 参数。

为什么重要：理解“只换随机源”这四个字，你就同时理解了它的两个属性——一是不降质量（不加隐藏字符、不多余 token、不影响速度与价格），二是它天然可被规避（第三节会讲）。

这条路线可以上溯到 Scott Aaronson 2022 年的提案，成熟于两个里程碑：Kirchenbauer 等人的 KGW（ICML 2023）把词表用“密钥 + 前文窗口”分成绿/红两集，给绿集 logits 加偏置；Google DeepMind 的 SynthID-Text（Nature 2024）改用 tournament sampling，每个解码步给候选 token 赋一层 g 值，层层打擂台，胜者输出。Claude 用的是 SynthID-Text 的一个版本。

用一段最小可运行代码感受一下检测的本质（KGW 简化版，教学用，非官方实现）：

```python
import hashlib, random

VOCAB = ["the", "cat", "dog", "sky", "is", "blue", "runs", "fast", "very", "quietly"]

def green_mask(key, ctx):
    """伪随机地把词表分成绿/红两集，归属由 key + 前文 决定"""
    return {
        t: int(hashlib.sha256(f"{key}|{ctx}|{t}".encode()).hexdigest(), 16) % 2 == 0
        for t in VOCAB
    }

def gen(key, n, watermark, bias=5.0, seed=7):
    """逐 token 生成；含水印时给绿色 token 加权（等价于 logits 加偏置）"""
    rng = random.Random(seed)
    toks = []
    for _ in range(n):
        ctx = " ".join(toks[-3:])
        mask = green_mask(key, ctx)
        weights = [bias if (watermark and mask[t]) else 1.0 for t in VOCAB]
        toks.append(rng.choices(VOCAB, weights=weights)[0])
    return " ".join(toks)

def detect(text, key):
    """统计绿色 token 占比，做单比例 z 检验（H0：绿比例 = 0.5）"""
    tokens = text.split()
    green = sum(
        1 for i, t in enumerate(tokens)
        if green_mask(key, " ".join(tokens[max(0, i - 3):i])).get(t, False)
    )
    T = len(tokens)
    return green / T, (green - 0.5 * T) / (0.25 * T) ** 0.5

key = "demo-secret"
for name, flag in [("含水印生成", True), ("随机生成", False)]:
    p, z = detect(gen(key, 1000, flag), key)
    print(f"{name}: green ratio={p:.3f}, z={z:+.2f}")
```

运行后你会看到两个量级截然不同的 z 值：含水印文本的绿色占比被系统性地顶到 50% 以上，随机文本则稳稳落在 50% 附近。这就是密钥化统计水印的全部魔法——检测时你不需要生成模型，只需要密钥和文本，重算各位置的绿/红归属，再做一次假设检验。

（这里的 z 值、密钥与 green_mask 逻辑属于本段代码；SynthID-Text 的真实实现用的是 tournament sampling 与 per-layer g 值，具体细节 Anthropic 未公开。）

## 二、熵约束：代码和短文本天生“无水印”

信号强度与下一个 token 的分布熵成正比。高熵的散文（创意写作、长文）附着点最多；低熵场景几乎无信号。官方的例子很直白：“Isaac Newton 的著作后必须接 Mathematica”，“2+2=” 后面只能是 4——这些位置没有“低风险选词”可供替换。

对开发者最相关的一条：代码基本无信号。语法必须精确，只有注释这类任意处可能残留一点痕迹，官方称对实际代码的影响可忽略。同理，短消息、评论、commit message 都不够长——KGW 论文里约 25 token 才达到 FPR < 10⁻⁵ 的理论下限，稳健检测需要 100+ token。

为什么重要：别指望水印能帮你分辨一段三行 diff 是不是 AI 写的。它在 Claude Code 产生的 prose——计划摘要、PR 描述——里才可能留下印记，代码本身大概率是“干净”的。这也意味着你的 format-on-save（linter/formatter）每次重排 token，对水印就是一次无意的对抗攻击。

## 三、检测与规避：公开 API 就是免费 oracle

检测公式很简单：密钥 + 文本 → 统计检验。Anthropic 已在 2026 年 8 月 12 日确认会提供一个“你自己就能调用的文本检测 API”，这是欧盟守则“支持第三方检测”义务的直接产物，但定价、限流、访问控制都还没公布。

问题在于：一个公开的检测 API，同时也是一个免费的规避 oracle。TechTimes 算过一笔账——对一篇一千词的文章做一次释义再回查，前沿 API 的价格约 4 美分；反复“释义 → 检测 → 再释义”直到水印消失，成本几乎可忽略。独立研究者 John Wang 说得更直接：用一个没水印的 paraphraser 绕开这种水印“相当容易”，水印的意义是“抬高作弊成本”，而不是防作弊。

量化证据更难看：2026 年 7 月的一份取证评测（arXiv 2607.16010，846 次运行）显示，对保持语义的释义攻击，KGW 与 Unigram 一次释义后 100% 去除水印，SynthID-Text 去除率 98.3%。还有研究者证明，花不到 50 美元的 API 查询就能恢复现行方案的水印规则，进而批量清除乃至伪造（陷害式误报）。理论层面，ICML 2024 的不可能性定理早已给出结论：攻击者握有质量 oracle 与扰动 oracle 时，强水印在理论上就不可能。

为什么重要：把水印当“防作弊锁”是误判，当“抬高作恶成本的合规门槛”才准确。设计依赖检测结果的产品时，请按“可被绕过”做威胁建模。

## 四、治理：合规工具的两难

这套水印是被法规“推”上线的。直接动因是欧盟 AI Act 第 50(2) 条透明度 Code of Practice——2026 年 7 月签署，约 190 个签署方，Anthropic 在列，2026 年 8 月 2 日生效，要求向欧盟市场提供服务的 AI 提供商标记 AI 生成内容。因为官方称暂无“按地区裁剪”的可靠方法，所以全球默认开启，覆盖 API、Claude、Claude Code、Cowork、Tag 以及 AWS/GCP/Microsoft Foundry 上的 Claude 模型，旧模型在 12 月 2 日宽限期前补上。Gemini 自 2024 年起已在用 SynthID-Text；OpenAI 未公布计划，但受同一法律约束。

争议集中在两个点。

其一是证据效度。同一份 Daubert 取证评测发现，KGW/Unigram/SynthID 三种方法均不满足美国 Daubert 五因素中的三项以上；SynthID 在释义过的人类文本上误报率（FPR）达 5.4%，悖论率 18.6%，80% 的自产水印输出落在它自己都无法判定的“不确定区”。换句话说，水印检测目前进不了美国法庭，机构若拿它做处分或雇佣决策，风险自负。

其二是密钥权力结构。检测密钥只掌握在少数机构手里，公众只能通过对方的 API 查询——John Wang 把这称作“particularly undemocratic”。加上几乎所有开源权重模型天然无水印，任何厂商检测工具都标不了它们，于是形成“闭源有水印、开源无从标”的双层溯源格局。

为什么重要：水印满足的是 provider 的 Art 50(2) 义务，不等于企业披露义务（Art 50(1)）自动满足——你拿 Claude 做客服机器人却不披露 AI 身份，有水印依然不合规。合规边界和技术边界是两回事。

## 小结

Claude 文本水印是采样级密钥化统计水印：把“掷骰子”换成“查密钥 + 上文算伪随机数”，观感不变，持密钥者可验证。它的边界要写清楚：只证明“Claude 可能参与过”，不证明作者身份，不证明人类创作，也不区分“写了”与“重度编辑”；轻编辑留存，全词重写去除；短文本、代码、事实性内容天然弱信号；检测 API 一旦公开即成为低成本规避 oracle。

对 AI 开发者，它的正确打开方式不是“防伪锁”，而是“合规标配”：理解它何时有效、何时失效，比相信它能验明正身更重要。

## 参考链接

1. [Anthropic 博客 · How Claude's text watermark works](https://www.anthropic.com/news/claude-text-watermark)（官方）
2. [Claude Help Center · How Claude marks AI-generated content](https://support.claude.com/en/articles/16266773-how-claude-marks-ai-generated-content)（官方）
3. [John Wang · How Claude's watermarking (probably) works](https://johnjwang.com/post/2026/08/12/how-claude-watermarking-probably-works/)
4. [The Verge · Anthropic 水印报道](https://www.theverge.com/ai-artificial-intelligence/980869/anthropic-claude-watermarks-synthid-text-system)
5. [TechCrunch · Claude 水印 FAQ 解读](https://techcrunch.com/2026/08/15/anthropic-shares-more-details-about-how-claudes-new-watermarks-will-work/)
6. [TechCrunch · 用户“工作/课堂被抓”争议](https://techcrunch.com/2026/08/12/some-claude-users-are-mad-that-anthropics-new-watermarks-will-catch-them-cheating-at-their-jobs-classes/)
7. [TechTimes · 4 美分规避 Claude 水印](https://www.techtimes.com/articles/324183/20260812/four-cents-strips-claude-watermark-anthropic-detection-api-confirms-evasion-oracle.htm)
8. [SynthID-Text 开源实现（DeepMind）](https://github.com/google-deepmind/synthid-text)
9. [DeepMind 博客 · Watermarking AI-generated text and video with SynthID](https://deepmind.google/blog/watermarking-ai-generated-text-and-video-with-synthid/)
10. Kirchenbauer et al., “A Watermark for Large Language Models”（ICML 2023）——经第三方转引，原文链接「需人工核实」
11. 取证评测 arXiv 2607.16010（2026-07）——释义去除率与 Daubert 评估，原文未直接抓取「需人工核实」
