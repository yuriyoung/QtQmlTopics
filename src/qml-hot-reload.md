## QML热加载
通过c++端侦听qml文件实时刷新界面而无须重新运行应用程序。
### demo:
    ![image](/resources/qml-hot-reload.gif)

### code:
```c++
#ifndef QMLLISTENER_H
#define QMLLISTENER_H

class QmlListenerPrivate;
class QmlListener : public QObject
{
    Q_OBJECT
    Q_DISABLE_COPY(QmlListener)
    Q_DECLARE_PRIVATE(QmlListener)
    QScopedPointer<QmlListenerPrivate> const d_ptr;
public:
    QmlListener(QObject *parent = nullptr);
    explicit QmlListener(QQuickView *view, QObject *parent = nullptr);
    explicit QmlListener(QQmlApplicationEngine *engine, QObject *parent = nullptr);

signals:

public slots:
    void reload(const QString &file);
}
#endif // QMLLISTENER_H
```

```c++
class QmlListenerPrivate
{
    Q_DECLARE_PUBLIC(QmlListenerPrivate)
    QmlListener *q_ptr;
public:
    QmlListenerPrivate(QmlListener *parent) : q_ptr(parent) { }
}
```
(待续...)
