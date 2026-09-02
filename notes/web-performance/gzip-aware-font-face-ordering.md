---
title: "@font-face 排序如何把 gzip 体积从 146 KB 降到 26 KB"
date: 2026-09-02
tags:
  - web-performance
  - css
  - web-fonts
  - gzip
  - compression
source: conversation
verified: 2026-09-02
tested_with:
  - "Node.js v22.22.3"
  - "zlib gzip level 9"
  - "Brotli quality 11"
---

# @font-face 排序如何把 gzip 体积从 146 KB 降到 26 KB

一个接近 0.5 MB 的字体样式表里，41 组 `unicode-range` 在 7 个字重中完整重复。直觉上，gzip 应该很擅长处理这种内容；但实测结果却是 146 KB，远大于 Brotli 的约 15 KB。

问题不在于重复不够多，而在于重复出现得太远。

只把 `@font-face` 从“字重优先”改成“切片优先”，gzip 就能从约 146 KB 降到 26 KB。这个案例揭示了一条容易忽略的规则：

> 对基于滑动窗口的压缩算法，重复内容的距离和重复次数同样重要。

## 为什么重复七次，gzip 仍然压不下来？

gzip 使用 DEFLATE。DEFLATE 会寻找前面出现过的相同字节序列，再用“长度 + 回退距离”引用它们，而不是重新保存完整内容。

