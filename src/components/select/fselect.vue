<template>
    <div :class="classes" v-clickoutside="handleClose">
        <div
            :class="[prefixCls + '-selection']"
            ref="reference"
            @click="toggleMenu">
            <div class="fv-tag" v-for="(item, index) in selectedMultiple">
                <span class="fv-tag-text">{{ item.label }}</span>
                <i-icon type="ios-close-empty" @click.native.stop="removeTag(index)"></i-icon>
            </div>
            <span :class="[prefixCls + '-placeholder']" v-show="showPlaceholder && !filterable">{{ localePlaceholder }}</span>
            <span :class="[prefixCls + '-selected-value']" v-show="!showPlaceholder && !multiple && !filterable">{{ selectedSingle }}</span>
            <input
                type="text"
                v-if="filterable"
                v-model="query"
                :class="[prefixCls + '-input']"
                :placeholder="showPlaceholder ? localePlaceholder : ''"
                :style="inputStyle"
                @blur="handleBlur"
                @keydown="resetInputState"
                @keydown.delete="handleInputDelete"
                ref="input">
            <i-icon type="ios-close" :class="[prefixCls + '-arrow']" v-show="showCloseIcon" @click.native.stop="clearSingleSelect"></i-icon>
            <i-icon type="arrow-down-b" :class="[prefixCls + '-arrow']"></i-icon>
        </div>
        <transition name="slide-up">
            <f-drop v-show="visible" :placement="placement" ref="dropdown">
                <ul :class="[prefixCls + '-dropdown-list']" ref="options">
                    <!-- <slot></slot> -->
                    <f-option v-for="item in model" :key="item.value" :value="item">{{ item }}</f-option>
                    <f-option ref="query" :value="query" query :list="model">{{ query }}</f-option>
                </ul>
            </f-drop>
        </transition>
    </div>
