---
title: '企业级 AI 题库引擎架构演进（二）：基于 AST 与 Pandoc 的 Word 高保真解析'
description: '剖析 Word 底层 XML 机制与传统 POI 抽取图片的脱节陷阱，详解如何利用 Pandoc AST 流式遍历在原位注入行内占位符，实现试卷图文时序保真与专业计算公式向 LaTeX 的无损转换。'
publishDate: '2026-08-17 18:00:00'
tags:
  - Pandoc
  - LaTeX
  - Java
category: 企业级AI题库
---

## 题干、配图与公式：专业试卷解析中的“时序对齐”难题

在上篇《企业级 AI 题库引擎架构演进（一）：初代系统的四大硬伤与重构反思》中，我复盘了早期的几个暗坑。其中，最让我抓狂的硬骨头之一就是：**如何保证 Word 试卷解析出的图片与公式，能够严丝合缝地嵌入到对应的题目文字流中？**

在企业技能考核、特种作业安全培训与工程资质认证场景中，一份标准的 Word 题库讲义往往呈现出如下排版：

```
12. 如图所示，在某低压配电系统中，断路器与熔断器的保护配合电路拓扑如下：
    [此处紧跟一张配电箱接线与熔断器接线原理图]
    已知负荷计算电流 I_js = 45A，整定电流满足 I_zd = K_rel * I_js...
    (1) 根据图示分析该接线方式是否存在短路保护死区；
    (2) 若可靠系数 K_rel 取 1.25，计算该线路应选配的熔断器额定电流。
```

当我把这类文档喂给大模型时，理想状态必须是：**题干描述 $\rightarrow$ 配图 URL 占位符 $\rightarrow$ LaTeX 参数公式 $\rightarrow$ 设问，按照视觉阅读顺序自然流淌**。

事实上，初期用常规的 Java POI 库读取，得到的是“图文脱节”的怪异结果。

---

## 传统 Apache POI 模式下的“图文剥离”陷阱

在开发初期，我犯了一个致命错误：把图片和文字作为独立集合提取。

```java
try (XWPFDocument doc = new XWPFDocument(inputStream)) {
    // 1. 提取所有正文段落
    List<XWPFParagraph> paragraphs = doc.getParagraphs();
    
    // 2. 提取文档中的所有图片
    List<XWPFPictureData> pictures = doc.getAllPictures();
}
```

这段代码看似没有问题，但实际跑起来后，我发现它在底层逻辑上切断了图文的所有纽带：
1. `doc.getAllPictures()` 底层仅仅是扫描了 `.docx` 解压包内 `word/media/` 目录下的所有二进制文件，返回一个无序的列表；
2. 图片在哪个段落、属于哪道题、出现在哪一句话后面，元数据信息在此全部丢失；
3. 最终我只能把所有文本拼成一个大字符串发给大模型，大模型看到“*如图所示*”，却根本看不到图，只能凭空臆造。

为了彻底解决这个问题，我意识到必须下潜到 **Word (.docx) 的底层文件规范** 中寻找解法。

---

## Word 底层 XML 的天然有序性

我把 `.docx` 文件直接解压，查看其中的 `word/document.xml`，终于看清了段落、文字、公式与图片在 XML 节点树中的底层真相 —— 它们是**严格按顺序嵌套**的：

```xml
<w:p>
    <!-- 文字块 1 -->
    <w:r>
        <w:t>12. 如图所示，在某低压配电系统中，整定电流满足 </w:t>
    </w:r>

    <!-- 内联数学公式 (OMML) -->
    <m:oMath>
        <m:r><m:t>I_{zd} = K_{rel} \cdot I_{js}</m:t></m:r>
    </m:oMath>

    <!-- 内联图片引用 (Drawing) -->
    <w:r>
        <w:drawing>
            <wp:inline>
                <a:graphic>
                    <a:graphicData>
                        <pic:pic>
                            <pic:blipFill>
                                <!-- 指向 rId5 对应的 media/image1.png -->
                                <a:blip r:embed="rId5"/>
                            </pic:blipFill>
                        </pic:pic>
                    </a:graphicData>
                </a:graphic>
            </wp:inline>
        </w:drawing>
    </w:r>

    <!-- 文字块 2 -->
    <w:r>
        <w:t> 已知负荷计算电流 I_js = 45A...</w:t>
    </w:r>
</w:p>
```

