<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget, review recommended items based on demand forecast, and submit a purchase order.</p>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-body">
        <div class="budget-label-row">
          <span class="budget-label">Available Budget</span>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          type="range"
          min="5000"
          max="250000"
          step="5000"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="sub-stats-row">
          <div class="sub-stat">
            <div class="sub-stat-label">Budget</div>
            <div class="sub-stat-value">{{ formatCurrency(budget) }}</div>
          </div>
          <div class="sub-stat">
            <div class="sub-stat-label">Allocated</div>
            <div class="sub-stat-value">{{ formatCurrency(editedAllocated) }}</div>
          </div>
          <div class="sub-stat">
            <div class="sub-stat-label">Remaining</div>
            <div class="sub-stat-value" :class="{ 'value-negative': remaining < 0 }">{{ formatCurrency(remaining) }}</div>
          </div>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items ({{ items.length }})</h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="items.length === 0" class="empty-state">
        No items recommended for this budget. Try increasing the budget.
      </div>
      <div v-else>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-sku">SKU</th>
                <th class="col-item">Item</th>
                <th class="col-category">Category</th>
                <th class="col-trend">Trend</th>
                <th class="col-lead">Lead Time</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-qty">Quantity</th>
                <th class="col-total">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in items" :key="row.sku">
                <td class="col-sku"><strong>{{ row.sku }}</strong></td>
                <td class="col-item">{{ row.name }}</td>
                <td class="col-category">{{ row.category }}</td>
                <td class="col-trend">
                  <span :class="['badge', row.trend]">{{ row.trend }}</span>
                </td>
                <td class="col-lead">{{ row.lead_time_days }} days</td>
                <td class="col-cost">${{ row.unit_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td class="col-qty">
                  <input
                    type="number"
                    min="0"
                    step="1"
                    v-model.number="row.quantity"
                    class="qty-input"
                  />
                </td>
                <td class="col-total">${{ computedLineTotal(row).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
              </tr>
              <tr class="totals-row">
                <td colspan="7" class="totals-label">Total</td>
                <td class="col-total"><strong>${{ editedAllocated.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="order-actions">
          <button
            class="place-order-btn"
            :disabled="!canSubmit"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>

          <div v-if="submitMessage" :class="['submit-message', submitMessage.type]">
            {{ submitMessage.text }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const items = ref([])
    const serverAllocated = ref(0)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submitMessage = ref(null)

    let debounceTimer = null

    const computedLineTotal = (row) => {
      return (row.quantity || 0) * row.unit_cost
    }

    const editedAllocated = computed(() => {
      return items.value.reduce((sum, row) => sum + computedLineTotal(row), 0)
    })

    const remaining = computed(() => {
      return budget.value - editedAllocated.value
    })

    const canSubmit = computed(() => {
      return !submitting.value && items.value.some(row => (row.quantity || 0) > 0)
    })

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', {
        style: 'currency',
        currency: 'USD',
        minimumFractionDigits: 0,
        maximumFractionDigits: 0
      })
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockRecommendations(budget.value)
        serverAllocated.value = data.allocated
        items.value = data.items.map(item => ({
          ...item,
          quantity: item.recommended_quantity
        }))
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      const orderItems = items.value
        .filter(i => (i.quantity || 0) > 0)
        .map(i => ({
          sku: i.sku,
          name: i.name,
          category: i.category,
          quantity: i.quantity,
          unit_cost: i.unit_cost
        }))

      if (orderItems.length === 0) return

      submitting.value = true
      submitMessage.value = null

      try {
        const result = await api.submitRestockOrder(orderItems)
        submitMessage.value = {
          type: 'success',
          text: `Order ${result.id} submitted — expected delivery ${result.expected_delivery}`
        }
        await loadRecommendations()
      } catch (err) {
        submitMessage.value = {
          type: 'error',
          text: 'Failed to submit order: ' + err.message
        }
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 250)
    })

    onMounted(() => loadRecommendations())

    return {
      budget,
      items,
      loading,
      error,
      submitting,
      submitMessage,
      editedAllocated,
      remaining,
      canSubmit,
      formatCurrency,
      computedLineTotal,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-body {
  padding: 0.5rem 0;
}

.budget-label-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.75rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 1.25rem;
}

.sub-stats-row {
  display: flex;
  gap: 2rem;
}

.sub-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.sub-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.sub-stat-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.sub-stat-value.value-negative {
  color: #dc2626;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-sku { width: 110px; }
.col-item { width: auto; }
.col-category { width: 130px; }
.col-trend { width: 110px; }
.col-lead { width: 110px; }
.col-cost { width: 110px; }
.col-qty { width: 110px; }
.col-total { width: 130px; }

.qty-input {
  width: 80px;
  padding: 0.25rem 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  font-size: 0.875rem;
  color: #0f172a;
  text-align: right;
  background: #f8fafc;
  transition: border-color 0.15s;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  background: white;
}

.totals-row {
  background: #f8fafc;
  font-weight: 700;
}

.totals-label {
  text-align: right;
  color: #475569;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.order-actions {
  padding: 1.25rem 0.75rem 0.25rem;
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.submit-message {
  padding: 0.625rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
}

.submit-message.success {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
}

.submit-message.error {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}
</style>
