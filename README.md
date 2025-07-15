<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>스마트 시티 시뮬레이션</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      background-color: #ccc;
      user-select: none;
    }
    canvas {
      border: 2px solid #333;
      display: block;
      margin: 0 auto;
      background-color: #ccc;
      cursor: default;
    }
    #exitButton {
      position: absolute;
      padding: 6px 12px;
      background-color: #e74c3c;
      color: white;
      font-weight: bold;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      display: none;
      z-index: 10;
      user-select: none;
      right: 20px;
      top: 20px;
    }
  </style>
</head>
<body>
<button id="exitButton">출구</button>
<canvas id="cityCanvas" width="800" height="600"></canvas>

<script>
const canvas = document.getElementById("cityCanvas");
const ctx = canvas.getContext("2d");
const exitBtn = document.getElementById("exitButton");

const canvasWidth = canvas.width;
const canvasHeight = canvas.height;

// 모드: "city" (외부 도시), "building" (건물 내부)
let mode = "city";
let currentBuilding = null;

// 건물 데이터 (외부 도시)
const buildings = [
  {
    x: 50, y: 60, w: 130, h: 140,
    name: "에너지 센터",
    type: "energyCenter",
    energyProduced: 5000,
    energyType: "태양광 + 풍력",
    cityPowerUsage: 12000
  },
  {
    x: 200, y: 60, w: 150, h: 130,
    name: "스마트 공장",
    type: "factory",
    productCount: 524,
    defectRate: 0.07,
    powerPerHour: 130,
    productInfo: "스마트폰 부품 생산 중"
  },
  {
    x: 370, y: 60, w: 130, h: 130,
    name: "아파트",
    type: "apartment",
    floors: 10,
    residents: 50
  },
  {
    x: 520, y: 60, w: 100, h: 90,
    name: "주택",
    type: "house",
    rooms: 4,
    residents: 3
  }
];

// 대형 상가 건물 (도로 아래)
const shoppingMall = {
  x: 50,
  y: 410,
  w: 700,
  h: 150,
  name: "대형 상가",
  floors: [
    { floor: 1, occupancy: "사무실 3개", powerUsage: 1200, waterUsage: 800, vacancy: false },
    { floor: 2, occupancy: "상점 5개", powerUsage: 1500, waterUsage: 1200, vacancy: false },
    { floor: 3, occupancy: "사무실 2개", powerUsage: 1000, waterUsage: 600, vacancy: true },
    { floor: 4, occupancy: "상점 4개", powerUsage: 1400, waterUsage: 1100, vacancy: false },
    { floor: 5, occupancy: "사무실 1개", powerUsage: 800, waterUsage: 500, vacancy: false }
  ]
};

// 가로등 설정 (도로 양 옆, 도로 바로 옆 위치)
// 총 7개 가로등, 간격 동일, 버스정류장 영역 제외
const lampRadius = 6;
const totalLamps = 7;
const gap = canvasWidth / (totalLamps - 1);

const lamps = [];
const busStop = {
  x: 580,
  y: 230,
  w: 70,
  h: 50,
  name: "버스 정류장"
};
const busStopX = busStop.x;
const busStopW = busStop.w;
const busStopEndX = busStopX + busStopW;

const roadTopY = 280;
const roadHeight = 60;

for (let i = 0; i < totalLamps; i++) {
  const xPos = i * gap;
  // 버스 정류장 영역 위에는 가로등 생성하지 않음
  if (!(xPos >= busStopX && xPos <= busStopEndX)) {
    lamps.push({ x: xPos, y: roadTopY - 10, radius: lampRadius });
    lamps.push({ x: xPos, y: roadTopY + roadHeight + 10, radius: lampRadius });
  }
}

// 버스 경로 및 상태
const busRoute = [
  { x: 0, y: 300 },
  { x: 750, y: 300 }
];
const bus = {
  x: busRoute[0].x,
  y: busRoute[0].y,
  w: 60,
  h: 30,
  speed: 1.5,
  direction: 1,
  stopped: false,
  stopDuration: 0,
  maxStopTime: 120
};

// 사람 상태 (외부 및 내부 공통)
const person = {
  x: 50,
  y: 50,
  r: 15,
  dragging: false,
  offsetX: 0,
  offsetY: 0
};

// 내부 건물 구조 (예시)
const buildingInteriors = {
  apartment: {
    width: 800,
    height: 600,
    rooms: [
      { x: 50, y: 50, w: 400, h: 300, name: "거실" },      // 거실 크게
      { x: 500, y: 50, w: 250, h: 150, name: "주방" },     // 주방 적당히
      { x: 500, y: 220, w: 250, h: 180, name: "침실" }     // 침실 크게
    ]
  },
  house: {
    width: 800,
    height: 600,
    rooms: [
      { x: 40, y: 40, w: 350, h: 300, name: "거실" },
      { x: 410, y: 40, w: 280, h: 200, name: "주방" },
      { x: 410, y: 260, w: 280, h: 280, name: "침실" }
    ]
  }
};

