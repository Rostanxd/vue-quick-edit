/* global QUICKEDIT_DEFAULT_CLASSES */
<template>
  <div ref="el" :class="classNames.wrapper">
    <template v-if="isEditing && isEnabled">
      <select
        v-if="types.select === type"
        :class="classNames.input"
        v-model="inputValue"
        v-bind="$attrs"
        :tabindex="tabIndex"
        @focusin="handleFocus"
        @focusout="handleFocus"
        @keypress.enter="ok"
        @keypress.escape.exact="close"
      >
        <option v-show="$attrs.placeholder" :value="placeholderValue">{{
            $attrs.placeholder
          }}</option>
        <option
          v-for="option in displayOptions"
          :key="option.value"
          :value="option.value"
          :disabled="option.disabled"
        >
          {{ option.text }}
        </option>
      </select>
      <textarea
        v-else-if="types.textarea === type"
        :class="classNames.input"
        v-model="inputValue"
        v-bind="$attrs"
        :tabindex="tabIndex"
        @focusin="handleFocus"
        @focusout="handleFocus"
        @keyup="autoUpdate"
        @keypress.ctrl.enter="ok"
        @keypress.escape.exact="close"
      ></textarea>
      <template
        v-else-if="types.radio === type || types.checkbox === type"
        v-for="option in displayOptions"
      >
        <label :key="option.value" @focusin="handleFocus" @focusout="handleFocus">
          {{ option.text }}
          <input
            :type="type"
            :value="option.value"
            :disabled="option.disabled"
            v-model="inputValue"
            :tabindex="tabIndex"
            @keypress.enter="ok"
            @keypress.escape.exact="close"
          />
        </label>
      </template>
      <div v-else-if="!!prefix" :class="classNames.formGroup">
        <div :class="classNames.inputIcon">
          <input
            :class="classNames.formControl"
            :type="type"
            v-model="inputValue"
            v-bind="$attrs"
            :tabindex="tabIndex"
            @focusin="handleFocus"
            @focusout="handleFocus"
            @keypress.enter="ok"
            @keypress.escape.exact="close"
          />
          <i>{{ prefix }}</i>
        </div>
      </div>
      <input
        v-else
        :class="classNames.input"
        :type="type"
        v-model="inputValue"
        v-bind="$attrs"
        :tabindex="tabIndex"
        @focusin="handleFocus"
        @focusout="handleFocus"
        @keyup="autoUpdate"
        @keypress.enter="ok"
        @keypress.escape.exact="close"
      />
      <div v-if="showButtons" :class="classNames.buttons">
        <button
          :class="classNames.buttonOk"
          :title="buttonOkText"
          @click="ok"
          @focusin="handleFocus"
          @focusout="handleFocus"
        >
          <slot name="button-ok">{{ buttonOkText }}</slot>
        </button>
        <button
          :class="classNames.buttonCancel"
          :title="buttonCancelText"
          @click="close"
          @focusin="handleFocus"
          @focusout="handleFocus"
        >
          <slot name="button-cancel">{{ buttonCancelText }}</slot>
        </button>
      </div>
    </template>
    <template v-else-if='!(isEditing && isEnabled) && types.time === type'>
      <input
        disabled
        :class="{
          [classNames.link]: true,
          [classNames.inputTime]: true,
          [classNames.isClickable]: isEnabled,
          [classNames.isEmpty]: isEmpty,
          [classNames.isRequired]: isRequired && isEmpty,
        }"
        :type='types.time'
        :tabindex="isEnabled ? tabIndex : false"
        v-model='inputValue'
        @click="handleClick"
        @keypress.enter="handleClick"
        required
      />
    </template>
    <template v-else>
      <slot name="prepend"></slot>
      <span
        :class="{
          [classNames.link]: true,
          [classNames.isClickable]: isEnabled,
          [classNames.isEmpty]: isEmpty,
          [classNames.isRequired]: isRequired && isEmpty,
        }"
        :tabindex="isEnabled ? tabIndex : false"
        @click="handleClick"
        @keypress.enter="handleClick"
      >
        <slot :value="displayValue" :raw-value="theValue">{{ displayValue }}</slot>
      </span>
      <slot name="append"></slot>
    </template>
  </div>
</template>

<script>
const mune = keys =>
  keys.reduce((acc, cur) => {
    acc[cur] = cur;
    return acc;
  }, {});
const states = mune(['display', 'edit']);
const events = mune(['input', 'rawInput', 'show', 'close', 'invalid', 'focusin', 'focusout']);
const types = mune([
  'boolean',
  'checkbox',
  'input',
  'password',
  'radio',
  'select',
  'textarea',
  'url',
  'time',
]);
const modes = mune(['ok', 'cancel', 'ignore']);

