<template>
  <div class="prism-editor-component">
    <div class="prism-editor-wrapper" ref="wrapper">
      <div
        class="prism-editor__line-numbers"
        aria-hidden="true"
        v-if="lineNumbers"
        :style="{ 'min-height': lineNumbersHeight }"
      >
        <div
          class="prism-editor__line-width-calc"
          style="height: 0px; visibility: hidden; pointer-events: none;"
        >
          999
        </div>
        <div
          class="prism-editor__line-number token comment"
          v-for="line in lineNumbersCount"
          :key="line"
        >
          {{ line }}
        </div>
      </div>
      <pre
        class="prism-editor__code"
        :class="{ ['language-' + language]: true }"
        ref="pre"
        v-html="content"
        :contenteditable="!readonly"
        @keydown="handleKeyDown"
        @keyup="handleKeyUp"
        @click="handleClick"
        spellCheck="false"
        autocapitalize="off"
        autocomplete="off"
        autocorrect="off"
        data-gramm="false"
        v-on="$listeners"
      ></pre>
    </div>

    <ul class="prism-editor__autocomplete" v-if="autocompleteOpen && autocompleteData.length" :style="{left: cursorOffset[0] + 'px', top: cursorOffset[1] + 'px'}">
      <li
        v-for="(suggestion, i) in autocompleteData"
        :key="suggestion.text" :class="{selected: i == autocompleteIndex}"
        @mousedown="acceptAutocomplete($event, i)"
      >{{ suggestion.label || suggestion.text }}</li>
    </ul>
  </div>
</template>

<script>
import prism from "../utils/prism";
import escapeHtml from "escape-html";
import normalizeHtml from "../utils/normalizeHtml.js";
import htmlToPlain from "../utils/htmlToPlain.js";
import selectionRange from "../utils/selection-range.js";
import { getIndent, getDeindentLevel } from "../utils/getIndent";
import { FORBIDDEN_KEYS } from "../utils/constant";

