# libpinyin 库
此库用于汉字与拼音之间的转换，api不太好用，链接：https://github.com/libpinyin/libpinyin

### 安装相应库
```
dnf install libpinyin-devel
```

### 配置
在 CMakeLists.txt 中进行如下配置：
```
# 查找库
find_package(PkgConfig REQUIRED)
pkg_check_modules(PINYIN REQUIRED libpinyin)


# 添加包括路径
include_directories(${PINYIN_INCLUDE_DIRS})

# 添加库链接路径
link_directories(${PINYIN_LIBRARY_DIRS})

target_link_libraries(xxx
    ${PINYIN_LIBRARIES}
)

```

如果有 spec 文件，则需添加如下配置
```
BuildRequires:  libpinyin-devel

Requires:       libpinyin-devel
```

### 使用
在 cpp 文件中如下使用
.h 文件如下：
```c++
#include <QObject>

#include <pinyin.h>
#include <QString>
#include <QStringList>
#include <QDebug>
#include <QChar>

// 由于编译问题，需要将源码中的 _ChewingKey 定义加入到我们自己的源码中
struct _ChewingKey
{
    guint16 m_initial : 5;
    guint16 m_middle  : 2;
    guint16 m_final   : 5;
    guint16 m_tone    : 3;
    guint16 m_zero_padding : 1;

    // _ChewingKey() {
    //     m_initial = CHEWING_ZERO_INITIAL;
    //     m_middle  = CHEWING_ZERO_MIDDLE;
    //     m_final   = CHEWING_ZERO_FINAL;
    //     m_tone    = CHEWING_ZERO_TONE;
    //     m_zero_padding = 0;
    // }

    // _ChewingKey(ChewingInitial initial, ChewingMiddle middle,
    //             ChewingFinal final) {
    //     m_initial = initial;
    //     m_middle = middle;
    //     m_final = final;
    //     m_tone = CHEWING_ZERO_TONE;
    //     m_zero_padding = 0;
    // }

public:
    gint get_table_index();
    bool is_valid_zhuyin();

    /* Note: the return value should be freed by g_free. */
    gchar * get_pinyin_string();
    gchar * get_shengmu_string();
    gchar * get_yunmu_string();
    gchar * get_zhuyin_string();
    gchar * get_luoma_pinyin_string();
    gchar * get_secondary_zhuyin_string();
};




class RuyiPinyinManager : public QObject
{
    Q_OBJECT
public:
    explicit RuyiPinyinManager(QObject *parent = nullptr);
    virtual ~RuyiPinyinManager();




    /**
 * 判断字符是否为汉字
 * @param ch 要判断的字符
 * @return 如果是汉字返回 true，否则返回 false
 */
    bool isChineseChar(const QChar &ch);

    /**
 * 将汉字转换为拼音
 * @param text 要转换的文本，可以包含汉字和非汉字
 * @return 转换后的字符数组，汉字部分转为拼音，非汉字部分保持不变
 */
    QStringList chineseToPinyinList(const QString &text);

    /**
 * 将汉字转换为拼音
 * @param text 要转换的汉字字符串
 * @return 拼音字符串，多个汉字的拼音用空格分隔
 */
    QString chineseToPinyin(const QString &text);

    /**
     * @brief pinyinToChinese
     * @param input
     */
    void pinyinToChinese(const std::string& input);




signals:

private:

    pinyin_instance_t * instance;
    pinyin_context_t * context;

    QStringList processChinese(pinyin_instance_t *instance, QString &chineseSegment);
};

#endif // RUYIPINYINMANAGER_H
```

.cpp 文件

