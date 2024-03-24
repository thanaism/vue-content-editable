<template>
  <QuillEditor
      ref="quill"
      :options="options"
      @blur="handleBlur"
      @compositionend="handleCompositionEnd"
      @compositionstart="handleCompositionStart"
      @ready="handleReady"
      @textChange="handleTextChange"
      @selectionChange="handleSelectionChange"
  />
</template>

<script>
import {QuillEditor} from "@vueup/vue-quill";

export default {
  components: {
    QuillEditor,
  },

  data() {
    return {
      isReady: false, // Quillの初期化が完了したか
      isComposing: false, // IMEが未確定状態か
      initialText: 'あいうえお', // 編集前のテキスト
      initialContents: null, // 初期状態の保存用
      currentText: null, // 現在の確定済みテキスト
      currentContents: null, // 確定済み状態の保存用
      options: {modules: {toolbar: false}}, // Quillのオプション
      addedFormat: {background: "cyan", bold: true}, // 追記箇所のスタイル
      deletedFormat: {background: "bisque", strike: true}, // 削除箇所のスタイル
    };
  },

  methods: {
    // IME入力の開始
    handleCompositionStart() {
      this.isComposing = true;
      console.log({isComposing: this.isComposing});
    },

    // IME入力の終了
    handleCompositionEnd(event) {
      this.isComposing = false;
      console.log({isComposing: this.isComposing, data: event.data});
      const quill = this.$refs.quill.getQuill();
      const delta = this.currentContents.diff(quill.getContents());
      this.handleTextChange({delta, source: 'user'});
    },

    // フォーカス外れ：APIへ更新処理を行う場合はこのタイミングを想定
    handleBlur() {
      console.log('handleBlur');
    },

    // Quillの初期化完了
    handleReady() {
      this.isReady = true;
      console.log({isReady: this.isReady})
      // 初期コンテンツを格納する
      const quill = this.$refs.quill.getQuill();
      quill.setText(this.initialText, 'api');
      this.initialContents = quill.getContents();
      this.currentContents = quill.getContents();
    },

    // 選択範囲変更時のフック：ここで選択範囲に対する処理を行う
    handleSelectionChange({range, oldRange, source}) {
      // 範囲選択の場合は処理しない
      if (range.length !== 0) return;

      const quill = this.$refs.quill.getQuill();
      const contents = quill.getContents();

      // 打ち消し線の範囲内にカーソルを移動させた場合、打ち消し線の外側に移動させる
      let retain = 0;
      for (const op of contents.ops) {
        if (op.retain) {
          retain += op.retain;
        } else if (op.insert) {
          if(op.attributes?.strike ) {
            console.log(JSON.stringify({op, range}))
            if (range.index > retain && range.index < retain + op.insert.length) {
              // 左から右への移動の場合、打ち消し線の右端までカーソルを移動させる、逆も然り
              const isLeftToRight = range.index > oldRange.index;
              if (isLeftToRight) {
                quill.setSelection(retain + op.insert.length, 0, 'api');
              } else {
                quill.setSelection(retain, 0, 'api');
              }
              break;
            }
            retain += op.insert.length;
          } else {
            retain += op.insert.length;
          }
        }
      }
    },

    // テキスト変更時のフック：ここで差分計算やスタイルの適用を行う
    handleTextChange({delta, source}) {
      // 初期化中、IME入力中、ユーザー以外の変更の場合は処理しない（QuillはQuill自身のAPIによる変更も検知する）
      if (!this.isReady || this.isComposing || source !== 'user') return;

      // Quillのインスタンスを取得（このオブジェクトはVueQuillではなくオリジナルのQuillのAPIを利用可能）
      const quill = this.$refs.quill.getQuill();

      // 削除箇所の途中や直後に追記すると、意図せず削除箇所と同じスタイルを引き継いでしまうので、スタイルを削除する
      let retain = 0;
      for (const op of delta.ops) {
        if (op.retain) {
          retain += op.retain;
        } else if (op.insert) {
          quill.removeFormat(retain, op.insert.length);
          retain += op.insert.length;
        }
      }

      // 意図せぬスタイル適用を取り除いたので、打ち消し線が適用されている部分を削除することで、前回までの削除箇所を除去する
      // この処理により、削除箇所を含まない純粋な更新後の平文の状態になる
      retain = 0;
      for (const op of quill.getContents().ops) {
        if (op.retain) {
          retain += op.retain;
        } else if (op.insert) {
          if (op.attributes?.strike) {
            quill.deleteText(retain, op.insert.length);
          } else {
            retain += op.insert.length;
          }
        }
      }
      // console.log(JSON.stringify({delRemoved: quill.getContents()}));

      // いったんすべてのスタイルをリセットする
      quill.removeFormat(0, quill.getLength() - 1);

      // 削除箇所を文字として含まない状態を得たため、更新後の平文 vs 更新前の平文 で差分が取得できるようになる
      const oldContents = this.initialContents;
      const newContents = quill.getContents();
      const diff = oldContents.diff(newContents);

      const oldText = oldContents.ops.filter(op => op.insert).map(op => op.insert).join('');

      this.currentText = quill.getText();
      console.log({currentText: this.currentText});
      // console.log(JSON.stringify({newContents}));
      // console.log(JSON.stringify({oldContents}));
      // console.log(JSON.stringify({diff}));
      // console.log(JSON.stringify({oldText}));

      let retainNew = 0; // newContentsで見たretainの累積値
      let retainOld = 0; // oldContentsで見たretainの累積値
      for (const op of diff.ops) {
        if (op.retain) {
          retainNew += op.retain;
          retainOld += op.retain;
        } else if (op.insert) {
          quill.formatText(retainNew, op.insert.length, this.addedFormat);
          retainNew += op.insert.length; // insertはnewContentsにしか存在しない
        } else if (op.delete) {
          const deleted = oldText.slice(retainOld, Math.min(oldText.length, retainOld + op.delete));
          if (deleted == null) continue; // 意図せずnullが入ることがあるので、その場合はスキップ
          quill.insertText(retainNew, deleted);
          quill.formatText(retainNew, deleted.length, this.deletedFormat);
          // 削除箇所はどちらにも含まれることになるので、両方のretainを進める必要がある
          retainNew += deleted.length;
          retainOld += deleted.length;
        }
      }

      // CompositionEndのタイミングにとってCompositionStartの際の状態を保持することになる
      this.currentContents = quill.getContents();
    },
  },
};
</script>