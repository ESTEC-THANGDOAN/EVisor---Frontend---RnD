<template>
  <div class="module-dashboard" style="padding: 20px">
    <h2>Quản lý Sản xuất (MMSX)</h2>

    <!-- KPI Cards -->
    <el-row :gutter="16" style="margin-bottom: 20px">
      <el-col :span="6">
        <el-card class="kpi-card">
          <div class="kpi-value">{{ stats.total }}</div>
          <div class="kpi-label">Tổng lệnh sản xuất</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-blue">
          <div class="kpi-value">{{ stats.in_progress }}</div>
          <div class="kpi-label">Đang sản xuất</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-green">
          <div class="kpi-value">{{ stats.completed }}</div>
          <div class="kpi-label">Hoàn thành</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-purple">
          <div class="kpi-value">{{ stats.completion_rate }}%</div>
          <div class="kpi-label">Tỷ lệ hoàn thành</div>
        </el-card>
      </el-col>
    </el-row>

    <!-- Toolbar -->
    <div style="margin-bottom: 16px; display: flex; gap: 8px; align-items: center">
      <el-button type="primary" :icon="Plus" @click="openCreate">Thêm lệnh sản xuất</el-button>
      <el-button :icon="Refresh" @click="fetchAll">Làm mới</el-button>
      <el-select v-model="filterStatus" placeholder="Lọc trạng thái" clearable style="width: 180px" @change="fetchOrders">
        <el-option label="Kế hoạch" :value="0" />
        <el-option label="Đang sản xuất" :value="1" />
        <el-option label="Hoàn thành" :value="2" />
        <el-option label="Đã hủy" :value="3" />
      </el-select>
    </div>

    <!-- Table -->
    <el-table :data="orders" v-loading="loading" border stripe style="width: 100%">
      <el-table-column prop="order_code" label="Mã lệnh" width="140" />
      <el-table-column prop="product_name" label="Tên sản phẩm" min-width="180" />
      <el-table-column label="Số lượng" width="110" align="right">
        <template #default="{ row }">{{ row.quantity }} {{ row.unit }}</template>
      </el-table-column>
      <el-table-column label="Kế hoạch bắt đầu" width="150">
        <template #default="{ row }">{{ formatDate(row.planned_start) }}</template>
      </el-table-column>
      <el-table-column label="Kế hoạch kết thúc" width="150">
        <template #default="{ row }">{{ formatDate(row.planned_end) }}</template>
      </el-table-column>
      <el-table-column label="Trạng thái" width="140" align="center">
        <template #default="{ row }">
          <el-tag :type="orderStatusType(row.status)">{{ orderStatusLabel(row.status) }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="Thao tác" width="160" align="center" fixed="right">
        <template #default="{ row }">
          <el-button size="small" @click="openEdit(row)">Sửa</el-button>
          <el-button size="small" type="danger" @click="handleDelete(row)">Xóa</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- Form Dialog -->
    <el-dialog v-model="dialogVisible" :title="isEdit ? 'Cập nhật lệnh sản xuất' : 'Thêm lệnh sản xuất'" width="600px" destroy-on-close>
      <el-form :model="form" label-width="160px" label-position="left">
        <el-form-item label="Mã lệnh" v-if="!isEdit">
          <el-input v-model="form.order_code" placeholder="LSX-2025-001" />
        </el-form-item>
        <el-form-item label="Tên sản phẩm">
          <el-input v-model="form.product_name" />
        </el-form-item>
        <el-form-item label="Số lượng">
          <el-input-number v-model="form.quantity" :min="0" style="width: 150px" />
          <el-input v-model="form.unit" placeholder="cái" style="width: 100px; margin-left: 8px" />
        </el-form-item>
        <el-form-item label="Ngày bắt đầu (KH)">
          <el-date-picker v-model="form.planned_start" type="datetime" placeholder="Chọn ngày" style="width: 100%" />
        </el-form-item>
        <el-form-item label="Ngày kết thúc (KH)">
          <el-date-picker v-model="form.planned_end" type="datetime" placeholder="Chọn ngày" style="width: 100%" />
        </el-form-item>
        <el-form-item label="Trạng thái" v-if="isEdit">
          <el-select v-model="form.status" style="width: 100%">
            <el-option label="Kế hoạch" :value="0" />
            <el-option label="Đang sản xuất" :value="1" />
            <el-option label="Hoàn thành" :value="2" />
            <el-option label="Đã hủy" :value="3" />
          </el-select>
        </el-form-item>
        <el-form-item label="Ghi chú">
          <el-input v-model="form.notes" type="textarea" :rows="2" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">Hủy</el-button>
        <el-button type="primary" :loading="formLoading" @click="handleSubmit">
          {{ isEdit ? 'Cập nhật' : 'Tạo lệnh' }}
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script>
import { Plus, Refresh } from "@element-plus/icons-vue";
import { ElMessage, ElMessageBox } from "element-plus";
import { computed, onMounted, ref } from "vue";
import { useAuthStore } from "../../stores/auth";
import { mmsxViewApi, mmsxDashboardApi, mmsxDmlApi } from "../../services/auth.service";

const ORDER_STATUS_LABEL = ["Kế hoạch", "Đang sản xuất", "Hoàn thành", "Đã hủy"];
const ORDER_STATUS_TYPE = ["info", "warning", "success", "danger"];

export default {
  name: "MMSXDashboard",
  setup() {
    const authStore = useAuthStore();
    const owner = computed(() => authStore.user?.id || "");

    const loading = ref(false);
    const formLoading = ref(false);
    const orders = ref([]);
    const filterStatus = ref(null);
    const dialogVisible = ref(false);
    const isEdit = ref(false);
    const stats = ref({ total: 0, planned: 0, in_progress: 0, completed: 0, cancelled: 0, completion_rate: 0 });

    const emptyForm = () => ({ order_code: "", product_name: "", quantity: 0, unit: "cái", planned_start: null, planned_end: null, actual_start: null, actual_end: null, status: 0, notes: "", id: null });
    const form = ref(emptyForm());

    const fetchOrders = async () => {
      loading.value = true;
      try {
        const payload = { request_id: "evisor-" + Date.now(), owner: owner.value };
        if (filterStatus.value !== null) payload.filter_status = filterStatus.value;
        const res = await mmsxViewApi(payload);
        if (res.status === "success") orders.value = res.data;
        else ElMessage.error(res.message || "Không thể tải dữ liệu.");
      } catch (e) { ElMessage.error(e.message); }
      finally { loading.value = false; }
    };

    const fetchStats = async () => {
      try {
        const res = await mmsxDashboardApi({ request_id: "evisor-" + Date.now(), owner: owner.value });
        if (res.status === "success") stats.value = res.data;
      } catch { /* silent */ }
    };

    const fetchAll = () => { fetchOrders(); fetchStats(); };

    const openCreate = () => { form.value = emptyForm(); isEdit.value = false; dialogVisible.value = true; };
    const openEdit = (row) => { form.value = { ...row }; isEdit.value = true; dialogVisible.value = true; };

    const handleSubmit = async () => {
      if (!form.value.product_name) return ElMessage.warning("Vui lòng nhập tên sản phẩm.");
      formLoading.value = true;
      try {
        const action = isEdit.value ? "update" : "insert";
        const payload = { request_id: "evisor-" + Date.now(), owner: owner.value, dml_action: action, form: { ...form.value } };
        const res = await mmsxDmlApi(payload);
        if (res.status === "success") {
          ElMessage.success(res.message);
          dialogVisible.value = false;
          fetchAll();
        } else ElMessage.error(res.message);
      } catch (e) { ElMessage.error(e.message); }
      finally { formLoading.value = false; }
    };

    const handleDelete = async (row) => {
      try {
        await ElMessageBox.confirm(`Xóa lệnh sản xuất "${row.order_code}"?`, "Xác nhận", { type: "warning" });
        const res = await mmsxDmlApi({ request_id: "evisor-" + Date.now(), owner: owner.value, dml_action: "delete", form: { id: row.id } });
        if (res.status === "success") { ElMessage.success(res.message); fetchAll(); }
        else ElMessage.error(res.message);
      } catch (e) { if (e !== "cancel") ElMessage.error(e.message); }
    };

    const formatDate = (iso) => iso ? iso.replace("T", " ").substring(0, 16) : "—";
    const orderStatusLabel = (s) => ORDER_STATUS_LABEL[s] || "—";
    const orderStatusType = (s) => ORDER_STATUS_TYPE[s] || "";

    onMounted(fetchAll);

    return { Plus, Refresh, loading, formLoading, orders, filterStatus, dialogVisible, isEdit, form, stats, fetchAll, fetchOrders, openCreate, openEdit, handleSubmit, handleDelete, formatDate, orderStatusLabel, orderStatusType };
  },
};
</script>

<style scoped>
.module-dashboard { background-color: white; min-height: calc(100vh - 60px); }
.kpi-card { text-align: center; cursor: default; }
.kpi-value { font-size: 2rem; font-weight: 700; color: #333; }
.kpi-label { font-size: 0.85rem; color: #888; margin-top: 4px; }
.kpi-blue .kpi-value { color: #409eff; }
.kpi-green .kpi-value { color: #67c23a; }
.kpi-purple .kpi-value { color: #9b59b6; }
</style>
