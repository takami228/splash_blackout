# splachblackout

https://github.com/flutter/flutter/issues/39494 の対処を検証するサンプル。

## 事象

Androidでスプラッシュ表示からrunAppの間にdelayを挟むと黒画面が表示される。

v1.16.3で修正がマージされた模様。 https://github.com/flutter/flutter/commit/9f4e5ad9c3a63a7e50f358b5cbf233872e9e077c

## 検証端末

- Android
  - Emulator(Android SDK 29)
  - HTC U11 life(Android 8.0)
- iOS
  - Emulator(iOS 13.3)
  - iPhone 6s(iOS 13.3)

flavorはdebugビルドにて検証する。

## 原因

1.9.1から1.12.13に上げた以降に発生している。

WidgetsFlutterBinding.ensureInitialized の処理を入れないと1.9.1では発生しない疑惑。

## 切り分け

- ○ : 黒画面が表示されない
- × : 黒画面が表示される

#### WidgetsFlutterBinding.ensureInitialized なし

| flutterバージョン | Android　Emulator | Android実機 |  iOSエミュレータ |  iOS実機 |
| ---- | ---- | ---- | ---- | ---- |
| v1.9.1+hotfix.6 | ○ | ○ | ○ | ○ |
| v1.12.13+hotfix.8 | ○ | × | ○ | ○　|
| v1.16.3 | ○ | × | ○　|  ○　|

#### WidgetsFlutterBinding.ensureInitialized あり

| flutterバージョン | Android　Emulator | Android実機 |  iOSエミュレータ |  iOS実機 |
| ---- | ---- | ---- | ---- | ---- |
| v1.9.1+hotfix.6 | × | × | × | × |
| v1.12.13+hotfix.8 | × | × | × | ○　|
| v1.16.3 | ○ | × | ○　|  ○　|

WidgetsFlutterBinding.ensureInitialized は必要なので`あり`の状態での対処を考える。

## 対処その1

`v1.12.13+hotfix.8`にバージョンを固定して、Issueで紹介されていた以下のwork aroundをやってみる。

- https://github.com/flutter/flutter/issues/39494#issuecomment-565627914
- https://github.com/flutter/flutter/issues/39494#issuecomment-596089595

```
diff --git a/packages/flutter/lib/src/rendering/binding.dart b/packages/flutter/lib/src/rendering/binding.dart
index adf10377f..f19cb44cc 100644
--- a/packages/flutter/lib/src/rendering/binding.dart
+++ b/packages/flutter/lib/src/rendering/binding.dart
@@ -42,10 +42,13 @@ mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, Gesture
     initRenderView();
     _handleSemanticsEnabledChanged();
     assert(renderView != null);
-    addPersistentFrameCallback(_handlePersistentFrameCallback);
     initMouseTracker();
   }

+  void initPersistentFrameCallback(){
+    addPersistentFrameCallback(_handlePersistentFrameCallback);
+  }
+
   /// The current [RendererBinding], if one has been created.
   static RendererBinding get instance => _instance;
   static RendererBinding _instance;
```

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  Future.delayed(Duration(seconds: 10), () {
    RendererBinding.instance.initPersistentFrameCallback();
    runApp(MyApp());
  });
}
```

### 結果

エミュレータでは治ったもののAndroidで変わらず...

| Android　Emulator | Android実機 |  iOSエミュレータ |  iOS実機 |
| ---- | ---- | ---- | ---- |
| ○ | × | ○ | ○　|

## 対処その2

https://github.com/flutter/flutter/issues/22007#issuecomment-605827915 を発見。

よく見たらAndroidManifest.xml のSplashScreenDrawableが抜けてた。

```xml
            <meta-data
                android:name="io.flutter.embedding.android.SplashScreenDrawable"
                android:resource="@drawable/launch_background"
                />
```

flutterのバージョンアップで新しく定義された模様　

https://github.com/flutter/engine/pull/9525/files

https://flutter.dev/docs/development/ui/splash-screen/android-splash-screen

### 結果

| Android　Emulator | Android実機 |  iOSエミュレータ |  iOS実機 |
| ---- | ---- | ---- | ---- |
| ○ | ○ | × | ○　|

## 結論

`WidgetsFlutterBinding.ensureInitialized` ありの `v1.12.13+hotfix.8` でAndroidでスプラッシュ後に黒画面がでたのは、 https://github.com/flutter/engine/pull/9525/files で追加されたスプラッシュ画面のdrawableの定義がなかったから黒画面になったから。

iOSの黒画面は https://github.com/flutter/flutter/commit/9f4e5ad9c3a63a7e50f358b5cbf233872e9e077c の対処が必要だが、実機では発生しないので放置でよさそう。

release noteを読もう。

https://flutter.dev/docs/development/tools/sdk/release-notes/release-notes-1.12.13

