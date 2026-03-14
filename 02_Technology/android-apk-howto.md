# ViteアプリをAndroid APK化するノウハウ（Capacitor）

対象：Vite + React + TypeScript で作られたWebアプリ

---

## 前提条件

- Node.js 18+、pnpm インストール済み
- [Android Studio](https://developer.android.com/studio) インストール済み
- Gemini APIキーなど `.env` の設定済み

---

## 初回セットアップ

### 1. Capacitorのインストール

プロジェクトのルートフォルダでターミナルを開いて実行：

```bash
pnpm add @capacitor/core @capacitor/cli @capacitor/android
```

### 2. Capacitorの初期化

```bash
npx cap init <アプリ名> com.example.<アプリ名小文字> --web-dir dist
```

例：`npx cap init MAGIsystem com.example.magisystem --web-dir dist`

→ `capacitor.config.ts` が生成される。すでに存在する場合は2回目は不要（エラーが出ても無視）。

### 3. ビルド → Androidプロジェクト追加 → 同期

```bash
pnpm build
npx cap add android
npx cap sync
```

### 4. Android Studioで開く

```bash
npx cap open android
```

---

## ⚠️ ハマりポイント：AGPバージョンエラー

Android Studioを開いたときに下記エラーが出ることがある：

```
The project is using an incompatible version (AGP X.X.X) of the Android Gradle plugin.
Latest supported version is AGP 8.10.1
```

### 対処法

`android/build.gradle` を開いて該当行を修正：

```groovy
// 修正前
classpath 'com.android.tools.build:gradle:X.X.X'
// 修正後（Android Studioが対応しているバージョンに合わせる）
classpath 'com.android.tools.build:gradle:8.10.1'
```

修正後、Android Studioで **File → Sync Project with Gradle Files** を実行。

> Android Studioをアップデートしたら対応AGPバージョンも変わるので、
> エラーメッセージに書いてある「Latest supported version」に合わせること。

---

## APKのビルド

Android Studioのメニューから：

**Build → Generate App Bundles or APKs → Generate APKs**

完了すると右下に通知が出る。「locate」をクリックするとAPKのフォルダが開く。

APKの場所：`android/app/build/outputs/apk/debug/app-debug.apk`

---

## スマホへのインストール

### Googleドライブ経由（一番手軽）
1. `app-debug.apk` をGoogleドライブにアップロード
2. スマホのGoogleドライブアプリから開いてダウンロード
3. 「提供元不明のアプリ」の許可を求められたら許可してインストール

### USB経由
1. スマホをUSBでPCに接続（ファイル転送モード）
2. APKをスマホのダウンロードフォルダにコピー
3. ファイルマネージャーからAPKをタップしてインストール

---

## コード更新後の再ビルド

```bash
pnpm build
npx cap sync
```

その後Android Studioで **Generate APKs** を再実行。
（`npx cap open android` は不要。すでに開いていればそのまま使える）

---

## ⚠️ APIキーの取り扱い注意

ViteはビルドするときにAPIキーをJSファイルへ**直接埋め込む**。
そのため生成されたAPKにはAPIキーが含まれる。

- `.env` は `.gitignore` で除外済みなのでgit自体は安全
- `android/` フォルダには埋め込み済みJSが含まれるため **`.gitignore` に追加して除外すること**
- APKは**信頼できる相手にのみ配布**する
- 不特定多数への配布後はAPIキーをGoogle AI Studioでローテーション（削除して再発行）する

### 各プロジェクトの `.gitignore` に必ず追加

```
# Androidビルド成果物（APIキーが埋め込まれたJSを含むため除外）
android/
```

---

## スマホ対応レスポンシブ対応のポイント

PC向けに作ったWebアプリをそのままAndroid化するとレイアウトが崩れることがある。

### よくある問題
- 固定幅サイドバー（`width: 280px` など）がスマホ幅に収まらず右側がはみ出す
- `position: fixed` のコーナー装飾が邪魔になる
- フォントサイズが大きすぎる

### 重要：ReactのインラインスタイルはCSSで上書きできない

`[style*="..."]` セレクタは React のインラインスタイルには効かない。
コンポーネントに `className` を追加して、`index.css` でメディアクエリを当てるのが確実。

```tsx
// コンポーネント側：classNameを追加するだけ
<div className="app-sidebar" style={{ width: '280px', ... }}>
```

```css
/* index.css側 */
@media (max-width: 600px) {
  .app-sidebar {
    width: 100% !important;
    border-right: none !important;
    border-bottom: 1px solid #333 !important;
  }
}
```

### 左右2カラム → スマホで縦積みにする典型パターン

```tsx
<div className="app-body" style={{ display: 'flex' }}>
  <div className="app-sidebar" style={{ width: '320px', ... }} />
  <div className="app-main" style={{ flex: 1, ... }} />
</div>
```

```css
@media (max-width: 600px) {
  .app-body    { flex-direction: column !important; }
  .app-sidebar {
    width: 100% !important;
    border-right: none !important;
    border-bottom: 1px solid #333 !important;
    max-height: 200px !important;
    overflow-y: auto !important;
  }
  .app-main { padding: 16px !important; }
}
```

---

## android/ フォルダの再生成方法

git clone 後や `android/` を削除した後は以下で再生成：

```bash
pnpm install
pnpm build
npx cap add android
npx cap sync
```

その後AGPバージョンの修正（上記参照）を行い、Android Studio でビルド。
