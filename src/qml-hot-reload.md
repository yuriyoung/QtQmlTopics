## 简单的QML热加载
通过c++端侦听qml文件实时刷新界面而无须重新运行应用程序。
### demo:
![demo](https://github.com/yuriyoung/QtQmlTopics/blob/master/resources/qml-hot-reload.gif)

### code:
* QmlListener.h

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
    explicit QmlListener(const QString mainQml = QString(), QObject *parent = nullptr);
    ~QmlListener();
    
    void setIndexQml(const QString &qmlFile);
    
    void listen(QQuickView *view);
    void listen(QQmlApplicationEngine *engine);
    
    void addPaths(const QStringList &paths);
    void appendPath(const QString &path);
    
signals:

public slots:
    void reload(const QString &file);
}
#endif // QMLLISTENER_H
```

* QmlListener.cpp

```c++
class QmlListenerPrivate
{
    Q_DECLARE_PUBLIC(QmlListenerPrivate)
    QmlListener *q_ptr;
public:
    QmlListenerPrivate(QmlListener *parent);
    
    void watching();
    
    QQuickView *view;
    QQmlApplicationEngine *engine;
    QFileSystemWatcher *watcher;
    QList<QObject *> qmlObjects;
    QString mainQml;
}

QmlListenerPrivate::QmlListenerPrivate(QmlListener *parent)
    : q_ptr(parent)
    , watcher(new QFileSystemWatcher())
{
}

QmlListenerPrivate::watching()
{
    Q_Q(QmlListener);
    if(!engine && !view)
        return ;
    
    const QString basePath = engine ? engine->baseUrl().toString() : view->engine()->baseUrl().toString();
    watcher->addPath(basePath);
    watcher->addPath(mainQml);
    QObject::connect(m_watcher, &QFileSystemWatcher::fileChanged, q, &QmlListener::reload);
}

QmlListener::QmlListener(const QString mainQml, QObject *parent)
    : QObject(parent)
    , d_ptr(new QmlListenerPrivate(this))
{
    Q_D(QmlListener);
    d->mainQml = mainQml;
}

QmlListener::~QmlListener()
{
}

void QmlListener::setIndexQml(const QString &qmlFile)
{
    Q_D(QmlListener);
    if(qmlFile == d->mainQml)
        return;

    d->mainQml = qmlFile;
}

void QmlListener::listen(QQuickView *view)
{
    Q_D(QmlListener);
    d->view = view;
    d->watching();
}

void QmlListener::listen(QQmlApplicationEngine *engine)
{
    Q_D(QmlListener);
    d->engine = engine;
    d->watching();
}

void QmlListener::addPaths(const QStringList &paths)
{
    Q_D(QmlListener);
    d->watcher->addPaths(paths);
}

void QmlListener::appendPath(const QString &path)
{
    Q_D(QmlListener);
    d->watcher->addPath(path);
}

void QmlListener::reload(const QString &file)
{
    Q_D(QmlListener);
    if(d->view)
    {
        d->view->setSource(QUrl());
        d->view->engine()->clearComponentCache();
        d->view->setSource(QUrl(d->mainQml));
    }
    else if(d->engine)
    {
        foreach(QObject *obj, d->engine->rootObjects())
        {
            if(d->qmlObjects.contains(obj))
                continue;

            d->qmlObjects.append(obj);
            d->engine->setObjectOwnership(obj, QQmlApplicationEngine::CppOwnership);
            obj->deleteLater();
        }
        d->engine->clearComponentCache();
        d->engine->load(d->mainQml);
    }
}
```

### 使用
* main.cpp

```c++
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QDir>
#include <QDirIterator>

#include "QmlListener.h"

static const QString UI_PATH = "path/to/your/project/qml";

int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);
    QGuiApplication app(argc, argv);
    
    QQmlApplicationEngine engine;
    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));
    if (engine.rootObjects().isEmpty())
        return -1;

#ifdef QT_DEBUG
    // scan all qml files
    QDir dir(UI_PATH);
    dir.setFilter(QDir::Dirs | QDir::Files | QDir::NoDotAndDotDot);
    dir.setSorting(QDir::Name);

    QStringList files;
    QDirIterator it(dir, QDirIterator::Subdirectories);
    while (it.hasNext())
    {
        it.next();
        QFileInfo info = it.fileInfo();
        if(info.isFile())
        {
            files << info.filePath();
        }
    }
    
    QmlListener listtener;
    listtener.setIndexQml(UI_PATH + "/main.qml");
    listtener.addPaths(files);
    listtener.listen(&engine);
#endif

    return app.exec();
}
```
### 待续
