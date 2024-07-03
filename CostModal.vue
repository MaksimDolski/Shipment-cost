<template>
    <div>
        <b-btn size="md" class="min-width-8getUnit" @click="handleCreate">
            + {{ $t('message.addCost') }}
        </b-btn>
        <div>
            <b-modal id="addCostModal" title="Add cost" size="lg" hide-footer @ok.prevent="onFormSubmit">
                <div>
                    <ValidationObserver ref="addCostRef" vid="addCost" tag="div">
                        <b-form ref="form" autocomplete="off" @submit.prevent="onFormSubmit">
                            <b-form-group label="">
                                <b-row>
                                    <b-col md="1" class="mr-3">
                                        <b-form-radio v-model="formNameChecking" disabled value="1">
                                            Planned
                                        </b-form-radio>
                                    </b-col>
                                    <b-col md="1">
                                        <b-form-radio v-model="formNameChecking" disabled value="2">
                                            Running
                                        </b-form-radio>
                                    </b-col>
                                </b-row>
                            </b-form-group>
                            <b-form-group>
                                <b-row>
                                    <b-col md="6">
                                        <b-form-group :label="$t('message.costsType')" class="required">
                                            <ValidationProvider v-slot="{ errors }" :name="$t('message.costsName')"
                                                :rules="{
            required: true,
        }">
                                                <select-charge :v-model="form.code || value[0]?.code"
                                                    :class="{ 'is-invalid': errors[0] }" :company-id="company.id"
                                                    @change-option="(option) => handleChargeChange(option)" />
                                                <b-form-invalid-feedback v-if="errors[0]">
                                                    {{ errors[0] }}
                                                </b-form-invalid-feedback>
                                            </ValidationProvider>
                                        </b-form-group>
                                    </b-col>
                                    <b-col md="3">
                                        <b-form-group label="Direction" class="required">
                                            <select-enum :v-model="form.applianceType || value[0]?.applianceType"
                                                placeholder="Search / Select Status" :enum="getDirectionOptions"
                                                @change="updateDirection" />
                                        </b-form-group>
                                    </b-col>
                                    <b-col v-if="!isDnd" md="3">
                                        <b-form-group label="Scale">
                                            <b-form-input :v-model="form.scale || value[0]?.scale" disabled type="text"
                                                placeholder="Automatic data" />
                                        </b-form-group>
                                    </b-col>
                                </b-row>
                                <cost-new-form-row v-model="value" :form="form" :shipment="shipment"
                                    :get-number-of-units="getNumberOfUnits" :is-dnd="isDnd"
                                    :is-ocean-freight="isOceanFreight" :is-container="isContainer" :is-teu="isTeu"
                                    :is-bill-of-lading="isBillOfLading" :is-container-type="isContainerType"
                                    :is-cost-row-clicked="isCostRowClicked" />
                                <b-row>
                                    <b-col md="6">
                                        <b-form-group :label="$t('message.vendor')" class="required">
                                            <ValidationProvider v-slot="{ errors }" :name="$t('message.vendor')" :rules="{
            required: true,
        }">
                                                <select-business-partner :class="{ 'is-invalid': errors[0] }"
                                                    :v-model="value[0]?.vendorId" :company-id="company.id"
                                                    :disabled="isCarrierVendorCheckBox"
                                                    :placeholder="isCarrierVendorCheckBox ? 'No data' : 'Search / Select Business Partner'"
                                                    :default-options="[
            {
                label: company.name,
                value: company.id,
                partnerCompanyId: company.id,
            },
        ]" @change="handleInput($event, 'vendor')" />
                                                <b-form-invalid-feedback v-if="errors[0]">
                                                    {{ errors[0] }}
                                                </b-form-invalid-feedback>
                                            </ValidationProvider>
                                        </b-form-group>
                                    </b-col>
                                    <b-col md="6">
                                        <ValidationProvider v-slot="{ errors }" :name="$t('message.payer')"
                                            :rules="{ required: true }">
                                            <b-form-group :label="$t('message.payer')" class="required">
                                                <select-business-partner :class="{ 'is-invalid': errors[0] }"
                                                    :company-id="company.id" :v-model="value[0]?.payerId"
                                                    :default-options="[
            {
                label: company.name,
                value: company.id,
                partnerCompanyId: company.id,
            },
        ]" @input="handleInput($event, 'payer')" />
                                                <b-form-invalid-feedback v-if="errors[0]">
                                                    {{ errors[0] }}
                                                </b-form-invalid-feedback>
                                            </b-form-group>
                                        </ValidationProvider>
                                    </b-col>
                                </b-row>
                                <b-row>
                                    <b-col>
                                        <b-form-checkbox v-model="isCarrierVendorCheckBox" class="no-text-select">
                                            To be paid to carrier
                                        </b-form-checkbox>
                                    </b-col>
                                </b-row>
                            </b-form-group>
                        </b-form>
                    </ValidationObserver>
                </div>
                <div>
                    <div v-if="!isCostRowClicked">
                        <b-button class="float-right w-15" @click="handleSubmit">
                            {{ $t('message.addCost') }}
                        </b-button>

                        <b-button class="float-right mr-2 w-15" variant="outline-primary"
                            @click="hideModal(); isCostRowClicked = false">
                            {{ $t('message.cancel') }}
                        </b-button>
                    </div>

                    <div v-if="isCostRowClicked">
                        <b-button class="float-right w-15" @click="fetchUpdateCosts">
                            {{ $t('message.updateCost') }}
                        </b-button>

                        <b-button class="float-right mr-2 w-15" variant="outline-primary"
                            @click="hideModal(); isCostRowClicked = false">
                            {{ $t('message.cancel') }}
                        </b-button>
                    </div>
                </div>
            </b-modal>
        </div>
    </div>