```c++
#include "RuyiPinyinManager.h"


#ifdef Q_OS_MAC
 const char*  systemdir = "/opt/homebrew/Cellar/libpinyin/2.10.1/lib/libpinyin/data";
 const char* userdir = "/opt/homebrew/Cellar/libpinyin/2.10.1/lib/libpinyin/data";
#else
 const char* systemdir = "/usr/lib64/libpinyin/data/";
 const char* userdir = "/usr/lib64/libpinyin/data/";
#endif

RuyiPinyinManager::RuyiPinyinManager(QObject *parent)
     : QObject{parent}, instance(nullptr), context(nullptr)
{
    // 初始化 libpinyin 上下文
    context = pinyin_init(systemdir, nullptr);
    if (!context) {
        qWarning() << "Failed to initialize pinyin context";
    } else {
        // 创建 pinyin 实例
        instance = pinyin_alloc_instance(context);
        if (!instance) {
            qWarning() << "Failed to allocate pinyin instance";
            pinyin_fini(context);
        } else {
            // 设置全拼方案
            pinyin_set_full_pinyin_scheme(context, FULL_PINYIN_DEFAULT);
        }
    }
}

RuyiPinyinManager::~RuyiPinyinManager() {
    // // 清理资源
    if (context) {
        pinyin_fini(context);
        context = nullptr;
    }
    if (instance) {
        pinyin_free_instance(instance);
        instance = nullptr;
    }
}

/**
 * 判断字符是否为汉字
 * @param ch 要判断的字符
 * @return 如果是汉字返回 true，否则返回 false
 */
bool RuyiPinyinManager::isChineseChar(const QChar &ch)
{
    ushort unicode = ch.unicode();
    return (unicode >= 0x4E00 && unicode <= 0x9FFF) || // 基本汉字
           (unicode >= 0x3400 && unicode <= 0x4DBF) || // 扩展A
           (unicode >= 0x20000 && unicode <= 0x2A6DF) || // 扩展B
           (unicode >= 0x2A700 && unicode <= 0x2B73F) || // 扩展C
           (unicode >= 0x2B740 && unicode <= 0x2B81F) || // 扩展D
           (unicode >= 0x2B820 && unicode <= 0x2CEAF) || // 扩展E
           (unicode >= 0xF900 && unicode <= 0xFAFF) || // 兼容汉字
           (unicode >= 0x2F800 && unicode <= 0x2FA1F); // 兼容扩展
}

/**
 * 将汉字转换为拼音
 * @param text 要转换的文本，可以包含汉字和非汉字
 * @return 转换后的字符串，汉字部分转为拼音，非汉字部分保持不变
 */
QStringList RuyiPinyinManager::chineseToPinyinList(const QString &text)
{
    QStringList resultList;
    if (text.isEmpty() || !this->instance) {
        return resultList;
    }

    QString chineseSegment;


    // 分段处理汉字和非汉字部分
    for (int i = 0; i < text.length(); i++) {
        QChar tChar = text.at(i);
        bool isCh = isChineseChar(tChar);
        if (isCh) {
            chineseSegment += tChar;
            if (i == text.length() - 1) {
                QStringList tmpList = processChinese(instance, chineseSegment);
                resultList.append(tmpList);
            }
        } else {
            if (!chineseSegment.isEmpty()) {
                QStringList tmpList = processChinese(instance, chineseSegment);
                resultList.append(tmpList);
            }

            // 追加非汉字字符
            resultList.append(tChar);
        }
    }

    return resultList;
}

QStringList RuyiPinyinManager::processChinese(pinyin_instance_t *instance, QString &chineseSegment) {
    QStringList resultList;
    if (!instance || chineseSegment.isEmpty()) {
        return resultList;
    }

    QByteArray utf8Text = chineseSegment.toUtf8();
    bool segmentResult = pinyin_phrase_segment(instance, utf8Text.constData());

    if (segmentResult) {
        // 获取分词结果
        guint numPhrases = 0;
        pinyin_get_n_phrase(instance, &numPhrases);

        for (guint j = 0; j < numPhrases; ++j) {
            phrase_token_t token;
            if (!pinyin_get_phrase_token(instance, j, &token)) {
                continue;
            }

            // 获取词组的长度和文本
            guint phraseLen = 0;
            gchar *phraseStr = nullptr;
            if (!pinyin_token_get_phrase(instance, token, &phraseLen, &phraseStr)) {
                continue;
            }

            // 获取拼音数量
            guint numPronunciations = 0;
            if (!pinyin_token_get_n_pronunciation(instance, token, &numPronunciations)
                || numPronunciations == 0) {
                g_free(phraseStr);
                continue;
            }

            // 创建 ChewingKeyVector 存储拼音键
            ChewingKeyVector keys = g_array_new(FALSE, FALSE, sizeof(ChewingKey));

            // 获取第一个发音
            if (!pinyin_token_get_nth_pronunciation(instance, token, 0, keys)) {
                g_array_free(keys, TRUE);
                g_free(phraseStr);
                continue;
            }

            // 逐个获取拼音
            for (guint k = 0; k < keys->len; ++k) {
                ChewingKey *key = &g_array_index(keys, ChewingKey, k);
                gchar *pinyinStr = nullptr;

                if (pinyin_get_pinyin_string(instance, key, &pinyinStr)) {
                    resultList.append(QString::fromUtf8(pinyinStr));
                    g_free(pinyinStr);
                }
            }

            g_array_free(keys, TRUE);
            g_free(phraseStr);
        }

        // 重置中文段
        chineseSegment.clear();
        // 重置 instance 状态
        pinyin_reset(instance);
    } else {
        // 如果分词失败，直接保留原文本
        resultList.append(chineseSegment);
    }
    return resultList;
}

/**
 * 将汉字转换为拼音
 * @param text 要转换的汉字字符串
 * @return 拼音字符串，多个汉字的拼音用空格分隔
 */
QString RuyiPinyinManager::chineseToPinyin(const QString &text)
{
    // 初始化 libpinyin 上下文
    pinyin_context_t *context = pinyin_init(systemdir, nullptr);
    if (!context) {
        qWarning() << "Failed to initialize pinyin context";
        return QString();
    }

    // 创建 pinyin 实例
    pinyin_instance_t *instance = pinyin_alloc_instance(context);
    if (!instance) {
        qWarning() << "Failed to allocate pinyin instance";
        pinyin_fini(context);
        return QString();
    }

    // 设置全拼方案
    pinyin_set_full_pinyin_scheme(context, FULL_PINYIN_DEFAULT);

    // 对输入文本进行分词
    QByteArray utf8Text = text.toUtf8();
    bool segmentResult = pinyin_phrase_segment(instance, utf8Text.constData());
    if (!segmentResult) {
        qWarning() << "Failed to segment phrase";
        pinyin_free_instance(instance);
        pinyin_fini(context);
        return QString();
    }

    // 获取分词结果
    guint numPhrases = 0;
    pinyin_get_n_phrase(instance, &numPhrases);

    QStringList pinyinList;

    // 遍历每个词组
    for (guint i = 0; i < numPhrases; ++i) {
        phrase_token_t token;
        if (!pinyin_get_phrase_token(instance, i, &token)) {
            continue;
        }

        // 获取词组的长度和文本
        guint phraseLen = 0;
        gchar *phraseStr = nullptr;
        if (!pinyin_token_get_phrase(instance, token, &phraseLen, &phraseStr)) {
            continue;
        }

        // 获取拼音数量
        guint numPronunciations = 0;
        if (!pinyin_token_get_n_pronunciation(instance, token, &numPronunciations)
            || numPronunciations == 0) {
            g_free(phraseStr);
            continue;
        }

        // 创建 ChewingKeyVector 存储拼音键
        ChewingKeyVector keys = g_array_new(FALSE, FALSE, sizeof(ChewingKey));

        // 获取第一个发音
        if (!pinyin_token_get_nth_pronunciation(instance, token, 0, keys)) {
            g_array_free(keys, TRUE);
            g_free(phraseStr);
            continue;
        }

        // 逐个获取拼音
        for (guint j = 0; j < keys->len; ++j) {
            ChewingKey *key = &g_array_index(keys, ChewingKey, j);
            gchar *pinyinStr = nullptr;

            if (pinyin_get_pinyin_string(instance, key, &pinyinStr)) {
                pinyinList.append(QString::fromUtf8(pinyinStr));
                g_free(pinyinStr);
            }
        }

        g_array_free(keys, TRUE);
        g_free(phraseStr);
    }

    // 清理资源
    pinyin_free_instance(instance);
    pinyin_fini(context);

    // 返回以空格分隔的拼音字符串
    return pinyinList.join(" ");
}

/**
     * @brief pinyinToChinese
     * @param input
     */
void RuyiPinyinManager::pinyinToChinese(const std::string& input) {
    pinyin_context_t *ctx = pinyin_init(systemdir, userdir);
    if (!ctx) {
        MyTools::writeLog("Failed to initialize libpinyin.");
        return;
    }


    // 设置全拼方案（这里以默认的全拼方案为例）
    if (!pinyin_set_full_pinyin_scheme(ctx, FULL_PINYIN_DEFAULT)) {
        MyTools::writeLog("Failed to set full pinyin scheme.");
        pinyin_fini(ctx);
        return;
    }

    // 分配一个 pinyin 实例
    pinyin_instance_t* instance = pinyin_alloc_instance(ctx);
    if (!instance) {
        MyTools::writeLog("Failed to allocate pinyin instance.");
        pinyin_fini(ctx);
        return;
    }

    // 输入拼音字符串
    const char* pinyin_str = input.c_str();
    // 解析全拼
    size_t parsed_length = pinyin_parse_more_full_pinyins(instance, pinyin_str);

    if (parsed_length == 0) {
        MyTools::writeLog("Failed to parse pinyin.");
        pinyin_free_instance(instance);
        pinyin_fini(ctx);
        return;
    }

    // 猜测候选词
    if (!pinyin_guess_candidates(instance, 0, SORT_BY_PHRASE_LENGTH_AND_FREQUENCY)) {
        MyTools::writeLog("Failed to guess candidates.");
        pinyin_free_instance(instance);
        pinyin_fini(ctx);
        return;
    }

    // 获取候选词数量
    guint num_candidates;
    if (!pinyin_get_n_candidate(instance, &num_candidates)) {
        MyTools::writeLog("Failed to get number of candidates.");
        pinyin_free_instance(instance);
        pinyin_fini(ctx);
        return;
    }

    // 输出候选词
    for (guint i = 0; i < num_candidates; ++i) {
        lookup_candidate_t* candidate;
        if (pinyin_get_candidate(instance, i, &candidate)) {
            const gchar* candidate_str;
            if (pinyin_get_candidate_string(instance, candidate, &candidate_str)) {                
                qDebug() << i + 1 << ". " << candidate_str;
            }
        }
    }

    // 释放 pinyin 实例
    pinyin_free_instance(instance);
    // 结束 libpinyin 上下文
    pinyin_fini(ctx);

}

```