// ----------------------
// 함수들
// ----------------------

function drawCity() {
  ctx.clearRect(0, 0, canvasWidth, canvasHeight);

  // 배경 아스팔트색
  ctx.fillStyle = "#999";
  ctx.fillRect(0, 0, canvasWidth, canvasHeight);

  // 도로
  ctx.fillStyle = "#666";
  ctx.fillRect(0, roadTopY, canvasWidth, roadHeight);

  // 건물 그리기
  buildings.forEach(b => {
    ctx.fillStyle = "#3b5998";
    if (b.type === "factory") ctx.fillStyle = "#8e44ad";
    else if (b.type === "energyCenter") ctx.fillStyle = "#27ae60";
    else if (b.type === "apartment") ctx.fillStyle = "#2980b9";
    else if (b.type === "house") ctx.fillStyle = "#f39c12";

    ctx.fillRect(b.x, b.y, b.w, b.h);

    ctx.fillStyle = "#fff";
    ctx.font = "14px sans-serif";
    ctx.fillText(b.name, b.x + 8, b.y + 20);
  });

  // 대형 상가 (도로 아래)
  ctx.fillStyle = "#d35400";
  ctx.fillRect(shoppingMall.x, shoppingMall.y, shoppingMall.w, shoppingMall.h);
  ctx.fillStyle = "#fff";
  ctx.font = "bold 18px sans-serif";
  ctx.fillText(shoppingMall.name, shoppingMall.x + 20, shoppingMall.y + 30);

  // 버스 정류장
  ctx.fillStyle = "#ecf0f1";
  ctx.fillRect(busStop.x, busStop.y, busStop.w, busStop.h);
  ctx.strokeStyle = "#2c3e50";
  ctx.strokeRect(busStop.x, busStop.y, busStop.w, busStop.h);
  ctx.fillStyle = "#2c3e50";
  ctx.font = "bold 16px sans-serif";
  ctx.fillText(busStop.name, busStop.x + 5, busStop.y + 30);

  // 가로등 그리기
  drawLamps();

  // 사람 그리기
  drawPerson();

  // 버스 그리기
  drawBus();

  // 상호작용 텍스트
  let interactionText = "";

  if (!person.dragging) {
    buildings.forEach(b => {
      if (isNear(person, b)) {
        if (b.type === "factory") {
          const defectNum = Math.round(b.productCount * b.defectRate);
          interactionText += `[스마트 공장]\n`;
          interactionText += `제품 수량: ${b.productCount}개\n`;
          interactionText += `불량률: ${(b.defectRate * 100).toFixed(1)}%\n`;
          interactionText += `불량품 수: ${defectNum}개\n`;
          interactionText += `전력 소비: ${b.powerPerHour}kWh/h\n`;
          interactionText += `제품 설명: ${b.productInfo}\n\n`;
        } else if (b.type === "energyCenter") {
          interactionText += `[에너지 센터]\n`;
          interactionText += `생산 에너지: ${b.energyProduced} kWh\n`;
          interactionText += `에너지 종류: ${b.energyType}\n`;
          interactionText += `도시 소비 전력: ${b.cityPowerUsage} kWh/h\n\n`;
        } else if (b.type === "apartment") {
          interactionText += `[아파트]\n층수: ${b.floors}층\n거주자 수: ${b.residents}명\n\n`;
        } else if (b.type === "house") {
          interactionText += `[주택]\n방 수: ${b.rooms}개\n거주자 수: ${b.residents}명\n\n`;
        } else {
          interactionText += `🏢 ${b.name} 근처에 있습니다.\n\n`;
        }
      }
    });

    // 대형 상가 정보 표시
    if (person.x > shoppingMall.x && person.x < shoppingMall.x + shoppingMall.w &&
        person.y > shoppingMall.y && person.y < shoppingMall.y + shoppingMall.h) {

      interactionText += `[대형 상가]\n`;

      const totalPowerUsage = shoppingMall.floors.reduce((sum, floor) => sum + (floor.powerUsage || 0), 0);
      const vacancyCount = shoppingMall.floors.reduce((count, floor) => floor.vacancy ? count + 1 : count, 0);

      interactionText += `총 전력 사용량: ${totalPowerUsage} kWh\n`;

      // 각 층 상세 정보 모두 출력 추가
      shoppingMall.floors.forEach(floor => {
        interactionText += `${floor.floor}층: ${floor.occupancy}, 전력: ${floor.powerUsage}kWh, 공실: ${floor.vacancy ? "O" : "X"}\n`;
      });
      interactionText += `\n`;
    }
  }

  // 사람이 버스 정류장 위에 있을 때 버스 도착 정보 표시
  if (isOnBusStop(person, busStop)) {
    interactionText += `[버스 정류장 정보]\n`;

    const busPos = bus.x;
    const busSpeed = bus.speed;
    const busDir = bus.direction;

    let arrivalTimeText = "도착 예정 정보 없음";

    if (bus.stopped) {
      arrivalTimeText = "도착함";
    } else if (busDir === 1 && busPos < busStop.x) {
      const distance = busStop.x - busPos;
      const timeMin = distance / busSpeed / 60;
      arrivalTimeText = formatTimeMin(timeMin);
    } else if (busDir === -1 && busPos > busStop.x + busStop.w) {
      const distance = busPos - (busStop.x + busStop.w);
      const timeMin = distance / busSpeed / 60;
      arrivalTimeText = formatTimeMin(timeMin);
    } else if (busPos >= busStop.x && busPos <= busStop.x + busStop.w) {
      arrivalTimeText = "곧 도착";
    } else {
      arrivalTimeText = "버스 운행 중";
    }

    interactionText += `다음 도착: ${arrivalTimeText}\n`;
    interactionText += `남은 좌석: 5석\n`;
  }

  if (interactionText !== "") {
    drawInfoBox(interactionText);
  }
}