export default {
  model: {
    prop: "code",
    event: "change"
  },
  props: {
    emitEvents: {
      type: Boolean,
      default: false
    },
    language: {
      type: String,
      default: "js"
    },
    lineNumbers: {
      type: Boolean,
      default: false
    },
    autoStyleLineNumbers: {
      type: Boolean,
      default: true
    },
    readonly: {
      type: Boolean,
      default: false
    },
    code: {
      type: String,
      default: ""
    },
    ignoreTabKey: {
      type: Boolean,
      default: false
    },
    autocomplete: {
      type: Function,
      default() {
        return []
      }
    }
  },
  data() {
    return {
      undoStack: [],
      selection: undefined,
      lineNumbersHeight: "20px",
      undoOffset: 0,
      undoTimestamp: 0,
      lastPos: 0,
      codeData: "",
      composing: false,
      autocompleteOpen: false,
      autocompleteIndex: 0,
      autocompleteData: []
    };
  },
  watch: {
    code: {
      immediate: true,
      handler(newVal) {
        if (!newVal) {
          this.codeData = "";
        } else {
          this.codeData = newVal;
        }
      }
    },
    content: {
      immediate: true,
      handler() {
        if (this.lineNumbers) {
          this.$nextTick(() => {
            this.setLineNumbersHeight();
          });
        }
      }
    },
    lineNumbers() {
      this.$nextTick(() => {
        this.styleLineNumbers();
        this.setLineNumbersHeight();
      });
    },
    autocompleteIndex() {
      Vue.nextTick(() => {
        let node = this.$el.querySelector('ul.prism-editor__autocomplete > li.selected');
        if (node) node.scrollIntoView();
      })
    }
  },
  computed: {
    content() {
      return prism(this.codeData || "", this.language);
    },
    lineNumbersCount() {
      let totalLines = this.codeData.split(/\r\n|\n/).length;
      // TODO: Find a better way of doing this - ignore last line break (os spesific etc.)
      if (this.codeData.endsWith("\n")) {
        totalLines--;
      }
      return totalLines;
    },
    cursorOffset() {
      let lines = this.codeData.substring(0, this.selection && this.selection.end || 0).split(/\r\n|\n/);
      let font_size = parseFloat(getComputedStyle(this.$refs.pre).getPropertyValue('font-size'));
      let line = lines.length;
      let column = lines[lines.length-1].length;
      return [
        (column * 8.85 * (font_size / 16))    - this.$refs.wrapper.scrollLeft,
        (line * 24.0 * (font_size / 16)) + 2  - this.$refs.wrapper.scrollTop,
      ]
    }
  },
  updated() {
    if (this.selection) {
      selectionRange(this.$refs.pre, this.selection);
    }
  },
  mounted() {
    this.recordChange(this.getPlain());
    this.undoTimestamp = 0; // Reset timestamp
    this.styleLineNumbers();

    const onPaste = e => {
      e.preventDefault();
      const currentCursorPos = selectionRange(this.$refs.pre);

      // get text representation of clipboard
      var text = (e.originalEvent || e).clipboardData.getData("Text");
      // insert text manually
      document.execCommand("insertHTML", false, escapeHtml(text));

      const newCursorPos = currentCursorPos.end + text.length;
      this.selection = { start: newCursorPos, end: newCursorPos };

      const plain = this.getPlain();
      this.recordChange(plain, this.selection);
      this.updateContent(plain);
      this.setLineNumbersHeight();
    };

    const Brackets = {
      '(':')',
      '{':'}',
      '[':']',
      '"':'"',
      "'":"'",
    }
    const onBracket = e => {
      let char = event.key;
      
      if (!Brackets[char]) return;
      if (this.selection.start == this.selection.end) return;
      
      event.preventDefault()

      let new_text = [
        this.code.substr(0, this.selection.start),
        char,
        this.code.substring(this.selection.start, this.selection.end),
        Brackets[char],
        this.code.substring(this.selection.end),
      ]
      this.code = new_text.join('');
      this.selection.start += 1;
      this.selection.end += 1;

      const plain = this.getPlain();
      this.recordChange(plain, this.selection);
      this.updateContent(plain);
      this.setLineNumbersHeight();
    };

    const onFocusOut = e => {
      this.autocompleteOpen = false;
      if (this.emitEvents) {
        this.$emit("keydown", evt);
      }
    }


    const $pre = this.$refs.pre;
    $pre.addEventListener("paste", onPaste);
    $pre.addEventListener("keypress", onBracket);
    $pre.addEventListener("focusout", onFocusOut);
    this.$once("hook:beforeDestroy", () => {
      $pre.removeEventListener("paste", onPaste);
      $pre.removeEventListener("keypress", onBracket);
      $pre.removeEventListener("focusout", onFocusOut);
    });
    $pre.addEventListener("compositionstart", () => {
      this.composing = true;
    });
    $pre.addEventListener("compositionend", () => {
      // for canceling input.
      this.composing = false;
    });
  },

  methods: {
    updateAutocompleteData() {
      let data = this.autocomplete(this.getPlain(), this.selection.end)
      let old_length = this.autocompleteData.length;
      this.autocompleteData.splice(0, Infinity, ...data);
      this.autocompleteOpen = true;
      this.autocompleteIndex = Math.max(0, Math.min(this.autocompleteIndex, this.autocompleteData.length-1));
      if (old_length > this.autocompleteData.length) this.autocompleteIndex = 0;
    },
    acceptAutocomplete(event, option = this.autocompleteIndex) {
      event.preventDefault()

      let suggestion = this.autocompleteData[option] || this.autocompleteData[0];
      if (!suggestion) return;
      
      let overlap = suggestion.overlap || 0;
      let new_text = [
        this.code.substr(0, this.selection.end - overlap),
        suggestion.text,
        this.code.substring(this.selection.end),
      ]
      this.code = new_text.join('');
      let cursor_pos = this.selection.end - overlap + suggestion.text.length + (suggestion.text.endsWith(')') ? -1 : 0);
      this.selection.start = this.selection.end = cursor_pos;

      const plain = this.getPlain();
      this.recordChange(plain, this.selection);
      this.updateContent(plain);
      this.setLineNumbersHeight();
      if (suggestion.text.endsWith('.')) {
        this.updateAutocompleteData();
      }{
        this.autocompleteOpen = false;
      }
    },
    setLineNumbersHeight() {
      this.lineNumbersHeight = getComputedStyle(this.$refs.pre).height;
    },
    styleLineNumbers() {
      if (!this.lineNumbers || !this.autoStyleLineNumbers) return;

      const $editor = this.$refs.pre;
      const $lineNumbers = this.$el.querySelector(
        ".prism-editor__line-numbers"
      );
      const editorStyles = window.getComputedStyle($editor);

      this.$nextTick(() => {
        const btlr = "border-top-left-radius";
        const bblr = "border-bottom-left-radius";
        $lineNumbers.style[btlr] = editorStyles[btlr];
        $lineNumbers.style[bblr] = editorStyles[bblr];
        $editor.style[btlr] = 0;
        $editor.style[bblr] = 0;

        const stylesList = [
          "background-color",
          "margin-top",
          "padding-top",
          "font-family",
          "font-size",
          "line-height"
        ];
        stylesList.forEach(style => {
          $lineNumbers.style[style] = editorStyles[style];
        });
        $lineNumbers.style["margin-bottom"] = "-" + editorStyles["padding-top"];
      });
    },
    handleClick(evt) {
      if (this.emitEvents) {
        this.$emit("editorClick", evt);
      }
      this.undoTimestamp = 0; // Reset timestamp
      this.selection = selectionRange(this.$refs.pre);
      this.autocompleteOpen = false;
    },
    getPlain() {
      if (this._innerHTML === this.$refs.pre.innerHTML) {
        return this._plain;
      }
      const plain = htmlToPlain(normalizeHtml(this.$refs.pre.innerHTML));
      this._innerHTML = this.$refs.pre.innerHTML;
      this._plain = plain;

      return this._plain;
    },
    recordChange(plain, selection) {
      if (plain === this.undoStack[this.undoStack.length - 1]) {
        return;
      }

      if (this.undoOffset > 0) {
        this.undoStack = this.undoStack.slice(0, -this.undoOffset);
        this.undoOffset = 0;
      }

      const timestamp = Date.now();
      const record = { plain, selection };

      // Overwrite last record if threshold is not crossed
      if (timestamp - this.undoTimestamp < 3000) {
        this.undoStack[this.undoStack.length - 1] = record;
      } else {
        this.undoStack.push(record);

        if (this.undoStack.length > 50) {
          this.undoStack.shift();
        }
      }

      this.undoTimestamp = timestamp;
    },
    updateContent(plain) {
      this.$emit("change", plain);
      this.$emit("update:code", plain);
    },
    restoreStackState(offset) {
      const { plain, selection } = this.undoStack[
        this.undoStack.length - 1 - offset
      ];

      this.selection = selection;
      this.undoOffset = offset;
      this.updateContent(plain);
    },
    undo() {
      const offset = this.undoOffset + 1;
      if (offset >= this.undoStack.length) {
        return;
      }

      this.restoreStackState(offset);
    },
    redo() {
      const offset = this.undoOffset - 1;
      if (offset < 0) {
        return;
      }

      this.restoreStackState(offset);
    },
    handleKeyDown(evt) {
      if (this.emitEvents) {
        this.$emit("keydown", evt);
      }

      if (evt.keyCode === 9 && !this.ignoreTabKey) {
        document.execCommand("insertHTML", false, "  ");
        evt.preventDefault();
      } else if (evt.keyCode === 8) {
        // Backspace Key
        const { start: cursorPos, end: cursorEndPos } = selectionRange(
          this.$refs.pre
        );
        if (cursorPos !== cursorEndPos) {
          return; // Bail on selections
        }

        const deindent = getDeindentLevel(this.$refs.pre.innerText, cursorPos);
        if (deindent <= 0) {
          return; // Bail when deindent level defaults to 0
        }

        // Delete chars `deindent` times
        for (let i = 0; i < deindent; i++) {
          document.execCommand("delete", false);
        }

        evt.preventDefault();
      } else if (evt.keyCode === 27) {
        // Escape
        if (this.autocompleteData.length && this.autocompleteOpen) {
          evt.preventDefault();
          this.autocompleteOpen = false;
        }
      } else if (evt.keyCode === 38) {
        // Up
        if (this.autocompleteData.length && this.autocompleteOpen) {
          evt.preventDefault();
          this.autocompleteIndex = (this.autocompleteIndex ? this.autocompleteIndex : this.autocompleteData.length) - 1;
        }
      } else if (evt.keyCode === 40) {
        // Down
        if (this.autocompleteData.length && this.autocompleteOpen) {
          evt.preventDefault();
          this.autocompleteIndex = (this.autocompleteIndex + 1) % this.autocompleteData.length;
        }
      } else if (evt.keyCode === 13) {

        if (this.autocompleteData.length && this.autocompleteOpen) {
          this.acceptAutocomplete(evt);

        } else {
          // Enter Key
          const { start: cursorPos } = selectionRange(this.$refs.pre);
          const indentation = getIndent(this.$refs.pre.innerText, cursorPos);

          // https://stackoverflow.com/questions/35585421
          // add a space and remove it. it works :/
          document.execCommand("insertHTML", false, "\n " + indentation);
          document.execCommand("delete", false);
        }

        evt.preventDefault();
      } else if (
        // Undo
        evt.keyCode === 90 &&
        evt.metaKey !== evt.ctrlKey &&
        !evt.altKey
      ) {
        this.undo();
        evt.preventDefault();
      } else if (
        // Redo
        evt.keyCode === 89 &&
        evt.metaKey !== evt.ctrlKey &&
        !evt.altKey
      ) {
        this.redo();
        evt.preventDefault();
      }
    },
    handleKeyUp(evt) {
      const keyupCode = evt.which;
      if (this.composing) {
        if (keyupCode === 13) {
          // finish inputting via IM.
          this.composing = false;
        } else {
          // now inputting words using IM.
          // must not update view.
          return;
        }
      }

      if (!this.code) {
        this.codeData = evt.target.innerText;
      }

      if (this.emitEvents) {
        this.$emit("keyup", evt);
      }
      if (
        evt.keyCode === 91 || // left cmd
        evt.keyCode === 93 || // right cmd
        (
          (evt.ctrlKey || evt.metaKey)
          && evt.keyCode !== 88
          )
      ) {
        return;
      }

      // Enter key
      if (evt.keyCode === 13) {
        this.undoTimestamp = 0;
      }

      this.selection = selectionRange(this.$refs.pre);


      if (!Object.values(FORBIDDEN_KEYS).includes(evt.keyCode)) {
        const plain = this.getPlain();

        this.recordChange(plain, this.selection);
        this.updateContent(plain);

        if (evt.keyCode !== 13) {
          this.updateAutocompleteData()
        }
      } else {
        this.undoTimestamp = 0;
      }
    }
  }
};
</script>

