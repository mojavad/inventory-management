<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner -->
      <div v-if="successOrder" class="success-banner">
        <div class="success-banner-content">
          <span class="success-banner-text">
            {{ t('restocking.successTitle') }} —
            {{ t('restocking.successBody', { orderNumber: successOrder.order_number, date: formatDate(successOrder.expected_delivery) }) }}
          </span>
          <button class="btn-link" @click="dismissSuccess">
            {{ t('restocking.placeAnother') }}
          </button>
        </div>
      </div>

      <!-- Submit error banner -->
      <div v-if="submitError" class="error">
        {{ t('restocking.errorTitle') }}: {{ submitError }}
      </div>

      <!-- Budget card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetTitle') }}</h3>
        </div>
        <div class="budget-section">
          <div class="budget-value">{{ formatCurrency(budget) }}</div>
          <p class="budget-hint">{{ t('restocking.budgetHint') }}</p>
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="500000"
            step="1000"
            v-model.number="budget"
          />
          <div class="slider-labels">
            <span>{{ formatCurrency(0) }}</span>
            <span>{{ formatCurrency(250000) }}</span>
            <span>{{ formatCurrency(500000) }}</span>
          </div>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedTitle') }}</h3>
          <span class="items-summary-label">
            {{ t('restocking.itemsRecommended', { count: recommendedCount }) }}
          </span>
        </div>

        <div v-if="recommendations.length === 0" class="no-items">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.item') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th class="col-num">{{ t('restocking.table.currentStock') }}</th>
                <th class="col-num">{{ t('restocking.table.forecast') }}</th>
                <th class="col-num">{{ t('restocking.table.gap') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-qty">{{ t('restocking.table.quantity') }}</th>
                <th class="col-num">{{ t('restocking.table.subtotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="row in recommendations"
                :key="row.sku"
                :class="{ 'row-zero': (quantities[row.sku] || 0) === 0 }"
              >
                <td><code class="sku-code">{{ row.sku }}</code></td>
                <td class="item-name">{{ row.name }}</td>
                <td>
                  <span :class="['badge', row.trend]">{{ row.trend }}</span>
                </td>
                <td class="col-num">{{ row.current_stock }}</td>
                <td class="col-num">{{ row.forecasted_demand }}</td>
                <td class="col-num">
                  <span :class="['gap-value', row.gap > 0 ? 'gap-positive' : '']">{{ row.gap }}</span>
                </td>
                <td class="col-num">{{ formatCurrency(row.unit_cost) }}</td>
                <td class="col-qty">
                  <input
                    type="number"
                    class="qty-input"
                    min="0"
                    :max="row.gap"
                    :value="quantities[row.sku] || 0"
                    @input="updateQuantity(row.sku, $event.target.value)"
                  />
                </td>
                <td class="col-num subtotal">
                  {{ formatCurrency((quantities[row.sku] || 0) * row.unit_cost) }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Footer card -->
      <div class="card footer-card">
        <div class="footer-row">
          <div class="footer-stats">
            <div class="footer-stat">
              <span class="footer-stat-label">{{ t('restocking.totalCost') }}</span>
              <span class="footer-stat-value">{{ formatCurrency(totalCost) }}</span>
            </div>
            <div class="footer-stat">
              <span class="footer-stat-label">{{ t('restocking.remainingBudget') }}</span>
              <span :class="['footer-stat-value', isOverBudget ? 'over-budget' : '']">
                {{ formatCurrency(remainingBudget) }}
              </span>
            </div>
          </div>
          <div class="footer-actions">
            <p v-if="isOverBudget" class="over-budget-warning">
              {{ t('restocking.overBudgetWarning', { amount: formatCurrency(Math.abs(remainingBudget)) }) }}
            </p>
            <button
              class="btn-primary"
              :class="{ 'btn-disabled': !hasItemsToOrder }"
              :disabled="!hasItemsToOrder || placing"
              @click="placeOrder"
            >
              {{ placing ? t('restocking.placing') : t('restocking.placeOrder') }}
            </button>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, reactive, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, currentLocale } = useI18n()

    // --- State ---
    const loading = ref(true)
    const error = ref(null)
    const demandData = ref([])
    const inventoryData = ref([])
    const budget = ref(100000)
    const quantities = reactive({})
    const placing = ref(false)
    const submitError = ref(null)
    const successOrder = ref(null)

    // --- Currency formatter ---
    const formatCurrency = (value) => {
      const currency = currentCurrency.value
      const fractionDigits = currency === 'JPY' ? 0 : 2
      return Number(value).toLocaleString('en-US', {
        style: 'currency',
        currency,
        minimumFractionDigits: fractionDigits,
        maximumFractionDigits: fractionDigits
      })
    }

    // --- Date formatter ---
    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    // --- Algorithm ---
    const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

    const recommendations = computed(() => {
      if (demandData.value.length === 0 || inventoryData.value.length === 0) return []

      // Build inventory map keyed by sku
      const inventoryMap = new Map()
      for (const item of inventoryData.value) {
        inventoryMap.set(item.sku, item)
      }

      // Join demand with inventory, compute gaps
      const candidates = []
      for (const demand of demandData.value) {
        const inv = inventoryMap.get(demand.item_sku)
        if (!inv) continue  // No inventory match — no unit_cost, skip

        const gap = Math.max(0, demand.forecasted_demand - inv.quantity_on_hand)
        if (gap === 0) continue  // Already stocked above forecast

        candidates.push({
          sku: demand.item_sku,
          name: demand.item_name,
          category: inv.category,
          warehouse: inv.warehouse,
          trend: demand.trend,
          current_stock: inv.quantity_on_hand,
          forecasted_demand: demand.forecasted_demand,
          gap,
          unit_cost: inv.unit_cost
        })
      }

      // Sort: trend tier first, then gap×cost desc within tier
      candidates.sort((a, b) => {
        const tierDiff = (TREND_ORDER[a.trend] ?? 99) - (TREND_ORDER[b.trend] ?? 99)
        if (tierDiff !== 0) return tierDiff
        return (b.gap * b.unit_cost) - (a.gap * a.unit_cost)
      })

      // Greedy fill — all items appear, past-budget items get qty 0
      let remaining = budget.value
      return candidates.map(row => {
        const affordableQty = row.unit_cost > 0
          ? Math.floor(remaining / row.unit_cost)
          : 0
        const recommended_qty = Math.min(row.gap, Math.max(0, affordableQty))
        remaining -= recommended_qty * row.unit_cost
        return { ...row, recommended_qty }
      })
    })

    // Reset quantities whenever recommendations change
    watch(
      recommendations,
      (rows) => {
        // Clear existing keys
        for (const key of Object.keys(quantities)) {
          delete quantities[key]
        }
        for (const row of rows) {
          quantities[row.sku] = row.recommended_qty
        }
      },
      { immediate: true }
    )

    // --- Derived ---
    const recommendedCount = computed(() =>
      recommendations.value.filter(r => r.recommended_qty > 0).length
    )

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, row) => sum + (quantities[row.sku] || 0) * row.unit_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)
    const isOverBudget = computed(() => remainingBudget.value < 0)
    const hasItemsToOrder = computed(() =>
      recommendations.value.some(r => (quantities[r.sku] || 0) > 0)
    )

    // --- Actions ---
    const updateQuantity = (sku, rawValue) => {
      const parsed = parseInt(rawValue, 10)
      quantities[sku] = isNaN(parsed) ? 0 : Math.max(0, parsed)
    }

    const placeOrder = async () => {
      submitError.value = null
      successOrder.value = null
      placing.value = true
      try {
        const items = recommendations.value
          .filter(r => (quantities[r.sku] || 0) > 0)
          .map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: quantities[r.sku],
            unit_cost: r.unit_cost
          }))
        const result = await api.createRestockingOrder({ items })
        successOrder.value = result
        // Reset quantities to recommendation defaults
        for (const row of recommendations.value) {
          quantities[row.sku] = row.recommended_qty
        }
      } catch (err) {
        submitError.value = err?.response?.data?.detail || err.message || 'Unknown error'
      } finally {
        placing.value = false
      }
    }

    const dismissSuccess = () => {
      successOrder.value = null
    }

    // --- Data loading ---
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [demand, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandData.value = demand
        inventoryData.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + (err.message || 'Unknown error')
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      budget,
      quantities,
      recommendations,
      recommendedCount,
      totalCost,
      remainingBudget,
      isOverBudget,
      hasItemsToOrder,
      placing,
      submitError,
      successOrder,
      formatCurrency,
      formatDate,
      updateQuantity,
      placeOrder,
      dismissSuccess
    }
  }
}
</script>

