<template>
    <enso-form class="box has-background-light"
        :path="path"
        :params="params"
        :key="key"
        @ready="setFields"
        v-bind="$attrs"
        disable-state>
        <template #actions-left
            v-if="canLocalize">
            <a class="button is-warning"
                :class="{'loading': loading}"
                @click="localize">
                <span class="is-hidden-mobile">
                    {{ i18n('Localize') }}
                </span>
                <span class="icon">
                    <fa icon="map-pin"/>
                </span>
                <span class="is-hidden-mobile"/>
            </a>
        </template>
        <template #country_id="{ field: countryId }">
            <form-field :field="countryId"
                @update:model-value="reRender"/>
        </template>
        <template #postcode="{ field: postcodeField, errors }">
            <div class="is-fullwidth">
                <label class="label">
                    {{ i18n(postcodeField.label) }}
                </label>
                <div class="field has-addons">
                    <div class="control is-expanded">
                        <input class="input"
                            :class="['input', { 'is-danger': errors.has(postcodeField.name) }]"
                            type="text"
                            :placeholder="i18n(postcodeField.meta.placeholder)"
                            v-model="postcodeField.value"
                            @input="errors.clear(postcodeField.name)">
                    </div>
                    <div class="control"
                        v-if="canAccess('core.addresses.postcode')">
                        <a :class="['button', postcodeCss]"
                            @click="loadAddress">
                        <span class="icon">
                            <fa icon="search-location"/>
                        </span>
                        </a>
                    </div>
                </div>
                <p class="help is-danger"
                    v-if="errors.has(postcodeField.name)">
                    {{ errors.get(postcodeField.name) }}
                </p>
            </div>
        </template>
        <template #region_id="{ field: regionId, errors }">
            <form-field :field="regionId"
                @update:model-value="
                    localityParams.region_id = $event;
                    errors.clear(regionId.name);
                "/>
        </template>
        <template #locality_id="{ field: localityId, errors }">
            <form-field :field="localityId"
                :params="localityParams"
                @update:model-value="
                    errors.clear(localityId.name)
                "/>
        </template>
    </enso-form>
</template>

<script>
import { FontAwesomeIcon as Fa } from '@fortawesome/vue-fontawesome';
import { library } from '@fortawesome/fontawesome-svg-core';
import { faLocationArrow, faMapPin, faSearchLocation } from '@fortawesome/free-solid-svg-icons';

import { EnsoForm, FormField } from '@enso-ui/forms/bulma';

library.add(faLocationArrow, faMapPin, faSearchLocation);

export default {
    name: 'AddressForm',

    components: {
        Fa, EnsoForm, FormField,
    },

    inject: ['canAccess', 'errorHandler', 'http', 'i18n', 'route'],

    inheritAttrs: false,

    props: {
        addressableId: {
            type: [String, Number],
            required: true,
        },
        id: {
            type: [Number, null],
            required: true,
        },
        type: {
            type: String,
            default: null,
        },
    },

    emits: ['form-loaded'],

    data: () => ({
        key: 1,
        form: null,
        loading: false,
        postcode: null,
        params: {
            countryId: null,
        },
        localityParams: {
            region_id: null,
        },
    }),

    computed: {
        canLocalize() {
            return this.form && this.form.routeParam('address')
                && this.canAccess('core.addresses.localize');
        },
        field() {
            return this.form && this.form.field;
        },
        path() {
            return this.id
                ? this.route('core.addresses.edit', this.id)
                : this.route('core.addresses.create', this.params);
        },
        postcodeCss() {
            if (this.postcode === null) {
                return 'is-info';
            }

            return this.postcode
                ? 'is-success'
                : 'is-danger';
        },
    },

    methods: {
        localize() {
            this.loading = true;
            const address = this.form.routeParam('address');

            this.http.get(this.route('core.addresses.localize', address))
                .then(({ data }) => {
                    const { lat, long } = data;
                    this.field('lat').value = lat;
                    this.field('long').value = long;
                }).catch(this.errorHandler)
                .finally(() => (this.loading = false));
        },
        reRender(countryId) {
            this.params.countryId = countryId;
            this.key++;
        },
        setFields({ form }) {
            this.form = form;
            this.field('addressable_id').value = this.addressableId;
            this.field('addressable_type').value = this.type;
            this.localityParams.region_id = this.field('region_id').value;
            this.$emit('form-loaded', form);
        },
        loadAddress() {
            this.loading = true;
            this.postcode = null;

            const params = {
                params: {
                    postcode: this.field('postcode').value,
                    country_id: this.field('country_id').value,
                },
            };

            this.http.get(this.route('core.addresses.postcode'), params)
                .then(({ data: { postcode } }) => {
                    ['lat', 'long', 'city', 'region_id', 'locality_id', 'street']
                        .forEach(key => (this.field(key).value = postcode[key]
                        || this.field(key).value));
                    this.postcode = true;
                }).catch(error => {
                    const { status, data } = error.response;
                    this.postcode = false;

                    if (status === 422) {
                        this.form.errors.set(data.errors);
                        this.$nextTick(this.form.focusError);
                    } else {
                        this.errorHandler(error);
                    }
                }).finally(() => (this.loading = false));
        },
    },
};
</script>
