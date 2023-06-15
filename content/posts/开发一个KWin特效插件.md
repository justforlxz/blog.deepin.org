---
title: "开发一个 KWin 特效插件"
date: 2023-06-15T20:58:45+08:00
draft: false
tags: ['KWin']
authors: ["justforlxz"]
---

## 概要

KWin 是 KDE 开发的窗口管理器，提供了非常丰富的插件，可以对功能进行大量的定制。

本篇文章是对窗口特效插件的开发介绍。

## 插件开发

### 插件定义

KWin 的插件通常可以使用一些宏辅助生成代码，例如使用 `KPluginFactory` 进行插件的定义，内容是用来生成插件的入口类。

```cpp
#define EffectPluginFactory_iid "org.kde.kwin.EffectPluginFactory" KWIN_PLUGIN_VERSION_STRING
#define KWIN_PLUGIN_FACTORY_NAME KPLUGINFACTORY_PLUGIN_CLASS_INTERNAL_NAME
#define KWIN_EFFECT_FACTORY_SUPPORTED_ENABLED(className, jsonFile, supported, enabled ) \
    class KWIN_PLUGIN_FACTORY_NAME : public KWin::EffectPluginFactory \
    { \
        Q_OBJECT \
        Q_PLUGIN_METADATA(IID EffectPluginFactory_iid FILE jsonFile) \
        Q_INTERFACES(KPluginFactory) \
    public: \
        explicit KWIN_PLUGIN_FACTORY_NAME() {} \
        ~KWIN_PLUGIN_FACTORY_NAME() {} \
        bool isSupported() const override { \
            supported \
        } \
        bool enabledByDefault() const override { \
            enabled \
        } \
        KWin::Effect *createEffect() const override { \
            return new className(); \
        } \
    };

#define KWIN_EFFECT_FACTORY_ENABLED(className, jsonFile, enabled ) \
    KWIN_EFFECT_FACTORY_SUPPORTED_ENABLED(className, jsonFile, return true;, enabled )

#define KWIN_EFFECT_FACTORY_SUPPORTED(className, jsonFile, supported ) \
    KWIN_EFFECT_FACTORY_SUPPORTED_ENABLED(className, jsonFile, supported, return true; )

#define KWIN_EFFECT_FACTORY(className, jsonFile ) \
    KWIN_EFFECT_FACTORY_SUPPORTED_ENABLED(className, jsonFile, return true;, return true; )
```

大部分宏只是为了方便结构修改，我们只需要使用 `K_PLUGIN_FACTORY` 进行插件定义即可。

假设我们开发了一个插件，名字叫 demo，我们只需要在 main.cpp 中使用 `KWIN_EFFECT_FACTORY_SUPPORTED` 定义

```cpp
KWIN_EFFECT_FACTORY_SUPPORTED(
    Demo,
    "metadata.json",
    return true;
)
```

代码展开后是这样的。

```cpp
class KPLUGINFACTORY_PLUGIN_CLASS_INTERNAL_NAME : public KWin::EffectPluginFactory \
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID EffectPluginFactory_iid FILE "metadata.json")
    Q_INTERFACES(KPluginFactory)
public: \
    explicit KPLUGINFACTORY_PLUGIN_CLASS_INTERNAL_NAME() {}
    ~KPLUGINFACTORY_PLUGIN_CLASS_INTERNAL_NAME() {}
    bool isSupported() const override {
        return true;
    }
    bool enabledByDefault() const override {
        return true;
    }
    KWin::Effect *createEffect() const override {
        return new Demo();
    }
};
```

可以看到，其实 `KWIN_EFFECT_FACTORY_SUPPORTED` 只是为我们生成了工厂函数，辅助生成了一些必要的重载。

`metadata.json` 文件是用来作为插件的描述信息使用的。

```json
{
    "KPlugin": {
        "Category": "Accessibility",
        "Description": "Allow clip of window content",
        "EnabledByDefault": true,
        "Id": "scissor",
        "License": "GPL",
        "Name": "ScissorWindow",
        "Name[zh_CN]": "窗口圆角"
    },
    "org.kde.kwin.effect": {
        "enabledByDefaultMethod": true
    }
}
```

### 特效插件

特效插件是一类可以改变窗口画面的插件，例如我们可以在插件里对窗口进行贴图、变形和裁切，在 DDE 中，就使用特效插件完成了圆角裁切和窗口模糊。

这里使用圆角裁切插件作为例子，首先使用 `KWIN_EFFECT_FACTORY_SUPPORTED` 宏对插件进行定义， `KWIN_EFFECT_FACTORY_SUPPORTED` 接受一个 class 作为返回的接口类，它需要继承自 `Effect`，第二个参数是元信息的 json 文件，第三个参数是返回是否支持，在启用插件时可对当前环境进行判断，例如插件需要使用 `OpenGL` 对图形进行一些操作，但是当前环境不支持 `OpenGL`，那么插件就不会启用。

```json
#include "scissorwindow.h"

namespace KWin
{

KWIN_EFFECT_FACTORY_SUPPORTED(ScissorWindow,
                              "metadata.json.stripped",
                              return ScissorWindow::supported();)

} // namespace KWin

#include "main.moc"
```

在 Effect 类中有几个不同阶段的方法可以重载。

- prePaintScreen
    - 设置是否变换窗口或整个屏幕
    - 更改将要绘制的屏幕区域
    - 做各种内务处理任务，比如初始化你的效果变量
    用于即将到来的绘画过程或更新动画的进度
