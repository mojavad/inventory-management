<template>
  <div class="orders">
    <div class="page-header">
      <h2>{{ t('orders.title') }}</h2>
      <p>{{ t('orders.description') }}</p>
    </div>

    <div class="view-toggle">
      <button
        :class="['toggle-pill', { active: view === 'customer' }]"
        @click="view = 'customer'"
      >
        {{ t('orders.toggleCustomer') }}
      </button>
      <button
        :class="['toggle-pill', { active: view === 'restocking' }]"
        @click="view = 'restocking'"
      >
        {{ t('orders.toggleRestocking') }}
      </button>
    </div>

    <template v-if="view === 'customer'">
      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div class="stats-grid">
          <div class="stat-card success">
            <div class="stat-label">{{ t('status.delivered') }}</div>
            <div class="stat-value">{{ getOrdersByStatus('Delivered').length }}</div>
          </div>
          <div class="stat-card info">
            <div class="stat-label">{{ t('status.shipped') }}</div>
            <div class="stat-value">{{ getOrdersByStatus('Shipped').length }}</div>
          </div>
          <div class="stat-card warning">
            <div class="stat-label">{{ t('status.processing') }}</div>
            <div class="stat-value">{{ getOrdersByStatus('Processing').length }}</div>
          </div>
          <div class="stat-card danger">
            <div class="stat-label">{{ t('status.backordered') }}</div>
            <div class="stat-value">{{ getOrdersByStatus('Backordered').length }}</div>
          </div>
        </div>

        <div class="card">
          <div class="card-header">
            <h3 class="card-title">{{ t('orders.allOrders') }} ({{ orders.length }})</h3>
          </div>
          <div class="table-container">
            <table class="orders-table">
              <thead>
                <tr>
                  <th class="col-order-number">{{ t('orders.table.orderNumber') }}</th>
                  <th class="col-customer">{{ t('orders.table.customer') }}</th>
                  <th class="col-items">{{ t('orders.table.items') }}</th>
                  <th class="col-status">{{ t('orders.table.status') }}</th>
                  <th class="col-date">{{ t('orders.table.orderDate') }}</th>
                  <th class="col-date">{{ t('orders.table.expectedDelivery') }}</th>
                  <th class="col-value">{{ t('orders.table.totalValue') }}</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="order in orders" :key="order.id">
                  <td class="col-order-number"><strong>{{ order.order_number }}</strong></td>
                  <td class="col-customer">{{ translateCustomerName(order.customer) }}</td>
                  <td class="col-items">
                    <details class="items-details">
                      <summary class="items-summary">
                        {{ t('orders.itemsCount', { count: order.items.length }) }}
                      </summary>
                      <div class="items-dropdown">
                        <div v-for="(item, idx) in order.items" :key="idx" class="item-entry">
                          <span class="item-name">{{ translateProductName(item.name) }}</span>
                          <span class="item-meta">{{ t('orders.quantity') }}: {{ item.quantity }} @ {{ currencySymbol }}{{ item.unit_price }}</span>
                        </div>
                      </div>
                    </details>
                  </td>
                  <td class="col-status">
                    <span :class="['badge', getOrderStatusClass(order.status)]">
                      {{ t(`status.${order.status.toLowerCase()}`) }}
                    </span>
                  </td>
                  <td class="col-date">{{ formatDate(order.order_date) }}</td>
                  <td class="col-date">{{ formatDate(order.expected_delivery) }}</td>
                  <td class="col-value"><strong>{{ currencySymbol }}{{ order.total_value.toLocaleString() }}</strong></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </template>

    <template v-else>
      <div v-if="restockingLoading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="restockingError" class="error">{{ restockingError }}</div>
      <div v-else>
        <div class="card">
          <div class="card-header">
            <h3 class="card-title">{{ t('orders.restockingTitle') }} ({{ sortedRestockingOrders.length }})</h3>
          </div>
          <div v-if="sortedRestockingOrders.length === 0" class="empty-state">
            {{ t('orders.restockingEmpty') }}
          </div>
          <div v-else class="table-container">
            <table class="restocking-table">
              <thead>
                <tr>
                  <th class="col-order-number">{{ t('orders.table.orderNumber') }}</th>
                  <th class="col-date">{{ t('orders.table.submittedDate') }}</th>
                  <th class="col-items">{{ t('orders.table.items') }}</th>
                  <th class="col-status">{{ t('orders.table.status') }}</th>
                  <th class="col-lead-time">{{ t('orders.table.leadTime') }}</th>
                  <th class="col-date">{{ t('orders.table.expectedDelivery') }}</th>
                  <th class="col-value">{{ t('orders.table.totalValue') }}</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="row in sortedRestockingOrders" :key="row.id">
                  <td class="col-order-number"><strong>{{ row.order_number }}</strong></td>
                  <td class="col-date">{{ formatDate(row.submitted_at) }}</td>
                  <td class="col-items">
                    <details class="items-details">
                      <summary class="items-summary">
                        {{ t('orders.itemsCount', { count: row.items.length }) }}
                      </summary>
                      <div class="items-dropdown">
                        <div v-for="item in row.items" :key="item.sku" class="item-entry">
                          <span class="item-name">{{ item.name }}</span>
                          <span class="item-meta">{{ t('orders.quantity') }}: {{ item.quantity }} @ {{ currencySymbol }}{{ item.unit_cost }}</span>
                        </div>
                      </div>
                    </details>
                  </td>
                  <td class="col-status">
                    <span class="badge info">{{ row.status }}</span>
                  </td>
                  <td class="col-lead-time">{{ t('orders.table.leadTimeDays', { days: row.lead_time_days }) }}</td>
                  <td class="col-date">{{ formatDate(row.expected_delivery) }}</td>
                  <td class="col-value"><strong>{{ currencySymbol }}{{ row.total_value.toLocaleString() }}</strong></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Orders',
  setup() {
    const { t, currentCurrency, translateProductName, translateCustomerName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    // --- view toggle ---
    const view = ref('customer')

    // --- customer orders ---
    const loading = ref(true)
    const error = ref(null)
    const orders = ref([])

    // --- restocking orders ---
    const restockingLoading = ref(false)
    const restockingError = ref(null)
    const restockingOrders = ref([])

    const sortedRestockingOrders = computed(() => {
      return [...restockingOrders.value].sort((a, b) => {
        const dateA = new Date(a.submitted_at)
        const dateB = new Date(b.submitted_at)
        if (isNaN(dateA.getTime())) return 1
        if (isNaN(dateB.getTime())) return -1
        return dateB - dateA
      })
    })

    // Use shared filters
    const {
      selectedPeriod,
      selectedLocation,
      selectedCategory,
      selectedStatus,
      getCurrentFilters
    } = useFilters()

    const loadOrders = async () => {
      try {
        loading.value = true
        const filters = getCurrentFilters()
        const fetchedOrders = await api.getOrders(filters)

        // Sort orders by order_date (earliest first)
        orders.value = fetchedOrders.sort((a, b) => {
          const dateA = new Date(a.order_date)
          const dateB = new Date(b.order_date)
          return dateA - dateB
        })
      } catch (err) {
        error.value = 'Failed to load orders: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const loadRestockingOrders = async () => {
      try {
        restockingLoading.value = true
        restockingError.value = null
        restockingOrders.value = await api.getRestockingOrders()
      } catch (err) {
        restockingError.value = 'Failed to load restocking orders: ' + err.message
      } finally {
        restockingLoading.value = false
      }
    }

    // Watch for filter changes and reload customer orders
    watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], () => {
      loadOrders()
    })

    // Re-fetch restocking orders when toggling to the restocking view
    watch(view, (newView) => {
      if (newView === 'restocking') {
        loadRestockingOrders()
      }
    })

    const getOrdersByStatus = (status) => {
      return orders.value.filter(order => order.status === status)
    }

    const getOrderStatusClass = (status) => {
      const statusMap = {
        'Delivered': 'success',
        'Shipped': 'info',
        'Processing': 'warning',
        'Backordered': 'danger'
      }
      return statusMap[status] || 'info'
    }

    const formatDate = (dateString) => {
      const { currentLocale } = useI18n()
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return '—'
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    onMounted(() => {
      loadOrders()
      loadRestockingOrders()
    })

    return {
      t,
      loading,
      error,
      orders,
      view,
      restockingLoading,
      restockingError,
      restockingOrders,
      sortedRestockingOrders,
      getOrdersByStatus,
      getOrderStatusClass,
      formatDate,
      currencySymbol,
      translateProductName,
      translateCustomerName
    }
  }
}
</script>

