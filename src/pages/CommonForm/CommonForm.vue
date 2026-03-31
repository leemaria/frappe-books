<template>
  <FormContainer :use-full-width="useFullWidth">
    <template v-if="hasDoc" #header-left>
      <Barcode
        v-if="canShowBarcode"
        class="h-8"
        @item-selected="(name:string) => {
          // @ts-ignore
          doc?.addItem(name);
        }"
      />
      <ExchangeRate
        v-if="canShowExchangeRate"
        :disabled="doc?.isSubmitted || doc?.isCancelled"
        :from-currency="fromCurrency"
        :to-currency="toCurrency"
        :exchange-rate="exchangeRate"
        @change="
          async (exchangeRate: number) =>
            await doc.set('exchangeRate', exchangeRate)
        "
      />
      <p
        v-if="schema.label && !(canShowBarcode || canShowExchangeRate)"
        class="text-xl font-semibold items-center text-gray-600"
      >
        {{ schema.label }}
      </p>
    </template>
    <template v-if="hasDoc" #header>
      <Button
        v-if="canShowLinks"
        :icon="true"
        :title="t`View linked entries`"
        @click="showLinks = true"
      >
        <feather-icon name="link" class="w-4 h-4"></feather-icon>
      </Button>
      <Button
        v-if="canPrint"
        ref="printButton"
        :icon="true"
        :title="t`Open Print View`"
        @click="routeTo(`/print/${doc.schemaName}/${doc.name}`)"
      >
        <feather-icon name="printer" class="w-4 h-4"></feather-icon>
      </Button>
      <Button
        :icon="true"
        :title="t`Toggle between form and full width`"
        @click="toggleWidth"
      >
        <feather-icon
          :name="useFullWidth ? 'minimize' : 'maximize'"
          class="w-4 h-4"
        ></feather-icon>
      </Button>
      <DropdownWithActions
        v-for="group of groupedActions"
        :key="group.label"
        :type="group.type"
        :actions="group.actions"
      >
        <p v-if="group.group">
          {{ group.group }}
        </p>
        <feather-icon v-else name="more-horizontal" class="w-4 h-4" />
      </DropdownWithActions>
      <Button v-if="doc?.canSave" type="primary" @click="sync">
        {{ t`Save` }}
      </Button>
      <Button v-else-if="doc?.canSubmit" type="primary" @click="submit">{{
        t`Submit`
      }}</Button>
    </template>
    <template #body>
      <FormHeader
        :form-title="title"
        class="
          sticky
          top-0
          bg-white
          dark:bg-gray-890
          border-b
          dark:border-gray-800
        "
      >
        <StatusPill v-if="hasDoc" :doc="doc" />
      </FormHeader>

      <!-- Section Container -->
      <div
        v-if="hasDoc"
        class="overflow-auto custom-scroll custom-scroll-thumb1"
      >
        <CommonFormSection
          v-for="([n, fields], idx) in activeGroup.entries()"
          :key="n + idx"
          ref="section"
          class="p-4"
          :class="
            idx !== 0 && activeGroup.size > 1
              ? 'border-t dark:border-gray-800'
              : ''
          "
          :show-title="activeGroup.size > 1 && n !== t`Default`"
          :title="n"
          :fields="fields"
          :doc="doc"
          :errors="errors"
          @editrow="(doc: Doc) => showRowEditForm(doc)"
          @value-change="onValueChange"
          @row-change="onRowChange"
        />
      </div>

      <!-- Manufacture Stock Movement Section -->
      <div
        v-if="isManufactureStockMovement"
        class="px-4 py-3 border-t dark:border-gray-800 flex-shrink-0 bg-gray-50 dark:bg-gray-850"
      >
        <div class="flex items-center justify-between mb-2">
          <h3 class="text-sm font-semibold text-gray-700 dark:text-gray-200">
            {{ t`Raw Material Values` }}
          </h3>
          <button
            v-if="rawMaterialValues.length > 0"
            class="text-xs text-gray-500 hover:text-gray-700 dark:hover:text-gray-300"
            @click="updateRawMaterialValues"
          >
            {{ t`Refresh` }}
          </button>
        </div>

        <div v-if="loadingValuation" class="text-sm text-gray-500">
          {{ t`Loading...` }}
        </div>

        <div
          v-else-if="rawMaterialValues.length === 0"
          class="text-sm text-gray-500 italic"
        >
          {{ t`No raw materials with From Location selected` }}
        </div>

        <div v-else class="overflow-x-auto">
          <table class="w-full text-sm">
            <thead>
              <tr class="text-gray-600 dark:text-gray-400 border-b dark:border-gray-700">
                <th class="text-left py-1 pr-4">{{ t`Item` }}</th>
                <th class="text-left py-1 pr-4">{{ t`Location` }}</th>
                <th v-if="hasBatches" class="text-left py-1 pr-4">{{ t`Batch` }}</th>
                <th class="text-right py-1 pr-4">{{ t`Quantity` }}</th>
                <th class="text-right py-1 pr-4">{{ t`Valuation Rate` }}</th>
                <th class="text-right py-1">{{ t`Total Value` }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(rm, idx) in rawMaterialValues"
                :key="idx"
                class="border-b dark:border-gray-800 last:border-0"
              >
                <td class="py-2 pr-4">{{ rm.item }}</td>
                <td class="py-2 pr-4">{{ rm.location }}</td>
                <td v-if="hasBatches" class="py-2 pr-4">{{ rm.batch || '-' }}</td>
                <td class="py-2 pr-4 text-right">{{ fyo.format(rm.quantity, 'Float') }}</td>
                <td class="py-2 pr-4 text-right">{{ fyo.format(rm.valuationRate, 'Currency') }}</td>
                <td class="py-2 pr-4 text-right font-semibold">
                  {{ fyo.format(rm.value, 'Currency') }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Finished Goods Section -->
      <div
        v-if="isManufactureStockMovement && finishedGoodsValues.length > 0"
        class="px-4 py-3 border-t dark:border-gray-800 flex-shrink-0 bg-gray-50 dark:bg-gray-850"
      >
        <h3 class="text-sm font-semibold text-gray-700 dark:text-gray-200 mb-2">
          {{ t`Finished Goods` }}
        </h3>
        <div class="overflow-x-auto">
          <table class="w-full text-sm">
            <thead>
              <tr class="text-gray-600 dark:text-gray-400 border-b dark:border-gray-700">
                <th class="text-left py-1 pr-4">{{ t`Item` }}</th>
                <th class="text-left py-1 pr-4">{{ t`Location` }}</th>
                <th v-if="hasBatches" class="text-left py-1 pr-4">{{ t`Batch` }}</th>
                <th class="text-right py-1 pr-4">{{ t`Quantity` }}</th>
                <th class="text-right py-1 pr-4">{{ t`Rate` }}</th>
                <th class="text-right py-1">{{ t`Total Value` }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(fg, idx) in finishedGoodsValues"
                :key="idx"
                class="border-b dark:border-gray-800 last:border-0"
              >
                <td class="py-2 pr-4">{{ fg.item }}</td>
                <td class="py-2 pr-4">{{ fg.location }}</td>
                <td v-if="hasBatches" class="py-2 pr-4">{{ fg.batch || '-' }}</td>
                <td class="py-2 pr-4 text-right">{{ fyo.format(fg.quantity, 'Float') }}</td>
                <td class="py-2 pr-4 text-right">{{ fyo.format(fg.rate, 'Currency') }}</td>
                <td class="py-2 pr-4 text-right font-semibold">
                  {{ fyo.format(fg.value, 'Currency') }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Tab Bar -->
      <div
        v-if="groupedFields && groupedFields.size > 1"
        class="
          mt-auto
          px-4
          pb-4
          flex
          gap-8
          border-t
          dark:border-gray-800
          flex-shrink-0
          sticky
          bottom-0
          bg-white
          dark:bg-gray-875
        "
      >
        <div
          v-for="key of groupedFields.keys()"
          :key="key"
          class="text-sm cursor-pointer"
          :class="
            key === activeTab
              ? 'text-gray-900 dark:text-gray-25 font-semibold border-t-2 border-gray-800 dark:border-gray-100'
              : 'text-gray-700 dark:text-gray-200 '
          "
          :style="{
            paddingTop: key === activeTab ? 'calc(1rem - 2px)' : '1rem',
          }"
          @click="activeTab = key"
        >
          {{ key }}
        </div>
      </div>
    </template>
    <template #quickedit>
      <Transition name="quickedit">
        <LinkedEntries
          v-if="showLinks && canShowLinks"
          :doc="doc"
          @close="showLinks = false"
        />
      </Transition>
      <Transition name="quickedit">
        <RowEditForm
          v-if="row && !showLinks"
          :doc="doc"
          :fieldname="row.fieldname"
          :index="row.index"
          @previous="(i:number) => row!.index = i"
          @next="(i:number) => row!.index = i"
          @close="() => (row = null)"
        />
      </Transition>
    </template>
  </FormContainer>
</template>
<script lang="ts">
import { DocValue } from 'fyo/core/types';
import { Doc } from 'fyo/model/doc';
import { DEFAULT_CURRENCY } from 'fyo/utils/consts';
import { ValidationError } from 'fyo/utils/errors';
import { getCurrentValuationDetails } from 'reports/inventory/helpers';
import { getDocStatus } from 'models/helpers';
import { ModelNameEnum } from 'models/types';
import { Field, Schema } from 'schemas/types';
import Button from 'src/components/Button.vue';
import Barcode from 'src/components/Controls/Barcode.vue';
import ExchangeRate from 'src/components/Controls/ExchangeRate.vue';
import DropdownWithActions from 'src/components/DropdownWithActions.vue';
import FormContainer from 'src/components/FormContainer.vue';
import FormHeader from 'src/components/FormHeader.vue';
import StatusPill from 'src/components/StatusPill.vue';
import { getErrorMessage } from 'src/utils';
import { shortcutsKey } from 'src/utils/injectionKeys';
import { docsPathMap } from 'src/utils/misc';
import { docsPathRef } from 'src/utils/refs';
import { ActionGroup, DocRef, UIGroupedFields } from 'src/utils/types';
import {
  commonDocSubmit,
  commonDocSync,
  getDocFromNameIfExistsElseNew,
  getFieldsGroupedByTabAndSection,
  getFormRoute,
  getGroupedActionsForDoc,
  isPrintable,
  routeTo,
} from 'src/utils/ui';
import { useDocShortcuts } from 'src/utils/vueUtils';
import { computed, defineComponent, inject, nextTick, ref } from 'vue';
import CommonFormSection from './CommonFormSection.vue';
import LinkedEntries from './LinkedEntries.vue';
import RowEditForm from './RowEditForm.vue';

export default defineComponent({
  components: {
    FormContainer,
    FormHeader,
    CommonFormSection,
    Button,
    DropdownWithActions,
    Barcode,
    ExchangeRate,
    LinkedEntries,
    RowEditForm,
    StatusPill,
  },
  provide() {
    return {
      doc: computed(() => this.docOrNull),
    };
  },
  props: {
    name: { type: String, default: '' },
    schemaName: { type: String, default: ModelNameEnum.SalesInvoice },
  },
  setup() {
    const shortcuts = inject(shortcutsKey);
    const docOrNull = ref(null) as DocRef;
    let context = 'CommonForm';
    if (shortcuts) {
      context = useDocShortcuts(shortcuts, docOrNull, 'CommonForm', true);
    }

    return {
      docOrNull,
      shortcuts,
      context,
      printButton: ref<InstanceType<typeof Button> | null>(null),
    };
  },
  data() {
    return {
      errors: {},
      activeTab: this.t`Default`,
      groupedFields: null,
      isPrintable: false,
      showLinks: false,
      useFullWidth: false,
      row: null,
      rawMaterialValues: [] as Array<{
        item: string;
        location: string;
        batch?: string;
        quantity: number;
        valuationRate: number;
        value: number;
      }>,
      finishedGoodsValues: [] as Array<{
        item: string;
        location: string;
        batch?: string;
        quantity: number;
        rate: number;
        value: number;
      }>,
      loadingValuation: false,
    } as {
      errors: Record<string, string>;
      activeTab: string;
      groupedFields: null | UIGroupedFields;
      isPrintable: boolean;
      showLinks: boolean;
      useFullWidth: boolean;
      row: null | { index: number; fieldname: string };
      rawMaterialValues: Array<{
        item: string;
        location: string;
        batch?: string;
        quantity: number;
        valuationRate: number;
        value: number;
      }>;
      finishedGoodsValues: Array<{
        item: string;
        location: string;
        batch?: string;
        quantity: number;
        rate: number;
        value: number;
      }>;
      loadingValuation: boolean;
    };
  },
  computed: {
    canShowBarcode(): boolean {
      if (!this.fyo.singles.InventorySettings?.enableBarcodes) {
        return false;
      }

      if (!this.hasDoc) {
        return false;
      }

      if (this.doc.isSubmitted || this.doc.isCancelled) {
        return false;
      }

      // @ts-ignore
      return typeof this.doc?.addItem === 'function';
    },
    canShowExchangeRate(): boolean {
      return this.hasDoc && !!this.doc.isMultiCurrency;
    },
    exchangeRate(): number {
      if (!this.hasDoc || typeof this.doc.exchangeRate !== 'number') {
        return 1;
      }

      return this.doc.exchangeRate;
    },
    fromCurrency(): string {
      const currency = this.doc?.currency;
      if (typeof currency !== 'string') {
        return this.toCurrency;
      }

      return currency;
    },
    toCurrency(): string {
      const currency = this.fyo.singles.SystemSettings?.currency;
      if (typeof currency !== 'string') {
        return DEFAULT_CURRENCY;
      }

      return currency;
    },
    canPrint(): boolean {
      if (!this.hasDoc) {
        return false;
      }

      return !this.doc.isCancelled && !this.doc.dirty && this.isPrintable;
    },
    canShowLinks(): boolean {
      if (!this.hasDoc) {
        return false;
      }

      if (this.doc.schema.isSubmittable && !this.doc.isSubmitted) {
        return false;
      }

      return this.doc.inserted;
    },
    hasDoc(): boolean {
      return this.docOrNull instanceof Doc;
    },
    status(): string {
      if (!this.hasDoc) {
        return '';
      }

      return getDocStatus(this.doc);
    },
    doc(): Doc {
      const doc = this.docOrNull;
      if (!doc) {
        throw new ValidationError(
          this.t`Doc ${this.schema.label} ${this.name} not set`
        );
      }
      return doc;
    },
    title(): string {
      if (this.schema.isSubmittable && this.docOrNull?.notInserted) {
        return this.t`New Entry`;
      }

      return this.docOrNull?.name || this.t`New Entry`;
    },
    schema(): Schema {
      const schema = this.fyo.schemaMap[this.schemaName];
      if (!schema) {
        throw new ValidationError(`no schema found with ${this.schemaName}`);
      }

      return schema;
    },
    activeGroup(): Map<string, Field[]> {
      if (!this.groupedFields) {
        return new Map();
      }

      const group = this.groupedFields.get(this.activeTab);
      if (!group) {
        const tab = [...this.groupedFields.keys()][0];
        return this.groupedFields.get(tab) ?? new Map<string, Field[]>();
      }

      return group;
    },
    groupedActions(): ActionGroup[] {
      if (!this.hasDoc) {
        return [];
      }

      return getGroupedActionsForDoc(this.doc);
    },
    isManufactureStockMovement(): boolean {
      if (!this.hasDoc) {
        return false;
      }
      return (
        this.schemaName === 'StockMovement' &&
        this.doc.movementType === 'Manufacture'
      );
    },
    hasBatches(): boolean {
      return !!this.fyo.singles.InventorySettings?.enableBatches;
    },
  },
  beforeMount() {
    this.useFullWidth = !!this.fyo.singles.Misc?.useFullWidth;
  },
  async mounted() {
    if (this.fyo.store.isDevelopment) {
      // @ts-ignore
      window.cf = this;
    }

    await this.setDoc();
    this.replacePathAfterSync();
    this.updateGroupedFields();
    if (this.groupedFields) {
      this.activeTab = [...this.groupedFields.keys()][0];
    }
    this.isPrintable = await isPrintable(this.schemaName);
  },
  activated(): void {
    this.useFullWidth = !!this.fyo.singles.Misc?.useFullWidth;
    docsPathRef.value = docsPathMap[this.schemaName] ?? '';
    this.shortcuts?.pmod.set(this.context, ['KeyP'], () => {
      if (!this.canPrint) {
        return;
      }

      this.printButton?.$el.click();
    });
    this.shortcuts?.pmod.set(this.context, ['KeyL'], () => {
      if (!this.canShowLinks && !this.showLinks) {
        return;
      }

      this.showLinks = !this.showLinks;
    });
  },
  deactivated(): void {
    docsPathRef.value = '';
    this.showLinks = false;
    this.row = null;
  },
  methods: {
    routeTo,
    async toggleWidth() {
      const value = !this.useFullWidth;
      await this.fyo.singles.Misc?.setAndSync('useFullWidth', value);
      this.useFullWidth = value;
    },
    updateGroupedFields(): void {
      if (!this.hasDoc) {
        return;
      }

      this.groupedFields = getFieldsGroupedByTabAndSection(
        this.schema,
        this.doc
      );
    },
    async sync(useDialog?: boolean) {
      if (await commonDocSync(this.doc, useDialog)) {
        this.updateGroupedFields();
      }
    },
    async submit() {
      if (await commonDocSubmit(this.doc)) {
        this.updateGroupedFields();
      }
    },
    async setDoc() {
      if (this.hasDoc) {
        return;
      }

      this.docOrNull = await getDocFromNameIfExistsElseNew(
        this.schemaName,
        this.name
      );
    },
    replacePathAfterSync() {
      if (!this.hasDoc || this.doc.inserted) {
        return;
      }

      this.doc.once('afterSync', async () => {
        const route = getFormRoute(this.schemaName, this.doc.name!);
        await this.$router.replace(route);
      });
    },
    async showRowEditForm(doc: Doc) {
      if (this.showLinks) {
        this.showLinks = false;
        await nextTick();
      }

      const index = doc.idx;
      const fieldname = doc.parentFieldname;

      if (typeof index === 'number' && typeof fieldname === 'string') {
        this.row = { index, fieldname };
      }
    },
    async onValueChange(field: Field, value: DocValue) {
      const { fieldname } = field;
      delete this.errors[fieldname];

      try {
        await this.doc.set(fieldname, value);
      } catch (err) {
        if (!(err instanceof Error)) {
          return;
        }

        this.errors[fieldname] = getErrorMessage(err, this.doc);
      }

      this.updateGroupedFields();

      // Update raw material values if items changed for Manufacture
      if (
        this.isManufactureStockMovement &&
        fieldname === 'items'
      ) {
        await this.updateRawMaterialValues();
      }
    },
    async updateRawMaterialValues() {
      if (!this.isManufactureStockMovement || !this.doc.items) {
        this.rawMaterialValues = [];
        this.finishedGoodsValues = [];
        return;
      }

      this.loadingValuation = true;
      const rawMaterials = this.doc.items.filter((item) => item.fromLocation);
      const finishedGoods = this.doc.items.filter((item) => item.toLocation);

      // Process raw materials
      const rawMaterialValues: Array<{
        item: string;
        location: string;
        batch?: string;
        quantity: number;
        valuationRate: number;
        value: number;
      }> = [];

      for (const row of rawMaterials) {
        if (!row.item || !row.fromLocation) {
          continue;
        }

        const valuationDetails = await getCurrentValuationDetails(
          this.fyo,
          row.item,
          row.fromLocation,
          row.batch ?? null,
          this.doc.date
        );

        if (valuationDetails) {
          const movementQty = row.quantity ?? 0;
          const movementValue = movementQty * valuationDetails.valuationRate;

          rawMaterialValues.push({
            item: row.item,
            location: row.fromLocation,
            batch: row.batch ?? undefined,
            quantity: movementQty,
            valuationRate: valuationDetails.valuationRate,
            value: movementValue,
          });
        }
      }

      // Process finished goods (use entered rate and quantity)
      const finishedGoodsValues: Array<{
        item: string;
        location: string;
        batch?: string;
        quantity: number;
        rate: number;
        value: number;
      }> = [];

      for (const row of finishedGoods) {
        if (!row.item || !row.toLocation) {
          continue;
        }

        const qty = row.quantity ?? 0;
        // Convert Money object to number using .float property
        const rateValue = row.rate;
        const rate =
          typeof rateValue === 'object' && rateValue !== null && 'float' in rateValue
            ? rateValue.float
            : Number(rateValue ?? 0);
        const value = qty * rate;

        finishedGoodsValues.push({
          item: row.item,
          location: row.toLocation,
          batch: row.batch ?? undefined,
          quantity: qty,
          rate: rate,
          value: value,
        });
      }

      this.rawMaterialValues = rawMaterialValues;
      this.finishedGoodsValues = finishedGoodsValues;
      this.loadingValuation = false;
    },
    async onRowChange(field: Field, value: DocValue, parentField: Field) {
      // Update grouped fields first
      this.updateGroupedFields();

      // Update raw material values if the changed row belongs to items table
      if (parentField.fieldname === 'items' && this.isManufactureStockMovement) {
        await this.updateRawMaterialValues();
      }
    },
  },
});
</script>
