<template>
    <div>
        <i-input-number :max="10" :min="-1" v-model="v1"></i-input-number>
        {{ v1 }}
        <div @click="c">change v1</div>
        <i-input-number disabled :max="10" :min="1" :step="1.2" v-model="v2"></i-input-number>
    </div>
</template>
<script>
    export default {
        props: {},
        data () {
            return {
                v1: 1,
                v2: 1
            };
        },
        computed: {},
        methods: {
            c () {
                this.v1 = 5;
            }
        }
    };
</script>