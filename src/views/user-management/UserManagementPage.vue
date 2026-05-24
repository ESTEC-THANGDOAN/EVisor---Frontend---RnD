<template>
  <div class="user-management-screen" style="padding: 20px">
    <h2>Quản lý Nhân sự</h2>

    <div style="margin-bottom: 20px">
      <el-button type="primary" :icon="Plus" @click="handleCreateUser">Thêm nhân sự mới</el-button>
      <el-button :icon="Refresh" @click="fetchUsers">Làm mới dữ liệu</el-button>
    </div>

    <el-tabs v-model="activeTab" type="border-card" lazy>
      <el-tab-pane label="Tất cả nhân sự" name="all">
        <UserListTable
          :users="allUsers"
          :loading="isLoading"
          @view="handleViewUser"
          @edit="handleEditUser"
          @delete="handleDeleteUser"
          @toggle-status="handleToggleStatus"
        />
      </el-tab-pane>
      <el-tab-pane label="Hồ Chí Minh" name="hcm">
        <UserListTable
          :users="hcmUsers"
          :loading="isLoading"
          @view="handleViewUser"
          @edit="handleEditUser"
          @delete="handleDeleteUser"
          @toggle-status="handleToggleStatus"
        />
      </el-tab-pane>
      <el-tab-pane label="Đà Nẵng" name="dn">
        <UserListTable
          :users="dnUsers"
          :loading="isLoading"
          @view="handleViewUser"
          @edit="handleEditUser"
          @delete="handleDeleteUser"
          @toggle-status="handleToggleStatus"
        />
      </el-tab-pane>
    </el-tabs>

    <UserDetailDialog
      :visible="detailDialogVisible"
      :user="selectedUser"
      @close="detailDialogVisible = false"
      @edit="handleEditUser"
    />

    <UserFormDialog
      :visible="formDialogVisible"
      :mode="formMode"
      :user-data="selectedUser"
      :loading="formLoading"
      @close="formDialogVisible = false"
      @submit="handleFormSubmit"
    />
  </div>
</template>

<script>
import { Plus, Refresh } from "@element-plus/icons-vue";
import { ElMessage, ElMessageBox } from "element-plus";
import { computed, onMounted, ref } from "vue";
import UserListTable from "../../components/table/user_management/UserListTable.vue";
import { useAuthStore } from "../../stores/auth";
import {
  dmlUserManagementApi,
  resetPasswordUserManagementApi,
  toggleActiveUserManagementApi,
  viewUserManagementApi,
} from "../../services/auth.service";
import UserDetailDialog from "./UserDetailDialog.vue";
import UserFormDialog from "./UserFormDialog.vue";

