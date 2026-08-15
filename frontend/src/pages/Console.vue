<template>
    <transition name="slide-fade" appear>
        <div v-if="!processing">
            <div class="d-flex flex-wrap justify-content-between align-items-center gap-2 mb-3">
                <div>
                    <h1 class="mb-1">{{ $t("console") }}</h1>
                    <div v-if="enableConsole" class="text-muted small">
                        Full xterm.js PTY: TAB, arrows, ncurses/TUI apps, resize, copy and paste.
                    </div>
                </div>

                <div v-if="enableConsole && targets.length > 1" class="btn-group" role="group" aria-label="Console target">
                    <button
                        v-for="target in targets"
                        :key="target.id"
                        type="button"
                        class="btn"
                        :class="selectedTarget === target.id ? 'btn-primary' : 'btn-normal'"
                        @click="selectedTarget = target.id"
                    >
                        {{ target.label }}
                    </button>
                </div>
            </div>

            <div v-if="enableConsole && selectedTarget === 'host'" class="alert alert-warning py-2 mb-3" role="alert">
                PVE Host Shell is an SSH-backed host session. Commands here run with the configured host account privileges.
            </div>

            <Terminal
                v-if="enableConsole"
                :key="terminalName"
                class="terminal"
                :rows="20"
                mode="mainTerminal"
                :name="terminalName"
                :endpoint="endpoint"
            ></Terminal>

            <div v-else class="alert alert-warning shadow-box" role="alert">
                <h4 class="alert-heading">{{ $t("Console is not enabled") }}</h4>
                <i18n-t keypath="ConsoleNotEnabledMSG1" tag="p">
                    <template #docker><code>{{ $t('dockerCode') }}</code></template>
                    <template #rm><code>{{ $t('rmCode') }}</code></template>
                </i18n-t>

                <i18n-t keypath="ConsoleNotEnabledMSG2" tag="p">
                    <template #rmRf>
                        <code>{{ $t('rmRfCode') }}</code>
                    </template>
                </i18n-t>

                <i18n-t keypath="ConsoleNotEnabledMSG3" tag="p">
                    <template #envVar>
                        <code>{{ $t('envVarCode') }}</code>
                    </template>
                </i18n-t>
            </div>
        </div>
    </transition>
</template>

<script>
export default {
    data() {
        return {
            processing: true,
            enableConsole: false,
            targets: [],
            selectedTarget: "local",
        };
    },
    computed: {
        endpoint() {
            return this.$route.params.endpoint || "";
        },
        terminalName() {
            return this.selectedTarget === "host" ? "console-host" : "console";
        },
    },
    mounted() {
        this.$root.emitAgent(this.endpoint, "checkMainTerminal", (res) => {
            this.enableConsole = res.ok;
            this.targets = Array.isArray(res.targets) ? res.targets : [ { id: "local", label: "Dockge Shell" } ];
            this.selectedTarget = res.defaultTarget || this.targets[0]?.id || "local";
            this.processing = false;
        });
    },
};
</script>

<style scoped lang="scss">
.terminal {
    height: min(70vh, 760px);
    min-height: 410px;
}
</style>
