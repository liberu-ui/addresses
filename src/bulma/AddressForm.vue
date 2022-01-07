<template>
    <modal>
        <enso-form class="box has-background-light"
            v-bind="$attrs"
            :params="params"
            :key="key"
            @ready="setFields"
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
            <template #country_id="{ field }">
                <form-field :field="field"
                    @update:model-value="rerender"/>
            </template>
            <template #postcode="{ field, errors }">
                <div class="is-fullwidth">
                    <label class="label">
                        {{ i18n(field.label) }}
                    </label>
                    <div class="field has-addons">
                        <div class="control is-expanded">
                            <input class="input"
                                :class="['input', { 'is-danger': errors.has(field.name) }]"
                                type="text"
                                :placeholder="i18n(field.meta.placeholder)"
                                v-model="field.value"
                                @input="errors.clear(field.name)">
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
                        v-if="errors.has(field.name)">
                        {{ errors.get(field.name) }}
                    </p>
                </div>
            </template>
            <template #region_id="{ field, errors }">
                <form-field :field="field"
                    @update:model-value="
                        localityParams.region_id = $event;
                        errors.clear(field.name);
                    "/>
            </template>
            <template #locality_id="{ field, errors }">
                <form-field :field="field"
                    :params="localityParams"
                    @update:model-value="
                        errors.clear(field.name)
                    "/>
            </template>
        </enso-form>
    </modal>
</template>

<script>
import { FontAwesomeIcon as Fa } from '@fortawesome/vue-fontawesome';
import { library } from '@fortawesome/fontawesome-svg-core';
import { faLocationArrow, faMapPin, faSearchLocation } from '@fortawesome/free-solid-svg-icons';
import { Modal } from '@enso-ui/modal/bulma';
import { EnsoForm, FormField } from '@enso-ui/forms/bulma';

library.add(faLocationArrow, faMapPin, faSearchLocation);

export default {
    name: 'AddressForm',

    components: {
        Fa, Modal, EnsoForm, FormField,
    },

    inject: ['canAccess', 'errorHandler', 'http', 'i18n', 'route'],

    inheritAttrs: false,

    props: {
        id: {
            type: [String, Number],
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
        rerender(countryId) {
            this.params.countryId = countryId;
            this.key++;
        },
        setFields({ form }) {
            this.form = form;
            this.field('addressable_id').value = this.id;
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
