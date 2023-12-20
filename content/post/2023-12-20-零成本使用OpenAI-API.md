---
title: "零成本使用OpenAI API"
date: 2023-12-20T15:25:52+08:00
tags: ["openai"]
draft: false
---

### 1. OpenAI ChatGPT

ChatGPT已经发布了一年有余，成为有史以来用户增长最快的互联网产品。如果到了2023年你还没有使用过ChatGPT，可能你已经远离了互联网的中心。ChatGPT的发布与更新深刻改变了我的工作方式。我学到了如何撰写高效的提示，发现了ChatGPT的最佳应用场景，并在GitHub上探索了最佳实践。

一些影响我ChatGPT之旅的值得一提的资源包括[OpenAI官方指南：如何提高ChatGPT的输出质量](https://www.huxiu.com/article/2440157.html)，一篇关于[2023年OpenAI GPT-3最热门应用案例的文章](https://blog.wordbot.io/ai-artificial-intelligence/openai-gpt-3-top-22-trending-use-case-ideas-in-2022/)，以及GitHub上的[GPT提示列表](https://github.com/linexjlin/GPTs)。

在我的日常工作中，我利用ChatGPT做了很多事情：

- 它成为我在编写代码过程中查找相关文档的首选工具，替代了传统搜索引擎的需求。
- 它轻松帮助我进行中英文翻译，充分发挥了其语言处理能力。
- 在数据分析中，它高效地协助我处理复杂的SQL查询。

尽管我已经是一个长时间的用户，但我仍然保持在免费计划上，没有选择Plus，也没有使用OpenAI API。由于ChatGPT的充值的限制，我尚未探索基于OpenAI API构建的众多AI工具和插件。

<!--more-->

### 2. Gemini Pro API

Gemini Pro是最近由Google发布的大型语言模型，作为ChatGPT的竞争对手。尽管从各项研究来看，它的性能略逊于`gpt-3.5-turbo`，但作为通用用途的语言模型已经足够出色。尤其值得一提的是，它提供了每分钟60次的免费API调用，使我们能够以零成本使用。

参考资源：[《Gemini的语言能力深度剖析》](https://arxiv.org/abs/2312.11444)

鉴于OpenAI API已经成为事实上大型模型API协议的标准，市面上许多AI工具和插件仅支持OpenAI API。然而，作为新兴竞品，Gemini Pro API尚未得到众多AI工具的支持。因此，我开发了Gemini-OpenAI-Proxy这个协议转换代理，以帮助各种AI工具能够使用Gemini Pro API。

GitHub链接：[Gemini-OpenAI-Proxy](https://github.com/Coderbaobao/gemini-openai-proxy)

这个工具具备优秀的OpenAI API兼容性，只需在[Google AI平台](https://ai.google.dev/)上申请一个API KEY，就可以像使用OpenAI API一样轻松使用Gemini Pro API。

作为一个服务端程序，我们需要将Gemini-OpenAI-Proxy部署在开放Gemini Pro API的国外服务器上。我推荐使用[fly.io](https://fly.io/)进行零成本部署到日本或新加坡，以便在中国大陆能够快速响应。

### 3. AI工具

拥有独立的OpenAI API服务后，我开始在工作和生活中广泛使用各种AI工具。在工作中，我倚赖以下一些工具和插件：

  - [Chatbox](https://chatboxai.app/)：桌面聊天客户端，直接与AI对话，内置了多种场景和prompt。
  - [Pal](https://apps.apple.com/us/app/pal-ai-chat-client/id6447545085)：iOS聊天客户端，方便随时随地与AI沟通。我会让AI帮忙创作睡前故事，然后播放给我的宝宝听。
  - [ChatGPT Summary Assistant](https://chromewebstore.google.com/detail/nnjcoododbeemlmmhbfmmkbneniepaog)：对于较长的文章，我首先使用文章总结助手进行摘要总结，然后再判断是否需要更深入的精读。
  - [OpenAI Translator](https://chrome.google.com/webstore/detail/openai-translator/ogjibjphoadhljaoicdnjnmgokohngcc)：AI翻译工具，在阅读英文文章时尤其实用。

这些工具的使用让我的工作效率得到了提升，同时也为生活增添了一些趣味。特别是在使用AI翻译工具时，它在阅读英文文章时发挥了巨大的帮助作用。

### 4. 总结

总的来说，以ChatGPT为代表的大型语言模型已经深刻地改变了我们的工作和生活方式，同时也对整个世界产生了深远的影响。这些先进的AI工具不仅提高了工作效率，还为创造性和创新性的任务提供了新的可能性。随着技术的不断进步，我们可以期待这些语言模型继续发挥更大的作用，推动着人工智能在各个领域的进一步发展。在这个语言驱动的时代，ChatGPT等大型语言模型正在引领着我们走向更加智能化、创新化的未来。



注: 这篇文章使用`gpt-3.5-turbo`模型进行润色!