从 XML 结构我发现：
* `<w:p>` 代表段落（Paragraph）；
* `<w:r>` 代表连续属性的文本块（Run）；
* `<m:oMath>` 是微软自研的 OMML 格式数学公式；
* `<w:drawing>` 代表内嵌的绘图或位图对象。

**它们在 XML 树形结构上完全是并列的行内节点**

因此，破局思路非常清晰：**必须放弃粗暴的纯文本截取，采用 AST（抽象语法树）进行流式遍历。当遍历到图片节点时，将图片提取落盘，并在文档流的原位注入 Markdown 图片占位符 `![图片描述](url)`。**

---

## 破局思路：基于 Pandoc AST 的全要素无损提取

在方案选型上，我最初尝试过在 Java 里自己手写 XML DOM 解析，但很快被 MathType 兼容、复杂多层表格嵌套以及各种画图对象劝退——边缘 case 实在太多，手写解析器的维护成本极高。

经过调研与压测对比，我最终选定了业界工业级文档转换标杆 —— **Pandoc**。

### 1. 为什么 Pandoc 能够完美胜任我的诉求？
通过向 Pandoc 传入 `--extract-media` 和 `--mathml` 核心参数：
1. **图片时序保真**：Pandoc 在解析 `<w:drawing>` 时，自动将二进制文件导出到指定目录，并在 Markdown 输出流的原位置输出 `![](temp_media/image1.png)`；
2. **公式标准转化**：Word 里的计算公式（OMML）被自动无损转译为标准的 LaTeX 格式（`$I_{zd} = K_{rel} \cdot I_{js}$`），这是大模型最容易理解的数学语言；
3. **表格与标题层级保留**：Word 表格被转为标准 Markdown 表格，大纲级别转为 `#` 标题。

---

## 媒体资产管理与 URL 动态重写管道

拿到初版 Markdown 后，我面临的下一个工程细节是：Pandoc 导出的图片路径是本地临时路径（例如 `task_dir/media/image1.png`）。在生产环境中，前端和外部服务无法直接访问这个本地临时路径。

为此，我在后端建立了一条**媒体资产转移与 URL 动态重写管道**：

```
[Pandoc 临时导出目录]  ───(文件移动/上传OSS)───►  [/app/static/media/{taskId}/image1.png]
         │                                                      │
         ▼                                                      ▼
[Markdown 内容]       ───(正则路径重写 Replace)───►    [![](/static/media/{taskId}/image1.png)]
```

重写后的最终 Markdown 格式如下：

```markdown
12. 如图所示，在某低压配电系统中，断路器与熔断器的保护配合电路拓扑如下：

![](/static/media/8f3a1b8c-5d2e-4b7f-9c0a-1e2d3f4a5b6c/image1.png)

已知负荷计算电流 $I_{js} = 45\text{A}$，整定电流满足 $I_{zd} = K_{rel} \cdot I_{js}$：

(1) 根据图示分析该接线方式是否存在短路保护死区；
(2) 若可靠系数 $K_{rel}$ 取 1.25，计算该线路应选配的熔断器额定电流。
```

当我把这个 Markdown 输入给大模型时：
* 大模型清晰地理解到：这道题不仅有题干和两个设问，中间还包含一张核心配图 `![](/static/media/.../image1.png)`；
* 大模型输出的题目 JSON 中，题干 `content` 字段就会**完整保留这个图片占位符**；
* 前端页面（如基于 Vue / React 的 Markdown 渲染器）拿到数据后，直接渲染出图片与 KaTeX 公式，图文严丝合缝。

---

## Java 生产级落地：`DocxConverter.java`

在 Java 模块中，我将上述整套机制封装为高内聚的组件 `DocxConverter.java`：

