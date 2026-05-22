<template>
  <div class="user-list-table">
    <div style="margin-bottom: 15px; display: flex; justify-content: flex-end">
      <el-input
        v-model="searchQuery"
        placeholder="Tìm kiếm tên, email..."
        style="width: 300px"
        clearable
        :prefix-icon="Search"
      />
    </div>

    <el-table :data="filteredData" border stripe style="width: 100%" v-loading="loading">
      <el-table-column prop="user_id" label="Mã NV" width="100" sortable />
      <el-table-column label="Họ tên" min-width="180">
        <template #default="{ row }">
          <div style="display: flex; align-items: center">
            <el-avatar
              :size="30"
              :src="
                row.avatar ||
                'https://cube.elemecdn.com/3/7c/3ea6beec64369c2642b92c6726f1epng.png'
              "
            />
            <div style="margin-left: 10px; display: flex; flex-direction: column">
              <span style="font-weight: bold">{{ row.full_name }}</span>
              <span style="font-size: 12px; color: #888">@{{ row.username }}</span>
            </div>
          </div>
        </template>
      </el-table-column>

      <el-table-column label="Liên hệ" min-width="200">
        <template #default="{ row }">
          <div>
            <el-icon><Message /></el-icon> {{ row.email }}
          </div>
          <div>
            <el-icon><Phone /></el-icon> {{ row.phone_number }}
          </div>
        </template>
      </el-table-column>

      <el-table-column label="Phòng ban" min-width="180">
        <template #default="{ row }">
          <span>{{ getDeptName(row.department_id) }}</span>
        </template>
      </el-table-column>

      <el-table-column prop="role_id" label="Vai trò" width="150">
        <template #default="{ row }">
          <el-tag :type="getRoleTagType(row.role_id)">
            {{ getRoleName(row.role_id) }}
          </el-tag>
        </template>
      </el-table-column>

      <el-table-column label="Trạng thái" width="150" align="center">
        <template #default="{ row }">
          <el-switch
            v-model="row.is_active"
            active-text="Active"
            inactive-text="Deactive"
            inline-prompt
            style="--el-switch-on-color: #13ce66; --el-switch-off-color: #ff4949"
            @change="handleStatusChange(row)"
          />
        </template>
      </el-table-column>

      <el-table-column label="Hành động" width="180" align="center" fixed="right">
        <template #default="{ row }">
          <el-tooltip content="Xem chi tiết" placement="top">
            <el-button
              type="info"
              :icon="View"
              circle
              size="small"
              @click="$emit('view', row)"
            />
          </el-tooltip>

          <el-tooltip content="Chỉnh sửa" placement="top">
            <el-button
              type="primary"
              :icon="Edit"
              circle
              size="small"
              @click="$emit('edit', row)"
            />
          </el-tooltip>

          <el-tooltip content="Xóa nhân sự" placement="top">
            <el-button
              type="danger"
              :icon="Delete"
              circle
              size="small"
              @click="confirmDelete(row)"
            />
          </el-tooltip>
        </template>
      </el-table-column>
    </el-table>
    <div style="margin-top: 20px; display: flex; justify-content: flex-end">
      <el-pagination
        background
        layout="prev, pager, next"
        :total="filteredData.length"
        :page-size="10"
      />
    </div>
  </div>
</template>

<script>
import { Delete, Edit, Message, Phone, Search, View } from "@element-plus/icons-vue";
import { ElMessage, ElMessageBox } from "element-plus";
import { computed, ref } from "vue";

export default {
  name: "UserListTable",
  components: {
    Search,
    Message,
    Phone,
    View,
    Edit,
    Delete,
  },
  props: {
    users: {
      type: Array,
      required: true,
      default: () => [],
    },
    loading: {
      type: Boolean,
      default: false,
    },
  },
  emits: ["view", "edit", "delete", "toggle-status"],
  setup(props, { emit }) {
    const searchQuery = ref("");
    // Filter search local
    const filteredData = computed(() => {
      if (!searchQuery.value) return props.users;
      const lowerQuery = searchQuery.value.toLowerCase();
      return props.users.filter(
        (user) =>
          user.full_name.toLowerCase().includes(lowerQuery) ||
          user.email.toLowerCase().includes(lowerQuery)
      );
    });

    const ROLE_NAMES = { 255: "Admin", 127: "Manager", 63: "Warehouse Manager", 1: "Employee" };
    const ROLE_TAG_TYPES = { 255: "danger", 127: "warning", 63: "info", 1: "" };
    const DEPT_NAMES = {
      1000: "Ban GĐ HCM", 1001: "TV Công nghệ HCM", 1002: "NS&HC HCM", 1003: "Mua hàng HCM",
      1004: "Tài chính HCM", 1005: "TC&KH HCM", 1006: "Marketing HCM", 1007: "KD Điện-TĐ HCM",
      1008: "Account Mgr HCM", 1009: "KD Hệ thống Điện HCM", 1010: "KD Tự động HCM",
      1011: "KD PIS HCM", 1012: "KD SP Công nghiệp HCM", 1013: "ESTEC Digital HCM",
      1014: "KD Digital HCM", 1015: "Thực hiện DA HCM", 1016: "RnD HCM",
      1017: "Thiết kế & Thi công HCM", 1018: "Tự động hóa HCM", 1019: "Điện HCM",
      1020: "Đo lường HCM", 1021: "Dịch vụ CN HCM", 1022: "KTV M&E HCM", 1023: "Back Office HCM",
      2000: "Ban GĐ ĐN", 2001: "NS&HC ĐN", 2002: "Tài chính ĐN", 2003: "Kinh doanh ĐN",
      2004: "Tự động hóa ĐN", 2005: "Điện ĐN", 2006: "Đo lường ĐN",
      2007: "Sửa chữa M&E ĐN", 2008: "ESTEC Digital ĐN", 2009: "TTĐT ĐN", 2010: "Back Office ĐN",
    };
    const getRoleName = (roleId) => ROLE_NAMES[roleId] || "Unknown";
    const getRoleTagType = (roleId) => ROLE_TAG_TYPES[roleId] || "";
    const getDeptName = (deptId) => DEPT_NAMES[deptId] || (deptId ? String(deptId) : "—");

    const handleStatusChange = (row) => {
      emit("toggle-status", row);
      ElMessage.success(`Đã cập nhập trạng thái của ${row.full_name}`);
    };

    const confirmDelete = (row) => {
      ElMessageBox.confirm(
        `Bạn có chắc chắc muốn xóa nhân sự ${row.full_name} ?`,
        "Cảnh báo",
        {
          confirmButtonText: "Xóa",
          cancelButtonText: "Hủy",
          type: "warning",
        }
      )
        .then(() => {
          emit("delete", row);
        })
        .catch(() => {});
    };

    return {
      Search,
      Message,
      Phone,
      View,
      Edit,
      Delete,
      searchQuery,
      filteredData,
      getRoleName,
      getRoleTagType,
      getDeptName,
      handleStatusChange,
      confirmDelete,
    };
  },
};
</script>

<style scoped></style>
