# منصة macOS

## النظرة عامة

إلكترون لديه واجهة برمجة التطبيقات لتهيئة أيقونة التطبيق في ماكوس. A macOS-only API exists to create a custom dock menu, but Electron also uses the app dock icon as the entry point for cross-platform features like [recent documents][recent-documents] and [application progress][progress-bar].

يتم استخدام قاعدة الإرساء المخصصة عادة لإضافة الاختصارات إلى المهام التي لن يتمكن المستخدم من فتح نافذة التطبيق بأكملها.

__قائمة الإرساء من Terminal.app:__

![قائمة الإرساء][3]

لتعيين قائمة الإرساء المخصصة الخاصة بك، تحتاج إلى استخدام [`app.dock.setMenu`](../api/dock.md#docksetmenumenu-macos) API المتاح فقط على macOS.

## مثال

بدءاً بتطبيق عمل من [دليل البداية السريعة](quick-start.md)، قم بتحديث ملف `main.js` مع الأسطر التالية:

```javascript
const { app, Menu } = require('electron')

const dockMenu = Menu.buildFromTemplate([
  {
    label: 'New Window',
    click () { console.log('New Window') }
  }, {
    label: 'New Window with Settings',
    submenu: [
      { label: 'Basic' },
      { label: 'Pro' }
    ]
  },
  { label: 'New Command...' }
])

app.whenReady().then(() => {
  app.dock.setMenu(dockMenu)
})
```

بعد بدء تطبيق إلكترون، انقر بزر الماوس الأيمن على رمز التطبيق. يجب أن ترى القائمة المخصصة التي عرفتها للتو:

![قائمة الإرساء لـ macOS](../images/macos-dock-menu.png)

[3]: https://cloud.githubusercontent.com/assets/639601/5069962/6032658a-6e9c-11e4-9953-aa84006bdfff.png
[recent-documents]: ./recent-documents.md
[progress-bar]: ./progress-bar.md