// 시간 분 단위 표시 + "곧 도착" 텍스트 변환 함수
function formatTimeMin(mins) {
  if (mins <= 1) return "곧 도착";
  return `${Math.ceil(mins)}분 후 도착`;
}

function drawLamps() {
  lamps.forEach(lamp => {
    const distance = Math.hypot(person.x - lamp.x, person.y - lamp.y);
    const isNearLamp = distance < 60;

    if (isNearLamp) {
      let gradient = ctx.createRadialGradient(lamp.x, lamp.y, lamp.radius, lamp.x, lamp.y, 100);
      gradient.addColorStop(0, "rgba(255, 255, 150, 0.8)");
      gradient.addColorStop(1, "rgba(255, 255, 150, 0)");
      ctx.fillStyle = gradient;
      ctx.beginPath();
      ctx.arc(lamp.x, lamp.y, 100, 0, Math.PI * 2);
      ctx.fill();
      ctx.closePath();
    }

    ctx.beginPath();
    ctx.arc(lamp.x, lamp.y, lamp.radius, 0, Math.PI * 2);
    ctx.fillStyle = isNearLamp ? "yellow" : "gray";
    ctx.fill();
    ctx.strokeStyle = "#555";
    ctx.stroke();
    ctx.closePath();
  });
}

function drawPerson() {
  ctx.beginPath();
  ctx.arc(person.x, person.y, person.r, 0, Math.PI * 2);
  ctx.fillStyle = "lime";
  ctx.fill();
  ctx.closePath();
}

function drawBus() {
  ctx.fillStyle = "orange";
  ctx.fillRect(bus.x, bus.y, bus.w, bus.h);
  ctx.fillStyle = "#000";
  ctx.font = "14px sans-serif";
  ctx.fillText("버스", bus.x + 15, bus.y + 20);
}

function isNear(p, obj) {
  const px = p.x;
  const py = p.y;
  const ox = obj.x;
  const oy = obj.y;
  const ow = obj.w || 0;
  const oh = obj.h || 0;
  return (px > ox - 30 && px < ox + ow + 30 &&
          py > oy - 30 && py < oy + oh + 30);
}

function isOnBusStop(p, stop) {
  return (p.x > stop.x && p.x < stop.x + stop.w &&
          p.y > stop.y && p.y < stop.y + stop.h);
}

// 버스 이동 및 정류장 정차 처리
function updateBus() {
  // 모드가 내부면 버스 멈춤 유지
  if (mode === "building") return;

  const personOnStop = isOnBusStop(person, busStop);
  const distanceToStop = Math.abs(bus.x - busStop.x);

  if (bus.stopped) {
    bus.stopDuration++;
    if (bus.stopDuration >= bus.maxStopTime) {
      bus.stopped = false;
      bus.stopDuration = 0;
      if (bus.direction === 1 && bus.x >= busRoute[1].x) bus.direction = -1;
      else if (bus.direction === -1 && bus.x <= busRoute[0].x) bus.direction = 1;
    }
  } else {
    if (distanceToStop < 5 && personOnStop) {
      if ((bus.direction === 1 && bus.x <= busStop.x) ||
          (bus.direction === -1 && bus.x >= busStop.x)) {
        bus.stopped = true;
        bus.stopDuration = 0;
      } else {
        bus.x += bus.speed * bus.direction;
      }
    } else {
      if (bus.x >= busRoute[1].x) bus.direction = -1;
      else if (bus.x <= busRoute[0].x) bus.direction = 1;
      bus.x += bus.speed * bus.direction;
    }
  }
}

