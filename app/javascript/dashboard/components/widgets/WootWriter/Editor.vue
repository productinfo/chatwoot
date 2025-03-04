<template>
  <div class="editor-root">
    <tag-agents
      v-if="showUserMentions && isPrivate"
      :search-key="mentionSearchKey"
      @click="insertMentionNode"
    />
    <canned-response
      v-if="showCannedMenu"
      :search-key="cannedSearchTerm"
      @click="insertCannedResponse"
    />
    <div ref="editor"></div>
  </div>
</template>

<script>
import { EditorView } from 'prosemirror-view';

import { defaultMarkdownSerializer } from 'prosemirror-markdown';
import {
  addMentionsToMarkdownSerializer,
  addMentionsToMarkdownParser,
  schemaWithMentions,
} from '@chatwoot/prosemirror-schema/src/mentions/schema';

import {
  suggestionsPlugin,
  triggerCharacters,
} from '@chatwoot/prosemirror-schema/src/mentions/plugin';
import { EditorState } from 'prosemirror-state';
import { defaultMarkdownParser } from 'prosemirror-markdown';
import { wootWriterSetup } from '@chatwoot/prosemirror-schema';

import TagAgents from '../conversation/TagAgents';
import CannedResponse from '../conversation/CannedResponse';

const TYPING_INDICATOR_IDLE_TIME = 4000;

import '@chatwoot/prosemirror-schema/src/woot-editor.css';
import {
  hasPressedAltAndPKey,
  hasPressedAltAndLKey,
} from 'shared/helpers/KeyboardHelpers';
import eventListenerMixins from 'shared/mixins/eventListenerMixins';

const createState = (content, placeholder, plugins = []) => {
  return EditorState.create({
    doc: addMentionsToMarkdownParser(defaultMarkdownParser).parse(content),
    plugins: wootWriterSetup({
      schema: schemaWithMentions,
      placeholder,
      plugins,
    }),
  });
};