<style scoped>
/* Budget section */
.budget-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-hint {
  color: #64748b;
  font-size: 0.875rem;
}

.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  margin: 0.5rem 0;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 6px rgba(37, 99, 235, 0.15);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Recommendations table */
.restocking-table {
  width: 100%;
  table-layout: auto;
}

.col-num {
  text-align: right;
  width: 100px;
}

.col-qty {
  text-align: center;
  width: 90px;
}

.sku-code {
  font-family: 'Courier New', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.item-name {
  font-weight: 500;
  color: #0f172a;
}

.gap-value {
  font-weight: 600;
}

.gap-positive {
  color: #dc2626;
}

.qty-input {
  width: 80px;
  padding: 0.25rem 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  text-align: center;
  color: #0f172a;
  background: white;
  outline: none;
  transition: border-color 0.15s ease;
}

.qty-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.subtotal {
  font-weight: 600;
  color: #0f172a;
}

.row-zero td {
  opacity: 0.45;
}

.row-zero .qty-input {
  opacity: 1;
}

/* No items state */
.no-items {
  text-align: center;
  padding: 2.5rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Items summary label in card header */
.items-summary-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

/* Footer card */
.footer-card {
  margin-top: 1rem;
}

.footer-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.footer-stats {
  display: flex;
  gap: 2.5rem;
}

.footer-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.footer-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.footer-stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.footer-stat-value.over-budget {
  color: #dc2626;
}

.footer-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.5rem;
}

.over-budget-warning {
  font-size: 0.813rem;
  color: #dc2626;
  max-width: 360px;
  text-align: right;
}

/* Primary button */
.btn-primary {
  padding: 0.625rem 1.75rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
  white-space: nowrap;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary.btn-disabled,
.btn-primary:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
}

.success-banner-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
}

.success-banner-text {
  font-size: 0.938rem;
  color: #065f46;
  font-weight: 500;
}

.btn-link {
  background: none;
  border: none;
  color: #065f46;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  text-decoration: underline;
  padding: 0;
  white-space: nowrap;
}

.btn-link:hover {
  color: #047857;
}

/* Mobile responsive */
@media (max-width: 640px) {
  .footer-row {
    flex-direction: column;
    align-items: stretch;
  }

  .footer-actions {
    align-items: stretch;
  }

  .btn-primary {
    width: 100%;
  }

  .over-budget-warning {
    text-align: left;
    max-width: 100%;
  }
}
</style>
