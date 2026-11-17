<script setup lang="ts">
import { loadTransactions } from '@/api/transactions';
import { VUE_QUERY_CACHE_KEYS } from '@/common/const';
import { usePayeeLookup } from '@/composable/data-queries/payees';
import { useFormatCurrency } from '@/composable/formatters';
import { useDateLocale } from '@/composable/use-date-locale';
import { cn } from '@/lib/utils';
import { useCurrenciesStore } from '@/stores';
import {
  SORT_DIRECTIONS,
  TRANSACTION_SORT_FIELD,
  TRANSACTION_TYPES,
  type BudgetPlanPeriodViewLine,
} from '@bt/shared/types';
import { useQuery } from '@tanstack/vue-query';
import { parseISO } from 'date-fns';
import { storeToRefs } from 'pinia';
import { computed } from 'vue';

import { amountToneClass } from '../helpers';

const props = defineProps<{ lines: BudgetPlanPeriodViewLine[] }>();

const { formatBaseCurrency, formatAmountByCurrencyCode } = useFormatCurrency();
const { baseCurrency } = storeToRefs(useCurrenciesStore());
const { format } = useDateLocale();
const { nameById: payeeNameById } = usePayeeLookup();

const categoryIds = computed(() => [...new Set(props.lines.flatMap((line) => line.categoryIds))]);

const { data: transactions, isFetching } = useQuery({
  queryKey: computed(() => [...VUE_QUERY_CACHE_KEYS.recentCategoryTransactions, categoryIds.value]),
  queryFn: () =>
    loadTransactions({
      categoryIds: categoryIds.value,
      limit: 8,
      sortBy: TRANSACTION_SORT_FIELD.time,
      order: SORT_DIRECTIONS.desc,
      isPlanned: false,
      excludeBalanceAdjustments: true,
    }),
  enabled: computed(() => categoryIds.value.length > 0),
});

const signed = ({ tx, amount }: { tx: { transactionType: TRANSACTION_TYPES }; amount: number }) =>
  tx.transactionType === TRANSACTION_TYPES.expense ? -amount : amount;

const rows = computed(() =>
  (transactions.value ?? []).map((tx) => {
    const base = baseCurrency.value?.currencyCode;
    const isForeign = Boolean(base && tx.currencyCode !== base);
    return {
      id: tx.id,
      name: (tx.payeeId && payeeNameById.value[tx.payeeId]) || tx.note || '—',
      date: format(parseISO(String(tx.time)), 'd MMM'),
      tone: amountToneClass({ amount: signed({ tx, amount: tx.amount }) }),
      native: formatAmountByCurrencyCode(signed({ tx, amount: tx.amount }), tx.currencyCode),
      // Billed currency first, base-currency estimate beside it (matches the
      // recurring-payments screens); omitted when the tx is already in CAD.
      converted: isForeign ? formatBaseCurrency(signed({ tx, amount: tx.refAmount })) : null,
    };
  }),
);
</script>

<template>
  <p v-if="!rows.length" class="text-muted-foreground text-sm">
    {{ isFetching ? '…' : $t('budgetPlans.panel.noRecentTransactions') }}
  </p>

  <div v-else class="grid gap-2 text-sm">
    <div v-for="row in rows" :key="row.id" class="min-w-0">
      <div class="flex justify-between gap-3">
        <span class="truncate">{{ row.name }}</span>
        <span :class="cn('shrink-0 tabular-nums', row.tone)">
          {{ row.native }}
          <span v-if="row.converted" class="text-muted-foreground text-xs font-normal">≈ {{ row.converted }}</span>
        </span>
      </div>
      <div class="text-muted-foreground text-xs">{{ row.date }}</div>
    </div>
  </div>
</template>
