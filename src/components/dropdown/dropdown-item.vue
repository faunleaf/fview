<template>
    <li :class="classes" @click="handleClick"><slot></slot></li>
</template>
<script>
    const prefixCls = 'fv-dropdown-item';

    export default {
        name: 'iDropdownItem',
        props: {
            name: {
                type: [String, Number]
            },
            disabled: {
                type: Boolean,
                default: false
            },
            selected: {
                type: Boolean,
                default: false
            },
            divided: {
                type: Boolean,
                default: false
            }
        },
        computed: {
            classes () {
                return [
                    `${prefixCls}`,
                    {
                        [`${prefixCls}-disabled`]: this.disabled,
                        [`${prefixCls}-selected`]: this.selected,
                        [`${prefixCls}-divided`]: this.divided
                    }
                ];
            }
        },
        methods: {
            handleClick () {
                const $parent = this.$parent.$parent.$parent;
                const hasChildren = this.$parent && this.$parent.$options.name === 'iDropdown';

                if (this.disabled) {
                    this.$nextTick(() => {
                        $parent.currentVisible = true;
                    });
                } else if (hasChildren) {
                    this.$parent.$emit('on-haschild-click');
                } else {
                    if ($parent && $parent.$options.name === 'iDropdown') {
                        $parent.$emit('on-hover-click');
                    }
                }
                $parent.$emit('on-click', this.name);
            }
        }
    };
</script>
