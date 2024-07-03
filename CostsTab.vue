<template>
    <div class="track-wrapper">
        <div v-if="loading">
            <loading :text="$t('message.loadingWithMoreText', { key: 'shipment' })" />
        </div>
        <div v-else>
            <ValidationObserver ref="shipmentCostTabRef" vid="shipmentCostTab" slim>
                <b-row>
                    <b-col>
                        <div v-if="hasSimplifiedShipmentCostsAccess">
                            <div v-if="editMode" class="d-flex w-50">
                                <ValidationProvider ref="shipmentCostRef" v-slot="{ errors }" :rules="{
            decimalValidator: { digits: 15, decimals: 2 },
        }" :name="$t('message.shipmentCost')" class="flex-grow-1" tag="div">
                                    <b-form-group :label="`${$t('message.shipmentCost')} (USD):`" class="mb-0"
                                        label-for="shipment-cost">
                                        <b-form-input id="shipment-cost" v-model="shipmentCost"
                                            :class="{ 'is-invalid': errors[0] }" data-qa-id="input-shipment-cost" />
                                        <b-form-invalid-feedback v-if="errors[0]">
                                            {{ errors[0] }}
                                        </b-form-invalid-feedback>
                                    </b-form-group>
                                </ValidationProvider>
                                <b-btn data-qa-id="btn-save-shipment-cost" class="ml-2 min-width-82 mT-23" size="md"
                                    :disabled="loading" @click="updateShipmentCostHandler">
                                    <i v-if="loading" aria-hidden="true" class="fa fa-spinner fa-spin" />
                                    {{ $t('message.save') }}
                                </b-btn>
                            </div>
                            <div v-else>
                                <span class="no-result">{{ $t('message.shipmentCost') }} (USD): {{ shipmentCostValue
                                    }}</span>
                            </div>
                        </div>
                        <div v-if="hasShipmentCostsAccess || hasAccessToShipmentCostsFromParties">
                            <b-table v-if="totalCosts" class="mb-3 mt-3 mt-3 w-50 custom-table text-center "
                                :items="totalCosts" />
                            <cost-modal v-model="containersData" class="mb-3 mt-3" :shipment="shipment"
                                :fetch-costs="fetchCosts" :is-cost-row-clicked="isCostRowClicked"
                                :is-carrier-vendor-check-box="isCarrierVendorCheckBox" />
                            <div v-if="isEmpty" data-qa-id="data-no-records"
                                class="no-result d-flex justify-content-start">
                                {{ $t('message.noCosts') }}
                            </div>
                            <b-row v-else class="m-0">
                                <b-table :items="items.costs" :fields="fields"
                                    class="custom-table table-has-child text-center">
                                    <template #cell(type)="data">
                                        <div>{{ getCostType(data.item.type) }}</div>
                                    </template>
                                    <template #cell(vendor)="data">
                                        <div v-if="data.item.vendorBusinessPartnerId || data.item.vendorId">
                                            <select-business-partner
                                                :value="data.item.vendorBusinessPartnerId || data.item.vendorId"
                                                :company-id="company.id" placeholder="" rules="required" label-only
                                                :default-options="[
            {
                label: company.name,
                value: company.id,
                partnerCompanyId: company.id,
            },
        ]" />
                                        </div>
                                        <div v-else-if="data.item.vendorName">
                                            {{ getVendorCarrier(data.item.vendorName) }}
                                        </div>
                                    </template>
                                    <template #cell(payer)="data">
                                        <select-business-partner
                                            :value="data.item.payerBusinessPartnerId || data.item.payerId"
                                            :company-id="company.id" placeholder="" label-only :default-options="[
            {
                label: company.name,
                value: company.id,
                partnerCompanyId: company.id,
            },
        ]" />
                                    </template>
                                    <template #cell(price)="data">
                                        {{ data.item.totalCostLocalCurrency }}
                                        {{ data.item.localCurrency }}
                                        {{ `(${data.item.quantity} x` }}
                                        {{ data.item.localCurrencyCostPerUnit }}
                                        {{ `${data.item.localCurrency})` }}
                                    </template>
                                    <template #cell(totalCostBaseCurrency)="data">
                                        {{ data.item?.totalCostBaseCurrency + ' USD' }}
                                    </template>
                                    <template #cell(approvalStatus)="data">
                                        <div class="br">
                                            {{ wordsSeparator(data.item.approvalStatus) }}
                                        </div>
                                        {{ data.item.approvedBy }}
                                    </template>
                                    <template #cell(referenceTo)="data">
                                        {{ getContainerTypeName(data.item.referenceTo) }}
                                    </template>
                                    <template #cell(created)="data">
                                        <div class="br">
                                            {{ dateFormat(data.item.createdAt) }}
                                        </div>
                                        {{ data.item.creatorUserId }}
                                    </template>
                                    <template #cell(updated)="data">
                                        <div class="br">
                                            {{ dateFormat(data.item.updatedAt) }}
                                        </div>
                                        {{ data.item.updatedByUserId }}
                                    </template>
                                    <template #cell(actions)="data">
                                        <i v-if="data.item.isActive && data.item.payerId !== companyId"
                                            v-b-tooltip.hover.top="'Active'" class="far fa-check-circle"
                                            :style="`font-size: 1.35rem; color: #28a745;`" />
                                        <i v-if="!data.item.isActive && data.item.payerId !== companyId"
                                            v-b-tooltip.hover.top="'Not Active'" class="far fa-times-circle"
                                            :style="`font-size: 1.35rem;`" />
                                        <div v-if="data.item.payerId === companyId">
                                            <b-form-checkbox :checked="data.item.isActive" switch
                                                @change="handleSubmit(data.item.id, data.item.isActive)" />
                                        </div>
                                    </template>
                                    <template #cell(moreData)="data">
                                        <i v-if="data.item.additionalData" :id="`more-data-${data.index}`"
                                            :style="`font-size: 1.35rem;`" aria-hidden="true" class="ca-icon-info" />
                                        <b-popover :target="`more-data-${data.index}`" triggers="hover">
                                            <div v-if="data.item.additionalData">
                                                <div><b>Additional Data:</b></div>
                                                <div>Start Date: {{ dateFormat(data.item.additionalData.startDate) }}
                                                </div>
                                                <div>End Date: {{ dateFormat(data.item.additionalData.endDate) }}</div>
                                                <div>Comment: {{ data.item.additionalData.comment || 'No data' }}</div>
                                            </div>
                                        </b-popover>
                                    </template>
                                    <template #cell(edit)="data">
                                        <i v-if="isCreator && data.item.approvalStatus === REQUESTED"
                                            :style="`font-size: 1.35rem;`" @click="getData(data.item)">
                                            <i aria-hidden="true" class="ca-icon-edit" />
                                        </i>
                                    </template>

                                    <template #cell(delete)="data">
                                        <div>
                                            <i v-if="isCreator && data.item.approvalStatus === REQUESTED"
                                                aria-hidden="true" class="ca-icon-delete" style="font-size: 1.35rem;"
                                                @click="showCostModal(data.item.id)" />
                                        </div>
                                    </template>
                                </b-table>
                            </b-row>
                            <b-modal v-model="costModal" centered lazy>
                                <template #modal-header>
                                    <h5 class="modal-title">
                                        {{ $t('message.deleteCostHeader') }}
                                    </h5>
                                </template>
                                {{ $t('message.deleteCostModal') }}
                                <template #modal-footer>
                                    <b-btn class="p-1 w-15" variant="primary" @click="fetchDeleteCosts()">
                                        Proceed
                                    </b-btn>
                                    <b-btn class="mr-1 p-1 w-15" @click="hideCostModal">
                                        Cancel
                                    </b-btn>
                                </template>
                            </b-modal>
                        </div>
                    </b-col>
                </b-row>
            </ValidationObserver>
        </div>
    </div>
