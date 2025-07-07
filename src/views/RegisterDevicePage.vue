<template>
  <div class="page-container">
    <h1>새 IoT 장비 등록</h1>
    <DeviceForm @submit="registerDevice" :isSubmitting="isRegistering" />
    <p v-if="error" class="error-message">{{ error }}</p>
  </div>
</template>

<script>
import axios from 'axios';
import DeviceForm from '../components/DeviceForm.vue';

// 1. axios 인스턴스 생성 및 기본 설정
// process.env.VUE_APP_BACKEND_API_URL은 APIM의 엔드포인트 URL이 되어야 합니다.
// process.env.VUE_APP_APIM_SUBSCRIPTION_KEY는 APIM 구독 키가 되어야 합니다.
const apiClient = axios.create({
  baseURL: process.env.VUE_APP_BACKEND_API_URL,
  headers: {
    'Content-Type': 'application/json',
    'Ocp-Apim-Subscription-Key': process.env.VUE_APP_APIM_SUBSCRIPTION_KEY // 모든 요청에 공통으로 추가될 헤더
  },
  timeout: 5000 // 5초 타임아웃 설정 (기본 인스턴스에 적용)
});

export default {
  name: 'RegisterDevicePage',
  components: {
    DeviceForm,
  },
  data() {
    return {
      error: null,
      isRegistering: false, // 등록 요청 진행 상태
    };
  },
  methods: {
    async registerDevice(deviceData) {
      this.error = null;
      this.isRegistering = true; // 등록 시작, 버튼 비활성화

      try {
        // 2. 생성된 apiClient 인스턴스를 사용하여 요청
        // timeout은 apiClient 생성 시점에 설정했으므로 여기서 다시 명시할 필요는 없습니다.
        await apiClient.post('/devices', deviceData);
        alert('장비가 성공적으로 등록되었습니다.');
        this.$router.push('/'); // 등록 후 목록 페이지로 이동
      } catch (err) {
        console.error('Error registering device:', err);

        // Axios Timeout Error 처리
        if (axios.isCancel(err)) {
          this.error = '장비 등록 요청이 취소되었습니다.'; // 요청이 수동으로 취소된 경우
        } else if (err.code === 'ECONNABORTED' && err.message.includes('timeout')) {
          this.error = '서버 응답이 없습니다. 서버가 실행 중인지 확인해주세요.';
        } else if (err.response) {
          // 서버에서 에러 응답이 온 경우
          // 401 Unauthorized 오류일 경우 메시지 추가
          if (err.response.status === 401) {
              this.error = `401 error.`;
          } else {
              this.error = `장비 등록에 실패했습니다: ${err.response.data?.message || err.response.data || err.message}`;
          }
        } else {
          // 기타 네트워크 오류 등
          this.error = '장비 등록에 실패했습니다. 네트워크 연결을 확인하거나 서버를 확인해주세요.';
        }
      } finally {
        this.isRegistering = false; // 등록 완료 (성공 또는 실패), 버튼 활성화
      }
    },
  },
};
</script>

<style scoped>
.page-container {
  padding: 20px;
  text-align: center;
}
h1 {
  color: #0056b3;
}
.error-message {
  color: red;
  margin-top: 15px;
}
</style>
