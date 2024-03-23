# vue-content-editable

Web上でスタイルを当てたテキストを直接編集したくなったが、通常のinput要素では当然できない。  
調べてみると、こういったケースではcontent-editableなdivを使うということを知った。

そこで、content-editableをこれまで使ったことがなかったので、Vue3と組み合わせて使ってみることにした。そして挫折した。

本当はcontent-editableな要素を使って自前ですべて実装するつもりだったが、わりと魔境であることに気付き、既存のWYSIWYGエディタを活用することにした。

## 想定要件

Googleドキュメントの「提案」モード、あるいはMicrosoft Wordの「変更履歴の記録」のような機能を実装する。

初期テキストに対して編集を行ったとき、

- 追記した部分に特定のスタイルを当てる
- 削除した場合も画面上からテキストを消すのではなく特定のスタイルを当てた表現にする

## バニラのcontent-editable

カーソル位置の制御がかなり困難であることがわかった。

onInputで入力された内容に応じてスタイルを当て直すとカーソル位置のリセットが走り、この復元だけでもかなりの手間がかかるため、断念。

## Quill

Quillはテキスト差分を抽象化したdelta形式でデータを保持するWYSIWYGライブラリ。

delta形式の特徴が差分に応じてスタイルを当てる要件に適していると考えたが、公式ドキュメント通りに組み込むとエラーが解消できず、断念。

### v1.4系

- 入力欄の最後の1文字を消そうとするとエラーになる
- 複数の改行を含んでいるとエラーになる

### v2.0-rc系

v1.4系の問題は解消されたが、新たに下記のエラーが発生。

- quill.formatTextやquill.setContentなど、Content系のAPIを実行しようとすると内部の関数でnull参照のエラーが発生する
- 同様の問題を報告するissueあり解決の糸口がないため断念 https://github.com/quilljs/quill/issues/4040

## VueQuill

Quillがうまく動作せず困っていたところ、Vue3向けにQuillをラップしたライブラリVueQuillがあるのを発見した。

本家のQuillを自前でVueに組み込んでいた時に発生していたエラーがなかったので、これを利用することにした。

懸念点としては、おそらくv1.4系のQuillに依存しているので、deprecatedなブラウザAPIを利用することになる。

> [Deprecation] Listener added for a synchronous 'DOMNodeInserted' DOM Mutation Event. 
> This event type is deprecated (https://w3c.github.io/uievents/#legacy-event-types) and work is underway to remove it from this browser. 
> Usage of this event listener will cause performance issues today, and represents a risk of future incompatibility. 
> Consider using MutationObserver instead.

## 未解決問題

削除済みテキストに打ち消し線のスタイルを当てて表示しているが、それら削除済みテキストを選択して削除しようとすると場合により問題発生する。

削除位置にもよるが、Quillがカーソル位置を見失うため、エディタ自体からフォーカスが外れる。


---

This template should help get you started developing with Vue 3 in Vite.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Customize configuration

See [Vite Configuration Reference](https://vitejs.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```
