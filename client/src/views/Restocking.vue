<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on your budget</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-amount">{{ formatCurrency(budget) }}</div>
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="500000"
          step="5000"
          class="budget-slider"
        />
        <div class="budget-meta">
          <div class="budget-meta-item">
            <span class="budget-meta-label">Items selected</span>
            <span class="budget-meta-value">{{ recommendedItems.length }}</span>
          </div>
          <div class="budget-meta-item">
            <span class="budget-meta-label">Total cost</span>
            <span class="budget-meta-value">{{ formatCurrency(totalCost) }}</span>
          </div>
          <div class="budget-meta-item">
            <span class="budget-meta-label">Remaining budget</span>
            <span class="budget-meta-value">{{ formatCurrency(remainingBudget) }}</span>
          </div>
        </div>
      </div>

      <!-- Success State -->
      <div v-if="submittedOrder" class="card success-card">
        <div class="success-title">Order Submitted</div>
        <div class="success-order-number">{{ submittedOrder.order_number }}</div>
        <div class="success-details">
          {{ submittedOrder.items.length }} items &middot; Expected delivery: {{ expectedDelivery }}
        </div>
        <div class="success-total">Total: {{ formatCurrency(submittedOrder.total_value) }}</div>
        <div class="success-actions">
          <router-link to="/orders" class="link-orders">View in Orders tab</router-link>
          <button class="btn-primary" @click="resetOrder">Place Another Order</button>
        </div>
      </div>

      <!-- Recommended Items Card -->
      <div v-else-if="recommendedItems.length > 0" class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <button
            class="btn-primary"
            :disabled="submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.quantity }}</td>
                <td>{{ formatCurrency(item.unit_price) }}</td>
                <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="summary-row">
                <td colspan="2"><strong>{{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }}</strong></td>
                <td>{{ totalQuantity }}</td>
                <td></td>
                <td><strong>{{ formatCurrency(totalCost) }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>
        <div v-if="orderError" class="error" style="margin-top: 1rem;">{{ orderError }}</div>
      </div>

      <!-- Empty State -->
      <div v-else class="card empty-state">
        <p>Increase your budget to see restocking recommendations</p>
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
    const loading = ref(true)
    const error = ref(null)
    const orderError = ref(null)
    const submitting = ref(false)
    const submittedOrder = ref(null)

    const budget = ref(100000)
    const forecasts = ref([])
    const inventoryItems = ref([])

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const recommendedItems = computed(() => {
      // Filter to increasing-trend forecasts that have an inventory match
      const inventoryBySku = {}
      for (const item of inventoryItems.value) {
        inventoryBySku[item.sku] = item
      }

      const candidates = forecasts.value
        .filter(f => f.trend === 'increasing')
        .filter(f => inventoryBySku[f.item_sku] !== undefined)
        .map(f => {
          const inv = inventoryBySku[f.item_sku]
          const gap = f.forecasted_demand - f.current_demand
          return {
            sku: f.item_sku,
            name: f.item_name,
            quantity: gap,
            unit_price: inv.unit_cost,
            gap
          }
        })
        .sort((a, b) => b.gap - a.gap)

      const result = []
      let spent = 0

      for (const item of candidates) {
        const itemCost = item.quantity * item.unit_price
        if (spent + itemCost <= budget.value) {
          result.push({
            sku: item.sku,
            name: item.name,
            quantity: item.quantity,
            unit_price: item.unit_price,
            total_cost: itemCost
          })
          spent += itemCost
        } else {
          const remaining = budget.value - spent
          const partialQty = Math.floor(remaining / item.unit_price)
          if (partialQty > 0) {
            result.push({
              sku: item.sku,
              name: item.name,
              quantity: partialQty,
              unit_price: item.unit_price,
              total_cost: partialQty * item.unit_price
            })
            spent += partialQty * item.unit_price
          }
          break
        }
      }

      return result
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const totalQuantity = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.quantity, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const expectedDelivery = computed(() => {
      if (!submittedOrder.value) return ''
      return new Date(submittedOrder.value.expected_delivery).toLocaleDateString('en-US', {
        year: 'numeric', month: 'long', day: 'numeric'
      })
    })

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })

    const placeOrder = async () => {
      submitting.value = true
      orderError.value = null
      try {
        const order = await api.submitRestockingOrder({
          items: recommendedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.quantity,
            unit_price: item.unit_price
          }))
        })
        submittedOrder.value = order
      } catch (err) {
        orderError.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const resetOrder = () => {
      submittedOrder.value = null
      budget.value = 100000
    }

    onMounted(loadData)

    return {
      loading,
      error,
      orderError,
      submitting,
      submittedOrder,
      budget,
      recommendedItems,
      totalCost,
      totalQuantity,
      remainingBudget,
      expectedDelivery,
      formatCurrency,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* Uses global .main-content padding */
}

/* Budget display */
.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 1.25rem;
}

.budget-meta {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.budget-meta-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-meta-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.budget-meta-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

/* Table footer summary row */
.summary-row td {
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
  font-size: 0.875rem;
  color: #0f172a;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 3rem 1.25rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Success card */
.success-card {
  border-left: 4px solid #10b981;
  background: #f0fdf4;
}

.success-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #065f46;
  margin-bottom: 0.5rem;
}

.success-order-number {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.375rem;
  font-family: monospace;
  letter-spacing: 0.05em;
}

.success-details {
  font-size: 0.875rem;
  color: #64748b;
  margin-bottom: 0.375rem;
}

.success-total {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1.25rem;
}

.success-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.link-orders {
  color: #2563eb;
  text-decoration: none;
  font-size: 0.938rem;
  font-weight: 500;
}

.link-orders:hover {
  text-decoration: underline;
}

/* Primary button */
.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.btn-primary:hover {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