</template>
<script>

import { mapActions, mapState } from 'vuex';
import SelectBusinessPartner from '@/views/Workflow/Form/Components/SelectBusinessPartner.vue';
import SelectCharge from '@/views/Workflow/Form/Components/SelectCharge.vue';
import _ from 'lodash';
import CostNewFormRow from '@/views/Shipments/Track/TrackTabs/CostsTab/CostNewFormRow.vue';
import SelectEnum from '@/views/Workflow/Form/Components/SelectEnum.vue';
import cleanDeep from 'clean-deep';

export default {
    name: 'CostModal',
    components: {
        SelectCharge,
        SelectBusinessPartner,
        CostNewFormRow,
        SelectEnum,
    },
    props: {
        shipment: {
            type: Object,
            required: true,
        },
        isCostRowClicked: {
            type: Boolean,
            default: false,
        },
        isCarrierVendorCheckBox: {
            type: Boolean,
            default: false,
        },
        fetchCosts: {
            type: Function,
            default: () => { },
        },
        value: {
            type: Array,
            required: true,
        },
    },
    data() {
        return {
            vendorId: null,
            payerId: null,
            vendorBusinessPartnerId: null,
            payerBusinessPartnerId: null,
            partners: [],
            partner: null,
            form: {
                code: null,
                name: null,
                applianceType: null,
                scale: null,
            },
            containersTypeDataFromEndpoint: [],
        };
    },
    computed: {
        ...mapState({
            company: (state) => state.application.sidebar.company,
            user: (state) => state.auth.user,
            containerTypes: (state) => state.masterdata.containerTypes,
        }),
        isContainerType() {
            if (this.isCostRowClicked) {
                return this.value[0].scale === 'ContainerType';
            }
            return this.form.scale === 'ContainerType';
        },
        isBillOfLading() {
            if (this.isCostRowClicked) {
                return this.value[0].scale === 'BL';
            }
            return this.form.scale === 'BL';
        },
        isTeu() {
            if (this.isCostRowClicked) {
                return this.value[0].scale === 'TEU';
            }
            return this.form.scale === 'TEU';
        },
        isContainer() {
            if (this.isCostRowClicked) {
                return this.value[0].scale === 'CNT';
            }
            return this.form.scale === 'CNT';
        },
        isOceanFreight() {
            if (this.isCostRowClicked) {
                return this.value[0].scale === 'Ocean Freight';
            }
            return this.form.name === 'Ocean Freight';
        },
        isDnd() {
            const dndNames = [
                'Demurrage',
                'Detention',
                'Storage',
                'Storage, Demurrage and Detention',
                'Storage and Demurrage',
                'Demurrage and Detention',
            ];
            if (this.isCostRowClicked) {
                return dndNames.includes(this.value[0].name);
            }
            return dndNames.includes(this.form.name);
        },
        getDirectionOptions() {
            const directionOptions = [
                { value: 'O', label: 'Origin' },
                { value: 'M', label: 'Base' },
                { value: 'D', label: 'Destination' },
            ];
            if (this.isDnd) {
                return directionOptions.filter((option) => option.value === 'O' || option.value === 'D');
            }
            if (this.isOceanFreight) {
                return directionOptions.filter((option) => option.value === 'M');
            }
            return directionOptions;
        },
        formNameChecking: {
            get() {
                if (this.isDnd) {
                    return '2';
                }
                if (this.shipment.currentStatus.code >= 330) {
                    return '2';
                }
                if (this.shipment.currentStatus.code < 330) {
                    return '1';
                }
                return '';
            },
            set(value) {
                return value;
            },
        },
    },
    created() {
        this.getContainerTypesList();
        this.getFilings();
        this.getContainerType();
        this.containersTypeDataFromEndpoint = this.getContainersTypeDataFromEndpoint();
    },
    methods: {
        ...mapActions([
            'getContainerTypesList',
            'showNotification',
            'postCostData',
            'getContainerTypes',
            'getPartners_Stateless',
            'updateCosts',
        ]),
        async fetchUpdateCosts() {
            const data = {
                shipmentId: this.shipment.id,
                companyId: this.company.id,
                costs: this.value.map((el) => ({
                    id: el.id,
                    type: el.type,
                    code: el.code,
                    name: el.name,
                    applianceType: el.applianceType,
                    scale: this.getScale(),
                    quantity: Number(this.isDnd ? 1 : el.quantity),
                    localCurrency: el.localCurrency,
                    localCurrencyCostPerUnit: Number(el.localCurrencyCostPerUnit),
                    vendorCarrier: !!this.isCarrierVendorCheckBox,
                    vendorId: this.getVendorId(),
                    vendorBusinessPartnerId: this.getVendorBusinessPartnerId(),
                    payerId: el.payerId,
                    payerBusinessPartnerId: el.payerBusinessPartnerId,
                    contractId: null,
                    contractNumber: null,
                    referenceTo: el.referenceTo,
                    updatedByUserId: el.updatedByUserId,
                    plannedContainerId: el.plannedContainerId,
                    actualContainerId: el.actualContainerId,
                    additionalData: {
                        startDate: el.startDate ? new Date(el.startDate).toISOString() : null,
                        endDate: el.endDate ? new Date(el.endDate).toISOString() : null,
                        comment: el.comment,
                    },
                })),
            };
            this.updateCosts(cleanDeep(data))
                .then(() => {
                    this.fetchCosts();
                    this.value = [];
                    this.$bvModal.hide('addCostModal');
                });
        },
        updateDirection(newValue) {
            if (this.form.applianceType) {
                this.form.applianceType = newValue;
            } else {
                this.value[0].applianceType = newValue;
            }
        },
        async getFilings() {
            this.partners = await this.getPartners_Stateless({ companyId: this.company.id });
            if (this.company.id) {
                this.partner = this.partners.unshift(this.company);
            }
        },
        isVendorOrPayer(option, businessPartner) {
            if (this.company && this.partner) {
                if (this.company.id === this.partner.id) {
                    this[option] = this.partner.id || null;
                    this[businessPartner] = null;
                } else {
                    this[option] = this.partner.partnerCompanyId || null;
                    this[businessPartner] = this.partner.id || null;
                }
            }
        },
        async handleInput(value, option) {
            if (_.isArray(this.partners) && !_.isEmpty(this.partners)) {
                this.partner = this.partners.find((el) => el.id === value);
            }
            if (option === 'vendor') {
                this.isVendorOrPayer('vendorId', 'vendorBusinessPartnerId');
            }
            if (option === 'payer') {
                this.isVendorOrPayer('payerId', 'payerBusinessPartnerId');
            }
        },
        getContainersData(el) {
            return {
                id: el?.id ?? null,
                quantity: el?.count ?? 1,
                referenceTo: el?.type ?? null,
                name: el?.name ?? null,
                localCurrencyCostPerUnit: null,
                localCurrency: null,
                containerNumber: el?.number ?? null,
            };
        },
        getContainersDataFromShipment() {
            if (this.isBillOfLading || this.isTeu || this.isContainer) {
                this.value.push(this.getContainersData());
                this.value = this.value.map((el) => {
                    el.count = this.getNumberOfUnits();
                    return el;
                });
                return this.value;
            }
            if (this.isDnd || this.isContainerType || this.isOceanFreight) {
                if (this.shipment.containers.length > 0) {
                    this.shipment.containers.forEach((el) => {
                        this.value.push(this.getContainersData(el));
                    });
                    return this.value;
                }
                if (this.shipment.containerTypes.length > 0) {
                    this.shipment.containerTypes.forEach((el) => {
                        this.value.push(this.getContainersData(el));
                    });
                }
            }
            return this.value;
        },
        getNumberOfUnits() {
            if (this.isBillOfLading) {
                return 1;
            }
            if (this.isContainer) {
                if (this.shipment.containers.length > 0) {
                    return this.shipment.containers.length;
                }
                if (this.shipment.containerTypes.length > 0) {
                    return Object.values(this.shipment.containerTypes.map((el) => el.count)).reduce((a, b) => a + b, 0);
                }
            }
            if (this.isTeu) {
                if (this.shipment.containers.length > 0) return this.getContainersSum(this.shipment.containers);
                if (this.shipment.containerTypes.length > 0) return this.getContainersSum(this.shipment.containerTypes);
            }
            return null;
        },
        async getContainerType() {
            if (_.isEmpty(this.getContainerTypes)) {
                await this.getContainerTypes(this.company.id);
            }
        },
        getContainersTypeDataFromEndpoint() {
            this.containerTypes
                .flatMap((el) => el.value)
                .map((el) => {
                    this.containersTypeDataFromEndpoint.push({
                        code: el.code,
                        description: el.description,
                        name: el.name,
                        teu: el.teu,
                    });
                    return this.containersTypeDataFromEndpoint;
                });
            return this.containersTypeDataFromEndpoint;
        },
        getContainersSum(container) {
            let sum = 0;
            const containerDataHash = {};
            container.forEach((el) => {
                containerDataHash[el.type] = el.count ? el.count : 1;
            });
            this.containersTypeDataFromEndpoint.forEach((el) => {
                if (containerDataHash[el.code]) {
                    sum += (el.teu * containerDataHash[el.code]);
                }
            });
            return sum;
        },
        handleCreate() {
            this.isCostRowClicked = false;
            this.value = [];
            this.$nextTick(() => {
                this.$root.$emit('bv::show::modal', 'addCostModal');
            });
        },
        handleChargeChange(option) {
            this.value = [];
            if (option) {
                if (this.isCostRowClicked) {
                    this.value[0].code = option.value;
                    this.value[0].name = option.label;
                    this.value[0].scale = option.scale;
                    this.value[0].applianceType = option.type;
                } else {
                    this.form.code = option.value;
                    this.form.name = option.label;
                    this.form.scale = option.scale;
                    this.form.applianceType = option.type;
                    this.getContainersDataFromShipment();
                }
            } else {
                this.form.code = null;
                this.form.name = null;
                this.form.scale = null;
                this.form.applianceType = null;
            }
        },
        hideModal() {
            this.form.applianceType = null;
            this.form.code = null;
            this.form.name = null;
            this.form.scale = null;
            this.isCarrierVendorCheckBox = false;
            this.value = [];
            this.$bvModal.hide('addCostModal');
        },
        getScale() {
            if (this.isDnd) {
                return null;
            }
            return this.form.scale;
        },
        getVendorId() {
            if (this.isCarrierVendorCheckBox) {
                this.vendorId = null;
            }
            return this.vendorId;
        },
        getVendorBusinessPartnerId() {
            if (this.isCarrierVendorCheckBox) {
                this.vendorBusinessPartnerId = null;
            }
            return this.vendorBusinessPartnerId;
        },
        getAdditionalData(el) {
            if (this.isDnd || this.isOceanFreight) {
                return {
                    startDate: el.startDate ? new Date(el.startDate).toISOString() : null,
                    endDate: el.endDate ? new Date(el.endDate).toISOString() : null,
                    comment: el.comment,
                };
            }
            return null;
        },
        getContainersId(el) {
            if (!this.isBillOfLading || !this.isTeu || !this.isContainer) {
                return {
                    actualContainerId: this.shipment.containers.length > 0 && el.id ? el.id : null,
                    plannedContainerId: this.shipment.containers.length > 0 ? null : el.id,
                };
            }
            return {
                actualContainerId: null,
                plannedContainerId: null,
            };
        },
        async handleSubmit() {
            this.$refs.addCostRef.validate().then(async (success) => {
                if (!success) {
                    this.showNotification({
                        type: 'error',
                        message: 'message.pleaseCheckAllFields',
                    });
                    return;
                }
                const data = {
                    shipmentId: this.shipment.id,
                    companyId: this.company.id,
                    costs: this.value.map((el) => ({
                        code: this.form.code,
                        name: this.form.name,
                        payerId: this.payerId,
                        payerBusinessPartnerId: this.payerBusinessPartnerId,
                        creatorUserId: null,
                        contractId: null,
                        contractNumber: null,
                        referenceTo: el.referenceTo,
                        vendorCarrier: this.isCarrierVendorCheckBox,
                        localCurrency: el.localCurrency,
                        applianceType: this.form.applianceType,
                        localCurrencyCostPerUnit: Number(el.localCurrencyCostPerUnit),
                        quantity: Number(this.isDnd ? 1 : el.quantity),
                        scale: this.getScale(),
                        type: Number(this.formNameChecking),
                        vendorId: this.getVendorId(),
                        vendorBusinessPartnerId: this.getVendorBusinessPartnerId(),
                        additionalData: this.getAdditionalData(el),
                        actualContainerId: this.getContainersId(el).actualContainerId,
                        plannedContainerId: this.getContainersId(el).plannedContainerId,
                    })),
                };
                this.postCostData(cleanDeep(data))
                    .then(() => {
                        this.fetchCosts();
                        this.value = [];
                        this.isCarrierVendorCheckBox = false;
                        this.$bvModal.hide('addCostModal');
                    });
            });
        },
    },
};

</script>
<style scoped>
.no-text-select {
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
}
</style>
