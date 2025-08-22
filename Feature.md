下面按模块介绍项目中各文件的职责，并列出各自的核心函数与用途（挑关键函数说明，非逐行罗列）。

### 核心入口与消息分发
- `Reader/Reader.cpp`
  - 作用：应用入口、主窗口创建、消息循环、菜单/快捷键/托盘等交互总控。
  - 关键函数：
    - `_tWinMain`：程序入口，初始化 GDI+/COM、加载缓存、注册窗口类、创建主窗体与消息循环。
    - `MyRegisterClass`：注册窗口类（图标、光标、菜单等）。
    - `InitInstance`/`Exit`：实例初始化/退出清理。
    - `WndProc`：主窗口过程，转发菜单与快捷键消息到各业务处理函数。
    - 一系列处理函数（在 `Reader.h` 声明，定义多在此文件）：
      - `OnCreate/OnSize/OnPaint/OnDraw/Invalidate/ResetLayerd`：窗口生命周期与重绘。
      - 打开/跳转/搜索/书签/自动翻页等：`OnOpenItem/OnOpenFile/OnPageUp/OnPageDown/OnLineUp/OnLineDown/OnJump/OnSearch/OnAddMark/OnAutoPage/OnEditMode/OnChapterUp/OnChapterDown/OnFontZoomIn/OnFontZoomOut`。
      - 系统行为：`OnFullScreen/OnTopmost/OnHideBorder/OnHideWin/OnDropFiles/OnFindText/OnUpdateMenu/OnUpdateChapters/OnClearFileList/OnRestoreDefault`。
      - 网络与升级（启用 ENABLE_NETWORK 时）：`UpgradeProc`、在线书相关窗口过程与异步消息。
    - 内部辅助：`_update_data/_free_resource` 等资源管理。
- `Reader/Reader.h`
  - 作用：全局状态/句柄/对象声明，窗口行为函数原型。
  - 重要全局：
    - `_Book` 当前书对象（`Book*`），`_Cache` 缓存，`_header` 显示配置，`_item` 当前书记录，`_hWnd` 主窗体，热键钩子等。
    - `loading_data_t` 动图加载状态（GIF 帧管理）。
  - 消息处理函数原型见上。

### 文本排版与分页
- `Reader/Page.h` / `Reader/Page.cpp`
  - 作用：纯文本分页、绘制与翻页逻辑的基础类。
  - 核心数据结构：`char_info_t/line_info_t/lines_t/page_info_t/dc_info_t/alpha_dc_info_t`。
  - 关键方法：
    - 导航：`PageUp/PageDown/LineUp/LineDown/IsFirstPage/IsLastPage/IsCoverPage/GetProgress`。
    - 渲染：`DrawPage/ReDraw/BeginDraw/EndDraw/DrawAlphaText/CreateAlphaTextBitmap/DeleteAlphaTextBitmap/SelectFont/SelectFontByDcIndex/GetTextAlpha`。
    - 页面数据：`GetCurPageText/SetCurPageText/GetPageLength/GetTextLength/IsBlankPage`。
    - 行处理：`GetLine`（抽取一行，返回行长、空白行、前后空格等信息）。
    - 初始化与绑定：`Init(int* p_index, header_t* header)`。

### 书籍抽象与格式实现
- `Reader/Book.h` / `Reader/Book.cpp`
  - 作用：书籍抽象类，管理书文本、章节、跳转、统一解码、格式化。
  - 关键类型：`chapter_item_t`（章节索引、标题、URL、长度）、`chapters_t`。
  - 核心方法：
    - 生命周期：`OpenBook/CloseBook/OpenBookThread/ForceKill/IsLoading`。
    - 编码与预处理：`DecodeText`（检测 BOM/UTF-8/ANSI → UTF-16）、`FormatText`（统一换行缩进等）。
    - 章节：`GetChapters/GetCurChapterIndex/JumpChapter/JumpPrevChapter/JumpNextChapter/IsChapterIndex/IsChapter/GetChapterInfo/GetChapterTitle/SetChapterRule`。
    - 抽象接口（子类实现）：`GetBookType/SaveBook/UpdateChapters/ParserBook`。
- `Reader/TextBook.h` / `Reader/TextBook.cpp`
  - 作用：TXT 文本书实现。
  - 关键点：
    - `ParserBook`：读取文件/内存 → `ReadBook` → `DecodeText` → `ParserChapters`。
    - `ReadBook`：读取二进制、补零尾，交给 `DecodeText` 做编码识别。
    - 章节解析：`ParserChaptersDefault/ParserChaptersKeyword/ParserChaptersRegex/IsChapter`（按“第X章/卷/节/序/楔子”等规则，或关键字/正则切分）。
    - `SaveBook`：以 UTF-16LE+BOM 写回。
    - `UpdateChapters`：编辑后调整章节索引。
- `Reader/EpubBook.h` / `Reader/EpubBook.cpp`
  - 作用：EPUB 支持（基于 zip+xml/html）。
  - 关键点：解压目录、解析 `opf`/`ncx`/HTML，通过 `DecodeText` 统一成 UTF-16，抽取章节与正文。
