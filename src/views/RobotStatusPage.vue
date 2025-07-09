<template>
  <div id="robot-status-page">
    <div id="map-section">
      <div id="map-container"></div>
    </div>
    <div id="table-section">      
      <div class="table-scroll-container">
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>이름</th>
              <th>위치 (위도, 경도)</th>
              <th>상태</th>
              <th>마지막 업데이트</th>
            </tr>
          </thead>
          <tbody>
            <tr v-if="robotsData.length === 0">
              <td colspan="5" style="text-align: center;">로봇 데이터를 불러올 수 없습니다.</td>
            </tr>
            <tr v-for="bot in robotsData" :key="bot.botId" @click="panToRobot(bot)">
              <td>{{ bot.botId }}</td>
              <td>{{ bot.botName }}</td>
              <td>{{ bot.location }}</td>
              <td>{{ bot.status }}</td>
              <td>{{ new Date(bot.lastUpdated).toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import axios from 'axios'; // axios 임포트

// 웹팩 환경에서 기본 아이콘이 깨지는 문제 해결
delete L.Icon.Default.prototype._getIconUrl;
L.Icon.Default.mergeOptions({
  iconRetinaUrl: require('leaflet/dist/images/marker-icon-2x.png'),
  iconUrl: require('leaflet/dist/images/marker-icon.png'),
  shadowUrl: require('leaflet/dist/images/marker-shadow.png'),
});

export default {
  name: 'RobotStatusPage',
  data() {
    return {
      map: null,
      robotMarkers: {}, // Leaflet 마커를 botId를 키로 저장
      pollingInterval: null,
      robotsData: [], // 로봇 데이터를 저장할 배열
      apiClient: null, // axios 인스턴스를 저장할 속성
    };
  },
  created() {
    // 컴포넌트 생성 시 axios 인스턴스 초기화
    const backendApiUrl = process.env.VUE_APP_BACKEND_API_URL;
    const apimSubscriptionKey = process.env.VUE_APP_APIM_SUBSCRIPTION_KEY;

    if (!backendApiUrl) {
      console.error('VUE_APP_BACKEND_API_URL 환경 변수가 정의되지 않았습니다. .env 파일을 확인하세요.');
      return;
    }
    if (!apimSubscriptionKey) {
      console.warn('VUE_APP_APIM_SUBSCRIPTION_KEY 환경 변수가 정의되지 않았습니다. API 호출에 문제가 있을 수 있습니다.');
    }

    this.apiClient = axios.create({
      baseURL: backendApiUrl,
      headers: {
        'Content-Type': 'application/json',
        ...(apimSubscriptionKey && { 'Ocp-Apim-Subscription-Key': apimSubscriptionKey })
      },
      timeout: 5000 // 5초 타임아웃 설정
    });
  },
  mounted() {
    this.initializeMap();
    this.startPollingRobotStatus();
    // 지도 이벤트 리스너 추가
    if (this.map) {
      this.map.on('zoomend', this.onMapInteraction); // 확대/축소 종료 시
      this.map.on('moveend', this.onMapInteraction); // 지도 이동 종료 시 (스크롤 포함)
    }
  },
  beforeUnmount() {
    this.stopPollingRobotStatus();
    if (this.map) {
      // 지도 이벤트 리스너 제거
      this.map.off('zoomend', this.onMapInteraction);
      this.map.off('moveend', this.onMapInteraction);

      // 1. 모든 마커 레이어를 명시적으로 제거
      for (const botId in this.robotMarkers) {
        if (this.robotMarkers[botId] && this.map.hasLayer(this.robotMarkers[botId])) {
          this.map.removeLayer(this.robotMarkers[botId]);
        }
      }
      this.robotMarkers = {}; // 마커 객체 초기화

      // 2. Leaflet의 내부 애니메이션이 완료될 시간을 주기 위해 지연을 줍니다.
      setTimeout(() => {
        if (this.map) {
          this.map.remove();
          this.map = null;
        }
      }, 100);
    }
  },
  methods: {
    initializeMap() {
      if (this.map) {
        return;
      }
      this.map = L.map('map-container').setView([37.5665, 126.978], 13);

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
      }).addTo(this.map);
    },
    // 지도 인터랙션 (확대/축소, 이동) 종료 시 마커를 업데이트하는 핸들러
    onMapInteraction() {
      if (this.map && this.robotsData.length > 0) {
        console.log('지도 인터랙션 종료, 마커 업데이트를 시도합니다.');
        this.updateRobotMarkers(this.robotsData);
      }
    },
    async fetchRobotStatus() {
      try {
        if (!this.apiClient) {
          console.error('API 클라이언트가 초기화되지 않았습니다. API 호출을 건너뜝니다.');
          this.robotsData = [];
          return;
        }

        const response = await this.apiClient.get('/waterbots');
        let robotData = response.data;

        if (!Array.isArray(robotData)) {
          robotData = [robotData];
        }

        this.updateRobotMarkers(robotData);
        this.robotsData = robotData;

      } catch (error) {
        console.error('로봇 상태를 가져오는 데 실패했습니다:', error);
        if (error.response) {
          console.error('오류 응답 상태:', error.response.status);
          console.error('오류 응답 데이터:', error.response.data);
          console.error('오류 응답 헤더:', error.response.headers);
        } else if (error.request) {
          console.error('응답을 받지 못했습니다. 요청:', error.request);
        } else {
          console.error('오류 메시지:', error.message);
        }
        this.robotsData = [];
      }
    },
    updateRobotMarkers(robotData) {
      if (!this.map) {
        console.warn('Map is null, skipping marker update.');
        return;
      }

      const currentBotIds = new Set(robotData.map(bot => bot.botId));

      for (const botId in this.robotMarkers) {
        if (!currentBotIds.has(botId)) {
          if (this.robotMarkers[botId] && this.map.hasLayer(this.robotMarkers[botId])) {
            this.map.removeLayer(this.robotMarkers[botId]);
          }
          delete this.robotMarkers[botId];
        }
      }

      robotData.forEach(bot => {
        const [lat, lng] = bot.location.split(',').map(Number);
        const robotId = bot.botId;

        if (isNaN(lat) || isNaN(lng)) {
          console.warn(`봇 ${robotId}에 대한 유효하지 않은 위치 데이터: ${bot.location}`);
          return;
        }

        const tooltipContent = `로봇 ID: ${bot.botId}<br>이름: ${bot.botName}<br>상태: ${bot.status}<br>마지막 업데이트: ${new Date(bot.lastUpdated).toLocaleString()}`;

        if (this.robotMarkers[robotId]) {
          this.robotMarkers[robotId].setLatLng([lat, lng]);
          this.robotMarkers[robotId].setTooltipContent(tooltipContent);
        } else {
          const robotIcon = L.icon({
            iconUrl: '/images/waterbot.png',
            iconSize: [100, 100],
            iconAnchor: [16, 32],
            popupAnchor: [0, -32]
          });

          const marker = L.marker([lat, lng], { icon: robotIcon })
            .addTo(this.map)
            .bindTooltip(tooltipContent, {
              permanent: false,
              direction: 'right'
            });
          this.robotMarkers[robotId] = marker;
        }
      });
    },
    startPollingRobotStatus() {
      this.fetchRobotStatus();
      this.pollingInterval = setInterval(this.fetchRobotStatus, 60000);
    },
    stopPollingRobotStatus() {
      if (this.pollingInterval) {
        clearInterval(this.pollingInterval);
        this.pollingInterval = null;
      }
    },
    // 새롭게 추가된 메서드: 로봇 위치로 지도 이동 및 확대
    panToRobot(bot) {
      if (!this.map) {
        console.warn('Map is not initialized, cannot pan to robot.');
        return;
      }

      if (!bot || !bot.location) {
        console.warn('Invalid robot data or location for pan operation:', bot);
        return;
      }

      const [lat, lng] = bot.location.split(',').map(Number);

      if (isNaN(lat) || isNaN(lng)) {
        console.warn(`Invalid latitude or longitude for bot ${bot.botId}: ${bot.location}`);
        return;
      }

      // 목표 줌 레벨 설정 (일반적으로 18-19는 거리 수준의 상세 보기를 제공합니다.)
      const targetZoomLevel = 18; 

      // flyTo를 사용하여 부드럽게 지도를 이동하고 확대합니다.
      this.map.flyTo([lat, lng], targetZoomLevel, {
        animate: true,   // 애니메이션 사용
        duration: 1.5    // 애니메이션 지속 시간 (초)
      });

      // 선택 사항: 해당 로봇 마커의 툴팁을 바로 열어주는 것도 좋은 사용자 경험이 될 수 있습니다.
      // if (this.robotMarkers[bot.botId]) {
      //   this.robotMarkers[bot.botId].openTooltip();
      // }
    },
  },
};
</script>

