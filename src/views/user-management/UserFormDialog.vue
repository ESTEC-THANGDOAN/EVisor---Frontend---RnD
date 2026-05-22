<template>
  <el-dialog
    :model-value="visible"
    :title="isEdit ? 'Chỉnh sửa thông tin nhân sự' : 'Thêm nhân sự mới'"
    width="620px"
    @close="handleClose"
  >
    <el-form ref="formRef" :model="form" :rules="rules" label-width="140px" style="padding-right: 20px">
      <template v-if="!isEdit">
        <el-form-item label="Mã nhân viên" prop="user_id">
          <el-input v-model="form.user_id" placeholder="VD: thangddn" />
        </el-form-item>
        <el-form-item label="Tên đăng nhập" prop="username">
          <el-input v-model="form.username" placeholder="VD: thangddn" />
        </el-form-item>
        <el-form-item label="Mật khẩu" prop="password">
          <el-input v-model="form.password" type="password" show-password placeholder="Ít nhất 6 ký tự" />
        </el-form-item>
      </template>
      <template v-else>
        <el-form-item label="Mã nhân viên">
          <el-input :value="form.user_id" disabled />
        </el-form-item>
        <el-form-item label="Tên đăng nhập">
          <el-input :value="form.username" disabled />
        </el-form-item>
      </template>

      <el-form-item label="Họ và tên" prop="full_name">
        <el-input v-model="form.full_name" placeholder="Nguyễn Văn A" />
      </el-form-item>
      <el-form-item label="Email" prop="email">
        <el-input v-model="form.email" placeholder="ten.ho@estec.vn" />
      </el-form-item>
      <el-form-item label="Số điện thoại">
        <el-input v-model="form.phone_number" placeholder="0901234567" />
      </el-form-item>
      <el-form-item label="Số máy nhánh">
        <el-input v-model="form.ext" placeholder="101" style="width: 120px" />
      </el-form-item>
      <el-form-item label="Vai trò" prop="role_id">
        <el-select v-model="form.role_id" placeholder="Chọn vai trò" style="width: 100%">
          <el-option label="Admin" :value="255" />
          <el-option label="Manager" :value="127" />
          <el-option label="Warehouse Manager" :value="63" />
          <el-option label="Employee" :value="1" />
        </el-select>
      </el-form-item>
      <el-form-item label="Phòng ban" prop="department_id">
        <el-select v-model="form.department_id" placeholder="Chọn phòng ban" filterable style="width: 100%">
          <el-option-group label="Hồ Chí Minh">
            <el-option v-for="dept in hcmDepts" :key="dept.id" :label="dept.label" :value="dept.id" />
          </el-option-group>
          <el-option-group label="Đà Nẵng">
            <el-option v-for="dept in dnDepts" :key="dept.id" :label="dept.label" :value="dept.id" />
          </el-option-group>
        </el-select>
      </el-form-item>

      <!-- Reset password section (edit mode only) -->
      <template v-if="isEdit">
        <el-divider content-position="left">Đặt lại mật khẩu</el-divider>
        <el-form-item label="Mật khẩu mới" prop="new_password">
          <el-input
            v-model="form.new_password"
            type="password"
            show-password
            placeholder="Để trống nếu không đổi"
          />
        </el-form-item>
      </template>
    </el-form>

    <template #footer>
      <el-button @click="handleClose">Hủy</el-button>
      <el-button type="primary" :loading="loading" @click="handleSubmit">
        {{ isEdit ? 'Lưu thay đổi' : 'Thêm mới' }}
      </el-button>
    </template>
  </el-dialog>
</template>

<script>
import { ElMessage } from "element-plus";
import { computed, reactive, ref, watch } from "vue";

