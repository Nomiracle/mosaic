# Mosaic Lab · 图片打码工作台

> 纯浏览器本地的图片打码工具：导入图片 → 框选敏感区域 → 不可逆打码 → 导出。
> **图片不会离开你的设备** —— 页面不发起任何网络请求，没有后端，没有上传，没有存储。

在线使用：**https://mosaic.tsuibo.xyz** · 若它帮到了你，欢迎点个 ⭐ Star

---

## 为什么再造一个打码工具

大多数在线打码工具把原图上传到服务器处理，这恰恰是敏感图片最不该发生的事。
Mosaic Lab 把处理全部放在浏览器里（Canvas + `ImageData`），服务器只负责发一个静态 HTML 文件，
连一张图片的字节都不会经过网络。

同时它认真对待"对手会用工具还原"这件事：

- **不叠加半透明层**：打码是破坏性重绘，预览与导出走同一条渲染路径，不存在"预览打了、导出没打"。
- **不做画布缩放马赛克**：不依赖 `imageSmoothingEnabled`，而是逐像素求块均值再平面填充，
  导出图中不存在任何插值残影（缩放式马赛克会留下可被利用的插值痕迹）。
- **块大小下限 12px**：小块马赛克对文字/人脸存在被 AI 复原的风险，工具层面直接锁死。
- **元数据彻底剥离**：Canvas 重新编码，不携带 EXIF / GPS / 设备信息 / 内嵌缩略图
  （JPEG 内嵌缩略图常保留原图内容，只删 EXIF 段是不够的）。

## 三种强度

| 模式 | 原理 | 不可逆性 | 适用 |
| --- | --- | --- | --- |
| **实心**（默认） | 纯色覆盖原像素 | 最强，不可逆 | 证件、车牌、文字、屏幕内容、任何敏感画面 |
| **马赛克** | 逐像素块均值平面填充（块 ≥ 12px，可选块内加噪） | 细节不可逆；保留块级色块结构 | 人脸、身体、需要保留画面观感时 |
| **模糊** ⚠ | 盒式模糊 + 细块像素化 | 弱，反卷积与 AI 去模糊可部分还原 | 仅美观用途，敏感内容请改实心 |

马赛克会把红色印章变成"红色块"——它保留块级结构，这正是它的上限。
面对"对手会试图还原"的场景，请用**实心**。

## 功能

- 导入：文件选择、拖拽到窗口、`Ctrl/⌘+V` 粘贴剪贴板、多选批量
- 区域：矩形框选、拖动移动、四角手柄缩放、多区域独立配置（实心/马赛克/模糊可混用）
- **区域同步到全部**：把当前图片的区域按比例复制到队列中其他图片，适合批量处理同版式截图
- 撤销/重做（`Ctrl+Z` / `Ctrl+Shift+Z` / `Ctrl+Y`）、`Delete` 删除选中区域、`Esc` 取消选中
- 缩放平移：滚轮缩放、双指捏合、`+`/`-`/`0`（适应窗口）；按住空格或点「对比原图」查看打码前
- 导出：PNG（无损）/ JPEG（质量可调）；批量导出打包 ZIP（本地生成，无第三方压缩库）

## 导出前自检

每次导出，页面会就地做两件事，并把结果显示在右下角：

1. **破坏度**：逐区域计算打码前后 PSNR。像素未被改变（PSNR = ∞）会报错；
   模糊模式或破坏度不足（PSNR > 28dB）会告警，建议改实心或增大块。
2. **元数据核对**：扫描导出字节，检查 JPEG `APP1/Exif` 段与 PNG `eXIf`/`tEXt`/`iTXt`/`zTXt` 块，
   检出即报错。

自检是提示，不是保证——最终请目视复核一次导出图。

## 本地运行 / 自托管

没有任何构建步骤，单个 `index.html` 即全部：

```bash
# 直接打开
open index.html          # 或双击文件

# 或本地起个静态服务
python3 -m http.server 8080
```

nginx 静态托管示例（含本项目实际使用的严格 CSP）：

```nginx
server {
    listen 443 ssl;
    server_name mosaic.example.com;
    root /var/www/mosaic;          # 放入 index.html
    index index.html;

    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Content-Security-Policy "default-src 'none'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; connect-src 'self'; object-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'self'; upgrade-insecure-requests" always;

    location / { try_files $uri $uri/ /index.html; }
}
```

CSP 中 `default-src 'none'` 是有意为之：即使反代层（如 Cloudflare）注入第三方脚本，
浏览器也会拦掉，实测注入的分析脚本 `transferSize=0`，页面的"零外部请求"承诺依然成立。

## 验证记录

交付时做过浏览器端到端验证（真实鼠标事件 + 真实下载落盘），再由独立 Python 脚本复核：

- 实心：600×260 区域全黑，破坏度 PSNR 2.0dB，导出无 EXIF
- 马赛克：299 个块 **0 个非纯色块**，每块颜色与从原图独立重算的块均值最大偏差 0.5（舍入误差），
  区域外逐像素完全一致
- 批量 ZIP：`zipfile.testzip()` 通过，条目完整，各文件均无 EXIF
- JPEG：无 `APP1/Exif` 段，仅 JFIF
- 撤销/重做/快捷键/批量同步/大图（2400×1600）路径回归通过

## 已知限制

- 动图仅处理首帧（GIF/WebP 动画），导出静态图
- HEIC 能否解码取决于浏览器；不支持时请先转 JPEG/PNG
- 超过 8192px 或 40MP 的图片会自动降采样并在界面提示
- 移动端触摸与双指缩放已实现，但未在真机充分验证
- 2400×1600 全幅打码导出约 8 秒（渲染 + 编码 + 自检）

## License

MIT © 2026 Nomiracle
