# EyesCare Theme for Obsidian

> 一个融合文学美学的护眼主题，浅色模式如川端康成文字般柔润温暖，深色模式如马尔克斯般梦幻迷人。

[![Version](https://img.shields.io/badge/version-2.0.0-blue.svg)](./manifest.json)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
[![Obsidian](https://img.shields.io/badge/Obsidian-0.16%2B-purple.svg)](https://obsidian.md)

## 🎨 设计理念

### 浅色模式 - 川端康成的柔润美学
- **温润如玉的配色**：采用米白、暖灰、金棕色系，营造温暖舒适的阅读环境
- **柔和的视觉层次**：通过细腻的色彩渐变和阴影，创造深度感而不失柔和
- **护眼优化**：降低蓝光成分，增加暖色调，减少长时间阅读的视觉疲劳

### 深色模式 - 马尔克斯的梦幻色彩
- **神秘的深蓝紫色系**：营造如魔幻现实主义般的梦幻氛围
- **月光般的文字色彩**：柔和的浅色文字，在深色背景中如月光般温柔
- **减少疲劳设计**：优化对比度，保护夜间阅读时的眼部健康

## ✨ 主要特色

### 🎯 护眼设计
- **科学配色**：基于色彩心理学和视觉工程学原理
- **动态适应**：支持系统主题自动切换
- **长时间阅读优化**：减少眼部疲劳，提升专注力

### 🎨 视觉美学
- **文学气息**：融合东西方文学大师的美学风格
- **优雅交互**：流畅的动画效果和微交互设计
- **层次分明**：清晰的信息架构和视觉层次

### 🔧 功能完善
- **全面覆盖**：支持所有Obsidian核心功能
- **响应式设计**：适配不同屏幕尺寸
- **可访问性**：支持高对比度和减少动画模式

### ⚡ 性能优化
- **GPU加速**：使用transform和opacity进行硬件加速动画
- **渲染优化**：CSS containment减少重绘区域
- **流畅体验**：精心调校的动画曲线和时长

## 🎪 独特交互效果

### 动态元素
- **光标闪烁动画**：优雅的呼吸式闪烁效果
- **悬浮效果**：微妙的阴影和位移动画
- **波纹点击效果**：按钮点击时的水波纹扩散
- **渐入动画**：页面和元素的流畅加载效果
- **标签切换动画**：底部指示器滑动效果

### 视觉细节
- **标题装饰**：六级标题独特的标记符号（◆ ◇ ○ ● ◐ ◑）
- **引用块美化**：诗意的引号装饰和渐变边框
- **链接样式**：优雅的下划线动画和外链标识
- **列表美化**：精致的项目符号和层次缩进
- **代码块装饰**：顶部渐变条和语言标签
- **分割线艺术**：居中菱形装饰

## 📦 安装方法

### 方法一：手动安装
1. 下载本仓库到本地
2. 将主题文件夹复制到Obsidian库的 `.obsidian/themes/EyesCare/` 目录
3. 在Obsidian设置 → 外观 → 主题中选择 "EyesCare"
4. 享受优雅的阅读写作体验

### 方法二：社区主题市场（即将推出）
- 在Obsidian设置 → 外观 → 社区主题中搜索 "EyesCare"
- 点击安装即可自动更新

## 🎨 配色方案详解

### 浅色模式配色
```css
/* 基础色板 */
--color-base-05: #faf8f5 (温润米白 - 主背景)
--color-base-10: #f5f2ed (柔和象牙 - 次背景)
--color-base-20: #f0ece5 (浅暖灰 - 侧边栏)
--color-base-100: #3d3529 (温润墨色 - 主文字)

/* 强调色 */
--color-accent: #c17817 (温暖金棕)
--color-accent-1: #d4892a (亮金棕)
--color-accent-2: #a56712 (深金棕)

/* 功能色 */
--text-error: #c45c4a (错误红)
--text-success: #5a8a6a (成功绿)
```

### 深色模式配色
```css
/* 基础色板 */
--color-base-05: #1a1625 (神秘深紫 - 主背景)
--color-base-10: #211d2e (梦幻紫灰 - 次背景)
--color-base-20: #151221 (深紫黑 - 侧边栏)
--color-base-100: #e6e1d3 (月光银白 - 主文字)

/* 强调色 */
--color-accent: #7fb3d3 (梦幻蓝色)
--color-accent-1: #9ac8e0 (浅梦幻蓝)
--color-accent-2: #6a9eb8 (深梦幻蓝)

/* 功能色 */
--text-error: #e08070 (柔和红)
--text-success: #80c090 (柔和绿)
```

## 🔧 自定义选项

### CSS变量自定义
主题提供了400+ CSS变量，您可以通过CSS片段进行个性化调整：

```css
/* 自定义浅色模式强调色 */
.theme-light {
  --color-accent: #your-color;
  --color-accent-1: #your-light-color;
  --color-accent-2: #your-dark-color;
}

/* 自定义深色模式背景 */
.theme-dark {
  --background-primary: #your-background;
  --background-primary-alt: #your-alt-background;
}

/* 自定义动画时长 */
:root {
  --duration-fast: 100ms;
  --duration-normal: 200ms;
  --duration-slow: 300ms;
}
```

### 字体自定义
```css
/* 自定义正文字体 */
body {
  --font-text-theme: "您的字体", "Noto Serif CJK SC", serif;
}

/* 自定义代码字体 */
body {
  --font-monospace-theme: "您的等宽字体", "Fira Code", monospace;
}
```

### 禁用动画（减少动画模式）
```css
/* 在CSS片段中添加 */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition: none !important;
  }
}
```

## 🌟 适用场景

- **学术研究**：长时间阅读论文和文献
- **创意写作**：小说、诗歌、散文创作
- **知识管理**：个人知识库建设
- **日常笔记**：工作学习记录
- **夜间阅读**：深色模式护眼设计
- **编程开发**：优化的代码高亮和等宽字体

## 📋 主题结构

```
obsidian-eyescare-theme/
├── manifest.json          # 主题配置
├── theme.css              # 主题样式（约1800行）
├── README.md              # 项目说明
├── LICENSE                # MIT许可证
└── .gitignore             # Git忽略文件
```

## 🎯 设计原则

1. **以人为本**：优先考虑用户的视觉健康和使用体验
2. **文学美学**：融合东西方文学大师的审美理念
3. **功能完善**：确保所有功能的完美呈现
4. **性能优先**：流畅的动画和高效的渲染
5. **可访问性**：支持各种辅助功能需求
6. **持续优化**：根据用户反馈不断改进

## 📝 更新日志

### v2.0.0 (2024-04-07)
- ✨ **全面升级**：400+ CSS变量完整覆盖
- ⚡ **性能优化**：GPU加速动画和CSS containment
- 🎨 **动画系统**：精心设计的微交互和过渡效果
- 🔧 **设计令牌**：统一的间距、圆角、阴影系统
- ♿ **无障碍支持**：减少动画和高对比度模式
- 📱 **响应式设计**：移动端适配优化

### v1.0.0 (2024-01-15)
- 🎉 初始版本发布
- ✨ 完整的浅色/深色模式支持
- 🎨 独特的文学美学设计
- 🔧 全面的功能覆盖
- 📱 响应式设计支持

## 🤝 贡献指南

欢迎提交Issue和Pull Request来帮助改进这个主题！

### 开发流程
1. Fork本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建Pull Request

### 反馈渠道
- GitHub Issues：报告bug和功能请求
- Discussions：分享使用心得和建议
- Pull Requests：提交代码改进

## 📄 许可证

MIT License - 自由使用和修改

详见 [LICENSE](./LICENSE) 文件

## 👨‍💻 作者

**SantaChains**
- 专注于用户体验和视觉设计
- 热爱文学与技术的完美融合
- 致力于创造更美好的数字阅读体验

## 🙏 致谢

感谢所有为开源社区贡献的开发者们，特别是：
- Obsidian团队提供的优秀平台
- 社区中提供反馈和建议的用户们
- 川端康成和马尔克斯的文学作品给予的美学启发
- 各大优秀Obsidian主题的设计灵感

## 🔗 相关链接

- [Obsidian官网](https://obsidian.md/)
- [主题开发文档](https://docs.obsidian.md/Themes/App+themes/Build+a+theme)
- [CSS变量参考](https://docs.obsidian.md/Reference/CSS+variables/Index)
- [CSS自定义指南](https://help.obsidian.md/Extending+Obsidian/CSS+snippets)

---

*让阅读成为一种美的享受，让写作成为一种艺术的表达。*

**EyesCare Theme - 为您的思想提供最舒适的栖息地。**
