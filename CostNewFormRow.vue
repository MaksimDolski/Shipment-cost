<template>
    <div>
        <div v-for="(item, index) in value" :key="index">
            <b-row>
                <b-col v-if="isContainerType || isOceanFreight" md="3">
                    <b-form-group label="Container Type">
                        <select-container-type v-model="item.referenceTo" :company-id="company.id" disabled type="text"
                            placeholder="Search / Select" @change="updateValue($event, 'referenceTo', index)" />
                    </b-form-group>
                </b-col>

                <b-col v-if="isDnd || isOceanFreight || isContainerType"
                    :md="isDnd ? 4 : (isOceanFreight || isContainerType ? 2 : '')">
                    <b-form-group label="Container Number">
                        <b-form-input v-model="item.containerNumber" disabled type="text"
                            placeholder="Automatic data" />
                    </b-form-group>
                </b-col>

                <b-col v-if="isBillOfLading || isTeu || isContainer" md="4" />

                <b-col v-if="isOceanFreight || isContainerType || isBillOfLading || isTeu || isContainer" md="2">
                    <b-form-group :label="$t('message.units')" class="required">
                        <ValidationProvider v-slot="{ errors }" :vid="`$t('message.units')-${index}`"
                            :name="$t('message.units')" :rules="{
            required: true,
            isNumber: true,
            decimalValidator: {
                digits: 14,
                decimals: 3,
            },
        }">
                            <b-form-input v-model="item.quantity" :class="{ 'is-invalid': errors[0] }" min="0"
                                type="number" placeholder="Select" :disabled="isBillOfLading"
                                @input="updateValue($event, 'quantity', index)" />
                            <b-form-invalid-feedback v-if="errors[0]">
                                {{ errors[0] }}
                            </b-form-invalid-feedback>
                        </ValidationProvider>
                    </b-form-group>
                </b-col>
                <b-col
                    :md="(isDnd ? 4 : (isOceanFreight || isContainerType ? 2 : (isBillOfLading || isTeu || isContainer ? 3 : '')))">
                    <b-form-group :label="$t('message.pricePerUnit')" class="required">
                        <ValidationProvider v-slot="{ errors }" :vid="`$t('message.price')-${index}`"
                            :name="$t('message.price')" :rules="{
            required: true,
            isNumber: true,
            decimalValidator: {
                digits: 15,
                decimals: 2,
            },
        }">
                            <b-form-input v-model="item.localCurrencyCostPerUnit" :class="{ 'is-invalid': errors[0] }"
                                min="0" type="number" placeholder="Select"
                                @input="updateValue($event, 'localCurrencyCostPerUnit', index)" />
                            <b-form-invalid-feedback v-if="errors[0]">
                                {{ errors[0] }}
                            </b-form-invalid-feedback>
                        </ValidationProvider>
                    </b-form-group>
                </b-col>
                <b-col :md="isDnd ? 4 : 3">
                    <b-form-group :label="`${$t('message.currency')}`" class="required">
                        <ValidationProvider v-slot="{ errors }" :vid="`$t('message.currency')-${index}`"
                            :name="$t('message.currency')" :rules="{ required: true }" tag="div">
                            <multiselect v-model="item.localCurrency" :name="$t('message.currency')"
                                data-qa-id="input-for-currency" :class="{ 'is-invalid': errors[0] }"
                                :options="currencies" :custom-label="({ name, code }) => `${name} (${code})`"
                                placeholder="Select" @input="updateValue($event, 'localCurrency', index)" />
                            <b-form-invalid-feedback v-if="errors[0]">
                                {{ errors[0] }}
                            </b-form-invalid-feedback>
                        </ValidationProvider>
                    </b-form-group>
                </b-col>
            </b-row>
            <b-row v-if="isDnd">
                <b-col md="12">
                    <b-form-group label="Comment">
                        <b-form-textarea v-model="item.comment" placeholder="Enter comment" rows="1" max-rows="5"
                            :maxlength="500" no-resize @input="updateValue($event, 'comment', index)" />
                    </b-form-group>
                </b-col>
            </b-row>

            <b-row v-if="isDnd">
                <b-col md="3">
                    <b-form-group label="Start date" class="required">
                        <select-date v-model="item.startDate" @change="updateValue($event, 'startDate', index)" />
                    </b-form-group>
                </b-col>

                <b-col md="3">
                    <b-form-group label="End date" class="required">
                        <select-date v-model="item.endDate" @change="updateValue($event, 'endDate', index)" />
                    </b-form-group>
                </b-col>
            </b-row>
        </div>
    </div>
</template>

<script>

import { mapActions, mapState } from 'vuex';
import SelectContainerType from '@/views/Workflow/Form/Components/SelectContainerType.vue';
import SelectDate from '@/views/Workflow/Form/Components/SelectDate.vue';
import Multiselect from 'vue-multiselect';

import _ from 'lodash';

export default {
    name: 'CostNewFormRow',
    components: {
        SelectContainerType,
        SelectDate,
        Multiselect,
    },
    props: {
        form: {
            type: Object,
            default: () => { },
        },
        shipment: {
            type: Object,
            default: () => { },
        },
        value: {
            type: Array,
            required: true,
        },
        getNumberOfUnits: {
            type: Function,
            default: () => { },
        },
        required: {
            type: Boolean,
            default: false,
        },
        isCostRowClicked: {
            type: Boolean,
            default: false,
        },
        isContainerType: {
            type: Boolean,
            required: true,
        },
        isBillOfLading: {
            type: Boolean,
            required: true,
        },
        isTeu: {
            type: Boolean,
            required: true,
        },
        isContainer: {
            type: Boolean,
            required: true,
        },
        isOceanFreight: {
            type: Boolean,
            required: true,
        },
        isDnd: {
            type: Boolean,
            required: true,
        },
    },
    data() {
        return {
            test: null,
        };
    },
    computed: {
        ...mapState({
            company: (state) => state.application.sidebar.company,
            currencies: (state) => state.masterdata.currencies,
        }),
    },
    async created() {
        this.getCurrencies();
    },
    methods: {
        ...mapActions([
            'getCurrencies',
        ]),
        updateValue(update, key, index) {
            const updatedValue = _.cloneDeep(this.value);
            updatedValue[index][key] = update;
            this.$emit('input', updatedValue);
        },
    },
};
</script>
