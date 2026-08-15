<template>
    <div class="shadow-box" @click="focusTerminal">
        <div v-pre ref="terminal" class="main-terminal"></div>
    </div>
</template>

<script>
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { TERMINAL_COLS, TERMINAL_ROWS } from "../../../common/util-common";

export default {
    /**
     * @type {Terminal}
     */
    terminal: null,
    terminalFitAddOn: null,
    resizeObserver: null,
    props: {
        name: {
            type: String,
            required: true,
        },

        endpoint: {
            type: String,
            required: true,
        },

        // Required if mode is interactive
        stackName: {
            type: String,
            default: "",
        },

        // Required if mode is interactive
        serviceName: {
            type: String,
            default: "",
        },

        // Required if mode is interactive
        shell: {
            type: String,
            default: "bash",
        },

        rows: {
            type: Number,
            default: TERMINAL_ROWS,
        },

        cols: {
            type: Number,
            default: TERMINAL_COLS,
        },

        // Mode
        // displayOnly: Only display terminal output
        // mainTerminal: Full interactive Dockge/host console
        // interactive: Full interactive container terminal
        mode: {
            type: String,
            default: "displayOnly",
        }
    },
    emits: [ "has-data" ],
    data() {
        return {
            first: true,
            sessionReady: false,
        };
    },
    mounted() {
        const cursorBlink = this.mode !== "displayOnly";

        this.terminal = new Terminal({
            fontSize: 14,
            fontFamily: "'JetBrains Mono', monospace",
            cursorBlink,
            cols: this.cols,
            rows: this.rows,
            scrollback: 5000,
        });

        if (this.mode === "mainTerminal" || this.mode === "interactive") {
            this.configureInteractiveInput();
        }

        // Bind xterm.js to the page before starting the PTY so startup output is visible.
        this.terminal.open(this.$refs.terminal);
        this.terminal.focus();

        // Use native browser clipboard events. They work from a user paste/copy
        // gesture on Chromium browsers even when navigator.clipboard is not
        // available on a plain HTTP LAN origin.
        this.$refs.terminal.addEventListener("paste", this.handlePasteEvent, true);
        this.$refs.terminal.addEventListener("copy", this.handleCopyEvent, true);
        this.$refs.terminal.addEventListener("contextmenu", this.handleContextMenu);

        // Notify parent component when data is received.
        this.terminal.onCursorMove(() => {
            if (this.first) {
                this.$emit("has-data");
                this.first = false;
            }
        });

        this.bind();
        this.updateTerminalSize(false);
        this.createSession();
        this.watchTerminalSize();
    },

    unmounted() {
        window.removeEventListener("resize", this.onResizeEvent);
        this.resizeObserver?.disconnect();
        if (this.mode !== "displayOnly") {
            this.$root.emitAgent(this.endpoint, "terminalLeave", this.name);
        }
        this.$root.unbindTerminal(this.name);
        this.$refs.terminal?.removeEventListener("paste", this.handlePasteEvent, true);
        this.$refs.terminal?.removeEventListener("copy", this.handleCopyEvent, true);
        this.$refs.terminal?.removeEventListener("contextmenu", this.handleContextMenu);
        this.terminal.dispose();
    },

    methods: {
        bind(endpoint, name) {
            // Workaround: normally this.name should be set, but it is not sometimes,
            // so we use the parameter. Eventually this.name and name must match.
            if (name) {
                this.$root.unbindTerminal(name);
                this.$root.bindTerminal(endpoint, name, this.terminal);
            } else if (this.name) {
                this.$root.unbindTerminal(this.name);
                this.$root.bindTerminal(this.endpoint, this.name, this.terminal);
            }
        },

        createSession() {
            if (this.mode === "mainTerminal") {
                this.$root.emitAgent(this.endpoint, "mainTerminal", this.name, (res) => {
                    if (!res.ok) {
                        this.$root.toastRes(res);
                        return;
                    }
                    this.sessionReady = true;
                    this.sendTerminalSize();
                });
            } else if (this.mode === "interactive") {
                this.$root.emitAgent(this.endpoint, "interactiveTerminal", this.stackName, this.serviceName, this.shell, (res) => {
                    if (!res.ok) {
                        this.$root.toastRes(res);
                        return;
                    }
                    this.sessionReady = true;
                    this.sendTerminalSize();
                });
            }
        },

        configureInteractiveInput() {
            // onData is the raw xterm.js input stream. Unlike the old main-console
            // line editor, it preserves TAB, arrows, escape sequences, Ctrl keys,
            // bracketed paste, and input required by ncurses applications.
            this.terminal.onData((data) => {
                this.sendInput(data);
            });

            this.terminal.attachCustomKeyEventHandler((event) => {
                if (event.type !== "keydown") {
                    return true;
                }

                const controlKey = event.ctrlKey || event.metaKey;
                const key = event.key.toLowerCase();

                // Terminal convention: Ctrl+Shift+C copies selected text. Keep plain
                // Ctrl+C available to the PTY for SIGINT.
                if (controlKey && event.shiftKey && key === "c" && this.terminal.hasSelection()) {
                    this.copySelection();
                    return false;
                }

                // Let the browser generate a real ClipboardEvent for paste. The
                // handlePasteEvent() path does not require navigator.clipboard.
                if (controlKey && key === "v") {
                    return false;
                }

                return true;
            });
        },

        sendInput(data) {
            if (!data || this.mode === "displayOnly") {
                return;
            }
            this.$root.emitAgent(this.endpoint, "terminalInput", this.name, data);
        },

        focusTerminal() {
            this.terminal?.focus();
        },

        watchTerminalSize() {
            if (typeof ResizeObserver !== "undefined") {
                this.resizeObserver = new ResizeObserver(() => {
                    this.onResizeEvent();
                });
                this.resizeObserver.observe(this.$refs.terminal);
            } else {
                window.addEventListener("resize", this.onResizeEvent);
            }
        },

        /**
         * Fit xterm.js to the current container and optionally update the PTY.
         * @param {boolean} notifyBackend Send rows/columns to the backend PTY.
         */
        updateTerminalSize(notifyBackend = true) {
            if (!this.terminalFitAddOn) {
                this.terminalFitAddOn = new FitAddon();
                this.terminal.loadAddon(this.terminalFitAddOn);
            }

            this.terminalFitAddOn.fit();
            if (notifyBackend) {
                this.sendTerminalSize();
            }
        },

        onResizeEvent() {
            this.updateTerminalSize(true);
        },

        sendTerminalSize() {
            if (!this.sessionReady || this.mode === "displayOnly") {
                return;
            }

            this.$root.emitAgent(
                this.endpoint,
                "terminalResize",
                this.name,
                this.terminal.rows,
                this.terminal.cols
            );
        },

        /**
         * Native paste event. This is more compatible on HTTP/LAN deployments than
         * navigator.clipboard.readText(), which requires a secure context.
         * @param {ClipboardEvent} event Browser paste event.
         */
        handlePasteEvent(event) {
            if (this.mode === "displayOnly") {
                return;
            }

            const text = event.clipboardData?.getData("text/plain") || "";
            if (!text) {
                return;
            }

            event.preventDefault();
            event.stopPropagation();

            // Route through xterm.js so bracketed-paste mode is respected. xterm
            // will emit the resulting input through onData(), which then reaches
            // the backend PTY.
            this.terminal.paste(text);
        },

        /**
         * Allow browser Copy/context-menu Copy to copy xterm.js selection without
         * depending on the asynchronous Clipboard API.
         * @param {ClipboardEvent} event Browser copy event.
         */
        handleCopyEvent(event) {
            const text = this.terminal.getSelection();
            if (!text || !event.clipboardData) {
                return;
            }

            event.clipboardData.setData("text/plain", text);
            event.preventDefault();
        },

        handleContextMenu() {
            // Do not suppress the browser context menu. On Chromium-based browsers,
            // its Paste action generates a ClipboardEvent that handlePasteEvent()
            // can consume, including on non-HTTPS LAN installations.
            this.terminal.focus();
        },

        async copySelection() {
            const text = this.terminal.getSelection();
            if (!text) {
                return;
            }

            // Prefer the modern API when it is permitted.
            try {
                if (window.isSecureContext && navigator.clipboard?.writeText) {
                    await navigator.clipboard.writeText(text);
                    return;
                }
            } catch (error) {
                console.debug("Clipboard API unavailable, using copy fallback", error);
            }

            // Chromium-compatible fallback for HTTP/LAN deployments.
            const textarea = document.createElement("textarea");
            textarea.value = text;
            textarea.setAttribute("readonly", "");
            textarea.style.position = "fixed";
            textarea.style.opacity = "0";
            document.body.appendChild(textarea);
            textarea.select();
            document.execCommand("copy");
            textarea.remove();
            this.terminal.focus();
        },
    }
};
</script>

<style scoped lang="scss">
.main-terminal {
    height: 100%;
}
</style>

<style lang="scss">
.terminal {
    background-color: black !important;
    height: 100%;
}
</style>