- paintScreen
    - 在窗口上画东西（调用后画 effect->paintScreen())
    - 绘制多个桌面和/或同一桌面的多个副本
- postPaintScreen
    - 在动画的情况下安排下一次重绘，不应该在这里画任何东西。
- prePaintWindow
    - 启用或禁用窗口的绘制（例如启用最小化窗口的绘制）
    - 将窗口设置为半透明
    - 设置要转换的窗口
    - 请求将窗口分成多个部分
- paintWindow
    - 做各种转换
    - 改变窗口的不透明度
    - 改变亮度和/或饱和度，如果支持的话
- postPaintWindow
    - 在动画的情况下为单个窗口安排下一次重绘
    不应该在这里画任何东西。
- paintEffectFrame
    - 在绘制 EffectFrame 之前直接调用此方法。
    - 如果需要绑定shader或者执行，可以实现这个方法帧渲染前的其他操作。
- drawWindow
    - 可以调用以绘制一个窗口的多个副本（例如缩略图）。
    - 可以在这里改变窗口的不透明度/亮度/等，但不能做任何转换。
    - 在基于 OpenGL 的合成中，框架确保上下文是最新的

在方法名称中可以看出，在场景及窗口绘制的过程中，分别可以在实际绘制的前后分别执行一些动作，圆角插件就是在 `drawWindow` 函数中，使用 `OpenGL` 对窗口使用着色器进行窗口裁切，并绘制到屏幕上。

```cpp
void ScissorWindow::drawWindow(EffectWindow *w, int mask, const QRegion& region, WindowPaintData &data) {
    if (w->isDesktop() || isMaximized(w)) {
        return effects->drawWindow(w, mask, region, data);
	  }

    QPointF cornerRadius;
    const QVariant valueRadius = w->data(WindowRadiusRole);
    if (valueRadius.isValid()) {
        cornerRadius = w->data(WindowRadiusRole).toPointF();
        const qreal xMin{ std::min(cornerRadius.x(), w->width() / 2.0) };
        const qreal yMin{ std::min(cornerRadius.y(), w->height() / 2.0) };
        const qreal minRadius{ std::min(xMin, yMin) };
        cornerRadius = QPointF(minRadius, minRadius);
    }

    if (cornerRadius.x() < 2 && cornerRadius.y() < 2) {
        return effects->drawWindow(w, mask, region, data);
    }

    const QString& key = QString("%1+%2").arg(cornerRadius.toPoint().x()).arg(cornerRadius.toPoint().y()
    );
    if (!m_texMaskMap.count(key)) {
				QImage img(QSize(radius.x() * 2, radius.y() * 2), QImage::Format_RGBA8888);
		    img.fill(QColor(0, 0, 0, 0));
		    QPainter painter(&img);
		    painter.setPen(Qt::NoPen);
		    painter.setBrush(QColor(255, 255, 255, 255));
		    painter.setRenderHint(QPainter::Antialiasing);
		    painter.drawEllipse(0, 0, radius.x() * 2, radius.y() * 2);
		    painter.end();
		
		    m_texMaskMap[key] = new GLTexture(img.copy(0, 0, radius.x(), radius.y()));
		    m_texMaskMap[key]->setFilter(GL_LINEAR);
		    m_texMaskMap[key]->setWrapMode(GL_CLAMP_TO_EDGE);
    }

    ShaderManager::instance()->pushShader(m_filletOptimizeShader);
    m_filletOptimizeShader->setUniform("typ1", 1);
    m_filletOptimizeShader->setUniform("sampler", 0);
    m_filletOptimizeShader->setUniform("msk1", 1);
    m_filletOptimizeShader->setUniform("k", QVector2D(w->width() / cornerRadius.x(), w->height() / cornerRadius.y()));
    if (w->hasDecoration()) {
        m_filletOptimizeShader->setUniform("typ2", 0);
    } else {
        m_filletOptimizeShader->setUniform("typ2", 1);
    }
    auto old_shader = data.shader;
    data.shader = m_filletOptimizeShader;

    glActiveTexture(GL_TEXTURE1);
    m_texMaskMap[key]->bind();
    glActiveTexture(GL_TEXTURE0);
    effects->drawWindow(w, mask, region, data);
    ShaderManager::instance()->popShader();
    data.shader = old_shader;
    glActiveTexture(GL_TEXTURE1);
    m_texMaskMap[key]->unbind();
    glActiveTexture(GL_TEXTURE0);
    return;
}
```

如果窗口是桌面类型，或者已经最大化了，则无需处理，直接返回 Effect 原本的处理函数。

之后尝试从窗口属性中取出圆角大小的值，如果没有设置圆角大小，或者值小于2，则无需处理。

尝试查询缓存，在这里为窗口的四个角构建一份遮罩对象并缓存，使用 `OpenGL` 将遮罩和着色器进行关联，激活两个材质分别绘制窗口内容和四个角的遮罩，在着色器中完成窗口圆角的半透明效果。

限于篇幅，本文不展开介绍如何实现圆角插件的全部实现过程，仅挑选关键步骤。

## 安装和调试

### 安装

将动态库复制到 `/usr/share/kwin/effects/plugins` ，并使用 `DBus` 激活插件。

```bash
qdbus --literal org.kde.KWin /Effects org.kde.kwin.Effects.loadEffect scissor
```

### 调试

使用 gdb attach 到 deepin-kwin 的进程，并对 drawWindow 断点进行单步调试。
