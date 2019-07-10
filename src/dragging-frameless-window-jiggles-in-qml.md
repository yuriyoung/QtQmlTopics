## 在QML中拖动无边框窗口时“抖动”的普通实现
通常拖动的实现方法如下，会导致在某些系统上窗口“抖动”：
```QML
    ApplicationWindow {
    id: mainWindow
    visible: true
    width: 640
    height: 480
    title: qsTr("Window")
    flags: Qt.FramelessWindowHint
    header: ToolBar {
        MouseArea{
            anchors.fill: parent
            property variant pressedPos: "0,0"

            onPressed: {
                pressedPos  = Qt.point(mouse.x, mouse.y)
            }

            onPositionChanged: {
                    var delta = Qt.point(mouse.x - pressedPos.x, mouse.y - pressedPos.y);
                    mainWindow.x += delta.x;
                    mainWindow.y += delta.y;
            }
        }
    }
}
```

## 解决方法1：
使用c++实现一个鼠标位置的帮助类，并将其对象暴露给QML上下文属性，来解决这个问题。这个方法在原先“抖动”的系统上，使得复杂的QML UI也能表现良好：
1. 一个c++帮助类的实现：
```cpp
class CursorPosProvider : public QObject
{
    Q_OBJECT
public:
    explicit CursorPosProvider(QObject *parent = nullptr) : QObject(parent)
    {
    }
    virtual ~CursorPosProvider() = default;

    Q_INVOKABLE QPointF cursorPos()
    {
        return QCursor::pos();
    }
};
```
非常简单的一个类，只从c++端提供一个鼠标位置。这有些奇怪，在QML中使用相同的操作时，在一些系统上会遇到“抖动”问题。

2. 将这个类注册为context属性暴露给QML：
```cpp
// ...
#include "CursorPosProvider.h"

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);

    CursorPosProvider mousePosProvider; 
    QQmlApplicationEngine engine;
    engine.rootContext()->setContextProperty("mousePosition", &mousePosProvider);
    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));

    return app.exec();
}
```
3. 给无边框窗口实现一个Qt Quick标题栏组件`TitleBar.qml`：
```qml
Rectangle {
    id: control
    width: parent.width
    color: "#2196F3" // 随便给的一个颜色

    property QtObject container
    // TODO: 一些扩展属性或信号
    // ...

    MouseArea {
        id: titleBarMouseRegion
        property var pressedPos
        anchors.fill: parent
        onPressed: {
            pressedPos = { x: mouse.x, y: mouse.y }
        }
        onPositionChanged: {
            container.x = mousePosition.cursorPos().x - pressedPos.x
            container.y = mousePosition.cursorPos().y - pressedPos.y
        }
    }
}
```
4.现在可以在Window组件下使用这个TitleBar组件了：
```qml
Window {
    id: root
    visible: true
    flags: Qt.FramelessWindowHint

    TitleBar {
        height: 20
        container: root
    }

    Text {
        text: qsTr("Hello World")
        anchors.centerIn: parent
    }
}
```

## 解决方法2：
使用系统API（待续...）