export default {
  name: 'WootMessageEditor',
  components: { TagAgents, CannedResponse },
  mixins: [eventListenerMixins],
  props: {
    value: { type: String, default: '' },
    placeholder: { type: String, default: '' },
    isPrivate: { type: Boolean, default: false },
    isFormatMode: { type: Boolean, default: false },
  },
  data() {
    return {
      lastValue: null,
      showUserMentions: false,
      showCannedMenu: false,
      mentionSearchKey: '',
      cannedSearchTerm: '',
      editorView: null,
      range: null,
    };
  },
  computed: {
    plugins() {
      return [
        suggestionsPlugin({
          matcher: triggerCharacters('@'),
          onEnter: args => {
            this.showUserMentions = true;
            this.range = args.range;
            this.editorView = args.view;
            return false;
          },
          onChange: args => {
            this.editorView = args.view;
            this.range = args.range;

            this.mentionSearchKey = args.text.replace('@', '');
            return false;
          },
          onExit: () => {
            this.mentionSearchKey = '';
            this.showUserMentions = false;
            this.editorView = null;
            return false;
          },
          onKeyDown: ({ event }) => {
            return event.keyCode === 13 && this.showUserMentions;
          },
        }),
        suggestionsPlugin({
          matcher: triggerCharacters('/'),
          suggestionClass: '',
          onEnter: args => {
            if (this.isPrivate) {
              return false;
            }
            this.showCannedMenu = true;
            this.range = args.range;
            this.editorView = args.view;
            return false;
          },
          onChange: args => {
            this.editorView = args.view;
            this.range = args.range;

            this.cannedSearchTerm = args.text.replace('/', '');
            return false;
          },
          onExit: () => {
            this.cannedSearchTerm = '';
            this.showCannedMenu = false;
            this.editorView = null;
            return false;
          },
          onKeyDown: ({ event }) => {
            return event.keyCode === 13 && this.showCannedMenu;
          },
        }),
      ];
    },
  },
  watch: {
    showUserMentions(updatedValue) {
      this.$emit('toggle-user-mention', this.isPrivate && updatedValue);
    },
    showCannedMenu(updatedValue) {
      this.$emit('toggle-canned-menu', !this.isPrivate && updatedValue);
    },
    value(newValue = '') {
      if (newValue !== this.lastValue) {
        const { tr } = this.state;
        if (this.isFormatMode) {
          this.state = createState(
            newValue,
            this.placeholder,
            this.plugins,
            this.isFormatMode
          );
        } else {
          tr.insertText(newValue, 0, tr.doc.content.size);
          this.state = this.view.state.apply(tr);
        }
        this.view.updateState(this.state);
      }
    },
  },
  created() {
    this.state = createState(this.value, this.placeholder, this.plugins);
  },
  mounted() {
    this.view = new EditorView(this.$refs.editor, {
      state: this.state,
      dispatchTransaction: tx => {
        this.state = this.state.apply(tx);
        this.emitOnChange();
      },
      handleDOMEvents: {
        keyup: () => {
          this.onKeyup();
        },
        focus: () => {
          this.onFocus();
        },
        blur: () => {
          this.onBlur();
        },
      },
    });
    this.focusEditorInputField();
  },
  methods: {
    handleKeyEvents(e) {
      if (hasPressedAltAndPKey(e)) {
        this.focusEditorInputField();
      }
      if (hasPressedAltAndLKey(e)) {
        this.focusEditorInputField();
      }
    },
    focusEditorInputField() {
      this.$refs.editor.querySelector('div.ProseMirror-woot-style').focus();
    },
    insertMentionNode(mentionItem) {
      if (!this.view) {
        return null;
      }
      const node = this.view.state.schema.nodes.mention.create({
        userId: mentionItem.key,
        userFullName: mentionItem.label,
      });

      const tr = this.view.state.tr.replaceWith(
        this.range.from,
        this.range.to,
        node
      );
      this.state = this.view.state.apply(tr);
      return this.emitOnChange();
    },

    insertCannedResponse(cannedItem) {
      if (!this.view) {
        return null;
      }

      const tr = this.view.state.tr.insertText(
        cannedItem,
        this.range.from,
        this.range.to
      );
      this.state = this.view.state.apply(tr);
      return this.emitOnChange();
    },

    emitOnChange() {
      this.view.updateState(this.state);
      this.lastValue = addMentionsToMarkdownSerializer(
        defaultMarkdownSerializer
      ).serialize(this.state.doc);
      this.$emit('input', this.lastValue);
    },
    hideMentions() {
      this.showUserMentions = false;
    },
    resetTyping() {
      this.$emit('typing-off');
      this.idleTimer = null;
    },
    turnOffIdleTimer() {
      if (this.idleTimer) {
        clearTimeout(this.idleTimer);
      }
    },
    onKeyup() {
      if (!this.idleTimer) {
        this.$emit('typing-on');
      }
      this.turnOffIdleTimer();
      this.idleTimer = setTimeout(
        () => this.resetTyping(),
        TYPING_INDICATOR_IDLE_TIME
      );
    },
    onBlur() {
      this.turnOffIdleTimer();
      this.resetTyping();
      this.$emit('blur');
    },
    onFocus() {
      this.$emit('focus');
    },
  },
};
</script>

<style lang="scss">
.ProseMirror-menubar-wrapper {
  display: flex;
  flex-direction: column;

  > .ProseMirror {
    padding: 0;
    word-break: break-word;
  }
}

.editor-root {
  width: 100%;
}

.ProseMirror-woot-style {
  min-height: 8rem;
  max-height: 12rem;
  overflow: auto;
}

.is-private {
  .prosemirror-mention-node {
    font-weight: var(--font-weight-medium);
    background: var(--s-50);
    color: var(--s-900);
    padding: 0 var(--space-smaller);
  }
}

.editor-wrap {
  margin-bottom: var(--space-normal);
}

.message-editor {
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius-normal);
  padding: 0 var(--space-slab);
  margin-bottom: 0;
}

.editor_warning {
  border: 1px solid var(--r-400);
}

.editor-warning__message {
  color: var(--r-400);
  font-weight: var(--font-weight-normal);
  padding: var(--space-smaller) 0 0 0;
}
</style>