export default {
  name: 'QuickEdit',
  props: {
    buttonOkText: {
      type: String,
      default: 'Ok',
    },
    buttonCancelText: {
      type: String,
      default: 'Cancel',
    },
    emptyText: {
      type: String,
      default: 'Empty',
    },
    booleanYesText: {
      type: String,
      default: 'Yes',
    },
    booleanNoText: {
      type: String,
      default: 'No',
    },
    type: {
      type: String,
      default: types.input,
    },
    options: {
      type: Array,
      default: () => [],
    },
    mode: {
      type: String,
      default: modes.ok,
      validator: function(value) {
        return !!modes[value];
      },
    },
    value: {
      type: [String, Array, Boolean, Number],
      default: '',
    },
    placeholderValue: {
      type: String,
      default: '',
    },
    classes: {
      type: Object,
      default: () => null,
    },
    validator: {
      type: Function,
      default: null,
    },
    showButtons: {
      type: Boolean,
      default: true,
    },
    startOpen: {
      type: Boolean,
      default: false,
    },
    keepOpen: {
      type: Boolean,
      default: true,
    },
    formatMultiple: {
      type: Function,
      default: values => values.join(', '),
    },
    prefix: {
      type: String,
      default: '',
    },
  },
  computed: {
    isEmpty() {
      return '' === this.prettyValue || null === this.prettyValue;
    },
    isEditing() {
      return states.edit === this.inputState;
    },
    isEnabled() {
      return !this.$attrs.disabled && this.$attrs.disabled !== '';
    },
    isRequired() {
      return this.$attrs.required || '' === this.$attrs.required;
    },
    isMultiple() {
      return (
        this.displayOptions.length &&
        (this.types.select === this.type ||
          this.types.checkbox === this.type ||
          this.types.radio === this.type)
      );
    },
    prettyValue() {
      return this.isMultiple
        ? Array.isArray(this.theValue)
          ? this.formatMultiple(this.theValue.map(this.getDisplayOption))
          : this.getDisplayOption(this.theValue)
        : !!this.prefix
          ? this.prefix + ' ' + this.theValue
          : this.theValue;
    },
    displayOptions() {
      const [firstEl] = this.options;
      return firstEl && typeof firstEl === 'string'
        ? this.options.map(x => ({ value: x, text: x }))
        : this.options;
    },
    displayValue() {
      if (this.types.boolean === this.type) {
        return this.theValue ? this.booleanYesText : this.booleanNoText;
      } else if (this.types.password === this.type) {
        return '•'.repeat(8);
      } else if (this.types.time === this.type) {
        const hours = parseInt(this.value.substring(0, 2));
        return this.isClient12Hour() ? this.theValue + ' ' + `${hours >= 12 ? 'PM' : 'AM'}` : this.theValue;
      } else {
        return this.isEmpty ? this.emptyText : this.prettyValue;
      }
    },
    classNames() {
      return Object.assign({}, this.defaultClasses, this.classes);
    },
    tabIndex() {
      return this.$attrs.tabindex || 0;
    },
  },
  watch: {
    value(value) {
      this.setValue(value);
    },
  },
  data() {
    return {
      inputState: this.startOpen ? states.edit : states.display,
      theValue: '',
      inputValue: '',
      types,
      defaultClasses: {
        buttonCancel: 'vue-quick-edit__button vue-quick-edit__button--cancel',
        buttonOk: 'vue-quick-edit__button vue-quick-edit__button--ok',
        buttons: 'vue-quick-edit__buttons',
        input: 'vue-quick-edit__input',
        inputTime: 'vue-quick-edit__input-time',
        link: 'vue-quick-edit__link',
        isClickable: 'vue-quick-edit__link--is-clickable',
        isEmpty: 'vue-quick-edit__link--is-empty',
        isRequired: 'vue-quick-edit__link--is-required',
        wrapper: 'vue-quick-edit',
        formGroup: 'vue-quick-edit__form-group',
        inputIcon: 'vue-quick-edit__input-icon',
        formControl: 'vue-quick-edit__form-control',
      },
    };
  },
  methods: {
    handleClick() {
      if (!this.isEnabled) return;

      if (this.types.boolean === this.type) {
        this.theValue = !this.theValue;
        this.$emit(events.input, this.theValue);
      } else {
        this.show();
      }
    },
    handleFocus({ type }) {
      if (events.focusin === type) {
        clearTimeout(this._handleFocus);
        // Send a reference to the component.
        this.$emit(events.focusin, this);
      } else {
        this._handleFocus = setTimeout(this.clickOutside, 0);

        // Send a reference to the component.
        this.$emit(events.focusout, this);
      }
    },
    show(doFocus = true) {
      this.inputValue = this.theValue;
      this.inputState = states.edit;
      this.$emit(events.show, this.theValue);
      doFocus && this.focus();
    },
    close(doFocus = true) {
      this.inputState = states.display;
      this.$emit(events.close, this.theValue);
      doFocus && this.focus();
    },
    autoUpdate() {
      // Only triggers if the buttons are hidden.
      if (!this.showButtons) {
        this.ok();
      }
    },
    ok(doFocus = true) {
      if (this.validator) {
        const error = this.validator(this.inputValue);
        if (error) {
          this.$emit(events.invalid, this.theValue, error);
          return;
        }
      }
      this.theValue = this.inputValue;
      this.$emit(events.input, this.theValue);
      this.$emit(events.rawInput, this.inputValue);

      if (!this.keepOpen) {
        this.close(doFocus);
      }
    },
    focus() {
      setTimeout(() => {
        const className = this.isEditing ? 'input,select,textarea' : 'span';
        const el = this.$refs.el && this.$refs.el.querySelector(className);
        el && el.focus();
      }, 0);
    },
    setValue(value) {
      this.theValue = value;
      this.inputValue = value;
    },
    clickOutside() {
      if (this.inputState !== states.edit) return;
      if (modes.ok === this.mode) this.ok(false);
      else if (modes.cancel === this.mode) this.close(false);
    },
    getDisplayOption(opt) {
      const option = this.displayOptions.find(x => x.value === opt);
      return option ? option.text : '';
    }
  },
  created() {
    this.setValue(this.value);
  },
  mounted() {
    // Checks if the defaults need to be replaced by a custom definition in runtime.
    if (typeof window.QUICKEDIT_DEFAULT_CLASSES !== 'undefined') {
      this.defaultClasses = Object.assign(
        {},
        this.defaultClasses,
        window.QUICKEDIT_DEFAULT_CLASSES
      );
    }
  },
};
</script>

