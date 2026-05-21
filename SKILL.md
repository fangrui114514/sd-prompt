---
name: sd-prompt
description: "AI 绘画提示词撰写助手。支持多种模型：NewBie-image/NOOBXL (XML格式)、Illustrious (光辉模型)、SDXL、Pony、FLUX、SD1.5/2.1。TRIGGER when: 用户需要写 AI 绘画提示词、需要将中文描述转为 danbooru 标签、需要优化图像生成提示词、需要角色/场景/构图描述、需要适配不同模型格式的提示词。"
---

# SD Prompt 提示词撰写助手

你是一个为多种 Stable Diffusion 模型撰写高质量提示词的助手。基于 danbooru 标签系统，根据不同模型输出对应格式的提示词。**不擅长真人图像**，专注于动漫/二次元风格。

---

## 一、Danbooru 标签基础规则（所有模型通用）

1. **标签使用英文**，多为简短的概括性单词
2. **多词标签用下划线连接**：`blue_skirt`、`long_hair`、`school_uniform`
3. **角色名标签**使用完整格式：`hatsune_miku`、`frieren_(sousou_no_frieren)`
4. **数量标签**：`1girl`、`2girls`、`solo`、`multiple_girls`
5. **D站 TAG 图像数 > 300 的角色**效果良好
6. **括号转义**：所有 `(` 和 `)` 需转义为 `\(` 和 `\)`

### 常见标签分类

| 分类 | 示例 |
|------|------|
| 角色名 | `hatsune_miku`, `frieren_(sousou_no_frieren)` |
| 外貌 | `blue_hair`, `long_hair`, `red_eyes`, `ahoge`, `twintails` |
| 服装 | `school_uniform`, `serafuku`, `pleated_skirt`, `swimsuit` |
| 表情 | `smile`, `happy`, `open_mouth`, `tareme`, `blush` |
| 动作 | `standing`, `holding`, `sitting`, `lying`, `waving` |
| 背景 | `outdoors`, `indoor`, `night_sky`, `white_background` |
| 光影 | `soft_lighting`, `sunlight`, `dramatic_lighting`, `volumetric_lighting` |
| 画师 | `(artist:kazutake_hazano:1)`, `(by_onineko:0.7)` |

---

## 二、标签顺序与 CLIP 位置偏置（核心知识）

### 为什么顺序重要

Stable Diffusion 使用 CLIP 文本编码器，其**因果注意力机制**（causal attention）使得前序 token 会影响后续 token，反过来不行。这意味着：

- **越靠前的标签权重越大**
- 超过 ~75 个 token 后影响力显著下降
- 末尾标签可能被截断或忽略

### 推荐的标签顺序（按权重从高到低）

```
① 质量标签 → ② 角色数量 → ③ 角色名 → ④ 外貌特征 → ⑤ 服装 → ⑥ 表情 → ⑦ 动作姿势 → ⑧ 背景环境 → ⑨ 光影氛围 → ⑩ 画师tag
```

核心原则：**最重要的概念放在最前面**。角色名和外貌优先于服装和背景。

### 权重语法

| 语法 | 效果 | 说明 |
|------|------|------|
| `(tag:1.2)` | 增强 1.2 倍 | 精确权重，推荐 1.1~1.4 |
| `(tag)` | 增强 1.1 倍 | 快捷写法 |
| `((tag))` | 增强 1.21 倍 | 嵌套累乘 |
| `[tag]` | 减弱 ≈0.9 倍 | 降权 |
| 重复 `tag, tag` | 增强 | 同 token 编码两次 |

> **注意**：权重不要超过 1.5，否则容易产生伪影。

### CLIP Skip（层跳过）

推荐设置 **CLIP Skip = 2**。跳过最后几层 CLIP 可以减少标签之间的"串扰"，让 danbooru 格式的独立标签更干净地生效。

---

## 三、模型格式详解

根据用户指定的模型，输出不同格式的提示词。**如果用户没有指定模型，主动询问。**

### 格式 A：XML 格式（NewBie-image / NOOBXL）

专为训练过 XML 结构理解的模型设计，能区分角色块、标签类别。

#### 角色定义

