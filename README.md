# findx
Windows用フォルダ除外指定findstr

## 概要(Overview) 

Windows の findstr コマンドと似たファイル内野路烈検索。　　
本プログラムは、Google Gemini に全て作らせました。　
findstr が特定のディレクトリを除外できないので作ってもらった。

## デモ画面(Demo)

```cmd
findx /s "VITE" *.ts /i node_modules
----------------------------------------
vite.config.ts(8):   const proxyPath = env.VITE_API_PROXY_PATH || '/api';
vite.config.ts(9):   const proxyContents = env.VITE_DOC_PROXY_PATH || '/uploads';
vite.config.ts(10):   const apiServer = env.VITE_API_SERVER || 'http://localhost:3000';
```

## 導入方法(Setup)

Visual Studioでコンパイル。 

## 操作方法(Usage)

引数無しでオプションが表示されます。　
使用方法: findx [/s] [/i 除外フォルダ] [/f utf8|cp932] <検索文字列> [<パス\ワイルドカード>]

※ ファイルの文字コードは自動判定しますが、80%位。/f でファイル内の文字コードを指定すればそれに従います。　
  除外フォルダは /i を並列で書いて複数指定可能。

# おわりに
10分で出来たので、Gemini にソースを読ませれば、拡張は容易。　
好みで機能追加してください。
