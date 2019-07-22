### 图像区域选择并裁剪
接上篇[可缩放边的Rectagle用于裁剪图片](./resizable-borders-of-Rectangle-to-selection-rect-for-crop-image.md),实现从已选择的区域截取图像。
- demo:

![demo](/resources/crop-image.gif)

1. 接上篇[可缩放边的Rectagle用于裁剪图片](./resizable-borders-of-Rectangle-to-selection-rect-for-crop-image.md)已经实现的区域选择做成独立的组件: `SelectionArea.qml`
```qml
// SelectionArea.qml
import QtQuick 2.12

Rectangle {
    id: selectionRect

    property bool squared: false
    property int rulerSize: 12
    property color rulerColor: "steelblue"
    property int minimumSize: 50
    property int maximumSize: Math.min(parent.width, parent.height)

    color: "#354682B4"
    border {
        width: 2
        color: rulerColor
    }

    MouseArea {
        id: dragMouseArea
        anchors.fill: parent
        drag{
            target: parent
            minimumX: 0
            minimumY: 0
            maximumX: parent.parent.width - parent.width
            maximumY: parent.parent.height - parent.height
            smoothed: true
        }

        onDoubleClicked: {
            parent.destroy() // destroy component
        }
    }

    Repeater {
        id: repeater

        //TODO: optimize this code
        readonly property var actions: {
            "left" : function(x, y) {
                selectionRect.width = selectionRect.width - x
                selectionRect.x = selectionRect.x + x
                if(selectionRect.width < minimumSize)
                    selectionRect.width = minimumSize
                if(selectionRect.width > maximumSize)
                    selectionRect.width = maximumSize

                if(squared) {
                    selectionRect.height = selectionRect.width
                }
            },
            "right" : function(x, y) {
                selectionRect.width = selectionRect.width + x
                if(selectionRect.width < minimumSize)
                    selectionRect.width = minimumSize
                if(selectionRect.width > maximumSize)
                    selectionRect.width = maximumSize

                if(squared) {
                    selectionRect.height = selectionRect.width
                }
            },
            "top" : function(x, y) {
                selectionRect.height = selectionRect.height - y
                selectionRect.y = selectionRect.y + y
                if(selectionRect.height < minimumSize)
                    selectionRect.height = minimumSize
                if(selectionRect.height > maximumSize)
                    selectionRect.height = maximumSize

                if(squared) {
                    selectionRect.width = selectionRect.height
                }
            },
            "bottom" : function(x, y) {
                selectionRect.height = selectionRect.height + y
                if(selectionRect.height < minimumSize)
                    selectionRect.height = minimumSize
                if(selectionRect.height > maximumSize)
                    selectionRect.height = maximumSize

                if(squared) {
                    selectionRect.width = selectionRect.height
                }
            },
            "lt" : function(x, y) {
                repeater.actions["left"](x, y);
                if(!squared) {
                    repeater.actions["top"](x, y);
                }
            },
            "lb" : function(x, y) {
                repeater.actions["left"](x, y);
                if(!squared) {
                    repeater.actions["bottom"](x, y);
                }
            },
            "rt" : function(x, y) {
                repeater.actions["right"](x, y);
                if(!squared) {
                    repeater.actions["top"](x, y);
                }
            },
            "rb" : function(x, y) {
                repeater.actions["right"](x, y);
                if(!squared) {
                    repeater.actions["bottom"](x, y);
                }
            },
        }

        model: [
            // edge rulers
            { horizontal: parent.left, vertical: parent.verticalCenter, axis: Drag.XAxis, callback: "left" },
            { horizontal: parent.right, vertical: parent.verticalCenter, axis: Drag.XAxis, callback: "right" },
            { horizontal: parent.horizontalCenter, vertical: parent.top, axis: Drag.YAxis, callback: "top" },
            { horizontal: parent.horizontalCenter, vertical: parent.bottom, axis: Drag.YAxis, callback: "bottom" },
            // corner rulers
            { horizontal: parent.left, vertical: parent.top, axis: Drag.YAxis | Drag.XAxis, callback: "lt" },
            { horizontal: parent.left, vertical: parent.bottom, axis: Drag.YAxis | Drag.XAxis, callback: "lb" },
            { horizontal: parent.right, vertical: parent.top, axis: Drag.YAxis | Drag.XAxis, callback: "rt" },
            { horizontal: parent.right, vertical: parent.bottom, axis: Drag.YAxis | Drag.XAxis, callback: "rb" },
        ]
        delegate: Rectangle {
            width: rulerSize
            height: rulerSize
            radius: rulerSize
            color: rulerColor
            anchors.horizontalCenter: modelData.horizontal
            anchors.verticalCenter: modelData.vertical

            MouseArea {
                anchors.fill: parent
                drag{ target: parent; axis: modelData.axis }
                onPositionChanged: {
                    if(drag.active) {
                        if(typeof repeater.actions[modelData.callback] === "function")
                            repeater.actions[modelData.callback](mouseX, mouseY)
                    }
                }
            }
        }
    }// Repeater
}
```

