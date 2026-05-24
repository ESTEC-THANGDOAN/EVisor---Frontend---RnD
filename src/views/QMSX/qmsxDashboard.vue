<template>
  <div class="module-dashboard" style="padding: 20px">
    <h2>Quản lý Chất lượng (QMSX)</h2>

    <!-- KPI Cards -->
    <el-row :gutter="16" style="margin-bottom: 20px">
      <el-col :span="6">
        <el-card class="kpi-card">
          <div class="kpi-value">{{ stats.total }}</div>
          <div class="kpi-label">Tổng phiếu kiểm tra</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-green">
          <div class="kpi-value">{{ stats.passed }}</div>
          <div class="kpi-label">Đạt</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-red">
          <div class="kpi-value">{{ stats.failed }}</div>
          <div class="kpi-label">Không đạt</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-blue">
          <div class="kpi-value">{{ stats.pass_rate }}%</div>
          <div class="kpi-label">Tỷ lệ đạt</div>
        </el-card>
      </el-col>
    </el-row>

    <!-- Toolbar -->
    <div style="margin-bottom: 16px; display: flex; gap: 8px; align-items: center">
      <el-button type="primary" :icon="Plus" @click="openCreate">Thêm phiếu kiểm tra</el-button>
      <el-button :icon="Refresh" @click="fetchAll">Làm mới</el-button>
      <el-select v-model="filterResult" placeholder="Lọc kết quả" clearable style="width: 180px" @change="fetchInspections">
        <el-option label="Chờ kiểm tra" :value="0" />
        <el-option label="Đạt" :value="1" />
        <el-option label="Không đạt" :value="2" />
      </el-select>
    </div>

    <!-- Table -->
    <el-table :data="inspections" v-loading="loading" border stripe style="width: 100%">
      <el-table-column prop="inspection_code" label="Mã phiếu" width="140" />
      <el-table-column prop="product_name" label="Tên sản phẩm" min-width="160" />
      <el-table-column prop="reference_order" label="Lệnh SX liên quan" width="150" />
      <el-table-column prop="inspector" label="Người kiểm" width="130" />
      <el-table-column label="KT / Đạt / Lỗi" width="130" align="center">
        <template #default="{ row }">{{ row.qty_checked }} / {{ row.qty_passed }} / {{ row.qty_failed }}</template>
      </el-table-column>
      <el-table-column label="Ngày kiểm" width="140">
        <template #default="{ row }">{{ formatDate(row.inspection_date) }}</template>
      </el-table-column>
      <el-table-column label="Kết quả" width="130" align="center">
        <template #default="{ row }">
          <el-tag :type="resultType(row.result)">{{ resultLabel(row.result) }}</el-tag>
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
    <el-dialog v-model="dialogVisible" :title="isEdit ? 'Cập nhật phiếu kiểm tra' : 'Thêm phiếu kiểm tra'" width="600px" destroy-on-close>
      <el-form :model="form" label-width="170px" label-position="left">
        <el-form-item label="Mã phiếu" v-if="!isEdit">
          <el-input v-model="form.inspection_code" placeholder="QC-2025-001" />
        </el-form-item>
        <el-form-item label="Tên sản phẩm">
          <el-input v-model="form.product_name" />
        </el-form-item>
        <el-form-item label="Mã lệnh SX liên quan">
          <el-input v-model="form.reference_order" placeholder="LSX-2025-001" />
        </el-form-item>
        <el-form-item label="Người kiểm tra">
          <el-input v-model="form.inspector" />
        </el-form-item>
        <el-form-item label="Ngày kiểm tra">
          <el-date-picker v-model="form.inspection_date" type="datetime" placeholder="Chọn ngày" style="width: 100%" />
        </el-form-item>
        <el-form-item label="Số lượng kiểm tra">
          <el-input-number v-model="form.qty_checked" :min="0" />
        </el-form-item>
        <el-form-item label="Số lượng đạt">
          <el-input-number v-model="form.qty_passed" :min="0" :max="form.qty_checked" />
        </el-form-item>
        <el-form-item label="Kết quả">
          <el-select v-model="form.result" style="width: 100%">
            <el-option label="Chờ kiểm tra" :value="0" />
            <el-option label="Đạt" :value="1" />
            <el-option label="Không đạt" :value="2" />
          </el-select>
        </el-form-item>
        <el-form-item label="Ghi chú">
          <el-input v-model="form.notes" type="textarea" :rows="2" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">Hủy</el-button>
        <el-button type="primary" :loading="formLoading" @click="handleSubmit">
          {{ isEdit ? 'Cập nhật' : 'Tạo phiếu' }}
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
import { qmsxViewApi, qmsxDashboardApi, qmsxDmlApi } from "../../services/auth.service";

