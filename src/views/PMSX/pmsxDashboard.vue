<template>
  <div class="module-dashboard" style="padding: 20px">
    <h2>Quản lý Dự án (PMSX)</h2>

    <!-- KPI Cards -->
    <el-row :gutter="16" style="margin-bottom: 20px">
      <el-col :span="6">
        <el-card class="kpi-card">
          <div class="kpi-value">{{ stats.total }}</div>
          <div class="kpi-label">Tổng dự án</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="kpi-card kpi-blue">
          <div class="kpi-value">{{ stats.active }}</div>
          <div class="kpi-label">Đang thực hiện</div>
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
          <div class="kpi-value">{{ stats.avg_progress }}%</div>
          <div class="kpi-label">Tiến độ trung bình</div>
        </el-card>
      </el-col>
    </el-row>

    <!-- Toolbar -->
    <div style="margin-bottom: 16px; display: flex; gap: 8px; align-items: center">
      <el-button type="primary" :icon="Plus" @click="openCreate">Thêm dự án</el-button>
      <el-button :icon="Refresh" @click="fetchAll">Làm mới</el-button>
      <el-select v-model="filterStatus" placeholder="Lọc trạng thái" clearable style="width: 180px" @change="fetchProjects">
        <el-option label="Lên kế hoạch" :value="0" />
        <el-option label="Đang thực hiện" :value="1" />
        <el-option label="Hoàn thành" :value="2" />
        <el-option label="Tạm dừng" :value="3" />
        <el-option label="Đã hủy" :value="4" />
      </el-select>
    </div>

    <!-- Table -->
    <el-table :data="projects" v-loading="loading" border stripe style="width: 100%">
      <el-table-column prop="project_code" label="Mã dự án" width="130" />
      <el-table-column prop="project_name" label="Tên dự án" min-width="200" />
      <el-table-column prop="client" label="Khách hàng" width="150" />
      <el-table-column prop="manager" label="Quản lý" width="130" />
      <el-table-column label="Tiến độ" width="120" align="center">
        <template #default="{ row }">
          <el-progress :percentage="row.progress || 0" :stroke-width="8" />
        </template>
      </el-table-column>
      <el-table-column label="Hạn hoàn thành" width="140">
        <template #default="{ row }">{{ formatDate(row.end_date) }}</template>
      </el-table-column>
      <el-table-column label="Trạng thái" width="140" align="center">
        <template #default="{ row }">
          <el-tag :type="projectStatusType(row.status)">{{ projectStatusLabel(row.status) }}</el-tag>
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
    <el-dialog v-model="dialogVisible" :title="isEdit ? 'Cập nhật dự án' : 'Thêm dự án'" width="620px" destroy-on-close>
      <el-form :model="form" label-width="150px" label-position="left">
        <el-form-item label="Mã dự án" v-if="!isEdit">
          <el-input v-model="form.project_code" placeholder="PRJ-2025-001" />
        </el-form-item>
        <el-form-item label="Tên dự án">
          <el-input v-model="form.project_name" />
        </el-form-item>
        <el-form-item label="Khách hàng">
          <el-input v-model="form.client" />
        </el-form-item>
        <el-form-item label="Người quản lý">
          <el-input v-model="form.manager" />
        </el-form-item>
        <el-form-item label="Ngày bắt đầu">
          <el-date-picker v-model="form.start_date" type="datetime" placeholder="Chọn ngày" style="width: 100%" />
        </el-form-item>
        <el-form-item label="Ngày kết thúc">
          <el-date-picker v-model="form.end_date" type="datetime" placeholder="Chọn ngày" style="width: 100%" />
        </el-form-item>
        <el-form-item label="Ngân sách (VNĐ)">
          <el-input-number v-model="form.budget" :min="0" :step="1000000" style="width: 100%" />
        </el-form-item>
        <el-form-item label="Trạng thái">
          <el-select v-model="form.status" style="width: 100%">
            <el-option label="Lên kế hoạch" :value="0" />
            <el-option label="Đang thực hiện" :value="1" />
            <el-option label="Hoàn thành" :value="2" />
            <el-option label="Tạm dừng" :value="3" />
            <el-option label="Đã hủy" :value="4" />
          </el-select>
        </el-form-item>
        <el-form-item label="Tiến độ (%)">
          <el-slider v-model="form.progress" :min="0" :max="100" show-input />
        </el-form-item>
        <el-form-item label="Ghi chú">
          <el-input v-model="form.notes" type="textarea" :rows="2" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">Hủy</el-button>
        <el-button type="primary" :loading="formLoading" @click="handleSubmit">
          {{ isEdit ? 'Cập nhật' : 'Tạo dự án' }}
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
import { pmsxViewApi, pmsxDashboardApi, pmsxDmlApi } from "../../services/auth.service";

