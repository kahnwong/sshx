<!-- @component Interactive terminal rendered with xterm.js -->
<script lang="ts" context="module">
  // Deduplicated terminal font loading.
  const waitForFonts = (() => {
    let state: "initial" | "loading" | "loaded" = "initial";
    const waitlist: (() => void)[] = [];

    return async function waitForFonts() {
      if (state === "loaded") return;
      else if (state === "initial") {
        const FontFaceObserver = (await import("fontfaceobserver")).default;
        state = "loading";
        try {
          await new FontFaceObserver("Fira Code VF").load();
        } catch (error) {
          console.warn("Could not load terminal font", error);
        }
        state = "loaded";
        for (const fn of waitlist) fn();
      } else {
        await new Promise<void>((resolve) => {
          if (state === "loaded") resolve();
          else waitlist.push(resolve);
        });
      }
    };
  })();

  // Patch xterm to remove data requests triggering spurious messages when replayed.
  //
  // This removes support for several commands, which is not great for full feature support, but
  // without the patch the requests cause problems because they cause the terminal to send data
  // before any user interactions, so the data is duplicated with multiple connections.
  //
  // Search the xterm.js source for calls to "triggerDataEvent" to understand why these specific
  // functions were patched.
  //
  // I'm so sorry about this. In the future we should parse ANSI sequences correctly on the server
  // side and pass them through a state machine that filters such status requests and replies to
  // them exactly once, while being transparent to the sshx client.
  const patchXTerm = (() => {
    let patched = false;

    /* eslint-disable @typescript-eslint/no-explicit-any, @typescript-eslint/no-empty-function */
    return function patchXTerm(term: any) {
      if (patched) return;
      patched = true;

      // Hack: This requires monkey-patching internal XTerm methods.
      const Terminal = term._core.constructor;
      const InputHandler = term._core._inputHandler.constructor;

      Terminal.prototype._handleColorEvent = () => {};
      Terminal.prototype._reportFocus = () => {};
      InputHandler.prototype.unhook = function () {
        this._data = new Uint32Array(0);
        return true;
      };
      InputHandler.prototype.sendDeviceAttributesPrimary = () => {};
      InputHandler.prototype.sendDeviceAttributesSecondary = () => {};
      InputHandler.prototype.deviceStatus = () => {};
      InputHandler.prototype.deviceStatusPrivate = () => {};
      const windowOptions = InputHandler.prototype.windowOptions;
      InputHandler.prototype.windowOptions = function (params: any): boolean {
        if (params.params[0] === 18) {
          return true; // GetWinSizeChars
        } else {
          return windowOptions.call(this, params);
        }
      };
    };
    /* eslint-enable @typescript-eslint/no-explicit-any, @typescript-eslint/no-empty-function */
  })();
</script>