// 정보 박스 그리기 (크기 확장)
function drawInfoBox(text) {
  const lines = text.trim().split("\n");
  const boxX = 20, boxY = 400, boxW = 380, lineHeight = 22;
  const boxH = lines.length * lineHeight + 20;

  ctx.fillStyle = "rgba(0,0,0,0.85)";
  ctx.fillRect(boxX, boxY, boxW, boxH);

  ctx.fillStyle = "#fff";
  ctx.font = "16px monospace";
  lines.forEach((line, i) => {
    ctx.fillText(line, boxX + 10, boxY + 30 + i * lineHeight);
  });
}

// ----------------
// 내부 건물 모드
// ----------------
function drawBuildingInterior() {
  if (!currentBuilding) return;

  const interior = buildingInteriors[currentBuilding.type];
  if (!interior) return;

  ctx.clearRect(0, 0, canvasWidth, canvasHeight);

  // 배경 - 밝은 연회색으로 변경
  ctx.fillStyle = "#f0f0f5";  // 밝은 연회색
  ctx.fillRect(0, 0, canvasWidth, canvasHeight);

  // 방 그리기 (밝은 하늘색 + 테두리 진한 회색)
  interior.rooms.forEach(room => {
    ctx.fillStyle = "#a8d0e6";  // 밝은 하늘색
    ctx.fillRect(room.x, room.y, room.w, room.h);

    ctx.strokeStyle = "#607d8b"; // 진한 회색 테두리
    ctx.lineWidth = 2;
    ctx.strokeRect(room.x, room.y, room.w, room.h);

    ctx.fillStyle = "#37474f";  // 진한 회색 텍스트
    ctx.font = "18px sans-serif";
    ctx.fillText(room.name, room.x + 10, room.y + 30);
  });

  // 사람 그리기 (내부)
  drawPerson();

  // 출구 버튼 표시
  exitBtn.style.display = "block";
}

// -----------------
// 이벤트 핸들링
// -----------------

// 마우스 드래그 시작
canvas.addEventListener("mousedown", e => {
  const rect = canvas.getBoundingClientRect();
  const mx = e.clientX - rect.left;
  const my = e.clientY - rect.top;

  // 외부 도시 모드에서만 드래그 가능
  if (mode === "city") {
    const dist = Math.hypot(mx - person.x, my - person.y);
    if (dist < person.r) {
      person.dragging = true;
      person.offsetX = mx - person.x;
      person.offsetY = my - person.y;
    }
  }
});

// 드래그 중
canvas.addEventListener("mousemove", e => {
  if (!person.dragging) return;

  const rect = canvas.getBoundingClientRect();
  let mx = e.clientX - rect.left;
  let my = e.clientY - rect.top;

  person.x = mx - person.offsetX;
  person.y = my - person.offsetY;

  // 경계 제한
  person.x = Math.max(0, Math.min(canvasWidth, person.x));
  person.y = Math.max(0, Math.min(canvasHeight, person.y));
});

// 드래그 끝
canvas.addEventListener("mouseup", e => {
  if (person.dragging) {
    person.dragging = false;
  }
});

// 건물 내부 진입 처리 (아파트, 주택만 가능)
canvas.addEventListener("click", e => {
  if (mode === "city") {
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left;
    const my = e.clientY - rect.top;

    for (const b of buildings) {
      if (mx >= b.x && mx <= b.x + b.w && my >= b.y && my <= b.y + b.h) {
        // 아파트 또는 주택 내부 진입
        if (b.type === "apartment" || b.type === "house") {
          mode = "building";
          currentBuilding = b;
          // 사람 초기 위치 내부 거실로 설정
          person.x = 50;
          person.y = 50;
          exitBtn.style.display = "block";
          break;
        }
      }
    }
  }
});

// 출구 버튼 클릭 시 내부에서 외부 도시로 복귀
exitBtn.addEventListener("click", () => {
  if (mode === "building") {
    mode = "city";
    currentBuilding = null;
    // 외부 도시 사람 초기 위치 설정
    person.x = 100;
    person.y = 100;
    exitBtn.style.display = "none";
  }
});

// -----------------
// 애니메이션 루프
// -----------------
function loop() {
  if (mode === "city") {
    drawCity();
    updateBus();
  } else if (mode === "building") {
    drawBuildingInterior();
  }
  requestAnimationFrame(loop);
}

// 초기화
exitBtn.style.display = "none";
loop();

</script>
</body>
</html>