```java
package com.questionbank.processor.service;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.FileUtils;
import org.springframework.stereotype.Component;

import java.io.BufferedReader;
import java.io.File;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.TimeUnit;

@Slf4j
@Component
public class DocxConverter {

    public String convertDocxToMarkdown(Path docxPath, Path outputMdPath, Path taskMediaDir, String webMediaPrefix) throws IOException {
        Files.createDirectories(outputMdPath.getParent());
        Files.createDirectories(taskMediaDir);

        Path tempMediaExtractDir = outputMdPath.getParent();

        // 1. 调用系统 Pandoc CLI
        ProcessBuilder pb = new ProcessBuilder(
                "pandoc",
                docxPath.toAbsolutePath().toString(),
                "-t", "markdown",
                "--extract-media=" + tempMediaExtractDir.toAbsolutePath().toString(),
                "--mathml",
                "-o", outputMdPath.toAbsolutePath().toString()
        );
        pb.redirectErrorStream(true);

        log.info("Executing Pandoc: {}", String.join(" ", pb.command()));

        try {
            Process process = pb.start();
            try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream(), StandardCharsets.UTF_8))) {
                String line;
                while ((line = reader.readLine()) != null) {
                    log.debug("[Pandoc] {}", line);
                }
            }

            boolean finished = process.waitFor(60, TimeUnit.SECONDS);
            if (!finished || process.exitValue() != 0) {
                if (!finished) {
                    process.destroyForcibly();
                }
                throw new RuntimeException("Pandoc conversion failed, exitCode=" + (finished ? process.exitValue() : "TIMEOUT"));
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Pandoc process interrupted", e);
        }

        // 2. 读取生成的 Markdown
        String markdownContent = Files.readString(outputMdPath, StandardCharsets.UTF_8);

        // 3. 移动图片至业务专属静态目录，并执行 URL 路径重写
        File pandocMediaFolder = new File(tempMediaExtractDir.toFile(), "media");
        if (pandocMediaFolder.exists() && pandocMediaFolder.isDirectory()) {
            File[] mediaFiles = pandocMediaFolder.listFiles();
            if (mediaFiles != null) {
                for (File mediaFile : mediaFiles) {
                    if (mediaFile.isFile()) {
                        File destFile = new File(taskMediaDir.toFile(), mediaFile.getName());
                        FileUtils.copyFile(mediaFile, destFile);
                    }
                }
            }
            FileUtils.deleteDirectory(pandocMediaFolder);

            // 替换 Markdown 中的本地临时路径为公开访问 URL
            String pandocMediaPrefix = tempMediaExtractDir.toAbsolutePath().toString().replace("\\", "/") + "/media/";
            markdownContent = markdownContent.replace(pandocMediaPrefix, webMediaPrefix);
            
            Files.writeString(outputMdPath, markdownContent, StandardCharsets.UTF_8);
        }

        return markdownContent;
    }
}
```

---

## 工业级落地演进：四项生产实战考量

当我们将这套 AST 流式解析方案推向真实复杂的企业生产环境（处理海量存量历史试卷、高并发文档上传）时，又遭遇了一系列更为隐蔽的工程挑战。以下是我们在落地过程中总结的四项高阶实战经验：

### 1. 排版规范与浮动图片（`wp:anchor` vs `wp:inline`）治理
* **现象与根因**：Word 底层有两种主要的图片排版结构：
  * `<wp:inline>`（嵌入型）：图片作为行内元素参与正常的文字流排版，其 AST 顺序与阅读顺序 100% 吻合；
  * `<wp:anchor>`（四周型、上下型、衬于文字下方等浮动对象）：在 XML 树中它只是“挂载”在某个段落上的绝对定位锚点。Pandoc 在扁平化遍历 AST 时，浮动图片的时序往往会“飘”到段落末尾或偏离视觉预期位置。
* **治理对策**：
  * **自动转换降级（技术流）**：在调用 Pandoc 前置管道中，通过 XML 预处理将正文 `<w:p>` 下的 `<wp:anchor>` 提取核心 `<a:graphic>` 数据，自动重构改写为 `<wp:inline>` 节点（同时过滤掉页眉页脚、背景水印等非题目资产），实现大部分浮动图片的原位内联化；
  * **空间偏移预警（兜底流）**：若检测到浮动图片的垂直绝对位移（`wp:positionV`）跨度过大（例如教研老师在第 1 题插入图片但鼠标拖拽到了第 3 题旁边），技术强制转 Inline 会导致图片“归位”到锚点段落。针对这类视觉与锚点严重错位的情况，系统主动在导入报告中输出 Warning 提示，便于人工二次复核；
  * **出题规范推行（管理流）**：在企业与教研出题排版规范中，建议推行“所有配图一律采用**嵌入式排版 (Inline Shape)**”，从输入源头保障 100% 的时序确定性。

