<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Review demand-driven restocking recommendations and place orders within your budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success State -->
      <div v-if="lastOrder" class="success-card">
        <div class="success-header">
          <span class="success-title">Order Placed Successfully</span>
          <span class="success-order-num">{{ lastOrder.order_number }}</span>
        </div>
        <p class="success-body">
          Your restocking order for {{ lastOrder.items.length }} items has been submitted to {{ lastOrder.warehouse || 'the warehouse' }}.
        </p>
        <button class="btn-secondary" @click="resetOrder">Place Another Order</button>
      </div>

      <template v-else>
        <!-- Budget Card -->
        <div class="card">
          <div class="card-header">
            <h3 class="card-title">Available Budget</h3>
          </div>

          <div class="budget-display">{{ formatCurrency(budget) }}</div>

          <input
            type="range"
            class="budget-slider"
            :min="0"
            :max="sliderMax"
            :step="500"
            v-model.number="budget"
          />

          <div class="budget-range-labels">
            <span>{{ formatCurrency(0) }}</span>
            <span>{{ formatCurrency(sliderMax) }}</span>
          </div>

          <div class="budget-usage">
            <div class="budget-usage-text">
              <span>{{ formatCurrency(totalSelected) }} used of {{ formatCurrency(budget) }}</span>
              <span class="item-count-label">{{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }} selected</span>
            </div>
            <div class="progress-bar-container">
              <div
                class="progress-bar-fill"
                :style="{ width: budgetUsedPercent + '%' }"
              ></div>
            </div>
          </div>
        </div>

        <!-- Recommendations Table Card -->
        <div class="card">
          <div class="card-header">
            <h3 class="card-title">Recommended Items</h3>
            <span class="badge info">{{ recommendations.length }} items</span>
          </div>

          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th class="col-check"></th>
                  <th>SKU</th>
                  <th>Item Name</th>
                  <th>Category</th>
                  <th>Gap Qty</th>
                  <th>Unit Cost</th>
                  <th>Total Cost</th>
                  <th>Trend</th>
                  <th>Period</th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="item in recommendations"
                  :key="item.item_sku"
                  :class="{ 'row-selected': isSelected(item.item_sku) }"
                  @click="toggleItem(item.item_sku)"
                  class="recommendation-row"
                >
                  <td class="col-check">
                    <input
                      type="checkbox"
                      :checked="isSelected(item.item_sku)"
                      @click.stop="toggleItem(item.item_sku)"
                    />
                  </td>
                  <td><strong>{{ item.item_sku }}</strong></td>
                  <td>{{ item.item_name }}</td>
                  <td>{{ item.category }}</td>
                  <td>{{ item.restock_quantity }}</td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
                  <td>
                    <span :class="['badge', item.trend]">{{ item.trend }}</span>
                  </td>
                  <td>{{ item.period }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Place Order Section -->
        <div class="order-section">
          <div class="order-total">
            <span class="order-total-label">Selected total:</span>
            <span class="order-total-value">{{ formatCurrency(totalSelected) }}</span>
          </div>
          <button
            :class="['btn-order', { 'btn-disabled': selectedItems.length === 0 }]"
            :disabled="selectedItems.length === 0"
            @click="placeOrder"
          >
            Place Order ({{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }})
          </button>
        </div>
      </template>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const selectedSkus = ref(new Set())
    const budget = ref(0)
    const sliderMax = ref(0)
    const lastOrder = ref(null)
    const placingOrder = ref(false)

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const formatCurrency = (value) => {
      return `${currencySymbol.value}${value.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
    }

    const isSelected = (sku) => {
      return selectedSkus.value.has(sku)
    }

    const toggleItem = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const selectedItems = computed(() => {
      return recommendations.value.filter(r => selectedSkus.value.has(r.item_sku))
    })

    const totalSelected = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const budgetUsedPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, (totalSelected.value / budget.value) * 100)
    })

    const autoSelect = (budgetValue) => {
      const next = new Set()
      let cumulative = 0
      for (const item of recommendations.value) {
        if (cumulative + item.total_cost <= budgetValue) {
          next.add(item.item_sku)
          cumulative += item.total_cost
        }
      }
      selectedSkus.value = next
    }

    watch(budget, (newBudget) => {
      autoSelect(newBudget)
    })

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations()
        recommendations.value = data

        const totalPotentialCost = data.reduce((sum, item) => sum + item.total_cost, 0)
        const rounded = Math.ceil(totalPotentialCost / 1000) * 1000
        sliderMax.value = rounded
        budget.value = rounded

        autoSelect(rounded)
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || placingOrder.value) return
      placingOrder.value = true
      error.value = null
      try {
        const warehouse = selectedItems.value[0]?.warehouse || 'San Francisco'
        const items = selectedItems.value.map(item => ({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.restock_quantity,
          unit_cost: item.unit_cost
        }))
        const result = await api.createRestockingOrder(items, warehouse)
        lastOrder.value = result
      } catch (err) {
        error.value = 'Failed to place restocking order'
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    const resetOrder = () => {
      lastOrder.value = null
      loadRecommendations()
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      recommendations,
      selectedSkus,
      budget,
      sliderMax,
      lastOrder,
      selectedItems,
      totalSelected,
      budgetUsedPercent,
      formatCurrency,
      isSelected,
      toggleItem,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget card */
.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.375rem;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1.25rem;
}

.budget-usage {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-usage-text {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
  color: #475569;
}

.item-count-label {
  font-weight: 600;
  color: #2563eb;
}

.progress-bar-container {
  background: #e2e8f0;
  border-radius: 9999px;
  height: 8px;
  overflow: hidden;
}

.progress-bar-fill {
  background: #2563eb;
  border-radius: 9999px;
  height: 8px;
  transition: width 0.3s ease;
}

/* Table */
.col-check {
  width: 40px;
  text-align: center;
}

.recommendation-row {
  cursor: pointer;
}

.row-selected {
  background: #eff6ff !important;
}

/* Order section */
.order-section {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1.5rem;
  margin-top: 0.5rem;
}

.order-total {
  display: flex;
  align-items: baseline;
  gap: 0.5rem;
}

.order-total-label {
  font-size: 0.875rem;
  color: #64748b;
}

.order-total-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.btn-order {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  border: none;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-order:hover:not(.btn-disabled) {
  background: #1d4ed8;
}

.btn-disabled {
  background: #e2e8f0;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Success card */
.success-card {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  border-radius: 10px;
  padding: 1.5rem;
  margin-bottom: 1.25rem;
}

.success-header {
  display: flex;
  align-items: baseline;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.success-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #065f46;
}

.success-order-num {
  font-size: 0.875rem;
  color: #047857;
  font-weight: 600;
}

.success-body {
  color: #065f46;
  font-size: 0.938rem;
  margin-bottom: 1rem;
}

.btn-secondary {
  background: white;
  color: #065f46;
  padding: 0.5rem 1.25rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  border: 1px solid #6ee7b7;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-secondary:hover {
  background: #f0fdf4;
}
</style>
