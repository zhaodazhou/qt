# cmark 库
cmark 库是将文本转换为 html 格式的文本转换为Markdown格式

### 安装相应库
```
dnf install cmark
```

### 配置
在 CMakeLists.txt 中进行如下配置：
```
# 查找cmark库
find_package(PkgConfig REQUIRED)
pkg_check_modules(CMARK REQUIRED libcmark)


# 添加包括路径
include_directories(${CMARK_INCLUDE_DIRS})

# 添加库链接路径
link_directories(${CMARK_LIBRARY_DIRS})


target_link_libraries(ruyi-ai-assistant
  ${CMARK_LIBRARIES}
)

```

如果有 spec 文件，则需添加如下配置
```
BuildRequires:  cmark-devel

Requires:       cmark-devel
```

### cmark使用
在 cpp 文件中如下使用
```c++
// 引入头文件
#include <cmark.h>

//  具体使用
QString markdownConvertHtml(const QString &markdownText)
{
    char *html = cmark_markdown_to_html(markdownText.toUtf8().constData(), markdownText.toUtf8().size(), CMARK_OPT_DEFAULT);
    QString result = QString::fromUtf8(html);
    free(html);
    return result;
}
```