const STATUS_LABEL = ["Lên kế hoạch", "Đang thực hiện", "Hoàn thành", "Tạm dừng", "Đã hủy"];
const STATUS_TYPE = ["info", "warning", "success", "", "danger"];

export default {
  name: "PMSXDashboard",
  setup() {
    const authStore = useAuthStore();
    const owner = computed(() => authStore.user?.id || "");

    const loading = ref(false);
    const formLoading = ref(false);
    const projects = ref([]);
    const filterStatus = ref(null);
    const dialogVisible = ref(false);
    const isEdit = ref(false);
    const stats = ref({ total: 0, planning: 0, active: 0, completed: 0, on_hold: 0, cancelled: 0, avg_progress: 0 });

    const emptyForm = () => ({ project_code: "", project_name: "", client: "", manager: "", start_date: null, end_date: null, budget: 0, status: 0, progress: 0, notes: "", id: null });
    const form = ref(emptyForm());

    const fetchProjects = async () => {
      loading.value = true;
      try {
        const payload = { request_id: "evisor-" + Date.now(), owner: owner.value };
        if (filterStatus.value !== null) payload.filter_status = filterStatus.value;
        const res = await pmsxViewApi(payload);
        if (res.status === "success") projects.value = res.data;
        else ElMessage.error(res.message || "Không thể tải dữ liệu.");
      } catch (e) { ElMessage.error(e.message); }
      finally { loading.value = false; }
    };

    const fetchStats = async () => {
      try {
        const res = await pmsxDashboardApi({ request_id: "evisor-" + Date.now(), owner: owner.value });
        if (res.status === "success") stats.value = res.data;
      } catch { /* silent */ }
    };

    const fetchAll = () => { fetchProjects(); fetchStats(); };

    const openCreate = () => { form.value = emptyForm(); isEdit.value = false; dialogVisible.value = true; };
    const openEdit = (row) => { form.value = { ...row }; isEdit.value = true; dialogVisible.value = true; };

    const handleSubmit = async () => {
      if (!form.value.project_name) return ElMessage.warning("Vui lòng nhập tên dự án.");
      formLoading.value = true;
      try {
        const action = isEdit.value ? "update" : "insert";
        const res = await pmsxDmlApi({ request_id: "evisor-" + Date.now(), owner: owner.value, dml_action: action, form: { ...form.value } });
        if (res.status === "success") { ElMessage.success(res.message); dialogVisible.value = false; fetchAll(); }
        else ElMessage.error(res.message);
      } catch (e) { ElMessage.error(e.message); }
      finally { formLoading.value = false; }
    };

    const handleDelete = async (row) => {
      try {
        await ElMessageBox.confirm(`Xóa dự án "${row.project_name}"?`, "Xác nhận", { type: "warning" });
        const res = await pmsxDmlApi({ request_id: "evisor-" + Date.now(), owner: owner.value, dml_action: "delete", form: { id: row.id } });
        if (res.status === "success") { ElMessage.success(res.message); fetchAll(); }
        else ElMessage.error(res.message);
      } catch (e) { if (e !== "cancel") ElMessage.error(e.message); }
    };

    const formatDate = (iso) => iso ? iso.replace("T", " ").substring(0, 10) : "—";
    const projectStatusLabel = (s) => STATUS_LABEL[s] || "—";
    const projectStatusType = (s) => STATUS_TYPE[s] || "";

    onMounted(fetchAll);

    return { Plus, Refresh, loading, formLoading, projects, filterStatus, dialogVisible, isEdit, form, stats, fetchAll, fetchProjects, openCreate, openEdit, handleSubmit, handleDelete, formatDate, projectStatusLabel, projectStatusType };
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