### 2. 历史包袱处置：老旧 MathType (OLE 对象) 与现代 OMML
* **现象与根因**：现代 Word 默认采用原生 `<m:oMath>`（Office Math），Pandoc 的 `--mathml` 参数能够将其极其丝滑地转为 LaTeX。然而，大量历史老旧试卷（Word 2003~2010 年代）中充斥着 MathType 5.x/6.x 生成的 OLE 嵌入对象（`<w:object>`）。Pandoc 无法将其作为数学公式解析，通常会退化为提取内部的预览图（`.wmf` 或 `.emf`）。
* **工业级兼容对策**：
  * **前置清洗（格式转换）**：在入库管道前置挂载轻量预处理脚本（基于 Word 自动化或 `docx4j`），批量将 MathType 对象升级转译为原生 OMML 结构；
  * **多模态/OCR 兜底（视觉挽救）**：若提取出的媒体资产为 `.wmf`/`.emf` 或尺寸符合行内公式特征，管道自动将其路由至公式 OCR 模型（如 LaTeX-OCR / 轻量多模态大模型），将公式图片反解为标准 LaTeX 文本并替换回原位。

### 3. 容器化部署与 OS 进程治理（Process Safety & 限流背压）
* **风险点**：在微服务容器化（如 Kubernetes Pod）环境中，Java 通过 `ProcessBuilder` 频繁调用宿主机 CLI（`pandoc`）存在显著的资源隐患：
  * **资源争抢**：突发高并发上传可能瞬时耗尽容器的 CPU、内存与 OS 线程句柄；
  * **僵尸进程**：当遇到超大复杂文档解析超时时，若未妥善终止子进程树，易造成进程泄露。
* **架构保障**：
  * **信号量限流（Backpressure）**：在 Java 内存中引入 `Semaphore` 控制同时并发执行 Pandoc 的最大进程数（例如基于容器 CPU 核心数限制为 4~8 并发），超出时排队；
  * **异步化解耦**：对于数十页的整套题库文档，采用 MQ + Worker 异步解析机制，避免长时间阻塞 Web 容器请求线程；
  * **强力超时销毁**：在 `process.waitFor()` 超时后，务必调用 `process.destroyForcibly()` 强制终止外部进程。

### 4. 字符串 Replace vs 基于 Markdown AST 的 URL 安全重写
* **潜在风险**：使用 `markdownContent.replace(prefix, webPrefix)` 是一种简单直接的做法，但在严苛的跨平台生产场景中存在偶发风险：
  * Windows 反斜杠 `\` 与 Linux 斜杠 `/` 的混用；
  * 图片文件名包含空格时产生的 URL 编码差异（`%20`）；
  * 正文中若恰好出现与临时路径相同的内容，可能引发误伤。
* **健壮实现**：引入 **Flexmark-Java** 或 **CommonMark-Java**，在内存中构建 Markdown AST，通过 Visitor 模式精准遍历所有 `Image` 节点重写 `getUrl()`，最后重新渲染。这使得整条流水线在“Word 结构树输入”与“Markdown 结构树输出”两端都保持了严格的语法树级确定性。

---

## 小结

在这次文档输入层的重构攻坚中，我最大的实战心得是：**不要试图在大模型阶段去弥补前端数据清洗的缺陷**。一旦文档在输入阶段就丢掉了公式语义或图片时序，后端的 Prompt 写得再精妙也只是无源之水。

通过引入 **基于 AST 的流式转换 + 行内占位符时序保真 + 媒体资产重写管道**，配合严密的**生产级进程与格式治理机制**，彻底攻克了传统文档解析中最让人头痛的“图文脱节”和“公式乱码”顽疾。

现在，系统已经能够将一份复杂的 Word 培训手册或技能试卷，转化为包含规范 LaTeX 计算公式和精准内联配图的高质量 Markdown 数据流。