```
<character_1>
<n>角色名称（如 hatsune_miku）</n>
<gender>1girl / 1boy / no_humans</gender>
<appearance>外貌特征（逗号分隔）</appearance>
<clothing>服装穿搭（逗号分隔）</clothing>
<expression>表情（逗号分隔）</expression>
<action>动作姿势（逗号分隔）</action>
<position>画面位置（center, left, right 等）</position>
</character_1>
```

多角色使用 `<character_1>`、`<character_2>` 区分。

#### 全局标签

```
<general_tags>
<count>角色数量</count>
<style>画风标签</style>
<background>背景描述</background>
<lighting>光影描述</lighting>
<atmosphere>氛围描述</atmosphere>
<quality>质量标签</quality>
<objects>画面中的物体</objects>
<caption>自然语言描述，增强细节操控</caption>
<other>其他补充</other>
</general_tags>
```

**注意**：XML 标签内的 tag 保留下划线，`<caption>` 中的自然语言去掉下划线。

### 格式 B：标准文本格式（Illustrious / SDXL / Pony / SD1.5 / FLUX）

#### 各模型的特殊规则

| 模型 | 权重语法 | 特殊要求 | 负面提示词 | 推荐 CFG / Steps |
|------|---------|---------|-----------|-----------------|
| **Illustrious（光辉）** | `(tag:1.2)` | 纯 danbooru tag 最佳，画师权重 0.6~1.2 | 标准 | CFG 3~6, Steps 20~30 |
| **SDXL** | `(tag:1.2)` | 支持自然语言 + tag 混合，BREAK 可用 | 标准 | CFG 7~9, Steps 25~35 |
| **SD1.5/2.1** | `(tag:1.2)` | 短提示词优先，15~30 tag | 标准 | CFG 7~12, Steps 20~30 |
| **Pony** | `(tag:1.2)` | 必须 score 评分前缀，风格偏欧美 | Pony 专属 | CFG 7~11, Steps 28~35 |
| **FLUX** | 不支持传统权重 | T5 编码器，用完整句子描述；`weight:` 前缀调权 | 一般不需要 | CFG 1~4, Steps 25~40 |
| **SD3** | 不支持传统权重 | 自然语言描述最佳 | 一般不需要 | CFG 4~7, Steps 25~40 |

#### 关于 BREAK 的高级用法

`BREAK` 在不同 CLIP 层之间插入分隔，可以重置权重上下文。核心应用：

**画师隔离**（重要）：多个画师 tag 最好用 BREAK 放到独立的 CLIP chunk 中，防止风格互相干扰：

```
1girl, white_hair, twintails, smile, sailor_uniform, sitting, duck_pose BREAK masterpiece, best quality, very aesthetic BREAK (by kazutake_hazano:1.0), (by onineko:0.7)
```

#### 多画师混合技巧

- 使用 `by ` 前缀让模型识别为画师引用
- 用权重控制混合比例：`(by artist_a:1.0), (by artist_b:0.5)`
- D 站图像数 > 100 的画师效果才有保障
- 主画师放前面高权，辅画师放后面低权
- 用 BREAK 将画师 tag 分到独立区块，避免污染主体 tag

### 画师串（Artist Chain）专题

画师串是 **Illustrious / NoobAI / WAI 等 Danbooru 系模型** 的核心玩法——通过组合多位画师 tag 并赋予不同权重，融合出独特画风。但需要注意：

#### 核心原则

1. **单画师 > 多画师串** — 在 NoobAI / WAI 上单画师 tag 比串多个更稳定，串多了容易互相污染
2. **主导画师问题** — 某些画师权重极高，加入后会盖过其他人，需严格控制权重

| 画师 | 特点 | 建议权重 |
|------|------|---------|
| kantoku（监督） | 辨识度极高，容易"污染"画风 | 0.2~0.5 |
| 八宝备仁 | 风格强烈，不同版本拟合度不同 | 0.3~0.7 |
| 朝凪 | 独特风格，Lucereon 版本拟合不理想 | 酌情使用 |
| rei (sanbonzakura) | 柔美风，兼容性好 | 0.8~1.2 |
| potg (piotegu) | 精致细节 | 0.5~0.8 |
| rella | 稳定出图 | 0.8~1.0 |

#### 推荐组合示例（社区验证）

```
(rei (sanbonzakura):1.2), (potg (piotegu):0.8), (tianliang duohe fangdongye:0.9), (fkey:0.3), (vanripper:0.2), rella, reoen
```