<style scoped>
#robot-status-page {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100vw;
}

#map-section {
  flex: 3;
  display: flex;
  min-height: 0;
}

#map-container {
  width: 100%;
  height: 100%;
}

#table-section {
  flex: 2;
  padding: 1rem;
  overflow-y: auto;
  border-top: 1px solid #eee;
  display: flex;
  flex-direction: column;
}

#table-section h2 {
  margin-top: 0;
  margin-bottom: 1rem;
  color: #333;
  text-align: center;
}

.table-scroll-container {
  flex-grow: 1;
  overflow-y: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
}

table th,
table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

/* 표 헤더 스타일 조정 */
table th {
  background-color: black; /* 배경 검은색 */
  color: white; /* 글자색 흰색 */
  font-weight: bold;
  position: sticky;
  top: 0;
  z-index: 1;
  text-align: center; /* 텍스트 가운데 정렬 */
}

table tr:nth-child(even) {
  background-color: #f9f9f9;
}

/* 표 행에 클릭 가능 시각적 피드백 추가 */
table tbody tr {
  cursor: pointer; /* 마우스 오버 시 포인터 커서로 변경 */
  transition: background-color 0.2s ease; /* 부드러운 배경색 전환 */
}

table tbody tr:hover {
  background-color: #e0e0e0; /* 호버 시 약간 더 어두운 배경색 */
}
</style>