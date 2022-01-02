# proposal: expose `runOnMain`

<!-- for more information about completing this template please see
https://github.com/fyne-io/fyne/wiki/Contributing%3A-Proposals -->

## Background

I wrote a [hotkey](https://github.com/golang-design/hotkey) package for public use. However, there is a limitation on plugging that package into any other GUI framework, such as fyne.

The main limitation is that the hotkey registration APIs from OS (i.e. darwin) must be executed on the main thread. For individual usage, a self-occupied mainthread scheduling, such as [mainthread](https://github.com/golang-design/mainthread) package works nicely, but this will require to call `[NSApp run]` that runs the Cocoa's main event loop, which completely conflicts with the fyne's window boot process. Two of them are impossible to live together.

Until recently, @ventsislav-georgiev repots [a fyne-based application](https://github.com/ventsislav-georgiev/prosper) could not collaborate with golang-design/hotkey package unless forking it, see https://github.com/golang-design/hotkey/issues/7#issuecomment-1003099355.

The mentioned reason fundamentally limits potential third-party authors to extend some unimplemented OS features that require to be called on the main thread.

## API Design

Calling on the main thread functionality is implicitly exposed via
type assertion. For drivers that supports running on the main thread,
developers can assert if an `fyne.App` implements `CallOnMainThread(func())` method. For instance,

```go
ap, ok := fyne.CurrentApp().(interface{ CallOnMainThread(func()) })
if !ok {
    panic("fyne: current driver does not support call on the main thread")
}

ap.CallOnMainThread(func() {
    println("this function is called on the main thread")
})
```

## Implementation

```diff
commit 9dcc94ba480cc02f54fddb234a9492efc4234fa7
Author: Changkun Ou <hi@changkun.de>
Date:   Sun Jan 2 11:04:09 2022 +0100

    app: expose CallOnMainThread implicitly

diff --git a/app/app.go b/app/app.go
index 50f241db..ea21f123 100644
--- a/app/app.go
+++ b/app/app.go
@@ -97,6 +97,15 @@ func (a *fyneApp) Lifecycle() fyne.Lifecycle {
 	return a.lifecycle
 }

+func (a *fyneApp) CallOnMainThread(f func()) {
+	if ap, ok := a.driver.(interface{ CallOnMainThread(func()) }); ok {
+		ap.CallOnMainThread(f)
+		return
+	}
+
+	panic("fyne: CallOnMain is not supported in the current driver.")
+}
+
 // New returns a new application instance with the default driver and no unique ID
 func New() fyne.App {
 	internal.LogHint("Applications should be created with a unique ID using app.NewWithID()")
diff --git a/internal/driver/glfw/driver.go b/internal/driver/glfw/driver.go
index 8e1cbaf4..f48c867b 100644
--- a/internal/driver/glfw/driver.go
+++ b/internal/driver/glfw/driver.go
@@ -132,6 +132,17 @@ func (d *gLDriver) initFailed(msg string, err error) {
 	}
 }

+// CallOnMainThread schedules the given function f to be executed on the main
+// thread. This method blocks until the called function is returned.
+//
+// This method is implicitly exposed to the public. For developers who would
+// like to call a function on the main thread, one can assert if the current
+// fyne.App have the support. For example,
+//
+// 	if ap, ok := fyne.CurrentApp().(interface{ CallOnMainThread(func()) }); ok {
+// 		ap.CallOnMainThread(func() { println("calls on the main thread")})
+// 	}
+func (d *gLDriver) CallOnMainThread(f func()) { runOnMain(f) }
+
 func goroutineID() int {
 	b := make([]byte, 64)
 	b = b[:runtime.Stack(b, false)]
```

## A Minimum Example

See an example at https://github.com/changkun/demos/blob/main/hotkeys/fyneapp/main.go.