const HCM_DEPTS = [
  { id: 1000, label: "Ban Giám Đốc HCM" },
  { id: 1001, label: "Hội đồng Tư vấn Công nghệ HCM" },
  { id: 1002, label: "Nhân sự & Hành chính HCM" },
  { id: 1003, label: "Mua hàng & Logistics HCM" },
  { id: 1004, label: "Tài chính & Kế toán HCM" },
  { id: 1005, label: "Tổ chức & Kế hoạch HCM" },
  { id: 1006, label: "Marketing HCM" },
  { id: 1007, label: "Kinh doanh Điện-Tự động HCM" },
  { id: 1008, label: "Account Manager HCM" },
  { id: 1009, label: "Kinh doanh Hệ thống Điện HCM" },
  { id: 1010, label: "Kinh doanh Tự động hóa HCM" },
  { id: 1011, label: "Kinh doanh PIS HCM" },
  { id: 1012, label: "Kinh doanh Sản phẩm Công nghiệp HCM" },
  { id: 1013, label: "ESTEC Digital HCM" },
  { id: 1014, label: "Kinh doanh Digital HCM" },
  { id: 1015, label: "Thực hiện Dự án HCM" },
  { id: 1016, label: "RnD HCM" },
  { id: 1017, label: "Thiết kế & Thi công HCM" },
  { id: 1018, label: "Tự động hóa HCM" },
  { id: 1019, label: "Điện HCM" },
  { id: 1020, label: "Đo lường HCM" },
  { id: 1021, label: "Dịch vụ Công nghiệp HCM" },
  { id: 1022, label: "Kỹ thuật viên M&E HCM" },
  { id: 1023, label: "Back Office HCM" },
];

const DN_DEPTS = [
  { id: 2000, label: "Ban Giám Đốc ĐN" },
  { id: 2001, label: "Nhân sự & Hành chính ĐN" },
  { id: 2002, label: "Tài chính & Kế toán ĐN" },
  { id: 2003, label: "Kinh doanh ĐN" },
  { id: 2004, label: "Tự động hóa ĐN" },
  { id: 2005, label: "Điện ĐN" },
  { id: 2006, label: "Đo lường ĐN" },
  { id: 2007, label: "Sửa chữa Hộp số M&E Flender ĐN" },
  { id: 2008, label: "ESTEC Digital ĐN" },
  { id: 2009, label: "Trung tâm Đào tạo ĐN" },
  { id: 2010, label: "Back Office ĐN" },
];

export default {
  name: "UserFormDialog",
  props: {
    visible: { type: Boolean, default: false },
    mode: { type: String, default: "create" },
    userData: { type: Object, default: null },
    loading: { type: Boolean, default: false },
  },
  emits: ["close", "submit"],
  setup(props, { emit }) {
    const formRef = ref(null);
    const isEdit = computed(() => props.mode === "edit");

    const emptyForm = () => ({
      user_id: "",
      username: "",
      password: "",
      full_name: "",
      email: "",
      phone_number: "",
      ext: "",
      role_id: 1,
      department_id: null,
      new_password: "",
    });

    const form = reactive(emptyForm());

    watch(
      () => props.userData,
      (val) => {
        if (val) {
          form.user_id = val.user_id || "";
          form.username = val.username || "";
          form.full_name = val.full_name || "";
          form.email = val.email || "";
          form.phone_number = val.phone_number || "";
          form.ext = val.ext || "";
          form.role_id = val.role_id ?? 1;
          form.department_id = val.department_id ?? null;
          form.password = "";
          form.new_password = "";
        } else {
          Object.assign(form, emptyForm());
        }
      },
      { immediate: true }
    );

    const rules = computed(() => {
      const base = {
        full_name: [{ required: true, message: "Vui lòng nhập họ tên", trigger: "blur" }],
        role_id: [{ required: true, message: "Vui lòng chọn vai trò", trigger: "change" }],
        department_id: [{ required: true, message: "Vui lòng chọn phòng ban", trigger: "change" }],
        email: [{ type: "email", message: "Email không hợp lệ", trigger: "blur" }],
        new_password: [
          {
            validator: (rule, val, cb) => {
              if (val && val.length < 6) cb(new Error("Mật khẩu phải có ít nhất 6 ký tự"));
              else cb();
            },
            trigger: "blur",
          },
        ],
      };
      if (!isEdit.value) {
        base.user_id = [{ required: true, message: "Vui lòng nhập mã nhân viên", trigger: "blur" }];
        base.username = [{ required: true, message: "Vui lòng nhập tên đăng nhập", trigger: "blur" }];
        base.password = [
          { required: true, message: "Vui lòng nhập mật khẩu", trigger: "blur" },
          { min: 6, message: "Mật khẩu phải có ít nhất 6 ký tự", trigger: "blur" },
        ];
      }
      return base;
    });

    const handleSubmit = () => {
      formRef.value.validate((valid) => {
        if (!valid) return;
        emit("submit", { ...form, mode: props.mode });
      });
    };

    const handleClose = () => {
      formRef.value?.resetFields();
      emit("close");
    };

    return {
      formRef,
      isEdit,
      form,
      rules,
      hcmDepts: HCM_DEPTS,
      dnDepts: DN_DEPTS,
      handleSubmit,
      handleClose,
    };
  },
};
</script>