</template>

<script>

import { mapActions, mapState } from 'vuex';
import _ from 'lodash';
import { findParty } from '@/infrastructure/mixins/common/methods';
import CostModal from '@/views/Shipments/Track/TrackTabs/CostsTab/CostModal.vue';
import { allowSimplifiedShipmentCostsAccess, allowShipmentCostsAccess } from '@/helpers/modules';
import SelectBusinessPartner from '@/views/Workflow/Form/Components/SelectBusinessPartner.vue';
import { PLANNING, RUNNING } from '@/constants/shipment/costsTypes';
import Loading from '@/components/Common/Loading.vue';
import { dateFormat } from '@/helpers/dates';
import { getActionIconColor, getFlowIconColor } from '@/constants/workflow/workflowLog';
import cleanDeep from 'clean-deep';
import { REQUESTED } from '@/constants/company/approvalStatusTypes';

export default {
    name: 'CostsTab',
    components: {
        CostModal,
        SelectBusinessPartner,
        Loading,
    },
    mixins: [
        findParty,
    ],
    props: {
        hasAccessToShipmentCostsFromParties: {
            type: Boolean,
            default: false,
        },
        shipment: {
            type: Object,
            required: true,
        },
        companyId: {
            type: Number,
            default: 0,
        },
        shipmentId: {
            type: Number,
            default: 0,
        },
        shipmentCostValue: {
            type: Number,
            default: null,
        },
        editMode: {
            type: Boolean,
            default: false,
        },
        isCreator: {
            type: Boolean,
            default: false,
        },
        action: {
            type: String,
            default: () => 'getCost_Stateless',
        },
    },
    data() {
        return {
            REQUESTED,
            shipmentCost: null,
            loading: false,
            items: {},
            popoverShow: false,
            costModal: false,
            deleteCostId: null,
            containersData: [],
            isCostRowClicked: false,
            isCarrierVendorCheckBox: false,
        };
    },
    computed: {
        ...mapState({
            company: (state) => state.application.sidebar.company,
            user: (state) => state.auth.user,
            containerTypes: (state) => state.masterdata.containerTypes,
            carriers: (state) => state.masterdata.carriers,
            isEmpty() {
                return _.isEmpty(this.items);
            },
        }),
        fields() {
            return [
                {
                    key: 'type',
                    label: this.$t('message.costsType'),
                },
                {
                    key: 'name',
                    label: this.$t('message.costsName'),
                },
                {
                    key: 'applianceType',
                    label: this.$t('message.applianceType'),
                },
                {
                    key: 'scale',
                    label: this.$t('message.scale'),
                },
                {
                    key: 'referenceTo',
                    label: this.$t('message.referenceTo'),
                },
                {
                    key: 'price',
                    label: this.$t('message.price'),
                },
                {
                    key: 'totalCostBaseCurrency',
                    label: this.$t('message.totalCostBaseCurrency'),
                },
                {
                    key: 'contractNumber',
                    label: this.$t('message.contractNumber'),
                },
                {
                    key: 'vendor',
                    label: this.$t('message.vendor'),
                },
                {
                    key: 'payer',
                    label: this.$t('message.payer'),
                },
                {
                    key: 'approvalStatus',
                    label: this.$t('message.approvalStatus'),
                },
                {
                    key: 'created',
                    label: this.$t('message.createdAt'),
                },
                {
                    key: 'updated',
                    label: this.$t('message.updatedAt'),
                },
                {
                    key: 'actions',
                    label: this.$t('message.actions'),
                },
                {
                    key: 'moreData',
                    label: this.$t('message.moreData'),
                },
                {
                    key: 'edit',
                    label: this.$t('message.edit'),
                },
                {
                    key: 'delete',
                    label: this.$t('message.delete'),
                },
            ];
        },
        hasSimplifiedShipmentCostsAccess() {
            return !!allowSimplifiedShipmentCostsAccess(
                this.user,
                this.company,
            );
        },
        hasShipmentCostsAccess() {
            return !!allowShipmentCostsAccess(
                this.user,
                this.company,
            );
        },
        totalCosts() {
            if (this.items?.activePlannedCostsTotal || this.items?.activeRunningCostsTotal) {
                return [
                    {
                        label: 'Total planned cost',
                        value: this.items.activePlannedCostsTotal ? this.items.activePlannedCostsTotal : '-',
                    },
                    {
                        label: 'Total running cost',
                        value: this.items.activeRunningCostsTotal ? this.items.activeRunningCostsTotal : '-',
                    },
                ];
            }
            return null;
        },
    },
    async created() {
        this.shipmentCost = this.shipmentCostValue;
        this.fetchCosts();
    },

    methods: {
        getActionIconColor,
        getFlowIconColor,
        ...mapActions([
            'showNotification',
            'updateShipmentCostAsync',
            'updateSelectedShipment',
            'getCosts',
            'deleteCosts',
            'getCost_Stateless',
            'putCostStatusData',
            'getCompanyNameById',
        ]),
        getData(data) {
            console.log(data);
            this.isCarrierVendorCheckBox = data.vendorCarrier;
            this.isCostRowClicked = true;
            this.$nextTick(() => {
                this.$root.$emit('bv::show::modal', 'addCostModal');
            });
            this.containersData = [{
                id: data?.id,
                type: data?.type,
                code: data?.code,
                name: data?.name,
                applianceType: data?.applianceType,
                scale: data?.scale ?? null,
                quantity: data?.quantity ?? null,
                localCurrency: data?.localCurrency,
                localCurrencyCostPerUnit: data?.localCurrencyCostPerUnit,
                vendorId: data?.vendorId || data?.vendorBusinessPartnerId,
                payerId: data?.payerId || data?.payerBusinessPartnerId,
                contractId: data.contractId ?? null,
                contractNumber: data.contractNumber ?? null,
                referenceTo: data.referenceTo ?? null,
                updatedByUserId: data.updatedByUserId ?? null,
                plannedContainerId: data.plannedContainerId ?? null,
                actualContainerId: data.actualContainerId ?? null,
                containerNumber: data.containerNumber ?? null,
                startDate: data.additionalData?.startDate ?? null,
                endDate: data.additionalData?.endDate ?? null,
                comment: data.additionalData?.comment ?? null,
            }];
        },
        showCostModal(id) {
            this.deleteCostId = id;
            this.costModal = true;
        },
        hideCostModal() {
            this.costModal = false;
            this.deleteCostId = null;
        },
        getVendorCarrier(vendorName) {
            const match = this.carriers.find((el) => el.code === vendorName);
            if (match) {
                return match.name;
            }
            return null;
        },
        getContainerTypeName(referenceTo) {
            const containerType = this.containerTypes
                .flatMap((el) => el.value)
                .find((el) => el.code === referenceTo);
            return containerType ? containerType.name : referenceTo;
        },
        dateFormat,
        async updateShipmentCostHandler() {
            await this.$refs.shipmentCostTabRef?.validate().then(async (success) => {
                if (success) {
                    this.loading = true;
                    await this.updateShipmentCostAsync({
                        companyId: this.companyId,
                        shipmentId: this.shipmentId,
                        body: {
                            shipmentCost: !_.isNaN(parseFloat(this.shipmentCost))
                                ? Number(this.shipmentCost) : null,
                        },
                    })
                        .then(() => {
                            this.updateSelectedShipment({
                                shipmentCost: !_.isNaN(parseFloat(this.shipmentCost))
                                    ? Number(this.shipmentCost) : null,
                            });
                        })
                        .finally(() => {
                            this.loading = false;
                        });
                }
            });
        },
        async fetchCosts() {
            this.loading = true;
            this.items = await this.getCosts({
                shipmentId: this.shipmentId,
                companyId: this.companyId,
            });
            this.loading = false;
        },
        async fetchDeleteCosts() {
            const id = this.deleteCostId;
            if (id !== null) {
                await this.deleteCosts({
                    companyId: this.companyId,
                    shipmentId: this.shipmentId,
                    costsIds: [id],
                });
            }
            this.fetchCosts();
            this.hideCostModal();
        },
        wordsSeparator(word) {
            return word.replace(/([A-Z])/g, ' $1').trim();
        },
        getCostType(type) {
            if (type === PLANNING) return 'Planning';
            if (type === RUNNING) return 'Running';
            return '-';
        },
        async handleSubmit(id, isActive) {
            this.$refs.shipmentCostTabRef.validate().then(async (success) => {
                if (!success) {
                    this.showNotification({
                        type: 'error',
                    });
                    return;
                }
                const data = {
                    companyId: this.company?.id,
                    shipmentId: this.shipment?.id,
                    costId: id || null,
                    isActive: isActive !== true,
                };
                this.putCostStatusData(cleanDeep(data))
                    .then(() => {
                        this.fetchCosts();
                    });
            });
        },
    },
};
</script>

<style scoped>
.br {
    word-break: break-word;
}
</style>
