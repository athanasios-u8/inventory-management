<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Budget-driven restock recommendations based on demand forecasts</p>
    </div>

    <div class="budget-card">
      <div class="budget-row">
        <span class="budget-label">Restock Budget</span>
        <input
          type="range"
          class="budget-slider"
          v-model.number="budget"
          min="1000"
          max="500000"
          step="1000"
        />
        <span class="budget-value">${{ budget.toLocaleString() }}</span>
      </div>
    </div>

    <div v-if="!loading && recommendations.length" class="allocation-bar">
      ${{ selectedTotal.toLocaleString() }} of ${{ budget.toLocaleString() }} allocated &mdash; {{ selectedItems.length }} items selected
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="recommendations.length" class="card">
        <div class="card-header">
          <h3 class="card-title">Restock Recommendations</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>Status</th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="item.selected ? 'row-selected' : 'row-over-budget'"
              >
                <td>
                  <span
                    class="status-dot"
                    :class="item.selected ? 'selected' : 'over-budget'"
                  ></span>
                </td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.restock_qty.toLocaleString() }}</td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td>${{ item.restock_cost.toLocaleString() }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-else class="empty-state">
        No demand forecasts available for the current filters.
      </div>

      <button
        class="btn-primary"
        @click="placeOrder"
        :disabled="selectedItems.length === 0 || placing"
      >
        {{ placing ? 'Placing Order...' : 'Place Order' }}
      </button>

      <div v-if="orderPlaced" class="success-banner">
        Order placed successfully.
        <router-link to="/orders">View in Orders tab</router-link>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const allForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const orderPlaced = ref(false)

    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    // Build a Map from SKU to inventory item for O(1) unit_cost lookup
    // instead of doing a linear search per forecast item
    const inventoryBySku = computed(() => {
      const map = new Map()
      inventoryItems.value.forEach(item => map.set(item.sku, item))
      return map
    })

    // Join demand forecasts with inventory by SKU to obtain unit_cost.
    // Forecasts with no matching inventory entry are excluded because
    // we cannot compute a restock cost without knowing the unit price.
    const forecastsWithCost = computed(() => {
      return allForecasts.value
        .filter(f => inventoryBySku.value.has(f.item_sku))
        .map(f => {
          const invItem = inventoryBySku.value.get(f.item_sku)
          const restock_qty = f.forecasted_demand
          const unit_cost = invItem.unit_cost
          return {
            ...f,
            unit_cost,
            restock_qty,
            restock_cost: restock_qty * unit_cost,
            demand_gap: f.forecasted_demand - f.current_demand
          }
        })
    })

    const recommendations = computed(() => {
      const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

      // Sort: items with increasing demand first (highest urgency),
      // then stable, then decreasing. Within the same trend, sort by
      // demand_gap descending so the largest unmet demand comes first.
      const sorted = [...forecastsWithCost.value].sort((a, b) => {
        const trendDiff = (TREND_PRIORITY[a.trend] ?? 2) - (TREND_PRIORITY[b.trend] ?? 2)
        if (trendDiff !== 0) return trendDiff
        return b.demand_gap - a.demand_gap
      })

      // Greedy selection: iterate in priority order and include each item
      // if its restock cost fits within the remaining budget. Items that
      // don't fit are marked over-budget but still returned for visibility.
      let running = 0
      return sorted.map(item => {
        if (running + item.restock_cost <= budget.value) {
          running += item.restock_cost
          return { ...item, selected: true, overBudget: false }
        }
        return { ...item, selected: false, overBudget: true }
      })
    })

    const selectedItems = computed(() => recommendations.value.filter(r => r.selected))

    const selectedTotal = computed(() =>
      selectedItems.value.reduce((sum, i) => sum + i.restock_cost, 0)
    )

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Reload when warehouse or category filter changes
    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    const placeOrder = async () => {
      placing.value = true
      try {
        await api.createOrder({
          customer: 'Internal Restocking',
          items: selectedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.restock_qty,
            unit_price: i.unit_cost
          }))
        })
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(() => loadData())

    return {
      budget,
      allForecasts,
      inventoryItems,
      loading,
      error,
      placing,
      orderPlaced,
      inventoryBySku,
      forecastsWithCost,
      recommendations,
      selectedItems,
      selectedTotal,
      loadData,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.budget-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-label {
  font-weight: 600;
  color: #0f172a;
  font-size: 0.938rem;
  min-width: 130px;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #2563eb;
  min-width: 120px;
  text-align: right;
}

.allocation-bar {
  background: #f1f5f9;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.row-over-budget td {
  opacity: 0.4;
}

.status-dot {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

.status-dot.selected {
  background: #059669;
}

.status-dot.over-budget {
  background: #cbd5e1;
}

.empty-state {
  color: #64748b;
  font-size: 0.938rem;
  padding: 2rem 0;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  margin-top: 1rem;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.success-banner a {
  color: #047857;
  font-weight: 600;
  margin-left: 0.5rem;
}
</style>