<script lang="ts">
  import { browser } from "$app/env";

  import { createEventDispatcher, onDestroy, onMount } from "svelte";
  import type { Terminal } from "xterm";
  import { Buffer } from "buffer";
  import { MinusIcon, PlusIcon, XIcon } from "svelte-feather-icons";

  import themes from "./themes";

  const theme = themes.defaultDark;

  /** Used to determine Cmd versus Ctrl keyboard shortcuts. */
  const isMac = browser && navigator.platform.startsWith("Mac");

  const dispatch = createEventDispatcher<{
    data: Uint8Array;
    move: { x: number; y: number };
    moving: { x: number; y: number };
    close: void;
    focus: void;
  }>();

  export let rows: number, cols: number;
  export let write: (data: string) => void; // bound function prop

  let termEl: HTMLDivElement;
  let term: Terminal | null = null;

  let loaded = false;
  let currentTitle = "Remote Terminal";

  const preloadBuffer: string[] = [];

  write = (data: string) => {
    if (!term) {
      // Before the terminal is loaded, push data into a buffer.
      preloadBuffer.push(data);
    } else {
      term.write(data);
    }
  };

  $: term?.resize(cols, rows);

  onMount(async () => {
    const { Terminal } = await import("xterm");
    const { WebLinksAddon } = await import("xterm-addon-web-links");

    await waitForFonts();

    term = new Terminal({
      allowTransparency: false,
      cursorBlink: false,
      cursorStyle: "block",
      // This is the monospace font family configured in Tailwind.
      fontFamily:
        '"Fira Code VF", ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace',
      fontSize: 14,
      fontWeight: 400,
      fontWeightBold: 500,
      lineHeight: 1.06,
      scrollback: 5000,
      theme,
    });
    patchXTerm(term);

    // Keyboard shortcuts for natural text editing.
    term.attachCustomKeyEventHandler((event) => {
      if (
        (isMac && event.metaKey && !event.ctrlKey && !event.altKey) ||
        (!isMac && !event.metaKey && event.ctrlKey && !event.altKey)
      ) {
        if (event.key === "ArrowLeft") {
          dispatch("data", new Uint8Array([0x01]));
          return false;
        } else if (event.key === "ArrowRight") {
          dispatch("data", new Uint8Array([0x05]));
          return false;
        } else if (event.key === "Backspace") {
          dispatch("data", new Uint8Array([0x15]));
          return false;
        }
      }
      return true;
    });

    term.loadAddon(new WebLinksAddon());

    term.open(termEl);
    term.resize(cols, rows);
    term.onTitleChange((title) => {
      currentTitle = title;
    });

    loaded = true;
    for (const data of preloadBuffer) {
      term.write(data);
    }

    const utf8 = new TextEncoder();
    term.onData((data: string) => {
      dispatch("data", utf8.encode(data));
    });
    term.onBinary((data: string) => {
      dispatch("data", Buffer.from(data, "binary"));
    });
  });

  onDestroy(() => term?.dispose());

  // Mouse handler logic
  let [dragging, prevMouseX, prevMouseY] = [false, 0, 0];

  function handleDrag(event: MouseEvent, start = false) {
    if (start) {
      dragging = true;
      [prevMouseX, prevMouseY] = [event.pageX, event.pageY];
    } else if (dragging) {
      dispatch("moving", {
        x: event.pageX - prevMouseX,
        y: event.pageY - prevMouseY,
      });
    }
  }

  function handleDragEnd(event: MouseEvent) {
    if (!dragging) return;
    dispatch("move", {
      x: event.pageX - prevMouseX,
      y: event.pageY - prevMouseY,
    });
    dragging = false;
  }

  onMount(() => {
    window.addEventListener("mousemove", handleDrag);
    window.addEventListener("mouseup", handleDragEnd);
    window.addEventListener("mouseleave", handleDragEnd);
    return () => {
      window.removeEventListener("mousemove", handleDrag);
      window.removeEventListener("mouseup", handleDragEnd);
      window.removeEventListener("mouseleave", handleDragEnd);
    };
  });
</script>

<div
  class="term-container opacity-95"
  style:background={theme.background}
  on:mousedown={() => dispatch("focus")}
>
  <div
    class="flex select-none"
    on:mousedown={(event) => handleDrag(event, true)}
  >
    <div class="flex-1 flex items-center px-3">
      <div class="flex space-x-2 text-transparent hover:text-black/75">
        <button
          class="w-3 h-3 p-[1px] rounded-full bg-red-500 active:bg-red-700"
          on:mousedown|stopPropagation
          on:click={() => dispatch("close")}
        >
          <XIcon class="w-full h-full" strokeWidth={3} />
        </button>
        <button
          class="w-3 h-3 p-[1px] rounded-full bg-yellow-500 active:bg-yellow-700"
          on:mousedown|stopPropagation
        >
          <MinusIcon class="w-full h-full" strokeWidth={3} />
        </button>
        <button
          class="w-3 h-3 p-[1px] rounded-full bg-green-500 active:bg-green-700"
          on:mousedown|stopPropagation
        >
          <PlusIcon class="w-full h-full" strokeWidth={3} />
        </button>
      </div>
    </div>
    <div class="flex-shrink-0 p-2 text-sm text-gray-300 font-bold">
      {currentTitle}
    </div>
    <div class="flex-1" />
  </div>
  <div
    class="inline-block px-4 py-2 transition-opacity duration-500"
    bind:this={termEl}
    style:opacity={loaded ? 1.0 : 0.0}
  />
</div>

<style lang="postcss">
  .term-container {
    @apply inline-block rounded-lg border border-gray-600 transition-transform duration-200;
  }
</style>