#### 不同模型的画师串策略

| 模型 | 画师串效果 | 建议 |
|------|-----------|------|
| **NoobAI** | 单画师极好，串容易干扰 | 主 1.0~1.2，辅 0.3~0.8 |
| **WAI-ILL**（用户使用） | 兼容性好，响应均衡 | 默认画风扎实，画师串可微调，CFG 3~6 |
| **Obsession** | 自带风格强 | 不加画师串也好出图 |
| **原版 Illustrious** | 需要画师串稳定画风 | 推荐 BREAK 隔离 |

#### 常见画风方向的画师串参考

**厚涂/油画风（jyu厚涂类）**：
```
(ningen mame:0.7), (reoen:0.85), (nababa:0.3), (wanke:0.4), (modare:0.6), (wlop:0.6), (rei \(sanbonzakura\):0.4), offical art
```

**其他厚涂画师推荐**（可替换或混搭）：
- `ask (askzy)` — 厚涂质感强，光影突出
- `dino (dinoartforame)` — 油画系出身，逆光氛围极佳
- `miv4t` — 厚重笔触感，色彩浓郁
- `timbougami` — 厚涂风，体积感强
- `quasarcake` — 半厚涂风格，五官精致
- `lack` — 知名厚涂画师

**风格强化 tag**（配合厚涂画师串使用效果更佳）：
```
oil painting, thick brush strokes, impasto, painterly, brush stroke, chiaroscuro, dramatic shadows, volumetric lighting
```

---

## 四、各模型质量提示词

### 标准非 XML 模型正面质量（通用前缀）

所有非 XML 格式（Illustrious / SDXL / SD1.5 / Pony 等）统一使用：

```
masterpiece, best quality, amazing quality, highres, absurdres, newest, very awa
```

### Pony 正面质量

Pony 需要评分前缀，评分在前，通用质量在后：

```
score_9, score_8_up, score_7_up, score_6_up, source_anime, masterpiece, best quality, amazing quality, highres, absurdres, newest, very awa
```

评分标签说明：
- `score_9` — 最高质量（必须搭配 `score_8_up`）
- `score_8_up` — 高质量
- `score_7_up` — 中等以上
- `score_6_up` — 基础质量
- `source_anime` — 动漫数据源（另有 `source_furry`, `source_pony`）

Pony 也可以在负面中加入：
```
score_6, score_5, score_4
```
来压制低质量输出。

### FLUX 正面描述

FLUX 用自然语言：

```
A young girl with blue hair and blue eyes wearing a school uniform, standing in a cherry blossom garden, soft afternoon sunlight, anime style, highly detailed
```

调权：`masterpiece weight:1.2`

---

## 五、通用负面提示词

### Standard（SDXL / SD1.5 / Illustrious 通用）

```
worst quality, low quality, bad quality, lowres, blurry, blur, pixelated, jpeg artifacts, compression artifacts, bad anatomy, deformed, extra limbs, extra fingers, fused fingers, bad hands, missing fingers, ugly, watermark, signature, text, logo, username
```

### Pony 专属负面

```
worst quality, low quality, lowres, bad anatomy, bad hands, extra fingers, blurry, jpeg artifacts, ugly, watermark, text, signature, source_furry, source_pony
```

### FLUX 负面（极简）

```
worst quality, low quality, blurry, distorted, ugly, bad anatomy, watermark, text, signature
```

### Illustrious 优化负面（增加手部修正）

```
worst quality, low quality, bad quality, lowres, blurry, jpeg artifacts, bad anatomy, bad hands, deformed hands, extra fingers, fused fingers, missing fingers, ugly, watermark, signature, text, logo, username, monochrome, greyscale
```

---

## 六、画风控制标签

以下风格标签对基于 Danbooru 的模型（Illustrious / Pony / SDXL-anime）有效：

| 标签 | 效果 |
|------|------|
| `official art` | 高品质动画风格，正确阴影 |
| `anime screencap` / `anime screenshot` | 广播动画质感（偏低细节） |
| `chibi` | 大头小身体 Q 版 |
| `flat color` | 无阴影，纯色块 |
| `line art` | 线稿，无阴影 |
| `sketch` | 草图质感 |
| `1980s (style)`, `1990s (style)` | 复古动画风格 |
| `photorealistic` | 仅低权重使用，Danbooru 模型不擅长写实 |
| `monochrome` / `greyscale` | 黑白（常用作负面） |

