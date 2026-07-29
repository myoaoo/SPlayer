<!-- 登录 - 手机号 -->
<template>
  <div class="login-phone">
    <n-form
      ref="phoneFormRef"
      :model="phoneFormData"
      :rules="phoneFormRules"
      :show-label="false"
      class="phone-form"
    >
      <n-form-item path="phone">
        <n-input v-model:value="phoneFormData.phone" placeholder="请输入手机号">
          <template #prefix>
            <n-icon>
              <SvgIcon icon="phone" />
            </n-icon>
          </template>
        </n-input>
      </n-form-item>
      <n-form-item path="captcha">
        <div class="captcha-input-wrapper">
          <n-input
            v-model:value="phoneFormData.captcha"
            style="width: 100%"
            placeholder="请输入短信验证码"
          >
            <template #prefix>
              <n-icon>
                <SvgIcon icon="password" />
              </n-icon>
            </template>
          </n-input>
          <n-button
            :disabled="captchaDisabled"
            type="primary"
            style="margin-left: 12px"
            @click="getCaptcha(phoneFormData.phone)"
          >
            {{ captchaText }}
          </n-button>
        </div>
      </n-form-item>
      <n-form-item>
        <n-button style="width: 100%" type="primary" @click="phoneLogin"> 登录 </n-button>
      </n-form-item>
    </n-form>
  </div>
</template>

<script setup>
import { sentCaptcha, verifyCaptcha, toLogin } from "@/api/login";
import { formRules } from "@/utils/formRules";

const emit = defineEmits(["setLoginData"]);
const { numberRule, mobileRule } = formRules();

// 手机号数据
const phoneFormRef = ref(null);
const phoneFormData = ref({
  phone: null,
  captcha: "",
});
const phoneFormRules = {
  phone: mobileRule,
  captcha: [
    {
      required: true,
      message: "请输入验证码",
      trigger: "blur",
    },
    {
      pattern: /^\d{4,6}$/,
      message: "验证码为4-6位数字",
      trigger: "blur",
    },
  ],
};
const captchaTimeOut = ref(null);
const captchaText = ref("获取验证码");
const captchaDisabled = ref(false);

// 获取验证码
const getCaptcha = (phone) => {
  clearInterval(captchaTimeOut.value);
  phoneFormRef.value?.validate(
    async (errors) => {
      if (!errors) {
        console.log(phone + "发送验证码");
        try {
          const result = await sentCaptcha(phone);
          console.log("验证码发送结果:", result);
          if (result.code === 200) {
            $message.success("验证码发送成功");
            let countDown = 60;
            captchaDisabled.value = true;
            captchaText.value = countDown + "s";
            captchaTimeOut.value = setInterval(() => {
              countDown--;
              captchaText.value = countDown + "s";
              if (countDown === 0) {
                clearInterval(captchaTimeOut.value);
                captchaText.value = "重新获取";
                captchaDisabled.value = false;
              }
            }, 1000);
          } else {
            $message.error(result.msg || result.message || "验证码发送失败，请重试");
          }
        } catch (error) {
          console.error("验证码发送出错：", error);
          const errorMsg = error?.response?.data?.message || error?.message || "网络错误，请重试";
          $message.error(`验证码发送失败：${errorMsg}`);
        }
      } else {
        $message.error("请检查你的手机号输入");
      }
    },
    (rule) => {
      return rule?.key === "phone";
    },
  );
};

// 手机号登录
const phoneLogin = (e) => {
  e.preventDefault();
  phoneFormRef.value?.validate(async (errors) => {
    if (!errors) {
      try {
        const verifyRes = await verifyCaptcha(
          phoneFormData.value.phone,
          String(phoneFormData.value.captcha),
        );
        console.log("验证码验证结果:", verifyRes);
        if (verifyRes.code === 200) {
          const result = await toLogin(
            phoneFormData.value.phone,
            String(phoneFormData.value.captcha),
          );
          console.log("登录结果:", result);
          if (result.code === 200) {
            // 去除 HTTPOnly
            if (result.cookie) {
              result.cookie = result.cookie.replaceAll(" HTTPOnly", "");
            }
            // 是否含有 MUSIC_U
            if (result.cookie && result.cookie.includes("MUSIC_U")) {
              // 储存登录信息
              emit("setLoginData", result);
            } else {
              $message.error("登录返回数据异常，请稍后重试");
            }
          } else {
            $message.error(result.msg || result.message || "登录失败，请重试");
          }
        } else {
          $message.error(verifyRes.msg || verifyRes.message || "验证码验证失败，请检查验证码是否正确");
        }
      } catch (error) {
        phoneFormData.value.captcha = "";
        console.error("登录出错：", error);
        // 显示更详细的错误信息
        const errorMsg = error?.response?.data?.message || error?.message || "网络错误，请重试";
        $message.error(`登录出错：${errorMsg}`);
      }
    } else {
      $message.error("请检查你的输入");
    }
  });
};
</script>

<style lang="scss" scoped>
.login-phone {
  .phone-form {
    margin-top: 20px;
  }
  .captcha-input-wrapper {
    display: flex;
    align-items: center;
  }
}
</style>
