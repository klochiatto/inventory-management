<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget control -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
          <span class="budget-value">{{ formatCurrency(budget, currentCurrency) }}</span>
        </div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="maxBudget"
          :step="sliderStep"
          v-model.number="budget"
        />
        <div class="slider-scale">
          <span>{{ formatCurrency(0, currentCurrency) }}</span>
          <span>{{ formatCurrency(maxBudget, currentCurrency) }}</span>
        </div>
        <p class="budget-help">{{ t('restocking.budgetHelp') }}</p>

        <div class="stats-grid summary-grid">
          <div class="stat-card info">
            <div class="stat-label">{{ t('restocking.itemsSelected') }}</div>
            <div class="stat-value">{{ summary.itemCount }}</div>
          </div>
          <div class="stat-card success">
            <div class="stat-label">{{ t('restocking.totalCost') }}</div>
            <div class="stat-value">{{ formatCurrency(summary.totalCost, currentCurrency) }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
            <div class="stat-value">{{ formatCurrency(summary.remaining, currentCurrency) }}</div>
          </div>
          <div class="stat-card warning">
            <div class="stat-label">{{ t('restocking.longestLeadTime') }}</div>
            <div class="stat-value">{{ summary.maxLead }}</div>
          </div>
        </div>
      </div>

      <!-- Recommendations -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }} ({{ recommendations.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.restockQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
                <th>{{ t('restocking.table.leadTime') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.id">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
                <td>{{ item.restock_qty.toLocaleString() }}</td>
                <td>{{ formatCurrencyWithDecimals(item.unit_cost, currentCurrency, 2) }}</td>
                <td><strong>{{ formatCurrency(item.line_total, currentCurrency) }}</strong></td>
                <td>{{ t('restocking.days', { count: item.lead_time_days }) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Submit -->
      <div class="submit-row">
        <div v-if="submittedOrder" class="success-banner">
          {{ t('restocking.orderPlaced', { orderNumber: submittedOrder.order_number }) }}
          <router-link to="/orders" class="view-link">{{ t('restocking.viewInOrders') }}</router-link>
        </div>
        <div v-if="submitError" class="error">{{ submitError }}</div>
        <button
          class="btn-primary"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency, formatCurrencyWithDecimals } from '../utils/currency'

// Order in which trends are prioritized when filling the budget
const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(0)
    const submitting = ref(false)
    const submitError = ref(null)
    const submittedOrder = ref(null)

    // Cost to restock every item to its forecasted demand — the slider's ceiling
    const maxBudget = computed(() => {
      const total = forecasts.value.reduce((sum, f) => {
        const gap = Math.max(0, f.forecasted_demand - f.current_demand)
        return sum + gap * f.unit_cost
      }, 0)
      // Round up to a clean value so the slider ends on a round number
      return Math.max(1000, Math.ceil(total / 500) * 500)
    })

    const sliderStep = computed(() => Math.max(50, Math.round(maxBudget.value / 100 / 50) * 50))

    // Budget-constrained recommendation: fill the demand gap, prioritizing
    // increasing-trend items and larger gaps, adding each whole line while it
    // still fits the remaining budget. Recomputes instantly as the slider moves.
    const recommendations = computed(() => {
      const candidates = forecasts.value
        .map(f => {
          const restockQty = Math.max(0, f.forecasted_demand - f.current_demand)
          return { ...f, restock_qty: restockQty, line_total: restockQty * f.unit_cost }
        })
        .filter(c => c.restock_qty > 0)
        .sort((a, b) => {
          const byTrend = TREND_PRIORITY[a.trend] - TREND_PRIORITY[b.trend]
          if (byTrend !== 0) return byTrend
          return b.line_total - a.line_total
        })

      let remaining = budget.value
      const selected = []
      for (const c of candidates) {
        if (c.line_total <= remaining) {
          selected.push(c)
          remaining -= c.line_total
        }
      }
      return selected
    })

    const summary = computed(() => {
      const items = recommendations.value
      const totalCost = items.reduce((s, i) => s + i.line_total, 0)
      const maxLead = items.reduce((m, i) => Math.max(m, i.lead_time_days), 0)
      return {
        itemCount: items.length,
        totalCost,
        remaining: budget.value - totalCost,
        maxLead
      }
    })

    const loadForecasts = async () => {
      try {
        loading.value = true
        forecasts.value = await api.getDemandForecasts()
        // Start the slider around the midpoint so the budget constraint is visible
        budget.value = Math.round(maxBudget.value / 2 / 100) * 100
      } catch (err) {
        error.value = 'Failed to load forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0) return
      try {
        submitting.value = true
        submitError.value = null
        submittedOrder.value = null
        const payload = {
          items: recommendations.value.map(r => ({ sku: r.item_sku, quantity: r.restock_qty })),
          budget: budget.value
        }
        submittedOrder.value = await api.createRestockingOrder(payload)
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      currentCurrency,
      loading,
      error,
      budget,
      maxBudget,
      sliderStep,
      recommendations,
      summary,
      submitting,
      submitError,
      submittedOrder,
      placeOrder,
      formatCurrency,
      formatCurrencyWithDecimals
    }
  }
}
</script>

<style scoped>
.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
  margin: 0.5rem 0 0.25rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.slider-scale {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.budget-help {
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.summary-grid {
  margin-top: 1.5rem;
  margin-bottom: 0;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.submit-row {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
}

.success-banner {
  width: 100%;
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.938rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.view-link {
  color: #047857;
  font-weight: 600;
  text-decoration: underline;
}

.btn-primary {
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 1.75rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
