<template>
    <div class="flex items-center">
        <div class="input-group w-auto">
            <popover name="gradients" class="gradient-field" placement="bottom-start" @opened="handlePopoverOpen">
                <template #trigger>
                    <div
                        class="input-group-prepend px-px"
                        v-tooltip="__('Pick Gradient')"
                    >
                        <div class="relative flex items-center outline-none">
                            <div class="inline-block cursor-pointer rounded m-0 p-[2px]" >
                                <div
                                    class="rounded-sm overflow-hidden w-8 h-8"
                                    :class="{ 'border dark:border-dark-900': !value, 'cursor-not-allowed': isReadOnly }"
                                    :style="{ 'background-image': 'url(\'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMElEQVQ4T2N89uzZfwY8QFJSEp80A+OoAcMiDP7//483HTx//hx/Ohg1gIFx6IcBALl+VXknOCvFAAAAAElFTkSuQmCC\')' }"
                                >
                                    <div
                                        class="w-8 h-8"
                                        :style="{ 'background': value }"
                                    />
                                </div>
                            </div>
                        </div>
                    </div>
                </template>

                <template #default="{ close: closePopover }">
                    <div class="gradient-field-popover">
                        <div class="p-2" v-if="config.allow_any_gradient">
                            <vue-gpickr v-if="gradient" v-model="gradient" @input="gradientSelected" />
                        </div>

                        <div class="px-4 pb-4" :class="{ 'pt-4': !config.allow_any_gradient }">
                            <div class="input-group">
                                <input
                                    class="input-text"
                                    type="text"
                                    :readonly="isReadOnly || !config.allow_any_gradient"
                                    :value="value"
                                    @input="customGradientEntered"
                                    @blur="sanitizeCustomGradient"
                                />

                                <div
                                    class="input-group-append px-px"
                                    v-tooltip="__('Copy to clipboard')"
                                >
                                    <button 
                                        class="copy-btn"
                                        type="button"
                                        v-clipboard:copy="value"
                                        v-clipboard:success="onCopy"
                                        >
                                        <svg v-if="copied" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" width="24" height="24">
                                            <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M5.75 12.8665L8.33995 16.4138C9.15171 17.5256 10.8179 17.504 11.6006 16.3715L18.25 6.75"></path>
                                        </svg>

                                        <svg v-else xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" width="24" height="24">
                                            <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M6.5 15.25V15.25C5.5335 15.25 4.75 14.4665 4.75 13.5V6.75C4.75 5.64543 5.64543 4.75 6.75 4.75H13.5C14.4665 4.75 15.25 5.5335 15.25 6.5V6.5"></path>
                                            <rect width="10.5" height="10.5" x="8.75" y="8.75" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" rx="2"></rect>
                                        </svg>
                                    </button>
                                </div>
                            </div>
                        </div>

                        <div v-if="formattedAndValidatedPresets.length" class="presets">
                            <div 
                                v-for="(preset, index) in formattedAndValidatedPresets" 
                                :index="index" 
                                v-tooltip="preset"
                                @click="selectPreset(preset)"
                                class="preset"
                                :class="{ 'border dark:border-dark-900': !value, 'cursor-not-allowed': isReadOnly }"
                                :style="{ 'background-image': 'url(\'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMElEQVQ4T2N89uzZfwY8QFJSEp80A+OoAcMiDP7//483HTx//hx/Ohg1gIFx6IcBALl+VXknOCvFAAAAAElFTkSuQmCC\')' }"
                                >
                                <div
                                    class="w-8 h-8"
                                    :style="{ 'background': preset }"
                                />
                            </div>
                        </div>
                    </div>
                </template>
            </popover>
        </div>

        <button v-if="value" class="btn-close rtl:mr-1 ltr:ml-1" :aria-label="__('Reset')" @click="resetGradient">&times;</button>
    </div>
</template>

<script>
import { VueGpickr, LinearGradient } from 'vue-gpickr';
import VueClipboard from 'vue-clipboard2'
import rgbHex from 'rgb-hex';

Vue.use(VueClipboard)

let globalOpacity = 100;

let copied = false

let gradient = null;

export default {

    mixins: [Fieldtype],

    components: {
        VueGpickr
    },

    data() {
        return {
            gradient,
            globalOpacity,
            copied
        };
    },

    mounted() {
        if(!this.value) {
            let defaultGradient = this.replaceRgbWithHex(this.config.default_gradient)

            if(this.isValidLinearGradient(defaultGradient)) {
                this.setGradient(defaultGradient)
            }
        }
    },

    computed: {
        orderedStops() {
            return this.gradient.stops.slice().sort((a, b) => a[1] - b[1]);
        },

        formattedAndValidatedPresets() {
            let presets = this.config.presets;
            presets = presets.map(preset => this.replaceRgbWithHex(preset));
            presets = presets.filter(preset => this.isValidLinearGradient(preset));
            return presets;
        },

        replicatorPreview() {
            if (! this.showFieldPreviews || ! this.config.replicator_preview) return;

            return this.value
                ? replicatorPreviewHtml(`<span class="little-dot" style="background:${this.value}"></span>`)
                : null;
        }
    },
    

    methods: {
        handlePopoverOpen () {
            if(!this.gradient) {
                this.setGradient('linear-gradient(90deg, #00ff00 0%, #ff0000 100%)')
            }
        },

        isValidLinearGradient(str) {
            const gradientRegex = /^linear-gradient\(\s*(?:-?\d+deg\s*,\s*)?(?:#[0-9a-fA-F]{3,8}\s+\d+%)(?:,\s*#[0-9a-fA-F]{3,8}\s+\d+%)*\s*\)$/;
            return gradientRegex.test(str);
        },
        
        replaceRgbWithHex(str) {
            return str.replace(/rgba?\(.*?\)/g, (match) => {
                return `#${rgbHex(match)}`;
            });
        },

        sanitizeCustomGradient () {
            if(this.isValidLinearGradient(this.value)) {
                this.update(this.value);
            }
        },

        customGradientEntered(value) {
            this.setGradient(value);
        },

        setGradient(value) {
            const regex = /\(([^)]*)\)/;
            const match = regex.exec(value);

            if (match && match.length > 1) {
                let parts = match[1].split(',').map(value => value.trim());
                let angle = parts[0];
                let stops = parts.slice(1);
                let stopsArray = [];

                stops.forEach((stop) => {
                    let parts = stop.split(' ')
                    stopsArray.push([parts[0], parts[1].replace('%','') / 100])
                })

                this.gradient = new LinearGradient({
                    angle: angle.replace('deg',''),
                    stops: stopsArray
                });

                this.updateDebounced(value)
            }
        },

        gradientSelected() {
            const stops = this.orderedStops.map(stop => `${stop[0].toString()} ${Math.round(stop[1] * 100)}%`).join(', ');
            this.value = `linear-gradient(${this.gradient.angle}deg, ${stops})`;
            this.updateDebounced(this.value)
        },

        selectPreset(preset) {
            if(!this.isReadOnly) {
                this.value = preset
                this.setGradient(preset)
            }
        },

        onCopy: function (e) {
            this.copied = true;
            setTimeout(() => this.copied = false, 2000)
        },

        resetGradient() {
            this.gradient = null;
            this.update(null);
        },
    },
};
</script>