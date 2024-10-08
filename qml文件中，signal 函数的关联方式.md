# 关联方式

比如有个 qml 文件定义如下；

```js
// AppGridView.qml

Rectangle {
    id: appGridViewItem
    anchors.fill: parent

    signal gridHeightSignal(var height)
}

```

若要监听其中的信号函数 gridHeightSignal ，至少有 2 种方式，如下

## 方式 1

对象初始化后，直接实现：

```javascript
// AppGridView.qml

AppGridView {
    onGridHeightSignal: {
        conslog.log(height)
    }
}

```

## 方式 2

有时对象是通过 Loader 加载的，那可以通过如下方式

```js
  Loader {
                id: loader
                source: "./AppGridView.qml"

                onLoaded: {
                    if (loader.item) {
                        // 关联的一种方式
                        gridConnections.target = loader.item

                        // 也可以通过像下面这样关联单个 signal，这就需要单独新建一个函数，与信号绑定
                        // loader.item.gridHeightSignal.connect(onGridHeightChanged)
                    }
                }
            }
```

在 qml 中定义 Connectiongs，如下：

```js
            Connections {
                id: gridConnections
                target: null

                function onGridHeightSignal(height) {
                    conslog.log(height)
                }   
            }
```