<template>
  <el-dialog :model-value="visible" title="Thông tin chi tiết nhân sự" width="500px" @close="$emit('close')">
    <template v-if="user">
      <div style="display: flex; align-items: center; margin-bottom: 20px">
        <el-avatar
          :size="64"
          :src="user.avatar || 'https://cube.elemecdn.com/3/7c/3ea6beec64369c2642b92c6726f1epng.png'"
        />
        <div style="margin-left: 16px">
          <div style="font-size: 18px; font-weight: bold">{{ user.full_name }}</div>
          <div style="color: #888; font-size: 13px">@{{ user.username }}</div>
          <el-tag :type="activeTagType" size="small" style="margin-top: 4px">
            {{ user.is_active ? "Đang hoạt động" : "Đã vô hiệu hóa" }}
          </el-tag>
        </div>
      </div>

      <el-descriptions :column="1" border size="small">
        <el-descriptions-item label="Mã nhân viên">{{ user.user_id }}</el-descriptions-item>
        <el-descriptions-item label="Email">{{ user.email || "—" }}</el-descriptions-item>
        <el-descriptions-item label="Điện thoại">{{ user.phone_number || "—" }}</el-descriptions-item>
        <el-descriptions-item label="Số máy nhánh">{{ user.ext || "—" }}</el-descriptions-item>
        <el-descriptions-item label="Vai trò">
          <el-tag :type="roleTagType">{{ roleName }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="Phòng ban">{{ deptName }}</el-descriptions-item>
        <el-descriptions-item label="Ngày tạo">{{ formatDate(user.created_at) }}</el-descriptions-item>
        <el-descriptions-item label="Lần đăng nhập cuối">{{ formatDate(user.last_active_at) }}</el-descriptions-item>
      </el-descriptions>
    </template>

    <template #footer>
      <el-button @click="$emit('close')">Đóng</el-button>
      <el-button type="primary" @click="$emit('edit', user)">Chỉnh sửa</el-button>
    </template>
  </el-dialog>
</template>

<script>
import { computed } from "vue";

const ALL_DEPTS = {
  1000: "Ban Giám Đốc HCM", 1001: "Hội đồng Tư vấn Công nghệ HCM", 1002: "Nhân sự & Hành chính HCM",
  1003: "Mua hàng & Logistics HCM", 1004: "Tài chính & Kế toán HCM", 1005: "Tổ chức & Kế hoạch HCM",
  1006: "Marketing HCM", 1007: "Kinh doanh Điện-Tự động HCM", 1008: "Account Manager HCM",
  1009: "Kinh doanh Hệ thống Điện HCM", 1010: "Kinh doanh Tự động hóa HCM", 1011: "Kinh doanh PIS HCM",
  1012: "Kinh doanh Sản phẩm Công nghiệp HCM", 1013: "ESTEC Digital HCM", 1014: "Kinh doanh Digital HCM",
  1015: "Thực hiện Dự án HCM", 1016: "RnD HCM", 1017: "Thiết kế & Thi công HCM",
  1018: "Tự động hóa HCM", 1019: "Điện HCM", 1020: "Đo lường HCM",
  1021: "Dịch vụ Công nghiệp HCM", 1022: "Kỹ thuật viên M&E HCM", 1023: "Back Office HCM",
  2000: "Ban Giám Đốc ĐN", 2001: "Nhân sự & Hành chính ĐN", 2002: "Tài chính & Kế toán ĐN",
  2003: "Kinh doanh ĐN", 2004: "Tự động hóa ĐN", 2005: "Điện ĐN", 2006: "Đo lường ĐN",
  2007: "Sửa chữa Hộp số M&E Flender ĐN", 2008: "ESTEC Digital ĐN",
  2009: "Trung tâm Đào tạo ĐN", 2010: "Back Office ĐN",
};

export default {
  name: "UserDetailDialog",
  props: {
    visible: { type: Boolean, default: false },
    user: { type: Object, default: null },
  },
  emits: ["close", "edit"],
  setup(props) {
    const roleName = computed(() => {
      const map = { 255: "Admin", 127: "Manager", 63: "Warehouse Manager", 1: "Employee" };
      return map[props.user?.role_id] || "—";
    });
    const roleTagType = computed(() => {
      const map = { 255: "danger", 127: "warning", 63: "info", 1: "" };
      return map[props.user?.role_id] || "";
    });
    const deptName = computed(() => ALL_DEPTS[props.user?.department_id] || props.user?.department_id || "—");
    const activeTagType = computed(() => (props.user?.is_active ? "success" : "danger"));
    const formatDate = (val) => {
      if (!val) return "—";
      return new Date(val).toLocaleString("vi-VN");
    };
    return { roleName, roleTagType, deptName, activeTagType, formatDate };
  },
};
</script>