const RESULT_LABEL = ["Chờ kiểm tra", "Đạt", "Không đạt"];
const RESULT_TYPE = ["info", "success", "danger"];

export default {
  name: "QMSXDashboard",
  setup() {
    const authStore = useAuthStore();
    const owner = computed(() => authStore.user?.id || "");

    const loading = ref(false);
    const formLoading = ref(false);
    const inspections = ref([]);
    const filterResult = ref(null);
    const dialogVisible = ref(false);
    const isEdit = ref(false);
    const stats = ref({ total: 0, pending: 0, passed: 0, failed: 0, qty_checked: 0, qty_passed: 0, qty_failed: 0, pass_rate: 0 });

    const emptyForm = () => ({ inspection_code: "", product_name: "", reference_order: "", inspector: "", inspection_date: null, qty_checked: 0, qty_passed: 0, result: 0, notes: "", id: null });
    const form = ref(emptyForm());

    const fetchInspections = async () => {
      loading.value = true;
      try {
        const payload = { request_id: "evisor-" + Date.now(), owner: owner.value };
        if (filterResult.value !== null) payload.filter_result = filterResult.value;
        const res = await qmsxViewApi(payload);
        if (res.status === "success") inspections.value = res.data;
        else ElMessage.error(res.message || "Không thể tải dữ liệu.");
      } catch (e) { ElMessage.error(e.message); }
      finally { loading.value = false; }
    };

    const fetchStats = async () => {
      try {
        const res = await qmsxDashboardApi({ request_id: "evisor-" + Date.now(), owner: owner.value });
        if (res.status === "success") stats.value = res.data;
      } catch { /* silent */ }
    };

    const fetchAll = () => { fetchInspections(); fetchStats(); };

    const openCreate = () => { form.value = emptyForm(); isEdit.value = false; dialogVisible.value = true; };
    const openEdit = (row) => { form.value = { ...row }; isEdit.value = true; dialogVisible.value = true; };

    const handleSubmit = async () => {
      if (!form.value.product_name) return ElMessage.warning("Vui lòng nhập tên sản phẩm.");
      formLoading.value = true;
      try {
        const action = isEdit.value ? "update" : "insert";
        const res = await qmsxDmlApi({ request_id: "evisor-" + Date.now(), owner: owner.value, dml_action: action, form: { ...form.value } });
        if (res.status === "success") { ElMessage.success(res.message); dialogVisible.value = false; fetchAll(); }
        else ElMessage.error(res.message);
      } catch (e) { ElMessage.error(e.message); }
      finally { formLoading.value = false; }
    };

    const handleDelete = async (row) => {
      try {
        await ElMessageBox.confirm(`Xóa phiếu kiểm tra "${row.inspection_code}"?`, "Xác nhận", { type: "warning" });
        const res = await qmsxDmlApi({ request_id: "evisor-" + Date.now(), owner: owner.value, dml_action: "delete", form: { id: row.id } });
        if (res.status === "success") { ElMessage.success(res.message); fetchAll(); }
        else ElMessage.error(res.message);
      } catch (e) { if (e !== "cancel") ElMessage.error(e.message); }
    };

    const formatDate = (iso) => iso ? iso.replace("T", " ").substring(0, 16) : "—";
    const resultLabel = (r) => RESULT_LABEL[r] || "—";
    const resultType = (r) => RESULT_TYPE[r] || "";

    onMounted(fetchAll);

    return { Plus, Refresh, loading, formLoading, inspections, filterResult, dialogVisible, isEdit, form, stats, fetchAll, fetchInspections, openCreate, openEdit, handleSubmit, handleDelete, formatDate, resultLabel, resultType };
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
.kpi-red .kpi-value { color: #f56c6c; }
</style>