<style scoped>
/* View toggle pills */
.view-toggle {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.toggle-pill {
  padding: 0.5rem 1.25rem;
  border-radius: 999px;
  border: 1px solid #cbd5e1;
  background: white;
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s ease;
}

.toggle-pill:hover {
  border-color: #94a3b8;
  color: #334155;
}

.toggle-pill.active {
  background: #eff6ff;
  color: #2563eb;
  border-color: #2563eb;
}

/* Fixed table layout to prevent column shifting */
.orders-table,
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

/* Column widths */
.col-order-number {
  width: 130px;
}

.col-customer {
  width: 180px;
}

.col-items {
  width: 200px;
}

.col-status {
  width: 130px;
}

.col-date {
  width: 140px;
}

.col-value {
  width: 120px;
}

.col-lead-time {
  width: 110px;
}

/* Items details styling */
.items-details {
  position: relative;
}

.items-summary {
  cursor: pointer;
  color: #3b82f6;
  font-weight: 500;
  list-style: none;
  user-select: none;
  display: inline-block;
}

.items-summary::-webkit-details-marker {
  display: none;
}

.items-summary::before {
  content: '▶';
  display: inline-block;
  margin-right: 0.375rem;
  font-size: 0.75rem;
  transition: transform 0.2s;
}

.items-details[open] .items-summary::before {
  transform: rotate(90deg);
}

.items-summary:hover {
  color: #2563eb;
  text-decoration: underline;
}

/* Dropdown container */
.items-dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  margin-top: 0.5rem;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  padding: 0.75rem;
  z-index: 10;
  min-width: 300px;
  max-width: 400px;
}

.item-entry {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 0.5rem;
  border-bottom: 1px solid #f1f5f9;
}

.item-entry:last-child {
  border-bottom: none;
}

.item-name {
  font-size: 0.875rem;
  font-weight: 500;
  color: #0f172a;
}

.item-meta {
  font-size: 0.813rem;
  color: #64748b;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 3rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
