<template>
    <span :class="wrapClasses" @click="toggle">
        <span :class="innerClasses">
            <slot name="open" v-if="currentValue"></slot>
            <slot name="close" v-if="!currentValue"></slot>
        </span>
    </span>
</template>
<script>
    import { oneOf } from '../../utils/assist';
    import Emitter from '../../mixins/emitter';

    const prefixCls = 'fv-switch';

    export default {
        name: 'iSwitch',
        mixins: [ Emitter ],
        props: {
            value: {
                type: Boolean,
                default: false
            },
            disabled: {
                type: Boolean,
                default: false
            },
            size: {
                validator (value) {
                    return oneOf(value, ['large', 'small']);
                }
            }
        },
        data () {
            return {
                currentValue: this.value
            };
        },
        computed: {
            wrapClasses () {
                return [
                    `${prefixCls}`,
                    {
                        [`${prefixCls}-checked`]: this.currentValue,
                        [`${prefixCls}-disabled`]: this.disabled,
                        [`${prefixCls}-${this.size}`]: !!this.size
                    }
                ];
            },
            innerClasses () {
                return `${prefixCls}-inner`;
            }
        },
        methods: {
            toggle () {
                if (this.disabled) {
                    return false;
                }

                const checked = !this.currentValue;
                this.currentValue = checked;
                this.$emit('input', checked);
                this.$emit('on-change', checked);
                this.dispatch('iFormItem', 'on-form-change', checked);
            }
        },
        watch: {
            value (val) {
                this.currentValue = val;
            }
        }
    };
</script>