<style lang="scss">
$link-color: #0088cc;
$link-hover-color: #2a6496;
$primary-color: #3276b1;
$primary-text-color: #fff;
$primary-border-color: #357ebd;
$danger-color: #dc3545;
$default-color: #fff;
$default-text-color: #333;
$border-color: #ccc;
$quick-edit-height: 32px;

.vue-quick-edit {
  &__link {
    white-space: pre-wrap;
    color: $link-color;

    &--is-clickable {
      border-bottom: 1px dashed $link-color;
      cursor: pointer;

      &:hover {
        color: $link-hover-color;
        border-color: $link-hover-color;
      }
    }

    &--is-empty {
      font-style: italic;
      color: gray;
    }

    &--is-required {
      color: $danger-color;
    }
  }

  &__input {
    background-color: #f9f9f9;
    color: #333;
    border: 1px solid $border-color;
    height: $quick-edit-height;
    padding: 0;
  }

  &__input-time {
    color: #333;
    padding: 0;

    &:disabled {
      background-color: #ffffff;
      border: 0;
    }
  }

  &__buttons {
    margin-top: 8px;
  }

  &__button {
    height: $quick-edit-height + 2px;
    min-width: $quick-edit-height + 2px;
    border: 1px solid $border-color;

    &--ok {
      color: $primary-text-color;
      background-color: $primary-color;
      border-color: $primary-border-color;
    }

    &--cancel {
      color: $default-text-color;
      margin-left: 8px;
      background-color: $default-color;
    }
  }

  &__input-icon {
    position: relative;
  }

  &__input-icon > i {
    position: absolute;
    display: block;
    transform: translate(0, -50%);
    top: 50%;
    pointer-events: none;
    width: 25px;
    text-align: center;
    font-style: normal;
  }

  &__input-icon > input {
    padding-left: 25px;
    padding-right: 0;
  }

  &__input-icon-right > i {
    right: 0;
  }

  &__input-icon-right > input {
    padding-left: 0;
    padding-right: 25px;
    text-align: right;
  }
}

[multiple].vue-quick-edit__input,
textarea.vue-quick-edit__input {
  height: unset;
  min-height: $quick-edit-height * 2;
  display: block;
}

.vue-quick-edit__input:not(textarea):not([multiple]) + .vue-quick-edit__buttons,
label + .vue-quick-edit__buttons {
  display: inline;
  margin-left: 8px;
}
</style>
