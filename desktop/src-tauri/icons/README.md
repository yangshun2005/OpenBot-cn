# OpenBot 桌面图标

这些资源导出了 [桌面样式表](../../src/styles.css) 中现有的静态 `.orb`、`.orb::before` 和 `.orb::after` 图形。欢迎屏幕使用了相同的图形。没有重绘任何调色板、渐变、高光或边缘。

本次导出所使用的源样式表 SHA-256 为
`d20db4a7bdc371fbb7a3f0780f42080dc08374811c75d2fe893f49fb962bbb1d`。

`icon.png` 是透明的 1024×1024 主图。源图形保持 30 CSS 像素宽，居中于 32×32 的透明视口中，设备缩放为 32。这保留了固定像素的边缘和阴影比例，每边留有 1 CSS 像素的透明边距。截图使用 sRGB，包含两个伪元素，且不加载任何应用程序或网络内容。

## 复现源导出

使用的工具版本：Node 24.16.0、Playwright 1.62.1、Chromium 151.0.7922.34、Bun 1.4.3，以及桌面 lockfile 中的 Tauri CLI 2.11.4。使用已安装的 Playwright 1.62.1 模块及其匹配的 Chromium 可执行文件。渲染器会记录可执行文件的哈希值，并拒绝使用不同版本的浏览器。

将以下内容保存为检出目录之外的 `export-orb.cjs`。它接受源检出目录、新的输出目录、Playwright 模块目录以及 Chromium 可执行文件路径作为显式参数：

```javascript
// Usage: node export-orb.cjs SOURCE_ROOT FRESH_OUTPUT PLAYWRIGHT_DIR CHROMIUM
// The output must not already exist. No source or application process is modified.
const fs = require('node:fs');
const path = require('node:path');
const crypto = require('node:crypto');

async function main() {
  const [sourceArg, outputArg, playwrightArg, executableArg] = process.argv.slice(2);
  if (!sourceArg || !outputArg || !playwrightArg || !executableArg) {
    throw new Error('Expected SOURCE_ROOT FRESH_OUTPUT PLAYWRIGHT_DIR CHROMIUM');
  }
  const source = fs.realpathSync(sourceArg);
  const output = path.resolve(outputArg);
  const executablePath = fs.realpathSync(executableArg);
  const playwrightDir = fs.realpathSync(playwrightArg);
  const version = require(path.join(playwrightDir, 'package.json')).version;
  if (version !== '1.62.1') throw new Error(`Expected Playwright 1.62.1, got ${version}`);
  const cssPath = path.join(source, 'desktop/src/styles.css');
  const css = fs.readFileSync(cssPath, 'utf8');
  const selectors = ['.orb', '.orb::before', '.orb::after'];
  const rules = selectors.map(selector => {
    const escaped = selector.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const matches = [...css.matchAll(new RegExp(`^${escaped} \\{[^}]*\\}`, 'gm'))];
    if (matches.length !== 1) throw new Error(`Expected exactly one ${selector} rule`);
    return matches[0][0];
  }).join('\n\n');
  fs.mkdirSync(output);
  fs.writeFileSync(path.join(output, 'orb-source.css'), rules + '\n');
  const { chromium } = require(playwrightDir);
  const browser = await chromium.launch({ executablePath, headless: true, args: ['--force-color-profile=srgb'] });
  try {
    if (browser.version() !== '151.0.7922.34') throw new Error(`Unexpected Chromium ${browser.version()}`);
    const page = await browser.newPage({ viewport: { width: 32, height: 32 }, deviceScaleFactor: 32, colorScheme: 'light' });
    await page.route('**/*', route => route.abort());
    await page.setContent(`<style>${rules}\nhtml, body { margin: 0; width: 32px; height: 32px; background: transparent; } body { display: grid; place-items: center; }</style><div class="orb"></div>`);
    const geometry = await page.locator('.orb').evaluate(element => {
      const rect = element.getBoundingClientRect();
      return { x: rect.x, y: rect.y, width: rect.width, height: rect.height, dpr: devicePixelRatio };
    });
    if (JSON.stringify(geometry) !== JSON.stringify({ x: 1, y: 1, width: 30, height: 30, dpr: 32 })) {
      throw new Error(`Unexpected source geometry: ${JSON.stringify(geometry)}`);
    }
    await page.screenshot({ path: path.join(output, 'icon.png'), omitBackground: true, animations: 'disabled', scale: 'device' });
    const sha256 = file => crypto.createHash('sha256').update(fs.readFileSync(file)).digest('hex');
    fs.writeFileSync(path.join(output, 'render.json'), JSON.stringify({
      source, cssPath, cssSha256: sha256(cssPath), sourceRulesSha256: sha256(path.join(output, 'orb-source.css')),
      playwright: version, chromium: browser.version(), executablePath, executableSha256: sha256(executablePath),
      colorProfile: 'sRGB', transparentBackground: true, network: 'all routes aborted', geometry,
      outputSize: [1024, 1024], iconSha256: sha256(path.join(output, 'icon.png')),
    }, null, 2) + '\n');
    console.log(fs.readFileSync(path.join(output, 'render.json'), 'utf8'));
  } finally { await browser.close(); }
}
main().catch(error => { console.error(error); process.exitCode = 1; });
```

