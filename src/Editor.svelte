<svelte:options accessors={true} />

<script context="module">
  const editors = [];
</script>

<script>
  import {
    getTagsRecursive,
    saveRange as _saveRange,
    restoreRange as _restoreRange,
    exec as _exec,
    cleanHtml,
    getActionBtns,
    getNewActionObj,
    removeBadTags,
    isEditorClick,
  } from "./helpers/util.js";

  import defaultActions from "./helpers/actions.js";
  import EditorModal from "./helpers/EditorModal.svelte";
  import EditorColorPicker from "./helpers/EditorColorPicker.svelte";

  import {
    onMount,
    createEventDispatcher,
    setContext,
    getContext,
  } from "svelte";
  import { createStateStore } from "./helpers/store.js";
  import { writable } from "svelte/store";

  let dispatcher = new createEventDispatcher();

  export let actions = [];
  export let height = "300px";
  export let html = "";
  export let contentId = "";
  export let removeFormatTags = ["h1", "h2", "blockquote"];

  /**
   * styling variables (only for use in svelte components)
   */
  export let cl_root = "cl";
  export let cl_textarea = "cl-textarea";
  export let cl_actionbar = "cl-actionbar";
  export let cl_content = "cl-content";
  export let cl_button = "cl-button";
  export let cl_activeButton = "cl-button active";

  let helper = writable({
    foreColor: false,
    backColor: false,
    foreColorModal: false,
    backColorModal: false,
    image: false,
    link: false,
    showEditor: true,
    blurActive: false,
  });

  editors.push({});
  let contextKey = "editor_" + editors.length;

  let state = createStateStore(contextKey);

  let references = writable({});
  $state.actionObj = getNewActionObj(defaultActions, actions);

  let context = {
    exec,
    getHtml,
    getText,
    setHtml,
    saveRange,
    restoreRange,
    helper,
    references,
    state,
    removeFormatTags,
  };

  setContext(contextKey, context);

  onMount(() => {
    $state.actionBtns = getActionBtns($state.actionObj);
    setHtml(html);
  });

  function _btnClicked(action) {
    $references.editor.focus();
    saveRange($references.editor);
    restoreRange($references.editor);
    action.result.call(context);
    _handleButtonStatus();
  }

  function _handleButtonStatus(clearBtns) {
    const tags = clearBtns
      ? []
      : getTagsRecursive(document.getSelection().focusNode);
    Object.keys($state.actionObj).forEach(
      (action) => ($state.actionObj[action].active = false)
    );
    tags.forEach(
      (tag) => (($state.actionObj[tag.toLowerCase()] || {}).active = true)
    );
    $state.actionBtns = getActionBtns($state.actionObj);
    $state.actionObj = $state.actionObj;
  }

  function _onPaste(event) {
    event.preventDefault();
    exec(
      "insertHTML",
      event.clipboardData.getData("text/html")
        ? cleanHtml(event.clipboardData.getData("text/html"))
        : event.clipboardData.getData("text")
    );
  }

  function _onChange(event) {
    dispatcher("change", event);
  }

  function _documentClick(event) {
    if (
      !isEditorClick(event.target, $references.editorWrapper) &&
      $helper.blurActive
    ) {
      dispatcher("blur", event);
    }
    $helper.blurActive = true;
  }

  export function exec(cmd, value) {
    _exec(cmd, value);
  }

  export function getHtml(sanitize) {
    return sanitize
      ? removeBadTags($references.editor.innerHTML)
      : $references.editor.innerHTML;
  }
  export function getText() {
    return $references.editor.innerText;
  }
  export function setHtml(html, sanitize) {
    const htmlData = sanitize ? removeBadTags(html) : html || "";
    $references.editor.innerHTML = htmlData;
    $references.raw.value = htmlData;
  }
  export function saveRange() {
    _saveRange($references.editor);
  }
  export function restoreRange() {
    _restoreRange($references.editor);
  }
  export const refs = $references;
</script>

<svelte:window on:click={(event) => _documentClick(event)} />

<div class={cl_root} bind:this={$references.editorWrapper}>
  <div class={cl_actionbar}>
    {#each $state.actionBtns as action}
      <button
        type="button"
        class={action.active ? cl_activeButton : cl_button}
        title={action.title}
        on:click={(event) => _btnClicked(action)}
        disabled={action.disabled}
      >
        {@html action.icon}
      </button>
    {/each}
  </div>
  <div
    bind:this={$references.editor}
    id={contentId}
    class={cl_content}
    style="height: {height}"
    contenteditable="true"
    on:input={(event) => _onChange(event.target.innerHTML)}
    on:mouseup={() => _handleButtonStatus()}
    on:keyup={() => _handleButtonStatus()}
    on:paste={(event) => _onPaste(event)}
  />

  <textarea
    bind:this={$references.raw}
    class={cl_textarea}
    style="max-height: {height}; min-height: {height}"
  />
  <EditorModal bind:this={$references.modal} />
  <EditorColorPicker bind:this={$references.colorPicker} />
</div>

<style>
  .cl * {
    box-sizing: border-box;
  }

  .cl {
    box-shadow: 0 2px 3px rgba(10, 10, 10, 0.1), 0 0 0 1px rgba(10, 10, 10, 0.1);
    box-sizing: border-box;
    width: 100%;
    position: relative;
  }

  .cl-content {
    height: 300px;
    outline: 0;
    overflow-y: auto;
    padding: 10px;
    width: 100%;
    background-color: white;
  }

  .cl-actionbar {
    background-color: #ecf0f1;
    border-bottom: 1px solid rgba(10, 10, 10, 0.1);
    width: 100%;
  }

  .cl-button {
    background-color: transparent;
    border: none;
    cursor: pointer;
    height: 35px;
    outline: 0;
    width: 35px;
    vertical-align: top;
    position: relative;
  }

  .cl-button:hover,
  .cl-button.active {
    background-color: #fff;
  }

  .cl-button:disabled {
    opacity: 0.5;
    pointer-events: none;
  }

  .cl-textarea {
    display: none;
    max-width: 100%;
    min-width: 100%;
    border: none;
    padding: 10px;
  }

  .cl-textarea:focus {
    outline: none;
  }
</style>