它的限制是只能引用此前 32 KiB 范围内的内容。根据 [RFC 1951](https://www.rfc-editor.org/rfc/rfc1951.html#section-3.2.5)，DEFLATE 的最大 backward distance 是 32,768 字节。超过这个距离后，即使字符串完全相同，压缩器也无法直接引用。

字体 CSS 常按字重生成：

```js
for (const weight of weights) {
  for (const slice of slices) {
    emitFontFace(weight, slice);
  }
}
```

输出结构类似：

```text
字重 200：切片 001、002、003 ... 041
字重 300：切片 001、002、003 ... 041
字重 400：切片 001、002、003 ... 041
```

虽然不同字重的切片 001 使用相同的 `unicode-range`，但一整个字重的声明已经超过 32 KiB。等压缩器再次遇到切片 001 时，上一份内容早已离开滑动窗口。

在一个包含 7 个字重、41 个切片的匿名化样本中，相同 `unicode-range` 的相邻距离为：

```text
最小：69,648 字节
中位数：69,975 字节
最大：70,058 字节
```

246 个重复间距中，没有一个落在 DEFLATE 的 32 KiB 窗口内。

Huffman 编码仍然能压缩反复出现的 `U+`、十六进制数字和标点，但它不能像 LZ77 那样直接复制整段范围。因此，文件虽然重复，gzip 却不得不反复编码每组长字符串。

## 改变循环顺序，为什么效果会这么大？

把生成顺序改成“切片优先”即可：

```js
for (const slice of slices) {
  for (const weight of weights) {
    emitFontFace(weight, slice);
  }
}
```

新的结构是：

```text
切片 001：字重 200、300、400、500、600、700、800
切片 002：字重 200、300、400、500、600、700、800
```

同一切片的七条规则现在紧挨在一起：

```css
@font-face {
  font-family: 'JapaneseBrandFont';
  font-weight: 200;
  src: url('/fonts/japanese/weight-200-slice-001.woff2') format('woff2');
  unicode-range: U+25A0-25A1, U+3000-30FF;
}

@font-face {
  font-family: 'JapaneseBrandFont';
  font-weight: 300;
  src: url('/fonts/japanese/weight-300-slice-001.woff2') format('woff2');
  unicode-range: U+25A0-25A1, U+3000-30FF;
}
```

以上名称、路径和 Unicode 内容均为脱敏示例。

压缩器第一次遇到这组 `unicode-range` 时仍需正常编码；后续六次则可以引用刚刚出现的字节。重复的 `font-family`、`font-style` 和 `font-display` 也进入了有效窗口。

单次确定性字节测量结果如下：

| 排列方式 | 解压体积 | gzip level 9 | Brotli quality 11 |
| --- | ---: | ---: | ---: |
| 字重优先 | 约 490 KB | 145.9 KB | 14.9 KB |
| 切片优先 | 约 490 KB | 26.2 KB | 15.3 KB |

测试使用 Node.js v22.22.3 内置 zlib。这里只测量生成字节数，没有测量网络耗时、CPU 时间或 Web Vitals，因此不能把 120 KB 的差值直接换算成页面加速时间。

结果说明：源码几乎没有变小，gzip 却减少约 82%。Brotli 没有受益，因为它本来就能识别更远距离的重复，排序后甚至有几百字节波动。

## 重排会不会改变字体匹配结果？

可能，所以排序必须保留同一字体面内部的相对顺序。

CSS Fonts 规范规定：如果一组具有相同字体描述符的 `@font-face` 存在重叠 `unicode-range`，后定义的规则会优先参与字符匹配。[CSS Fonts Module Level 4](https://drafts.csswg.org/css-fonts/#unicode-range-desc)

安全的重排应满足：

```text
原始：
200: 001 → 002 → 003 → ... → 041
300: 001 → 002 → 003 → ... → 041

重排后：
200: 001 → 002 → 003 → ... → 041
300: 001 → 002 → 003 → ... → 041
```

不同字重可以相互交错，但下面这组描述符确定的规则序列不能改变：

```text
font-family + font-style + font-weight + font-stretch
```

因此排序键可以是：

```js
faces.sort((a, b) => {
  return a.slice - b.slice || a.weight - b.weight || a.originalIndex - b.originalIndex;
});
```

这段代码只是排序规则示意。生产实现还需要处理解析失败、非目标字体、注释和未来新增的描述符。

最重要的验证不是“新文件仍有相同数量的规则”，而是同时证明：

1. 每条规则归一化后的描述符和 URL 集合完全一致。
2. 同一字体面内部的切片顺序完全一致。
3. 每个切片仍具有预期的全部字重。
4. 输出能被真正的 CSS 解析器读取。
5. gzip 体积落在预设阈值内。

只比较规则数量无法发现 URL 串位、字重丢失或重叠范围优先级变化。

## 为什么不使用 CSS 变量消除重复？

因为 `@font-face` 中的是 descriptors，不是应用在元素上的普通属性。规范明确指出，这些描述符只在各自的 `@font-face` 内生效，不属于任何文档元素，也不存在对子元素的继承。

因此，不应把自定义属性当作跨 `@font-face` 复用 `unicode-range` 或 URL 片段的可移植方案。

Sass、Less、PostCSS 或 JavaScript 生成器可以让源数据只保存一份范围，但浏览器最终仍需要看到完整、合法的字体声明。它们解决的是维护重复，不会自动减少运行时规则。

比较稳妥的数据结构是让每个切片只保存一次范围：

```js
{
  id: '001',
  unicodeRange: 'U+25A0-25A1, U+3000-30FF',
  files: {
    200: '/fonts/japanese/weight-200-slice-001.woff2',
    300: '/fonts/japanese/weight-300-slice-001.woff2',
  },
}
```

生成器再按照“切片 → 字重”输出标准 CSS。这样既消除源数据重复，也能稳定获得适合 gzip 的排列。

如果无法控制原始生成器，应使用 PostCSS 等 CSS 解析器移动完整 AST 节点，而不是用正则重写规则。移动完整节点可以保留未来新增的 `size-adjust`、`font-feature-settings` 等描述符。

## 什么时候值得做，什么时候不值得？

这是一个传输编码优化，不是 CSS 解析优化。

浏览器解压后仍然得到约 0.5 MB CSS，并处理相同数量的 `@font-face`。如果瓶颈是 CSS 解析、字体匹配或主线程工作，重排不会解决问题；此时需要真正减少首屏字体声明数量。

它对 Brotli 客户端也几乎没有收益。以 CloudFront 为例，当客户端同时支持 `br` 和 `gzip` 时，CloudFront 会优先使用 Brotli。[AWS CloudFront 文档](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html)

因此，决定是否实施前应先查看真实的 `Content-Encoding` 分布：

| 场景 | 重排价值 |
| --- | --- |
| 大量响应使用 gzip | 高 |
| 现代浏览器主要使用 Brotli | 很低 |
| CDN 没有启用文本压缩 | 应先启用压缩 |
| 目标是降低 CSS 解析成本 | 无法解决 |
| 生成器可以免费调整循环顺序 | 值得作为低成本优化 |

如果样式表使用长期 immutable 缓存，内容重排后也应更新资源 URL。即使 CSS 语义相同，复用旧 URL 仍会让不同用户长期获得不同版本。

## 用局部性判断下一次压缩优化

遇到“大量重复但 gzip 仍然很大”的文本资源，可以按这个顺序判断：

1. 确认线上实际使用 gzip、Brotli 还是未压缩响应。
2. 测量相同长字符串之间的字节距离。
3. 比较距离是否超过压缩算法的匹配窗口。
4. 寻找不会改变语义的局部性重排。
5. 分别验证原始体积、gzip、Brotli 和浏览器行为。

不要只数重复字符串，也不要只看未压缩文件大小。

真正可复用的判断是：如果数据本来就具有二维结构，输出循环的嵌套顺序可能决定压缩器能否利用其中一个维度的重复。调整循环之前，先证明另一维度的语义顺序仍然保持不变。
