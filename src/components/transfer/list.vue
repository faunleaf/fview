<template>
    <div :class="classes" :style="style">
        <div :class="prefixCls + '-header'">
            <i-checkbox :value="checkedAll" :disabled="checkedAllDisabled" @on-change="toggleSelectAll"></i-checkbox>
            <span>{{ title }}</span>
            <span :class="prefixCls + '-header-count'">{{ count }}</span>
        </div>
        <div :class="bodyClasses">
            <div :class="prefixCls + '-body-search-wrapper'" v-if="filterable">
                <i-search
                    :prefix-cls="prefixCls + '-search'"
                    :query="query"
                    @on-query-clear="handleQueryClear"
                    @on-query-change="handleQueryChange"
                    :placeholder="filterPlaceholder"></i-search>
            </div>
            <ul :class="prefixCls + '-content'">
                <li
                    v-for="item in filterData"
                    :class="itemClasses(item)"
                    @click.prevent="select(item)">
                    <i-checkbox :value="isCheck(item)" :disabled="item.disabled"></i-checkbox>
                    <span v-html="showLabel(item)"></span>
                </li>
                <li :class="prefixCls + '-content-not-found'">{{ notFoundText }}</li>
            </ul>
        </div>
        <div :class="prefixCls + '-footer'" v-if="showFooter"><slot></slot></div>
    </div>
</template>
<script>
    import iSearch from './search.vue';
    import iCheckbox from '../checkbox/checkbox.vue';

    export default {
        name: 'iTransferList',
        components: { iSearch, iCheckbox },
        props: {
            prefixCls: String,
            data: Array,
            renderFormat: Function,
            checkedKeys: Array,
            style: Object,
            title: [String, Number],
            filterable: Boolean,
            filterPlaceholder: String,
            filterMethod: Function,
            notFoundText: String,
            validKeysCount: Number
        },
        data () {
            return {
                showItems: [],
                query: '',
                showFooter: true
            };
        },
        watch: {
            data () {
                this.updateFilteredData();
            }
        },
        computed: {
            classes () {
                return [
                    `${this.prefixCls}`,
                    {
                        [`${this.prefixCls}-with-footer`]: this.showFooter
                    }
                ];
            },
            bodyClasses () {
                return [
                    `${this.prefixCls}-body`,
                    {
                        [`${this.prefixCls}-body-with-search`]: this.filterable,
                        [`${this.prefixCls}-body-with-footer`]: this.showFooter
                    }
                ];
            },
            count () {
                const validKeysCount = this.validKeysCount;
                return (validKeysCount > 0 ? `${validKeysCount}/` : '') + `${this.data.length}`;
            },
            checkedAll () {
                return this.data.filter(data => !data.disabled).length === this.validKeysCount && this.validKeysCount !== 0;
            },
            checkedAllDisabled () {
                return this.data.filter(data => !data.disabled).length <= 0;
            },
            filterData () {
                return this.showItems.filter(item => this.filterMethod(item, this.query));
            }
        },
        methods: {
            itemClasses (item) {
                return [
                    `${this.prefixCls}-content-item`,
                    {
                        [`${this.prefixCls}-content-item-disabled`]: item.disabled
                    }
                ];
            },
            showLabel (item) {
                return this.renderFormat(item);
            },
            isCheck (item) {
                return this.checkedKeys.some(key => key === item.key);
            },
            select (item) {
                if (item.disabled) return;
                const index = this.checkedKeys.indexOf(item.key);
                index > -1 ? this.checkedKeys.splice(index, 1) : this.checkedKeys.push(item.key);
            },
            updateFilteredData () {
                this.showItems = this.data;
            },
            toggleSelectAll (status) {
                const keys = status ?
                        this.data.filter(data => !data.disabled || this.checkedKeys.indexOf(data.key) > -1).map(data => data.key) :
                        this.data.filter(data => data.disabled && this.checkedKeys.indexOf(data.key) > -1).map(data => data.key);
                this.$emit('on-checked-keys-change', keys);
            },
            handleQueryClear () {
                this.query = '';
            },
            handleQueryChange (val) {
                this.query = val;
            }
        },
        created () {
            this.updateFilteredData();
        },
        mounted () {
            this.showFooter = this.$slots.default !== undefined;
        }
    };
</script>