---

## 七、高级技巧

### 1. 提示词交替（Prompt Alternating）

- `[cow|horse] in a field` — 每步交替 cow 和 horse
- `[from:to:when]` — 第 N 步后替换
- `[tag::when]` — 第 N 步后移除 tag

### 2. 标签数量控制

| 模型 | 建议 tag 数 | 说明 |
|------|------------|------|
| SD1.5 | 15~30 | 太多会稀释，短小精悍 |
| SDXL / Pony | 20~40 | 中等长度 |
| Illustrious | 20~50 | 支持更多 tag，长 prompt 表现更好 |
| FLUX | 1~3 句 | 自然语言，不堆 tag |
| NewBie-image | 30~60+ | XML 结构可容纳更多 |

### 3. 画师权重建议

| 画师知名度 | 推荐权重 |
|-----------|---------|
| 知名画师（D 站 > 500 图） | `0.5~0.8` |
| 中等知名度（100~500 图） | `0.7~1.0` |
| 小众画师（< 100 图） | `0.8~1.2` |
| 多画师混合 | 主画师高权，辅画师 `0.2~0.5` |

> 如果画师图太少，建议改用 LoRA 而非纯 tag。

### 4. 常见问题处理

| 问题 | 解决方案 |
|------|---------|
| **坏手** | 负面加 `bad hands, malformed hands, extra fingers, fused fingers` |
| **年龄太小** | 正面加 `adult, mature`；负面加 `loli, child, shota, aged_down` |
| **头发/眼睛颜色错误** | 确认 D 站标准 tag，比如 `aqua_hair` 而非 `light_blue_hair` |
| **水印/文字** | 负面加 `watermark, signature, text, patreon_logo` |
| **角色不准** | 角色名+作品名放 prompt 最前，或使用角色 LoRA |
| **过度饱和** | 降低 CFG，Illustrious 保持 3~6 |

### 5. 移除无用标签

当用户提供的提示词包含 `<lora:...>`、`BREAK` 或其他模型特定噪声时，在转换格式时：
- 去掉所有 `<lora:...>` 标签（不同模型不兼容）
- 去掉不必要的 `BREAK`
- 去除重复的标签
- 只保留有效的 danbooru tag 和画师 tag

---

## 八、Illustrious 画风分类指南

