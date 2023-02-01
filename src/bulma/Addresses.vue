<template>
    <div class="addresses-wrapper">
        <div class="field is-grouped">
            <slot name="controls"
                :create="() => (form = true)"
                :internal-query="internalQuery"
                :fetch="fetch">
                <p class="control">
                    <a class="button is-small is-info is-rounded is-bold"
                        @click="form = true">
                        <span>
                            {{ i18n('New Address') }}
                        </span>
                        <span class="icon">
                            <fa icon="plus"/>
                        </span>
                    </a>
                </p>
                <p class="control has-icons-left has-icons-right is-expanded">
                    <input class="input is-rounded is-small"
                        v-model="internalQuery"
                        :placeholder="i18n('Filter')">
                    <span class="icon is-small is-left">
                        <fa icon="search"/>
                    </span>
                    <span class="icon is-small is-right clear-button"
                        @click="internalQuery = ''"
                        v-if="internalQuery">
                        <a class="delete is-small"/>
                    </span>
                </p>
                <p class="control">
                    <a class="button is-small is-rounded is-bold"
                        @click="fetch()">
                        <span>
                            {{ i18n('Reload') }}
                        </span>
                        <span class="icon">
                            <fa icon="sync"/>
                        </span>
                    </a>
                </p>
            </slot>
        </div>
        <div class="columns is-multiline mt-3">
            <div class="column is-half-tablet"
                v-for="(address, index) in filteredAddresses"
                :key="index">
                <address-card :address="address"
                    @make-default="make('default', address)"
                    @make-shipping="make('shipping', address)"
                    @make-billing="make('billing', address)"
                    @edit="addressId = address.id; form = true"
                    @delete="destroy(address, index)"/>
            </div>
        </div>
        <modal @close="form = null; addressId = null; fetch()"
            v-if="form">
            <address-form :id="addressId"
                @submit="submit"
                :addressable-id="id"
                :type="type"/>
        </modal>
    </div>
</template>

<script>
import { faPlus, faSync, faSearch } from '@fortawesome/free-solid-svg-icons';
import { FontAwesomeIcon as Fa } from '@fortawesome/vue-fontawesome';
import { library } from '@fortawesome/fontawesome-svg-core';
import { Modal } from '@enso-ui/modal/bulma';
import AddressCard from './AddressCard.vue';
import AddressForm from './AddressForm.vue';

library.add(faPlus, faSync, faSearch);

export default {
    name: 'Addresses',

    components: { Modal, Fa, AddressCard, AddressForm },

    inject: ['errorHandler', 'i18n', 'http', 'route'],

    props: {
        id: {
            type: [String, Number],
            required: true,
        },
        type: {
            type: String,
            default: null,
        },
        query: {
            type: String,
            default: '',
        },
    },

    emits: ['update'],

    data: () => ({
        addressId: null,
        form: false,
        loading: false,
        addresses: [],
        internalQuery: '',
    }),

    computed: {
        filteredAddresses() {
            const query = this.internalQuery.toLowerCase();

            return query
                ? this.addresses.filter(
                    ({ city, locality, street }) => street.toLowerCase().indexOf(query) > -1
                        || locality?.toLowerCase().indexOf(query) > -1
                        || city?.toLowerCase().indexOf(query) > -1,
                )
                : this.addresses;
        },
        count() {
            return this.filteredAddresses.length;
        },
        params() {
            return {
                addressable_id: this.id,
                addressable_type: this.type,
            };
        },
    },

    watch: {
        query() {
            this.internalQuery = this.query;
        },
    },

    created() {
        this.fetch();
    },

    methods: {
        fetch() {
            this.loading = true;

            this.http.get(this.route('core.addresses.index'), { params: this.params })
                .then(({ data }) => {
                    this.addresses = data;
                    this.$emit('update');
                    this.loading = false;
                }).catch(this.errorHandler);
        },
        make(type, address) {
            this.loading = true;
            const method = type.charAt(0).toUpperCase() + type.slice(1);

            this.http.patch(this.route(`core.addresses.make${method}`, address.id))
                .then(() => this.fetch())
                .catch(this.errorHandler);
        },
        destroy(address, index) {
            this.loading = true;

            this.http.delete(this.route('core.addresses.destroy', address.id))
                .then(() => {
                    this.addresses.splice(index, 1);
                    this.$emit('update');
                    this.loading = false;
                }).catch(this.errorHandler);
        },
        submit({ address }) {
            if (address) {
                this.addressId = address.id;
            }
        },
    },
};
</script>