export default {
  name: "UserManagementPage",
  components: { UserListTable, UserDetailDialog, UserFormDialog, Plus, Refresh },
  setup() {
    const authStore = useAuthStore();
    const owner = computed(() => authStore.user?.id);

    const isLoading = ref(false);
    const allUsers = ref([]);
    const activeTab = ref("all");

    const hcmUsers = computed(() => allUsers.value.filter((u) => u.department_id >= 1000 && u.department_id <= 1023));
    const dnUsers = computed(() => allUsers.value.filter((u) => u.department_id >= 2000 && u.department_id <= 2010));

    // Detail dialog
    const detailDialogVisible = ref(false);
    const selectedUser = ref(null);

    // Form dialog (create / edit)
    const formDialogVisible = ref(false);
    const formMode = ref("create");
    const formLoading = ref(false);

    const fetchUsers = async () => {
      isLoading.value = true;
      try {
        const payload = {
          request_id: "evisor-" + Date.now(),
          owner: owner.value,
        };
        const res = await viewUserManagementApi(payload);
        if (res.status === "success") {
          allUsers.value = res.data;
        } else {
          ElMessage.error(res.message || "Không thể tải danh sách nhân sự.");
        }
      } catch (e) {
        ElMessage.error("Lỗi kết nối: " + e.message);
      } finally {
        isLoading.value = false;
      }
    };

    const handleCreateUser = () => {
      selectedUser.value = null;
      formMode.value = "create";
      formDialogVisible.value = true;
    };

    const handleViewUser = (user) => {
      selectedUser.value = user;
      detailDialogVisible.value = true;
    };

    const handleEditUser = (user) => {
      detailDialogVisible.value = false;
      selectedUser.value = { ...user };
      formMode.value = "edit";
      formDialogVisible.value = true;
    };

    const handleDeleteUser = async (user) => {
      try {
        const payload = {
          request_id: "evisor-" + Date.now(),
          owner: owner.value,
          dml_action: "delete",
          form: { user_id: user.user_id },
        };
        const res = await dmlUserManagementApi(payload);
        if (res.status === "success") {
          ElMessage.success(res.message || "Xóa nhân sự thành công!");
          await fetchUsers();
        } else {
          ElMessage.error(res.message || "Xóa thất bại.");
        }
      } catch (e) {
        ElMessage.error("Lỗi kết nối: " + e.message);
      }
    };

    const handleToggleStatus = async (user) => {
      try {
        const payload = {
          request_id: "evisor-" + Date.now(),
          owner: owner.value,
          user_id: user.user_id,
        };
        const res = await toggleActiveUserManagementApi(payload);
        if (res.status === "success") {
          ElMessage.success(res.message || "Cập nhật trạng thái thành công!");
          await fetchUsers();
        } else {
          ElMessage.error(res.message || "Cập nhật thất bại.");
          // Revert switch state by refreshing data
          await fetchUsers();
        }
      } catch (e) {
        ElMessage.error("Lỗi kết nối: " + e.message);
        await fetchUsers();
      }
    };

    const handleFormSubmit = async (formData) => {
      formLoading.value = true;
      try {
        if (formData.mode === "create") {
          const payload = {
            request_id: "evisor-" + Date.now(),
            owner: owner.value,
            dml_action: "insert",
            form: {
              user_id: formData.user_id,
              username: formData.username,
              password: formData.password,
              full_name: formData.full_name,
              email: formData.email || null,
              phone_number: formData.phone_number || null,
              ext: formData.ext || null,
              role_id: formData.role_id,
              department_id: formData.department_id,
            },
          };
          const res = await dmlUserManagementApi(payload);
          if (res.status === "success") {
            ElMessage.success(res.message || "Thêm nhân sự thành công!");
            formDialogVisible.value = false;
            await fetchUsers();
          } else {
            ElMessage.error(res.message || "Thêm thất bại.");
          }
        } else {
          // Update profile
          const updatePayload = {
            request_id: "evisor-" + Date.now(),
            owner: owner.value,
            dml_action: "update",
            form: {
              user_id: formData.user_id,
              full_name: formData.full_name,
              email: formData.email || null,
              phone_number: formData.phone_number || null,
              ext: formData.ext || null,
              role_id: formData.role_id,
              department_id: formData.department_id,
            },
          };
          const res = await dmlUserManagementApi(updatePayload);
          if (res.status !== "success") {
            ElMessage.error(res.message || "Cập nhật thất bại.");
            return;
          }

          // Reset password if new_password is provided
          if (formData.new_password) {
            const resetPayload = {
              request_id: "evisor-" + Date.now(),
              owner: owner.value,
              user_id: formData.user_id,
              new_password: formData.new_password,
            };
            const resetRes = await resetPasswordUserManagementApi(resetPayload);
            if (resetRes.status !== "success") {
              ElMessage.warning(
                "Cập nhật thông tin thành công nhưng đặt lại mật khẩu thất bại: " + (resetRes.message || "")
              );
              formDialogVisible.value = false;
              await fetchUsers();
              return;
            }
          }

          ElMessage.success("Cập nhật nhân sự thành công!");
          formDialogVisible.value = false;
          await fetchUsers();
        }
      } catch (e) {
        ElMessage.error("Lỗi kết nối: " + e.message);
      } finally {
        formLoading.value = false;
      }
    };

    onMounted(() => {
      fetchUsers();
    });

    return {
      Plus,
      Refresh,
      activeTab,
      isLoading,
      allUsers,
      hcmUsers,
      dnUsers,
      detailDialogVisible,
      formDialogVisible,
      formMode,
      formLoading,
      selectedUser,
      fetchUsers,
      handleCreateUser,
      handleViewUser,
      handleEditUser,
      handleDeleteUser,
      handleToggleStatus,
      handleFormSubmit,
    };
  },
};
</script>

<style scoped>
.user-management-screen {
  background-color: white;
}
</style>
