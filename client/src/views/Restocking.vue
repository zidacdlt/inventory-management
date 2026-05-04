<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and review AI-recommended restocking orders based on demand forecasts.</p>
    </div>

    <!-- Success banner -->
    <div v-if="successMessage" class="banner banner-success">
      {{ successMessage }}
    </div>

    <!-- Error banner -->
    <div v-if="orderError" class="banner banner-error">
      {{ orderError }}
    </div>

    <!-- Budget control card -->
    <div class="card budget-card">
      <div class="budget-header">
        <div class="budget-label">Restocking Budget</div>
        <div class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
      </div>
      <div class="slider-row">
        <span class="slider-bound">{{ currencySymbol }}0</span>
        <div class="slider-track-wrapper">
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
          />
          <div class="budget-fill-bar">
            <div
              class="budget-fill-used"
              :style="{ width: budgetUsedPercent + '%' }"
              :class="{ 'over-budget': budgetUsedPercent >= 100 }"
            ></div>
          </div>
        </div>
        <span class="slider-bound">{{ currencySymbol }}50,000</span>
      </div>
      <div class="budget-usage-row">
        <span class="budget-usage-text">
          Recommended spend: <strong>{{ currencySymbol }}{{ totalRecommendedCost.toLocaleString() }}</strong>
          &mdash; {{ budgetUsedPercent }}% of budget
        </span>
        <span v-if="budgetUsedPercent >= 100" class="over-budget-label">Budget exceeded</span>
      </div>
    </div>

    <!-- Recommendations table -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items</h3>
        <button
          class="place-order-btn"
          :disabled="recommendedItems.length === 0 || loading || placingOrder"
          @click="placeOrder"
        >
          {{ placingOrder ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendedItems.length === 0" class="empty-state">
        No items fit within the current budget. Try increasing the budget.
      </div>
      <div v-else>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th class="col-right">Qty</th>
                <th class="col-right">Unit Cost</th>
                <th class="col-right">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td><strong>{{ item.name }}</strong></td>
                <td class="sku-cell">{{ item.sku }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-right">{{ item.quantity }}</td>
                <td class="col-right">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-right"><strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="recommendations-footer">
          <span class="rec-count">{{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }} recommended</span>
          <span class="rec-total">
            Total: <strong>{{ currencySymbol }}{{ totalRecommendedCost.toLocaleString() }}</strong>
          </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const TREND_PRIORITY = { increasing: 1, stable: 2, decreasing: 3 }

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(25000)
    const recommendedItems = ref([])
    const loading = ref(false)
    const error = ref(null)
    const placingOrder = ref(false)
    const successMessage = ref(null)
    const orderError = ref(null)
    let successTimer = null

    const totalRecommendedCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const budgetUsedPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(Math.round((totalRecommendedCost.value / budget.value) * 100), 100)
    })

    const buildRecommendations = (forecasts, inventory) => {
      // Build a SKU → unit_cost map from inventory
      const costMap = {}
      for (const inv of inventory) {
        costMap[inv.sku] = inv.unit_cost
      }

      // Join demand forecasts with inventory unit costs
      const joined = forecasts
        .filter(f => costMap[f.item_sku] !== undefined)
        .map(f => ({
          sku: f.item_sku,
          name: f.item_name,
          trend: f.trend,
          quantity: f.forecasted_demand,
          unit_cost: costMap[f.item_sku],
          total_cost: f.forecasted_demand * costMap[f.item_sku]
        }))

      // Sort by trend priority: increasing → stable → decreasing
      joined.sort((a, b) => {
        const pa = TREND_PRIORITY[a.trend] || 99
        const pb = TREND_PRIORITY[b.trend] || 99
        return pa - pb
      })

      // Greedy fill within budget
      let remaining = budget.value
      const selected = []
      for (const item of joined) {
        if (item.total_cost <= remaining) {
          selected.push(item)
          remaining -= item.total_cost
        }
      }

      recommendedItems.value = selected
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        buildRecommendations(forecasts, inventory)
      } catch (err) {
        error.value = 'Failed to load recommendations. Please try again.'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || placingOrder.value) return
      placingOrder.value = true
      orderError.value = null
      successMessage.value = null

      const items = recommendedItems.value.map(item => ({
        sku: item.sku,
        name: item.name,
        quantity: item.quantity,
        unit_cost: item.unit_cost,
        total_cost: item.total_cost
      }))

      try {
        const result = await api.submitRestockingOrder(items, totalRecommendedCost.value)
        const deliveryDate = result.expected_delivery
          ? new Date(result.expected_delivery).toLocaleDateString('en-US', {
              year: 'numeric', month: 'short', day: 'numeric'
            })
          : 'TBD'
        successMessage.value = `Order ${result.id} placed successfully. Expected delivery: ${deliveryDate}`

        if (successTimer) clearTimeout(successTimer)
        successTimer = setTimeout(() => {
          successMessage.value = null
        }, 4000)
      } catch (err) {
        orderError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    // Recompute recommendations when budget changes (debounced slightly)
    let budgetTimer = null
    watch(budget, () => {
      if (budgetTimer) clearTimeout(budgetTimer)
      budgetTimer = setTimeout(() => loadRecommendations(), 300)
    })

    onMounted(() => loadRecommendations())

    return {
      currencySymbol,
      budget,
      recommendedItems,
      loading,
      error,
      placingOrder,
      successMessage,
      orderError,
      totalRecommendedCost,
      budgetUsedPercent,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Banners */
.banner {
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.banner-success {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
}

.banner-error {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}

/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  align-items: baseline;
  gap: 1.25rem;
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.slider-bound {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
  min-width: 3.5rem;
}

.slider-bound:last-child {
  text-align: right;
}

.slider-track-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: transform 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  transform: scale(1.15);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-fill-bar {
  width: 100%;
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
}

.budget-fill-used {
  height: 100%;
  background: #2563eb;
  border-radius: 3px;
  transition: width 0.3s ease, background 0.2s ease;
}

.budget-fill-used.over-budget {
  background: #dc2626;
}

.budget-usage-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 0.25rem;
}

.budget-usage-text {
  font-size: 0.875rem;
  color: #64748b;
}

.over-budget-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #dc2626;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 4px;
  padding: 0.188rem 0.5rem;
}

/* Table */
.col-right {
  text-align: right;
}

.sku-cell {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  color: #64748b;
}

/* Footer totals */
.recommendations-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.875rem 0.75rem 0.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.25rem;
}

.rec-count {
  font-size: 0.875rem;
  color: #64748b;
}

.rec-total {
  font-size: 0.938rem;
  color: #0f172a;
}

/* Place order button */
.place-order-btn {
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