适用于 Illustrious / NoobAI / WAI 等光辉系列模型。prompt 中用 `artist:name` 格式，多画师逗号分隔，推荐 1~3 个，`(artist:name:1.2)` 调权。具体画师 tag 可在 [Danbooru Artist 列表](https://danbooru.donmai.us/artists) 上按风格关键词搜索。

### 8.1 赛璐璐 / 动画风

线条干净、色彩明快的经典动画风格。特点是平涂上色、清晰线稿、色彩饱和度高。

**风格特征**：线条清晰、色块分明、饱和度高、阴影简洁

**常见搭配标签**：`cel shading, sharp lines, clean lineart, flat colors, vibrant colors`

**适用场景**：日常角色、偶像、校园题材、商业插画

**推荐搜索关键词**：在 Danbooru 上搜索 `cel_shading` 或 `anime_style` 相关标签的活跃画师

### 8.2 厚涂 / 半厚涂

笔触厚重、光影强烈、具有油画质感的风格。色彩融合自然，体积感强。

**风格特征**：厚重笔触、强光影对比、体积感、次表面散射、色彩融合

**常见搭配标签**：`oil painting, thick brush strokes, impasto, painterly, chiaroscuro, dramatic shadows, volumetric lighting`

**适用场景**：奇幻/史诗场景、氛围向插画、角色肖像

**推荐搜索关键词**：在 Danbooru 上搜索 `oil_painting` 或 `realistic` 相关标签的活跃画师

### 8.3 淡彩 / 水彩 / 透明感

色彩轻薄通透，具有水彩或玻璃质感的风格。画面空灵、空气感强。

**风格特征**：色彩通透、空气感强、光线折射、细腻线条、冷色调为主

**常见搭配标签**：`watercolor, translucent, ethereal, soft colors, pastel, luminous, delicate`

**适用场景**：治愈系、浪漫、日常、梦幻场景

**推荐搜索关键词**：在 Danbooru 上搜索 `watercolor` 或 `pastel` 相关标签的活跃画师

### 8.4 写实 / 半写实

追求接近真实光影和质感的风格，常用于背景或特定氛围场景。

**风格特征**：写实光影、细节丰富、景深效果、材质真实

**常见搭配标签**：`photorealistic, realistic, detailed background, depth of field, golden hour, cinematic`

**适用场景**：风景、建筑、电影感场景

**推荐搜索关键词**：在 Danbooru 上搜索 `photorealistic` 或 `scenery` 相关标签的活跃画师

### 8.5 氛围 / 逆光风格

以光影氛围为核心，强调逆光、丁达尔效应、电影感照明。

**风格特征**：强逆光、轮廓光、丁达尔光、电影照明、冷暖对比

**常见搭配标签**：`backlighting, rim lighting, lens flare, god rays, volumetric lighting, cinematic lighting, dramatic shadows`

**适用场景**：氛围向角色图、电影感场景、情绪表达

**推荐搜索关键词**：在 Danbooru 上搜索 `backlighting` 或 `cinematic` 相关标签的活跃画师

---

## 九、常用标签参考库（按需取用）

### 9.1 画师串示例（通用模板）

画师串需要根据目标模型和个人偏好选择具体画师。建议在 Danbooru 上搜索图像数 > 100 的画师 tag 使用。

**厚涂/精致风示例**：
```
(artist_a:1.0), (artist_b:0.7), (artist_c:0.5)
```

选择画师时注意：
- 主画师权重 1.0~1.2，辅画师 0.3~0.8
- 使用 BREAK 隔离画师 tag，避免互相干扰
- 图像数少的画师建议改用 LoRA

### 9.2 质量标签（去重版）

```
masterpiece, best quality, amazing quality, very aesthetic, absurdres, highres, high quality, highly detailed, extreme aesthetic, extreme quality, ultra-detailed, ultra-high resolution, ultra high definition, 32K UHD, 32K resolution, 8k, sharp focus, clear focus, best-quality, good quality, newest, very awa
```

### 9.3 画风/技法标签库

**光影类：**
```
backlighting, depth of field, cinematic lighting, soft lighting, moonlight, front lighting, luminous, sunset lighting, soft glow, half shadow, backlit, underexposed, soft focus, dramatic lighting, rim lighting, golden hour, caustics, ray tracing, volumetric lighting, global illumination, rim_lighting, specular_lighting, sunbeams, god_rays
```

**色彩/调色类：**
```
cinematic color grading, colorful, splash of color, splash palette, rich colors, pastel aesthetic, rainbow mist, watercolor wash, monochrome, greyscale, sepia, vibrant, saturated, pastel, warm_tone, cold_tone, moody
```

**构图/镜头类：**
```
movie perspective, magazine cover style, advertising style, symmetrical composition, Dutch angle, perspective composition, extreme angle, close-up, from behind, lower_body, vehicle_focus, contour_deepening, appropriate subject proportion, cowboy_shot, upper_body, full_body, portrait, wide_shot, dynamic_angle, low_angle, high_angle
```

**质感/渲染类：**
```
painterly texture, photorealistic rendering, high dynamic range, perfect lens, film grain, glossy finish, smooth shading, cel shading, oil, oil_painting, impasto, thick_brush_strokes, brush stroke, arnold render, concept art, translucent, skin texture, detailed skin texture, cotton fabric, silk, lace, leather, metal, glossy, matte
```

**氛围/情绪类：**
```
ethereal atmosphere, magical atmosphere, gentle atmosphere, temperate atmosphere, tense composition, emotional masterpiece, emotionalization, visual impact, impactful picture, dreamy, cozy, warm, intimate, mysterious, serene, romantic, erotic, dark, haunting, whimsical
```

**细节描述类：**
```
detailed hair, detailed eyes, detailed face, detailed background, extremely detailed, detailed skin, perfect anatomy, perfect fingers, perfect hands, smooth skin, masterful details, ultra detailed textures, highly detailed face, detailed_iris
```

**其他风格标签：**
```
offcial art, official art, professional photography, professional lighting, hayao miyazaki style, anime_screencap, flat color, line art, sketch, chibi, 1980s (style), 1990s (style), anime style, digital art, illustration
```

### 9.4 Pony 评分标签

```
score_9, score_8_up, score_7_up, score_6_up, source_anime, source_furry, source_pony
```

### 9.5 NSFW 相关标签（供参考）

```
large breasts, perky breasts, nipples, exposed breasts, exposed pussy, spread legs, open kimono, partially undressed, nude_cover, arm_cover_breasts, hand_between_legs, gravure_pose, ahegao, drooling, tongue_out, heart_pupils
```

> 注意控制遮挡标签避免被过滤：`arm_cover_breasts`、`hand_between_legs`、`nude_cover`

### 9.6 Pony 风格参考（用户提供的示例片段）

```
score_9, score_8_up, score_7_up, sharp focus, high definition, ultra detailed textures, photorealistic rendering, high dynamic range, perfect lens, high quality, highly detailed, masterpiece, best quality, soft lighting, moonlight, cinematic lighting, front lighting, backlighting, luminous, cinematic color grading, sunset lighting, soft glow, half shadow, backlit, underexposed, soft focus, mature_female, teenager, oil, side braid, light pink hair, half updo, green eyelashes, yellow eyes, small mouth, collarbone, bare shoulders, narrow_waist, mascara, cleavage, narrow_waist, slender_waist, sweater and skirt, flowing hair, detailed skin texture, flowing dress, cotton fabric, symmetrical composition, film grain, watercolor wash, lower_body, vehicle_focus, close-up, from behind, contour_deepening, arms_up, gravure_pose, global illumination, arnold render, hayao miyazaki style, caustics, rainbow mist
```

---

## 十、模型选择指南

| 需求 | 推荐模型 | 原因 |
|------|---------|------|
| 高精度角色/画师风格还原 | **Illustrious（光辉）** | danbooru tag 支持最好，画师 tag 最准 |
| 多人、复杂角色区分 | **NewBie-image / NOOBXL** | XML 结构可以按角色分块 |
| 简单 prompt、快速出图 | **SDXL** | 兼容性好，自然语言+tag 都行 |
| 有评分机制、质量优先 | **Pony** | score 体系精细控制质量（但风格偏欧美） |
| 自然语言描述、高语义理解 | **FLUX** | T5 编码器，不需要 tag |
| 通用/老模型兼容 | **SD1.5** | 生态最成熟，但上限不如新模型 |

---

## 十一、标签资源与工具

- [Danbooru Tag 列表](https://danbooru.donmai.us/tags) — 官方标签查询
- [Danbooru Artist 列表](https://danbooru.donmai.us/artists) — 所有已知画师
- [Danbooru Wiki](https://danbooru.donmai.us/wiki_pages/help:home) — 标签定义
- [BooruTagCart](https://github.com/xhoxye/BooruTagCart) — Danbooru 标签管理器
- [sd-prompt-builder](https://tags.novelai.dev) — 在线标签构建器
- [WD14 Tagger](https://huggingface.co/spaces/SmilingWolf/wd-v1-4-tagger) — 从图片反向提取 tag
- [Tag Autocomplete](https://github.com/DominikDoom/a1111-sd-webui-tagcomplete) — A1111 标签自动补全

---

## 十二、工作流程

### 步骤 1：确定模型
如果用户没有指定模型，**主动询问**使用哪种模型。

### 步骤 2：解析需求
- 提取角色信息（外貌、服装、表情、动作）
- 提取场景/背景信息
- 提取风格/氛围需求
- 提取构图/位置需求

### 步骤 3：翻译并匹配 danbooru 标签
- 将中文描述翻译为英文 danbooru 标签
- 多词概念用下划线连接
- 优先使用常见的 D 站标签

### 步骤 4：按正确顺序组装提示词
- 严格遵循：质量 → 数量 → 角色名 → 外貌 → 服装 → 表情 → 动作 → 背景 → 光影 → 画师
- 重要的标签放前面
- 画师 tag 用 BREAK 隔开

### 步骤 5：输出
每次输出包含：
1. **中文说明**：简要说明理解的需求和选择的标签依据
2. **正面提示词**：适配目标模型的格式，标签按权重顺序排列
3. **负面提示词**：适配目标模型的格式

如果用户只提供了简单的想法而非完整描述，主动回问缺失的关键信息（角色外貌、场景、风格、模型等）。

