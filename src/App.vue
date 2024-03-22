<template>
    <div class="foo" contenteditable="true" ref="approverInput" v-html="inputText" @input="handleApproverInput"></div>
</template>

<script>
import diff from "fast-diff";

export default {
  data() {
    return {
      inputText: 'あああああああ',
      originalText: 'いいいいいい',
      deletions: [],
      additions: []
    };
  },
  computed: {
    formattedText() {
      let text = this.originalText;
      // 削除された部分に打ち消し線を適用
      this.deletions.forEach(deletion => {
        text = text.replace(deletion, '<del>' + deletion + '</del>');
      });
      // 追加された部分を太字に変換
      this.additions.forEach(addition => {
        text = text.replace(addition, '<strong>' + addition + '</strong>');
      });
      return text;
    }
  },
  methods: {
    handleApproverInput() {
      const newText = this.$refs.approverInput.innerText;
      const changes = diff(this.originalText, newText);
      console.log(changes);
      this.inputText = this.applyTextStyles(changes);
    },
    applyTextStyles(changes) {
      return changes.map(change => {
        const [type, text] = change;
        if (type === 0) {
          return text;
        } else if (type === 1) {
          this.additions.push(text);
          return '<strong>' + text + '</strong>';
        } else if (type === -1) {
          this.deletions.push(text);
          return '<del>' + text + '</del>';
        }
      }).join('');
    },
  }
};
</script>

<style scoped>
.foo >>> strong {
    font-weight: bold;
    color: red;
  }

.foo >>> del {
    text-decoration: line-through;
    color: red;
}
</style>