- `Reader/OnlineBook.h` / `Reader/OnlineBook.cpp`
  - 作用：在线书籍抓取与分页（需网络）。
  - 关键点：根据 `bs.json` 或 `.ol` 书源配置，抓章节列表、正文，处理分页、合并分页；请求发送、编码识别、内容拼接与清洗。

### UI 配置与对话框
- `Reader/DisplaySet.h` / `Reader/DisplaySet.cpp`
  - 作用：显示设置对话框（字体/颜色/背景图/字距/行距/段距/边距/换行/缩进/空行/章节独立页等）。
  - 关键：`OpenDisplaySetDlg`、`_init_*` 初始化 UI，`_valid_*` 校验，`_update_preview` 预览绘制，变更后回写到 `header_t` 并触发重排/重绘。
- `Reader/Advset.h` / `Reader/Advset.cpp`
  - 作用：高级设置（更细的行为选项）。
- `Reader/Keyset.h` / `Reader/Keyset.cpp`
  - 作用：按键绑定设置，消息到行为的映射配置。
- `Reader/OnlineDlg.h` / `Reader/OnlineDlg.cpp`
  - 作用：在线书源搜索、选择、拉取章节/内容的 UI。
- `Reader/BooksourceDlg.h` / `Reader/BooksourceDlg.cpp`
  - 作用：书源规则编辑（XPath/JSON 路径、分页、下一页关键字、编码等），导入/导出/同步书源。
- `Reader/Editctrl.h` / `Reader/Editctrl.cpp`
  - 作用：编辑框/查找替换等辅助控件行为。
- `Reader/DPIAwareness.h` / `Reader/DPIAwareness.cpp`
  - 作用：高 DPI 支持（Per-Monitor Aware），缩放字体/布局。

### 数据与工具
- `Reader/Cache.h` / `Reader/Cache.cpp`
  - 作用：缓存管理（最近文件、书签、阅读位置等，默认 `.cache.dat`）。
- `Reader/Jsondata.h` / `Reader/Jsondata.cpp`
  - 作用：配置数据的 JSON 读写（依赖 cJSON），如书源配置导入/导出。
- `Reader/HtmlParser.h` / `Reader/HtmlParser.cpp`
  - 作用：HTML 解析清洗（结合 libxml2），抽取正文、去广告等。
- `Reader/Utils.h` / `Reader/Utils.cpp`
  - 作用：通用工具函数集合。
  - 编码转换：`ansi_to_utf16/utf16_to_ansi/utf8_to_utf16/utf16_to_utf8/utf16_to_utf8_bom`、`Utf16ToUtf8/Utf16ToAnsi/Utf8ToUtf16`、BOM/UTF-8 检测：`check_bom/is_ascii/is_utf8`。
  - 字节序转换：`le_to_be/be_to_le`。
  - Base64：`b64_encode/b64_decode`。
  - URL：`url_encode/url_decode`。
  - 版本/平台：`Is_WinXP_SP2_or_Later/GetApplicationVersion`。
- `Reader/types.h`
  - 作用：全局常量/结构体/枚举。
  - 例如：`header_t`（字体、颜色、边距、背景、换行/缩进等显示配置）、`item_t`（书本记录）、`bg_image_t`、`auto_page_mode_t`、自定义菜单 ID 与消息常量等。
- `Reader/Upgrade.h` / `Reader/Upgrade.cpp`
  - 作用：版本检查与升级（仅网络启用时）。

### 资源与工程
- `Reader/Reader.rc`、`Reader/Reader_zh-cn.rc`
  - 作用：资源清单（菜单、对话框、字符串表、图标、游标、位图、PNG、GIF、版本信息等）。
- `Reader/resource.h`
  - 作用：所有资源 ID。
- `Reader/Reader.vcxproj` / `Reader.vcxproj.filters` / `Reader.sln`
  - 作用：工程/过滤器与解决方案文件。
- `ReadMe.txt`（工程说明）、`readme.txt`（发布说明）、`rebuild.bat`（批量构建与打包）。

### 第三方依赖（`opensrc/`）
- `cjson/`：cJSON（源码编译），用于 JSON 解析。
- `libxml2/`：HTML/XML 解析。
- `zlib/`、`miniz/`：压缩/解压。
- `openssl/`、`wolfssl/`、`libhttps/`：HTTPS/SSL（启用网络功能时）。

### 运行时资源（`tool/`）
- `7z.exe/ol2txt.exe/repstr.exe`：构建与书源辅助工具（不参与编译链接，供脚本或运维用）。

### 典型调用链（TXT 打开与显示）
1) 用户操作/拖拽 → `OnOpenItem/OnOpenFile`  
2) 创建具体书对象（如 `TextBook`）→ `OpenBook`（新线程）  
3) `ReadBook` 读取 → `DecodeText` 编码到 UTF-16 → `ParserChapters*` 解析章节  
4) `Page::Init` 绑定 `header_t` → 首次 `DrawPage`  
5) 翻页/跳转调用 `PageUp/PageDown/JumpChapter` → 重排/重绘  

如果您希望我对某个具体文件或某段函数作更细的参数/返回值/边界条件说明，请告诉我文件名与函数名，我再展开到实现级别。