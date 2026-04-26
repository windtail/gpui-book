# 第14章：图片元素

## 14.1 `img()` 基础

### 14.1.1 从资源加载

```rust
use gpui::{img, div, IntoElement, Context, Window, Pixels};

struct ImageView {
    image_path: String,
}

impl Render for ImageView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(
                img("icons/logo.png")  // 从嵌入资源加载
                    .size(Pixels(64.0))
            )
    }
}
```

`img()` 接受任何可以转换为 `ImageSource` 的类型。字符串字面量被识别为嵌入资源路径。

### 14.1.2 从路径加载

```rust
use std::path::PathBuf;

let path: PathBuf = PathBuf::from("/home/user/photo.jpg");
div().child(img(path.clone()))
```

GPUI 自动区分 URI 和本地路径：如果字符串包含 `://`，按 URI 处理；否则按本地文件路径处理。

```rust
img("https://example.com/image.png")   // 网络 URI
img("/home/user/photo.jpg")            // 绝对路径
img("icons/logo.png")                  // 嵌入资源
```

### 14.1.3 支持的格式

GPUI 使用 `image` crate 解码，支持以下格式：

| 格式 | 说明 |
|------|------|
| PNG | 无损压缩，支持透明 |
| JPG | 有损压缩 |
| GIF | 支持动画 |
| WebP | Google 的现代化格式 |
| SVG | 矢量图（作为静态光栅化） |
| AVIF | 高效压缩格式 |

## 14.2 `ImageSource` 枚举

```rust
pub enum ImageSource {
    Resource(Resource),    // 从资源加载
    Render(Arc<RenderImage>),  // 运行时生成
    Image(Arc<Image>),     // 预加载的图片
    Custom(Arc<dyn Fn(&mut Window, &mut App) -> Option<Result<Arc<RenderImage>, ImageCacheError>>>),
}
```

### 14.2.1 `Resource` —— 编译时嵌入

```rust
// 通过字符串或路径自动识别
img("assets/icon.png")
img(PathBuf::from("/path/to/image.jpg"))
```

### 14.2.2 `Render` —— 运行时生成

```rust
use gpui::RenderImage;

// 从 DynamicImage 生成
let dynamic_image: DynamicImage = image::open("photo.jpg").unwrap();
let render_image = Arc::new(RenderImage::new(dynamic_image));

div().child(
    img(ImageSource::Render(render_image))
        .size(Pixels(200.0))
)
```

### 14.2.3 `Image` —— 预加载的图片

```rust
use gpui::Image;

// Image 是更底层的类型，通常由 ImageCache 返回
let image: Arc<Image> = ...;
div().child(img(ImageSource::Image(image)))
```

### 14.2.4 `Custom` —— 自定义加载

```rust
img(ImageSource::Custom(Arc::new(|window, cx| {
    // 自定义加载逻辑
    // 返回 Result<Arc<RenderImage>, ImageCacheError>
    Some(Ok(render_image))
})))
```

## 14.3 `ObjectFit` 模式

```rust
pub enum ObjectFit {
    Fill,       // 拉伸填充，可能变形
    Contain,    // 完整显示，保持比例，可能有空白
    Cover,      // 填满区域，保持比例，可能裁剪
    ScaleDown,  // 类似 Contain，但不放大
    None,       // 原始尺寸
}
```

```rust
div()
    .size(Pixels(100.0))
    .overflow_hidden()
    .child(
        img("photo.jpg")
            .size_full()
            .object_fit(ObjectFit::Cover)  // 裁剪适配
    )
```

选择指南：
- **头像/缩略图**：`Cover`（填满正方形）
- **产品图片**：`Contain`（完整显示）
- **背景图**：`Cover`（覆盖容器）
- **图标**：`None` 或 `Contain`

## 14.4 图片加载与缓存

### 14.4.1 异步加载流程

图片加载是异步的：

```
img("photo.jpg")
    ↓
GPUI 检查缓存
    ↓
    ├─ 命中 → 直接渲染
    └─ 未命中 → 启动异步加载任务
                    ↓
              读取文件/网络请求
                    ↓
              解码图片
                    ↓
              存入缓存
                    ↓
              触发重新渲染
```

### 14.4.2 错误处理与 fallback

```rust
struct SafeImage {
    source: String,
    fallback: String,
}

impl Render for SafeImage {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(img(self.source.as_str()))
            // GPUI 在图片加载失败时会显示空白
            // 可以通过条件渲染添加 fallback
    }
}
```

对于需要更精细图片缓存控制的场景，使用 `image_cache()` 并传入一个 `ImageCacheProvider`：

```rust
// 使用 retain_all 缓存策略，所有加载过的图片都保留在缓存中
div()
    .image_cache(retain_all("gallery-cache"))
    .child(img(self.source.as_str()))
```

### 14.4.3 GIF 动画支持

```rust
div().child(
    img("animation.gif")
        .size(Pixels(100.0))
)
```

GPUI 自动解码并播放 GIF 动画帧。需要保持事件循环运行以触发动画帧更新。

## 14.5 图片元素与 `image_cache()`

```rust
// 在 div 上设置图片缓存
struct ImageGallery {
    photos: Vec<String>,
}

impl Render for ImageGallery {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .grid()
            .grid_cols(3)
            .gap_2()
            .image_cache(retain_all("gallery-cache"))  // 在此 div 子树中启用缓存
            .children(
                self.photos.iter().map(|path| {
                    div()
                        .size(Pixels(200.0))
                        .overflow_hidden()
                        .child(
                            img(path.as_str())
                                .size_full()
                                .object_fit(ObjectFit::Cover)
                        )
                })
            )
    }
}
```

`image_cache()` 将图片缓存绑定到 div 子树中。相同路径的图片只加载一次，避免重复的网络请求和解码操作。

### `StyledImage` trait

```rust
pub trait StyledImage: Styled {
    fn object_fit(mut self, fit: ObjectFit) -> Self {
        self.style().object_fit = Some(fit);
        self
    }

    fn grayscale(mut self) -> Self {
        self.style().grayscale = Some(true);
        self
    }
}
```

通过 `StyledImage` trait 为 div 添加图片样式支持。