2. 建立一个裁剪工作的对话框组件`ImageCropDialog.qml`。对话框上放置一个用于指定大小的Rectangle，作为操作图片的工作viewport（视口），如显示，缩放，旋转等。Image组件的填充模式(`fillMode)`应设置为`Image.Pad`以填充整个Image组件，让图片完全显示，并且方便计算裁剪的坐标映射到源图片。
```qml
// ImageCropDialog.qml
import QtQuick 2.12
import QtQuick.Controls 2.12

Dialog {
    id: control

    property rect selectedRect
    property alias source: imageItem.source
    property var selection: undefined

    function createSelection()
    {
        if(!selection) {
            // centre in parent
            var ajustWidth = Math.min(parent.width, parent.height) / 2;
            var properties = {
                "width": ajustWidth, "height": ajustWidth,
                "x": (viewport.width - ajustWidth) / 2, "y": (viewport.height - ajustWidth) / 2
            }
            selection = selectionComponent.createObject(viewport, properties)
        }
    }

    title: qsTr("Crop Image Dialog")
    modal: true
    padding: 0
    standardButtons: Dialog.Apply | Dialog.Ok | Dialog.Cancel
    closePolicy: Popup.CloseOnEscape
    onApplied: {
        // mapping parent coordinate to current coordinate
        selectedRect = imageItem.mapFromItem(viewport, selection.x, selection.y,selection.width, selection.height);
    }

    Component {
        id: selectionComponent
        SelectionArea {
            id: selectionRect
            squared: true
        }
    }

    // iamge viewport
    Rectangle {
        id: viewport
        anchors.fill: parent
        color: "black"

        Image {
            id: imageItem

            // zoom in or zoom out
            scale: 1 / (sourceSize.height / parent.height)
            fillMode: Image.Pad
            anchors.centerIn: parent
            onStatusChanged: {
                if(status === Image.Ready) {
                    createSelection();
                }
            }
            onSourceChanged: {
                // reset selection area
                if(selection) {
                    selection.destroy();
                    selection = undefined;
                }
            }

            MouseArea {
                anchors.fill: parent
                onClicked: {
                    if(imageItem.status === Image.Ready) {
                        createSelection();
                    }
                }
            }
        } // Image item
    } // viewport
}
```
3. 建立一个c++图片帮助类ImageHelper，增加一个裁剪函数`crop`，非常简单，只传入一个图片文件名和一个裁剪的大小作为参数。并使用qmlRegisterSingletonType`注册单例对象到QML。
```c++
// ImageHelper.h
#ifndef IMAGEHELPER_H
#define IMAGEHELPER_H

#include <QObject>
#include <QCryptographicHash>
#include <QImage>
#include <QDateTime>
#include <QDir>
#include <QUrl>

class ImageHelper : public QObject
{
    Q_OBJECT
public:
    // ...

    // return cropped image filename, return empty string if failed
    Q_INVOKABLE QUrl crop(const QUrl &url, const QRect &rect)
    {
        QUrl resultUrl;
        if(url.isEmpty())
            return resultUrl;
        QString fileName = url.isLocalFile() ? url.toLocalFile() : url.toString();

        QImage original(fileName);
        if(original.isNull())
            return resultUrl;

        // do crop image
        QImage cropped = original.copy(rect);

        // using the current timestamps as the filename
        qint64 ms = QDateTime::currentMSecsSinceEpoch();
        QByteArray md5 = QCryptographicHash::hash(QByteArray().setNum(ms), QCryptographicHash::Md5);

        // save to temp directory
        QDir tempDir = QDir::current();
        if(!tempDir.cd(QStringLiteral("temp")))
        {
            tempDir.mkdir(QStringLiteral("temp"));
            tempDir.cd(QStringLiteral("temp"));
        }
        // not suffix
        QString name = tempDir.path() + "/" + QString(md5.toHex());
        cropped.save(name, "PNG");
        resultUrl = QUrl::fromLocalFile(name);

        return resultUrl;
    }

    // other member functions
    // ...
};

#endif // IMAGEHELPER_H

```

注册单例对象到QML的代码:
```c++
qmlRegisterSingletonType<ImageHelper>("App", 1, 0, "ImageHelper",
                                    [](QQmlEngine *, QJSEngine *) -> QObject* {
    return new ImageHelper;
});
```
注册的模块名在本例中为App，版本号1.0。注册的对象名`ImageHelper`，在QML使用`ImageHelper`访问该对象使用Q_INVOKABL修饰的成员函数或public slot修饰的槽。

4. 在main.qml组件中，组装必要的组件来完成整个程序的功能：从磁盘中打开图片，显示到UI，选择裁剪区域，应用裁剪，显示裁剪结果。
```qml
// main.qml
import QtQuick 2.12
import QtQuick.Controls 2.12
import QtQuick.Dialogs 1.3
import App 1.0

ApplicationWindow {
    id: appWindow
    title: qsTr("Window")
    width: 640
    height: 480
    visible: true

    property var selection: undefined

    // select a image from disk
    FileDialog {
        id: appImagesSelectDialog

        modality: Qt.WindowModal
        title: qsTr(""Choose a image"")
        selectMultiple: false
        selectFolder: false
        nameFilters: [ "Image files (*.jpeg *.png *.jpg *.gif)", "All files (*)" ]
        selectedNameFilter: nameFilters[0]
        sidebarVisible: true
        onAccepted: {
            // open image crop dialog
            cropDialg.open();
            cropDialg.source = fileUrls[0];
        }
    }

    ImageCropDialog {
        id: cropDialg
        width: parent.width
        height: parent.height
        onApplied: {
            var croppedFile = ImageHelper.crop(source, selectedRect);
            // apply crop result
            source = Qt.resolvedUrl(croppedFile);
            image.source = croppedFile;
        }
        onAccepted: {
            // show result
            image.source = cropDialg.source;
        }
    }

    Column {
        anchors.centerIn: parent
        // show the result image
        Rectangle {
            width: 256; height: 256
            border.color: "blue"; border.width: 1
            Image { id: image; anchors.fill: parent }
        }
        Button {
            text: qsTr("Choose a image")
            onClicked: appImagesSelectDialog.open();
        }
    }
}
```

5.最后，在main.cpp中启动应用程序,在加载main.qml之前注册`ImageHelper`类型到QML,本例子中注册为单例，当然也可以注册为QML组件类型。
>     qmlRegisterType<ImageHelper>("App", 1, 0, "ImageHelper");

```c++
// main.cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>

// include crop helper
#include "ImageHelper.h"

int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);
    QGuiApplication app(argc, argv);

    // register image crop helper
    qmlRegisterSingletonType<ImageHelper>("App", 1, 0, "ImageHelper",
                                          [](QQmlEngine *, QJSEngine *) -> QObject* {
        return new ImageHelper;
    });

    QQmlApplicationEngine engine;
    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));
    if (engine.rootObjects().isEmpty())
        return -1;

    return app.exec();
}

```

### 结束
- 待续...