<style>
.prism-editor-wrapper code {
  font-family: inherit;
  line-height: inherit;
}
.prism-editor-component {
	width: 100%;
	height: auto;
  max-height: 100%;
	display: -webkit-box;
	display: -ms-flexbox;
	display: flex;
  align-items: flex-start;
  position: relative;
}
.prism-editor-wrapper {
  /* position: absolute; */
  width: 100%;
  height: 100%;
  overflow: auto;
  tab-size: 1.5em;
  -moz-tab-size: 1.5em;
}
.prism-editor__line-numbers {
  height: 100%;
  overflow: hidden;
  flex-shrink: 0;
  padding-top: 4px;
  margin-top: 0;
}
.prism-editor__line-number {
  /* padding: 0 3px 0 5px; */
  text-align: right;
  white-space: nowrap;
}

.prism-editor__autocomplete {
  position: absolute;
  min-width: 100px;
  width: 100%;
  max-width: 250px;
  top: 25px;
  min-height: 12px;
  max-height: 180px;
  overflow-y: auto;
  z-index: 4;
}
.prism-editor__autocomplete li {
	overflow: hidden;
	white-space: nowrap;
	padding: 1px 5px;
  cursor: pointer;
}

.prism-editor__code {
  margin-top: 0 !important;
  margin-bottom: 0 !important;
  flex-grow: 2;
  min-height: 100%;
  box-sizing: border-box;
  /* padding: 4px 2px 4px 8px; */
  tab-size: 4;
  -moz-tab-size: 4;
  outline: none;
}
pre.prism-editor__code:focus {
  outline: none;
}
</style>
