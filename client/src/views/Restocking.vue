<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Manage inventory restocking based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success Banner -->
      <div v-if="successOrder" class="success-banner">
        <div class="success-banner-content">
          <div>
            <strong>Order placed successfully.</strong>
            Order <strong>{{ successOrder.order_number }}</strong> confirmed —
            expected delivery <strong>{{ successOrder.expected_delivery }}</strong>.
            Total value: <strong>${{ Number(successOrder.total_value).toLocaleString() }}</strong>.
          </div>
          <button class="success-dismiss" @click="successOrder = null" aria-label="Dismiss">
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 20 20" fill="currentColor">
              <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
            </svg>
          </button>
        </div>
      </div>

      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget Control</h3>
        </div>
        <div class="budget-display">${{ budgetSlider.toLocaleString() }}</div>
        <input
          type="range"
          class="budget-slider"
          v-model.number="budgetSlider"
          :min="0"
          :max="totalPotentialCost"
          step="100"
        />
        <div class="budget-summary">
          {{ selectedItems.length }} items selected —
          ${{ currentSelectedCost.toLocaleString() }} of ${{ totalPotentialCost.toLocaleString() }} total potential
        </div>
        <div class="progress-bar-track">
          <div
            class="progress-bar-fill"
            :style="{ width: totalPotentialCost > 0 ? (currentSelectedCost / totalPotentialCost * 100) + '%' : '0%' }"
          ></div>
        </div>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Restock Recommendations ({{ sortedRecommendations.length }} items)</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>Include</th>
                <th>Priority</th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>On Hand</th>
                <th>Reorder Pt</th>
                <th>Forecasted</th>
                <th>Rec. Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Reason</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedRecommendations"
                :key="item.forecast_id"
                :class="{ 'row-selected': greedySelection.has(item.forecast_id) }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="greedySelection.has(item.forecast_id)"
                    @change="toggleItem(item.forecast_id)"
                  />
                </td>
                <td>
                  <span :class="['badge', item.priority]">{{ item.priority }}</span>
                </td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td>{{ item.reorder_point.toLocaleString() }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ item.recommended_quantity.toLocaleString() }}</td>
                <td>${{ Number(item.unit_cost).toLocaleString() }}</td>
                <td>${{ Number(item.total_cost).toLocaleString() }}</td>
                <td>
                  <span class="reason-text">{{ formatReason(item.reason) }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Row -->
      <div class="place-order-row">
        <button
          class="place-order-btn"
          @click="placeOrder"
          :disabled="selectedItems.length === 0 || submitting"
        >
          {{ submitting ? 'Placing...' : `Place Order (${selectedItems.length} items — $${currentSelectedCost.toLocaleString()})` }}
        </button>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    // Raw state — never mutate recommendations directly
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const budgetSlider = ref(0)
    // { [forecast_id]: true | false } — only set when user explicitly clicks
    const manualOverrides = ref({})
    const submitting = ref(false)
    const successOrder = ref(null)

    // Priority rank for sorting: high=0, medium=1, low=2
    const priorityRank = { high: 0, medium: 1, low: 2 }

    const sortedRecommendations = computed(() => {
      return [...recommendations.value].sort((a, b) => {
        const rankDiff = (priorityRank[a.priority] ?? 3) - (priorityRank[b.priority] ?? 3)
        if (rankDiff !== 0) return rankDiff
        // Within same priority tier, sort ascending by total_cost
        return Number(a.total_cost) - Number(b.total_cost)
      })
    })

    const totalPotentialCost = computed(() => {
      return recommendations.value.reduce((sum, item) => sum + Number(item.total_cost), 0)
    })

    // Greedy selection: iterate sorted list, apply overrides, fill within budget
    const greedySelection = computed(() => {
      const selected = new Set()
      let remaining = budgetSlider.value

      for (const item of sortedRecommendations.value) {
        const override = manualOverrides.value[item.forecast_id]
        if (override === false) {
          // User explicitly excluded — skip
          continue
        } else if (override === true) {
          // User explicitly included — always add regardless of budget
          selected.add(item.forecast_id)
          remaining -= Number(item.total_cost)
        } else {
          // Automatic: add only if it fits in remaining budget
          if (Number(item.total_cost) <= remaining) {
            selected.add(item.forecast_id)
            remaining -= Number(item.total_cost)
          }
        }
      }

      return selected
    })

    const selectedItems = computed(() => {
      return recommendations.value.filter(item => greedySelection.value.has(item.forecast_id))
    })

    const currentSelectedCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + Number(item.total_cost), 0)
    })

    // Toggle: if currently in greedy set → force-exclude; if not → force-include
    const toggleItem = (id) => {
      manualOverrides.value = {
        ...manualOverrides.value,
        [id]: !greedySelection.value.has(id)
      }
    }

    const formatReason = (reason) => {
      if (reason === 'both') return 'Demand Gap + Low Stock'
      if (reason === 'demand gap') return 'Demand Gap'
      if (reason === 'low stock') return 'Low Stock'
      return reason
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockRecommendations()
        recommendations.value = data
        // Initialize slider to full potential cost after load
        const total = data.reduce((sum, item) => sum + Number(item.total_cost), 0)
        budgetSlider.value = total
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (!selectedItems.value.length) return
      try {
        submitting.value = true
        const payload = {
          items: selectedItems.value.map(r => ({
            item_sku: r.item_sku,
            item_name: r.item_name,
            quantity: r.recommended_quantity,
            unit_cost: r.unit_cost,
            total_cost: r.total_cost
          })),
          budget: budgetSlider.value
        }
        const order = await api.placeRestockOrder(payload)
        successOrder.value = order
        manualOverrides.value = {}
        // Reload so slider re-initializes to new potential cost
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      recommendations,
      loading,
      error,
      budgetSlider,
      manualOverrides,
      submitting,
      successOrder,
      sortedRecommendations,
      totalPotentialCost,
      greedySelection,
      selectedItems,
      currentSelectedCost,
      toggleItem,
      formatReason,
      loadRecommendations,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.75rem;
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.progress-bar-track {
  width: 100%;
  height: 6px;
  background: #e2e8f0;
  border-radius: 9999px;
  overflow: hidden;
  margin-top: 0.25rem;
}

.progress-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 9999px;
  transition: width 0.2s ease;
}

.row-selected td:first-child {
  border-left: 3px solid #2563eb;
}

.row-selected {
  background: #eff6ff;
}

.place-order-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 1rem;
  margin-bottom: 1.5rem;
}

.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  font-size: 0.938rem;
  font-weight: 600;
  padding: 0.625rem 1.5rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #065f46;
  font-size: 0.938rem;
}

.success-banner-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.success-dismiss {
  flex-shrink: 0;
  background: transparent;
  border: none;
  cursor: pointer;
  color: #065f46;
  display: flex;
  align-items: center;
  padding: 0.25rem;
  border-radius: 4px;
  transition: background 0.15s ease;
}

.success-dismiss:hover {
  background: #a7f3d0;
}

.reason-text {
  font-size: 0.8125rem;
  color: #64748b;
}
</style>
