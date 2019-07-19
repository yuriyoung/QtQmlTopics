## 可自由缩放Rectangle的边，拖拽选择裁剪的图片范围。
- demo:

![demo](/resources/resizable-borders-rectangle.gif)

```qml
import QtQuick 2.12
import QtQuick.Controls 2.12

ApplicationWindow {
    id: appWindow
    title: qsTr("Window")
    width: 640
    height: 480
    visible: true

    property var selection: undefined

    Image {
        id: image
        anchors.fill: parent
        source: "file:///demo.jpg" // change a valid url

        MouseArea {
            anchors.fill: parent
            onClicked: {
                if(!selection) {
                    var properties = {
                        "x": parent.width / 4, "y": parent.height / 4,
                        "width": parent.width / 2, "height": parent.width / 2
                    }
                    selection = selectionComponent.createObject(parent, properties)

                }
            }
        }
    }

    Component {
        id: selectionComponent

        Rectangle {
            id: selectionRect

            property bool squared: true
            property int rulerSize: 12
            property color rulerColor: "steelblue"

            border {
                width: 2
                color: rulerColor
            }
            color: "#354682B4"

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

                readonly property var actions: {
                    "left" : function(x, y) {
                        selectionRect.width = selectionRect.width - x
                        selectionRect.x = selectionRect.x + x
                        if(selectionRect.width < 50)
                            selectionRect.width = 50

                        if(squared) {
                            selectionRect.height = selectionRect.width
                        }
                    },
                    "right" : function(x, y) {
                        selectionRect.width = selectionRect.width + x
                        if(selectionRect.width < 50)
                            selectionRect.width = 50

                        if(squared) {
                            selectionRect.height = selectionRect.width
                        }
                    },
                    "top" : function(x, y) {
                        selectionRect.height = selectionRect.height - y
                        selectionRect.y = selectionRect.y + y
                        if(selectionRect.height < 50)
                            selectionRect.height = 50

                        if(squared) {
                            selectionRect.width = selectionRect.height
                        }
                    },
                    "bottom" : function(x, y) {
                        selectionRect.height = selectionRect.height + y
                        if(selectionRect.height < 50)
                            selectionRect.height = 50

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
                        onMouseXChanged: {
                            if(drag.active) {
                                if(typeof repeater. actions[modelData.callback] === "function")
                                    repeater.actions[modelData.callback](mouseX, mouseY)
                            }
                        }
                    }
                }
            }// Repeater
        } // Selected Rectangle
    }
}
```
