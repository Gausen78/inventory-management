<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget to get restocking recommendations from the demand forecast.</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
        <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
      </div>
      <div class="slider-section">
        <div class="slider-labels">
          <span class="slider-label-min">{{ currencySymbol }}0</span>
          <span class="slider-label-max">{{ currencySymbol }}500K</span>
        </div>
        <input
          type="range"
          class="budget-slider"
          min="0"
          max="500000"
          step="5000"
          v-model.number="budget"
        />
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error && !successMessage" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Recommended Total</div>
          <div class="stat-value">{{ currencySymbol }}{{ (recommendations.recommended_total || 0).toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Remaining Budget</div>
          <div class="stat-value">{{ currencySymbol }}{{ (recommendations.remaining_budget || 0).toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Items Recommended</div>
          <div class="stat-value">{{ recommendations.item_count || 0 }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ (recommendations.items || []).length }})</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item</th>
                <th>Trend</th>
                <th>Recommended Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="!recommendations.items || recommendations.items.length === 0">
                <td colspan="7" class="empty-state">No items fit this budget — increase your budget.</td>
              </tr>
              <tr v-for="item in recommendations.items" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', getTrendBadgeClass(item.trend)]">{{ item.trend }}</span>
                </td>
                <td>{{ item.recommended_quantity.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
                <td>{{ item.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-if="successMessage" class="success-banner">
        {{ successMessage }}
      </div>

      <div v-if="submitError" class="error">{{ submitError }}</div>

      <div class="actions-row">
        <button
          class="btn-primary"
          :disabled="!recommendations.items || recommendations.items.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>
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

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(100000)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submitError = ref(null)
    const successMessage = ref(null)
    const recommendations = ref({
      budget: 0,
      recommended_total: 0,
      remaining_budget: 0,
      item_count: 0,
      items: []
    })

    let debounceTimer = null

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockRecommendations(budget.value)
        recommendations.value = data
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    watch(budget, () => {
      successMessage.value = null
      submitError.value = null
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 250)
    })

    const getTrendBadgeClass = (trend) => {
      const map = {
        increasing: 'success',
        decreasing: 'danger',
        stable: 'info'
      }
      return map[trend] || 'info'
    }

    const placeOrder = async () => {
      if (!recommendations.value.items || recommendations.value.items.length === 0) return

      submitting.value = true
      submitError.value = null
      successMessage.value = null

      try {
        const orderData = {
          budget: budget.value,
          items: recommendations.value.items.map(item => ({
            item_sku: item.item_sku,
            item_name: item.item_name,
            quantity: item.recommended_quantity,
            unit_cost: item.unit_cost,
            line_total: item.line_total,
            trend: item.trend,
            lead_time_days: item.lead_time_days
          }))
        }

        const result = await api.submitRestockOrder(orderData)
        successMessage.value = `Order ${result.order_number} placed successfully. It now appears in the Orders tab under "Submitted Orders".`
        await loadRecommendations()
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      currencySymbol,
      budget,
      loading,
      error,
      submitting,
      submitError,
      successMessage,
      recommendations,
      getTrendBadgeClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  align-items: center;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-section {
  padding-top: 0.5rem;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
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
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(59, 130, 246, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 1px 8px rgba(59, 130, 246, 0.6);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(59, 130, 246, 0.4);
}

.restock-table {
  width: 100%;
}

.empty-state {
  text-align: center;
  color: #64748b;
  font-style: italic;
  padding: 2rem 0.75rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.actions-row {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 1.25rem;
}

.btn-primary {
  background: #3b82f6;
  color: #ffffff;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
