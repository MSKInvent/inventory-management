<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'view' ? 'Purchase Order' : 'Create Purchase Order' }}
            </h3>
            <button class="close-button" @click="close" aria-label="Close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body" v-if="backlogItem">
            <div class="item-summary">
              <strong>{{ backlogItem.item_name }}</strong>
              <span>{{ backlogItem.item_sku }} · Order {{ backlogItem.order_id }}</span>
              <span>Shortage: {{ shortage }} units</span>
            </div>

            <div v-if="error" class="error-message">{{ error }}</div>

            <!-- View mode -->
            <template v-if="mode === 'view'">
              <div v-if="loading" class="status-text">Loading...</div>
              <dl v-else-if="existingPO" class="po-details">
                <dt>PO Number</dt><dd>{{ existingPO.id }}</dd>
                <dt>Supplier</dt><dd>{{ existingPO.supplier_name }}</dd>
                <dt>Quantity</dt><dd>{{ existingPO.quantity }}</dd>
                <dt>Unit Cost</dt><dd>${{ existingPO.unit_cost.toFixed(2) }}</dd>
                <dt>Total</dt><dd>${{ (existingPO.quantity * existingPO.unit_cost).toFixed(2) }}</dd>
                <dt>Expected Delivery</dt><dd>{{ existingPO.expected_delivery_date }}</dd>
                <dt>Status</dt><dd>{{ existingPO.status }}</dd>
                <dt>Created</dt><dd>{{ existingPO.created_date }}</dd>
                <template v-if="existingPO.notes">
                  <dt>Notes</dt><dd>{{ existingPO.notes }}</dd>
                </template>
              </dl>
            </template>

            <!-- Create mode -->
            <form v-else class="po-form" @submit.prevent="submit">
              <div class="form-group">
                <label for="po-supplier">Supplier</label>
                <input id="po-supplier" v-model="form.supplier_name" type="text" required />
              </div>
              <div class="form-row">
                <div class="form-group">
                  <label for="po-quantity">Quantity</label>
                  <input id="po-quantity" v-model.number="form.quantity" type="number" min="1" required />
                </div>
                <div class="form-group">
                  <label for="po-cost">Unit Cost ($)</label>
                  <input id="po-cost" v-model.number="form.unit_cost" type="number" min="0" step="0.01" required />
                </div>
              </div>
              <div class="form-group">
                <label for="po-delivery">Expected Delivery</label>
                <input id="po-delivery" v-model="form.expected_delivery_date" type="date" required />
              </div>
              <div class="form-group">
                <label for="po-notes">Notes</label>
                <textarea id="po-notes" v-model="form.notes" rows="2"></textarea>
              </div>
              <div class="modal-footer">
                <button type="button" class="btn-secondary" @click="close">Cancel</button>
                <button type="submit" class="btn-primary" :disabled="submitting || !canSubmit">
                  {{ submitting ? 'Creating...' : 'Create PO' }}
                </button>
              </div>
            </form>
          </div>

          <div v-if="mode === 'view'" class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

const emptyForm = () => ({
  supplier_name: '',
  quantity: 1,
  unit_cost: 0,
  expected_delivery_date: '',
  notes: ''
})

export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: { type: Boolean, required: true },
    backlogItem: { type: Object, default: null },
    mode: { type: String, default: 'create' } // 'create' | 'view'
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const form = ref(emptyForm())
    const existingPO = ref(null)
    const loading = ref(false)
    const submitting = ref(false)
    const error = ref('')

    const shortage = computed(() => {
      const item = props.backlogItem
      return item ? Math.max(item.quantity_needed - item.quantity_available, 0) : 0
    })

    const canSubmit = computed(() =>
      form.value.supplier_name.trim() &&
      form.value.quantity > 0 &&
      form.value.unit_cost >= 0 &&
      form.value.expected_delivery_date
    )

    // Reset state each time the modal opens so a previous item's values (or
    // error) never leak into the next one; default quantity to the shortage.
    watch(() => props.isOpen, async (open) => {
      if (!open || !props.backlogItem) return
      error.value = ''
      existingPO.value = null
      form.value = { ...emptyForm(), quantity: shortage.value || 1 }

      if (props.mode === 'view') {
        loading.value = true
        try {
          existingPO.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
        } catch (err) {
          error.value = 'Failed to load purchase order: ' + err.message
        } finally {
          loading.value = false
        }
      }
    })

    const close = () => emit('close')

    const submit = async () => {
      if (!canSubmit.value || submitting.value) return
      submitting.value = true
      error.value = ''
      try {
        const po = await api.createPurchaseOrder({
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name.trim(),
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes.trim() || null
        })
        emit('po-created', po)
      } catch (err) {
        error.value = err.response?.data?.detail || err.message
      } finally {
        submitting.value = false
      }
    }

    return { form, existingPO, loading, submitting, error, shortage, canSubmit, close, submit }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  width: 90%;
  max-width: 560px;
  max-height: 85vh;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.5rem 2rem;
  border-bottom: 2px solid #e2e8f0;
}

.modal-title {
  font-size: 1.5rem;
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  border-radius: 6px;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  padding: 1.5rem 2rem;
  overflow-y: auto;
}

.item-summary {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  background: #f8fafc;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1.25rem;
  color: #475569;
  font-size: 0.9rem;
}

.item-summary strong {
  color: #0f172a;
  font-size: 1rem;
}

.error-message {
  background: #fecaca;
  color: #991b1b;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  margin-bottom: 1rem;
  font-size: 0.9rem;
}

.status-text {
  color: #64748b;
  text-align: center;
  padding: 1rem;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.form-row {
  display: flex;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  flex: 1;
}

label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

input,
textarea {
  padding: 0.75rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  font-family: inherit;
}

input:focus,
textarea:focus {
  outline: none;
  border-color: #667eea;
}

.po-details {
  display: grid;
  grid-template-columns: max-content 1fr;
  gap: 0.5rem 1.5rem;
  margin: 0;
}

.po-details dt {
  font-weight: 600;
  color: #475569;
}

.po-details dd {
  margin: 0;
  color: #0f172a;
}

.modal-footer {
  padding: 1rem 2rem 1.5rem;
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
}

.po-form .modal-footer {
  padding: 0.5rem 0 0;
}

.btn-secondary,
.btn-primary {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
}

.btn-secondary {
  background: #f1f5f9;
  color: #475569;
}

.btn-secondary:hover {
  background: #e2e8f0;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
</style>