</template>
<script>
    import iIcon from '../icon/icon.vue';
    import fDrop from './dropdown.vue';
    import fOption from './foption.vue';
    import clickoutside from '../../directives/clickoutside';
    import { oneOf, findComponentDownward } from '../../utils/assist';
    import Emitter from '../../mixins/emitter';
    import Locale from '../../mixins/locale';

    const prefixCls = 'fv-fselect';

    export default {
        name: 'fSelect',
        mixins: [ Emitter, Locale ],
        components: { iIcon, fDrop, fOption },
        directives: { clickoutside },
        props: {
            value: {
                type: [String, Number, Array],
                default: ''
            },
            placement: {
                type: String,
                default: 'bottom'
            },
            multiple: {
                type: Boolean,
                default: false
            },
            disabled: {
                type: Boolean,
                default: false
            },
            clearable: {
                type: Boolean,
                default: false
            },
            placeholder: {
                type: String
            },
            filterable: {
                type: Boolean,
                default: false
            },
            filterMethod: {
                type: Function
            },
            size: {
                validator (value) {
                    return oneOf(value, ['small', 'large', 'default']);
                }
            },
            labelInValue: {
                type: Boolean,
                default: false
            }
        },
        data () {
            return {
                prefixCls: prefixCls,
                visible: false,
                options: [],
                optionInstances: [],
                selectedSingle: '',    // label
                selectedMultiple: [],
                focusIndex: 0,
                query: '',
                inputLength: 20,
                slotChangeDuration: false,    // if slot change duration and in multiple, set true and after slot change, set false
                model: this.value
            };
        },
        computed: {
            classes () {
                return [
                    `${prefixCls}`,
                    {
                        [`${prefixCls}-visible`]: this.visible,
                        [`${prefixCls}-disabled`]: this.disabled,
                        [`${prefixCls}-multiple`]: this.multiple,
                        [`${prefixCls}-single`]: !this.multiple,
                        [`${prefixCls}-show-clear`]: this.showCloseIcon,
                        [`${prefixCls}-${this.size}`]: !!this.size
                    }
                ];
            },
            showPlaceholder () {
                let status = false;

                if ((typeof this.model) === 'string') {
                    if (this.model === '') {
                        status = true;
                    }
                } else if (Array.isArray(this.model)) {
                    if (!this.model.length) {
                        status = true;
                    }
                }

                return status;
            },
            showCloseIcon () {
                return !this.multiple && this.clearable && !this.showPlaceholder;
            },
            inputStyle () {
                let style = {};

                if (this.multiple) {
                    if (this.showPlaceholder) {
                        style.width = '100%';
                    } else {
                        style.width = `${this.inputLength}px`;
                    }
                }

                return style;
            },
            localePlaceholder () {
                if (this.placeholder === undefined) {
                    return this.t('i.select.placeholder');
                } else {
                    return this.placeholder;
                }
            }
        },
        methods: {
            toggleMenu () {
                if (this.disabled) {
                    return false;
                }

                this.visible = !this.visible;
            },
            hideMenu () {
                this.visible = false;
                this.focusIndex = 0;
                this.broadcast('fOption', 'on-select-close');
            },
            // find option component
            findChild (cb) {
                const find = function (child) {
                    const name = child.$options.componentName;

                    if (name) {
                        cb(child);
                    } else if (child.$children.length) {
                        child.$children.forEach((innerChild) => {
                            find(innerChild, cb);
                        });
                    }
                };

                if (this.optionInstances.length) {
                    this.optionInstances.forEach((child) => {
                        find(child);
                    });
                } else {
                    this.$children.forEach((child) => {
                        find(child);
                    });
                }
            },
            updateOptions (init, slot = false) {
                let options = [];
                let index = 1;

                this.findChild((child) => {
                    options.push({
                        value: child.value,
                        label: (child.label === undefined) ? child.$el.innerHTML : child.label
                    });

                    child.index = index++;

                    if (init) {
                        this.optionInstances.push(child);
                    }
                });

                this.options = options;

                if (init) {
                    this.updateSingleSelected(true, slot);
                    this.updateMultipleSelected(true, slot);
                }
            },
            updateSingleSelected (init = false, slot = false) {
                const type = typeof this.model;

                if (type === 'string' || type === 'number') {
                    let findModel = false;

                    for (let i = 0; i < this.options.length; i++) {
                        if (this.model === this.options[i].value) {
                            this.selectedSingle = this.options[i].label;
                            findModel = true;
                            break;
                        }
                    }

                    if (slot && !findModel) {
                        this.model = '';
                        this.query = '';
                    }
                }

                this.toggleSingleSelected(this.model, init);
            },
            clearSingleSelect () {
                if (this.showCloseIcon) {
                    this.findChild((child) => {
                        child.selected = false;
                    });
                    this.model = '';

                    if (this.filterable) {
                        this.query = '';
                    }
                }
            },
            updateMultipleSelected (init = false, slot = false) {
                if (this.multiple && Array.isArray(this.model)) {
                    let selected = [];

                    for (let i = 0; i < this.model.length; i++) {
                        const model = this.model[i];

                        for (let j = 0; j < this.options.length; j++) {
                            const option = this.options[j];

                            if (model === option.value) {
                                selected.push({
                                    value: option.value,
                                    label: option.label
                                });
                            }
                        }
                    }

                    this.selectedMultiple = selected;

                    if (slot) {
                        let selectedModel = [];

                        for (let i = 0; i < selected.length; i++) {
                            selectedModel.push(selected[i].value);
                        }

                        // if slot change and remove a selected option, emit user
                        if (this.model.length === selectedModel.length) {
                            this.slotChangeDuration = true;
                        }

                        // this.model = selectedModel;
                    }
                }
                this.toggleMultipleSelected(this.model, init);
            },
            removeTag (index) {
                if (this.disabled) {
                    return false;
                }
                this.model.splice(index, 1);

                if (this.filterable && this.visible) {
                    this.$refs.input.focus();
                }

                this.broadcast('fDrop', 'on-update-popper');
            },
            // to select option for single
            toggleSingleSelected (value, init = false) {
                if (!this.multiple) {
                    let label = '';

                    this.findChild((child) => {
                        if (child.value === value) {
                            child.selected = true;
                            label = (child.label === undefined) ? child.$el.innerHTML : child.label;
                        } else {
                            child.selected = false;
                        }
                    });

                    this.hideMenu();

                    if (!init) {
                        if (this.labelInValue) {
                            this.$emit('on-change', {
                                value: value,
                                label: label
                            });
                            this.dispatch('iFormItem', 'on-form-change', {
                                value: value,
                                label: label
                            });
                        } else {
                            this.$emit('on-change', value);
                            this.dispatch('iFormItem', 'on-form-change', value);
                        }
                    }
                }
            },
            // to select option for multiple
            toggleMultipleSelected (value, init = false) {
                if (this.multiple) {
                    let hybridValue = [];
                    for (let i = 0; i < value.length; i++) {
                        hybridValue.push({
                            value: value[i]
                        });
                    }

                    this.findChild((child) => {
                        const index = value.indexOf(child.value);

                        if (index >= 0) {
                            child.selected = true;
                            hybridValue[index].label = (child.label === undefined) ? child.$el.innerHTML : child.label;
                        } else {
                            child.selected = false;
                        }
                    });

                    if (!init) {
                        if (this.labelInValue) {
                            this.$emit('on-change', hybridValue);
                            this.dispatch('iFormItem', 'on-form-change', hybridValue);
                        } else {
                            this.$emit('on-change', value);
                            this.dispatch('iFormItem', 'on-form-change', value);
                        }
                    }
                }
            },
            handleClose () {
                this.hideMenu();
            },
            handleKeydown (e) {
                if (this.visible) {
                    const keyCode = e.keyCode;
                    // Esc slide-up
                    if (keyCode === 27) {
                        e.preventDefault();
                        this.hideMenu();
                    }
                    // next
                    if (keyCode === 40) {
                        e.preventDefault();
                        this.navigateOptions('next', true);
                    }
                    // prev
                    if (keyCode === 38) {
                        e.preventDefault();
                        this.navigateOptions('prev', true);
                    }
                    // enter
                    if (keyCode === 13) {
                        e.preventDefault();

                        let selectChild = null;
                        this.findChild((child) => {
                            if (child.isFocus && selectChild == null) {
                                selectChild = child;
                            }
                        });
                        selectChild.select();
                    }
                }
            },
            navigateOptions (direction, init = false) {
                if (init) {
                    let focusChild = null;
                    this.findChild((child) => {
                        if (focusChild == null && child.isFocus)
                            focusChild = child;
                    });
                    if (focusChild != null)
                        this.focusIndex = focusChild.index;
                    else
                        this.focusIndex = 0;
                }

                if (direction === 'next') {
                    const next = this.focusIndex + 1;
                    this.focusIndex = (this.focusIndex > this.options.length - 1) ? 1 : next;
                } else if (direction === 'prev') {
                    const prev = this.focusIndex - 1;
                    this.focusIndex = (this.focusIndex <= 1) ? this.options.length : prev;
                }

                let child_status = {
                    disabled: false,
                    hidden: false
                };

                this.findChild((child) => {
                    if (child.index === this.focusIndex) {
                        child_status.disabled = child.disabled;
                        child_status.hidden = child.hidden;

                        if (!child.disabled && !child.hidden) {
                            child.isFocus = true;
                        }
                    } else {
                        child.isFocus = false;
                    }
                });

                this.resetScrollTop();

                if (child_status.disabled || child_status.hidden) {
                    console.log(child_status.hidden);
                    this.navigateOptions(direction);
                }
            },
            resetScrollTop () {
                const index = this.focusIndex - 1;
                let bottomOverflowDistance = this.optionInstances[index].$el.getBoundingClientRect().bottom - this.$refs.dropdown.$el.getBoundingClientRect().bottom;
                let topOverflowDistance = this.optionInstances[index].$el.getBoundingClientRect().top - this.$refs.dropdown.$el.getBoundingClientRect().top;

                if (bottomOverflowDistance > 0) {
                    this.$refs.dropdown.$el.scrollTop += bottomOverflowDistance;
                }
                if (topOverflowDistance < 0) {
                    this.$refs.dropdown.$el.scrollTop += topOverflowDistance;
                }
            },
            handleBlur () {
                setTimeout(() => {
                    const model = this.model;

                    if (this.multiple) {
                        this.query = '';
                    } else {
                        if (model !== '') {
                            this.findChild((child) => {
                                if (child.value === model) {
                                    this.query = child.label === undefined ? child.searchLabel : child.label;
                                }
                            });
                        } else {
                            this.query = '';
                        }
                    }
                }, 300);
            },
            resetInputState () {
                this.inputLength = this.$refs.input.value.length * 12 + 20;
            },
            handleInputDelete () {
                if (this.multiple && this.model.length && this.query === '') {
                    this.removeTag(this.model.length - 1);
                }
            },
            // use when slot changed
            slotChange () {
                this.options = [];
                this.optionInstances = [];
            },
            setQuery (query) {
                if (!this.filterable) return;
                this.query = query;
            },
            modelToQuery() {
                if (!this.multiple && this.filterable && this.model) {
                    this.findChild((child) => {
                        if (this.model === child.value) {
                            if (child.label) {
                                this.query = child.label;
                            } else if (child.searchLabel) {
                                this.query = child.searchLabel;
                            } else {
                                this.query = child.value;
                            }
                        }
                    });
                }
            }
        },
        mounted () {
            this.modelToQuery();

            this.updateOptions(true);
            document.addEventListener('keydown', this.handleKeydown);

            this.$on('append', () => {
                this.modelToQuery();
                this.slotChange();
                this.updateOptions(true, true);
            });
            this.$on('remove', () => {
                this.modelToQuery();
                this.slotChange();
                this.updateOptions(true, true);
            });

            this.$on('on-select-selected', (value) => {
                if (this.model === value) {
                    this.hideMenu();
                } else {
                    if (this.multiple) {
                        const index = this.model.indexOf(value);
                        if (index >= 0) {
                            this.removeTag(index);
                        } else {
                            this.model.push(value);
                            this.broadcast('fDrop', 'on-update-popper');
                        }

                        if (this.filterable) {
                            this.query = '';
                            this.$refs.input.focus();
                        }
                    } else {
                        this.model = value;

                        if (this.filterable) {
                            this.findChild((child) => {
                                if (child.value === value) {
                                    this.query = child.label === undefined ? child.searchLabel : child.label;
                                }
                            });
                        }
                    }
                }
            });
        },
        beforeDestroy () {
            document.removeEventListener('keydown', this.handleKeydown);
        },
        watch: {
            value (val) {
                this.model = val;
                if (val === '') this.query = '';
            },
            model () {
                this.$emit('input', this.model);
                this.$emit('change', this.model);
                this.modelToQuery();
                if (this.multiple) {
                    if (this.slotChangeDuration) {
                        this.slotChangeDuration = false;
                    } else {
                        this.updateMultipleSelected();
                    }
                } else {
                    this.updateSingleSelected();
                }
            },
            visible (val) {
                if (val) {
                    if (this.multiple && this.filterable) {
                        this.$refs.input.focus();
                    }
                    this.broadcast('fDrop', 'on-update-popper');
                } else {
                    if (this.filterable) {
                        this.$refs.input.blur();
                    }
                    this.broadcast('fDrop', 'on-destroy-popper');
                }
            },
            query (val) {
                this.$emit('on-query-change', val);

                if (findComponentDownward(this, 'fOptionGroup')) {
                    this.broadcast('fOptionGroup', 'on-query-change', val);
                    this.broadcast('fOption', 'on-query-change', val);
                } else {
                    this.broadcast('fOption', 'on-query-change', val);
                }
                
                this.broadcast('fDrop', 'on-update-popper');
            }
        }
    };
</script>
