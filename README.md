# splachblackout

https://github.com/flutter/flutter/issues/39494 の対処を検証するサンプル。

## 事象

Androidでスプラッシュ表示からrunAppの間にdelayを挟むと黒画面が表示される。

v1.16.3で修正がマージされた模様。 https://github.com/flutter/flutter/commit/9f4e5ad9c3a63a7e50f358b5cbf233872e9e077c

## 検証端末

- Android
  - Emulator(AndroidSDK 29)
  - HTC U11 life(Android 8.1)
- iOS
  - Emulator(iOS 13.2)
  - iPhone 6s(iOS 13.3)

flavorはdebugビルドにて検証する。

## 切り分け

- ○ : 黒画面が表示されない
- × : 黒画面が表示される

| flutterバージョン | Android　Emulator | Android実機 |  iOSエミュレータ |  iOS実機 |
| ---- | ---- | ---- | ---- | ---- |
| v1.9.1+hotfix.6 | NG | NG | NG | × |
| v1.12.13+hotfix.8 | NG | NG | NG | ○　|
| v1.16.3 | ○ | NG | ○　|  ○　|

## 原因？

1.9.1から1.12.13に上げたときにいれた `WidgetsFlutterBinding.ensureInitialized()` の処理を入れないと発生しない。

```dart
void main() {

  WidgetsFlutterBinding.ensureInitialized();

  Future.delayed(Duration(seconds: 10), () {
    runApp(MyApp());
  });
}

```

## 対処その1

以下のwork aroundをやってみる。

- https://github.com/flutter/flutter/issues/39494#issuecomment-565627914
- https://github.com/flutter/flutter/issues/39494#issuecomment-596089595