从检出根目录运行，替换为已安装的工具路径：

```sh
node /path/to/export-orb.cjs "$PWD" /tmp/openbot-orb-export \
  /path/to/playwright /path/to/chromium
cd desktop
bun run tauri icon /tmp/openbot-orb-export/icon.png \
  --output /tmp/openbot-orb-generated
cp /tmp/openbot-orb-export/icon.png src-tauri/icons/icon.png
for asset in 32x32.png 64x64.png 128x128.png 128x128@2x.png icon.ico icon.icns; do
  cp "/tmp/openbot-orb-generated/$asset" "src-tauri/icons/$asset"
done
```

每次运行都使用全新的临时目录。将直接导出的文件保留为主图：CLI 还会生成一个较低分辨率的 `icon.png`。只复制列出的六个衍生文件；其 Store、Square、Android 和 iOS 输出在此处不使用。锁定的 CLI 默认图标导出为每个衍生文件提供了重采样，包括所有平台上由 `tray::icon()` 嵌入的 64×64 RGBA 图像。

对于保真度检查，请在新目录中重新运行渲染器和图标命令，并比较主图和 PNG 衍生文件的解码 RGBA 像素。ICO 包含 16、24、32、48、64 和 256 像素的 PNG 帧。在 macOS 上使用
`iconutil --convert iconset --output /tmp/openbot-orb.iconset src-tauri/icons/icon.icns`
解码 ICNS。ICNS 块顺序在不同 CLI 运行之间会有所变化；应比较带类型的块载荷和解码帧像素，而不是要求整个文件哈希完全一致。用于这些资源的导出和重放具有相同的解码像素。

Rust 库测试会检验生产环境嵌入的图像：尺寸和 RGBA 长度、透明角落、可见内部区域以及多样的颜色。原始的不透明占位符无法通过透明度和颜色变化测试。像素检查确立了源保真度；实际的本地托盘渲染需要单独的操作系统桌面检查。

## 生成的资源哈希

这些 SHA-256 哈希标识本次导出，包括其特定的 ICNS 块顺序。

| 文件 | SHA-256 |
| --- | --- |
| `icon.png` | `455392ca6a53eddaf8b08b990eb404be53de3e9154a1bdc5d4bf9daa38b5ae1d` |
| `32x32.png` | `b3b2d402cb3cf50b7140746688675da308f3e06ff321888a987efd64dc9339ba` |
| `64x64.png` | `b5ced22dcc63d0b8881dba384a715ac3e74f52d18c46a40d923f112b5e335573` |
| `128x128.png` | `ffc8fe777c84074355d08410c39bdebe29d16a7e59aabb3902154bb421d9449a` |
| `128x128@2x.png` | `046319324c303eef04542bdb2efe3b1e75563618ad104db24f9b2552c4dc522d` |
| `icon.ico` | `44dab03a0b7a92f5f87b7f33e5823f4d6f8b5687efb115686290dded58e27a55` |
| `icon.icns` | `45926bd67fc5deab949d7d21148da3c3a81cc15890cc111619b447b04c606c48` |