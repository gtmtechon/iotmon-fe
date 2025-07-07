<template>
    <div class="page-container">
      <h1>장비 수정: {{ device ? device.deviceName : '로딩 중...' }}</h1>
      <DeviceForm v-if="device" :initialDevice="device" :isEditMode="true" @submit="updateDevice" />
      <div v-else-if="loading" class="loading-message">장비 정보를 불러오는 중...</div>
      <div v-else-if="error" class="error-message">{{ error }}</div>
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
    }
  });
  
  export default {
    name: 'EditDevicePage',
    components: {
      DeviceForm,
    },
    props: ['deviceId'], // 라우터에서 전달받은 deviceId
    data() {
      return {
        device: null,
        loading: true,
        error: null,
      };
    },
    mounted() {
      this.fetchDevice();
    },
    methods: {
      async fetchDevice() {
        this.loading = true;
        this.error = null;
        try {
          // 2. 생성된 apiClient 인스턴스를 사용하여 요청
          const response = await apiClient.get(`/devices/${this.deviceId}`);
          this.device = response.data;
        } catch (err) {
          console.error('Error fetching device for edit:', err);
          // 오류 메시지에 HTTP 상태 코드를 포함하여 디버깅에 도움
          this.error = `장비 정보를 불러오는 데 실패했습니다: ${err.response?.status ? `HTTP ${err.response.status} - ` : ''}${err.response?.data?.message || err.message}`;
          this.device = null; // 에러 발생 시 폼 숨김
        } finally {
          this.loading = false;
        }
      },
      async updateDevice(deviceData) {
        this.error = null;
        try {
          // 3. 생성된 apiClient 인스턴스를 사용하여 요청
          await apiClient.put(`/devices/${this.deviceId}`, deviceData);
          alert('장비가 성공적으로 수정되었습니다.');
          this.$router.push('/'); // 수정 후 목록 페이지로 이동
        } catch (err) {
          console.error('Error updating device:', err);
          // 401 Unauthorized 오류일 경우 메시지 추가
          if (err.response && err.response.status === 401) {
              this.error = `장비 수정에 실패했습니다: 인증 오류 (Subscription Key 누락 또는 잘못됨). 관리자에게 문의하세요.`;
          } else if (err.response && err.response.data) {
            this.error = `장비 수정에 실패했습니다: ${err.response.data.message || err.response.data}`;
          } else {
            this.error = '장비 수정에 실패했습니다. 서버를 확인해주세요.';
          }
        }
      },
    },
  };
  </script>
  
  <style scoped>
  /* 기존 스타일은 변경 없음 */
  .page-container {
    padding: 20px;
    text-align: center;
  }
  h1 {
    color: #0056b3;
  }
  .loading-message, .error-message {
    margin-top: 20px;
    font-size: 1.1em;
  }
  .error-message {
    color: red;
  }
  </style>