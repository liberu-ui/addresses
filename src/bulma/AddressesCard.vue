<template>
    <card collapsible
        :collapsed="collapsed">
        <card-header class="has-background-light">
            <template #title>
                <span class="icon is-small mr-1">
                    <fa :icon="icon"/>
                </span>
                {{ displayTitle }}
            </template>
            <template #controls>
                <card-refresh @refresh="fetch"/>
                <card-badge :label="count"/>
                <card-collapse/>
            </template>
        </card-header>
        <card-content class="is-paddingless">
            <addresses :id="id"
                :type="type"
                :query="query"
                @update="count = $refs.addresses.count; $refs.card.resize()"
                ref="addresses"/>
        </card-content>
    </card>
</template>

<script>
import { mapState } from 'vuex';
import {
    Card, CardHeader, CardRefresh, CardCollapse, CardBadge, CardContent,
} from '@liberu-ui/card/bulma';
import { FontAwesomeIcon as Fa } from '@fortawesome/vue-fontawesome';
import { library } from '@fortawesome/fontawesome-svg-core';
import { faMapSigns, faPlusSquare } from '@fortawesome/free-solid-svg-icons';
import Addresses from './Addresses.vue';

library.add(faMapSigns, faPlusSquare);

export default {
    name: 'AddressesCard',

    components: {
        Fa,
        Card,
        CardHeader,
        CardRefresh,
        CardCollapse,
        CardBadge,
        CardContent,
        Addresses,
    },

    inject: ['i18n'],

    props: {
        icon: {
            type: [String, Array, Object],
            default: () => faMapSigns,
        },
        id: {
            type: [String, Number],
            required: true,
        },
        type: {
            type: String,
            required: true,
        },
        collapsed: {
            type: Boolean,
            default: false,
        },
        title: {
            type: String,
            default: null,
        },
    },

    data: () => ({
        query: '',
        count: 0,
    }),

    computed: {
        ...mapState('layout', ['isMobile']),
        displayTitle() {
            return !this.isMobile
                ? this.title || this.i18n('Addresses')
                : null;
        },
        isEmpty() {
            return this.count === 0;
        },
    },

    watch: {
        count() {
            this.$refs.card.resize();
        },
    },
};
</script>
