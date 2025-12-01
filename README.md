# 网易云音乐解析器

一个小而美的网页应用，用来解析和下载网易云音乐链接。界面清爽，上手即用。

## 亮点

- 支持歌曲、歌单、专辑链接解析
- 多音质选择（标准/极高/无损/母带等）
- 在线播放与下载（智能识别 FLAC/MP3）
- 可选写入音频元数据（MP3/FLAC）
- 下载 ZIP 包时可附带封面、歌词（支持 LRC→SRT 转换）
- 主题切换（浅色/深色），移动端适配

## 技术栈

- Vue 3 + Element Plus
- 原生 `fetch` 封装请求
- Vite 构建

## 支持的链接格式

- `https://music.163.com/song?id=123456`
- `https://y.music.163.com/m/song/123456`
- `https://music.163.com/#/song?id=123456`
- `https://163cn.tv/abc123`

## 音质说明

| 等级 | 格式 | 说明 |
|------|------|------|
| jymaster | FLAC | 母带级 |
| dolby    | MP4 | 杜比 |
| sky      | FLAC | 环绕/沉浸 |
| jyeffect | FLAC | 空间音效 |
| hires    | FLAC | 高解析度 |
| lossless | FLAC | 无损 |
| exhigh   | MP3  | 极高 |
| standard | MP3  | 标准 |

## 快速开始

```bash
npm install
npm run dev
```

构建与预览：

```bash
npm run build
npm run preview
```

开发命令：

```bash
# 本地局域网调试
npm run dev:lan

# 代码检查
npm run lint
```

环境要求：

- Node.js 20.19+ 或 22.12+
- npm 10+（推荐）

## 使用方式

- 粘贴网易云链接，选择音质后解析
- 成功后可在线播放或下载
- 需要 ZIP 包、字幕或写入元数据的，可在设置里开启

## 设置说明

- 文件命名：支持三种命名格式
- 元数据写入：MP3/FLAC 可选开启
- ZIP 下载：打包音频、封面、歌词、信息文件
- 歌词字幕：LRC 转 SRT 可开启
- 播放链接缓存：可按分钟设置 TTL


## 项目结构

```
src/
├── components/         # 组件
├── services/           # 接口与下载
├── utils/              # 工具函数（分页、主题、歌词转换等）
├── App.vue             # 根组件
└── main.js             # 入口
```

## 免责声明

本项目用于学习与交流。请支持正版音乐，遵守相关法律法规。

## 贡献

欢迎提 Issue / PR，一起打磨体验。

## 许可

MIT，详见 [LICENSE](LICENSE)。
