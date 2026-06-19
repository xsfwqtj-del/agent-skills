---
name: comfyui-wildcard-expander
description: 从 Danbooru wiki 扩充 ComfyUI ImpactWildcardProcessor 的 wildcard txt 文件。当用户说：扩充wildcard、充实提示词库、追加标签、wildcard补充、找标签、丰富XX类别、XX.txt 加标签时触发。支持体位/发型/场景/服装/氛围/画质/表情/视角/身材 9 个类别。
---

# ComfyUI Wildcard 扩充器

从 Danbooru 官方 tag_group wiki 提取标签，过滤用户雷区后追加到 wildcard txt 文件。

## 前置信息

### Wildcard 目录

```
F:\ComfyUI\ComfyUI-aki-v3\ComfyUI\custom_nodes\ComfyUI-Impact-Pack\wildcards\一键随机\
```

### 9 个类别 ↔ Danbooru wiki 映射

| 文件 | Danbooru tag_group 页面 |
|------|------------------------|
| 体位.txt | `tag_group:sexual_positions` |
| 发型.txt | `tag_group:hair_styles` |
| 场景.txt | `tag_group:locations` |
| 服装.txt | `tag_group:attire` + `tag_group:sexual_attire` |
| 氛围.txt | `tag_group:lighting` |
| 画质.txt | `tag_group:metatags` + 社区质量标签组合 |
| 表情.txt | `tag_group:face_tags` |
| 视角.txt | `tag_group:image_composition` (含 focus_tags, framing) |
| 身材.txt | `tag_group:breasts_tags` + `tag_group:posture` |

Danbooru wiki 直接可访问：`https://danbooru.donmai.us/wiki_pages/tag_group:XXX`

### 文件格式

每个文件每行是一个完整 wildcard 选项。ImpactWildcardProcessor 随机抽取一行替换到 prompt 中。行格式为逗号分隔的 Danbooru 标签序列，例如：

```
bedroom, bed, messy sheets, pillows, warm light
```

## 工作流

### 第一步：收集要求

向用户确认：
1. 要扩充哪个类别？（体位/发型/场景/服装/氛围/画质/表情/视角/身材）
2. 扩充方向偏好（如：更色气、更日系、某个抖音滤镜风格等）。如果用户没说方向，按"色气/日系/不失一般性"默认处理。
3. 有没有额外的雷区（默认雷区见下方）

### 第二步：从 Danbooru wiki 获取标签

用 WebFetch 抓取对应 tag_group 页面。URL 格式：`https://danbooru.donmai.us/wiki_pages/tag_group:XXX`

提示 WebFetch 用这样的话术：
> 列出此页面所有tag名称。每行一个。全部列出，不要省略。

### 第三步：过滤

按以下规则过滤提取到的标签。这些规则来自用户明确表达的偏好，**必须严格遵守**：

#### 全局雷区（所有类别）
- **西漫/美式漫画风**：删除 `comic`、`toon`、`cartoon`、`western_comics`、`kubrick_stare`、`troll_face`、`awesome_face`、emoji 表情符（`:D`, `:p`, `uwu`, `XD`, `x3`, `;3`, `3:`, `O_O`, `^_^` 等）、`glasgow_smile`、`foodgasm`
- **写实/照片风**：删除 `photo_background`、`photorealistic`、`realistic`、`3D`、`photo_(medium)`
- **西方美术媒介**：删除 `oil_painting`、`watercolor_(medium)`、`sumi-e`（除非用户明确要日系传统）、`ukiyo-e`、`traditional_media`、`impressionism`、`ink_brush`、`line_art`、`sketch`
- **Pony 标签**：删除 `score_9`、`score_8_up`、`score_7_up`、`source_pony`、`source_furry`
- **R18G**：删除 `gore`、`blood`、`violence`、`scar`、`wound`、`death`、`torture`、`guro`、`vomit`、`scat`、`feces`、`urine`

#### 类别专属雷区
- **发型**：删除 `short_hair`、`very_short_hair`、`bald`、`bald_female`、`balding`、`buzz_cut`、`crew_cut`、`mohawk`、`flattop`、`dreadlocks`、`cornrows`（最短只到 `medium_hair` / `bob_cut`）
- **身材**：删除 `fat`、`obese`、`chubby`、`plump`、`emaciated`、`anorexic`
- **表情**：删除 `crying`（眼角的泪水 `tears` 可以，但大哭/哭脸不行）、`cold_stare`（用户已明确删除）、`scared`、`panicking`、`horrified`、`screaming`

#### 色气方向加权
用户目标是产出更色的图。在 Danbooru 标签中：
- 表情优先保留：`seductive_smile`、`bedroom_eyes`、`ahegao`、`torogao`、`parted_lips`、`heavy_breathing`、`blush`、`afterglow`、`fucked_silly`、`lust`、`aroused`、`naughty_face`
- 删除无助于色气的中性表情：`expressionless`、`bored`、`frustrated`、`disappointed`、`lonely`

### 第四步：格式化

Danbooru wiki 返回的是单标签名（如 `cowgirl_position`、`hime_cut`）。需要格式化为完整 wildcard 行。

**标签类（发型/身材/画质）**：把相关标签组合成一行，加入细节渲染词确保出图质量。
- 例：Danbooru 返回值 `hime_cut` → 写为 `long hair, hime cut, blunt bangs, sidelocks, straight hair, elegant`
- 例：Danbooru 返回值 `medium_breasts, narrow_waist` → 写为 `medium breasts, narrow waist, slender, soft curves`

**场景类（体位/场景/氛围/视角/服装）**：将标签扩展为完整场景描述行。
- 例：Danbooru 返回值 `backlighting` → 写为 `backlighting, backlit silhouette, glowing outline, rim light, dramatic`

**画质类**：保持 `masterpiece, best quality` 前缀格式（Illustrious 标准），组合不同的品质标签变体。

不要造不存在的 Danbooru 标签。所有核心标签必须来自 Danbooru wiki。

### 第五步：去重后追加

1. 读取目标文件当前全文
2. 逐行比对，标记与现有行内容高度相似的新行（相似度 >80% 视为重复）
3. 只追加确认不重复的新行
4. 用 Edit 工具追加到文件末尾

追加时用文件的最后一行作为 old_string 锚点。

### 第六步：汇报

报告：
- 从哪个 Danbooru wiki 页面获取
- 提取了多少个标签
- 过滤掉了多少个（按过滤原因分类）
- 去重了多少个
- 最终追加了多少行
- 文件现在总行数
