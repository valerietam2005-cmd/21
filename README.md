<!DOCTYPE html>
<html lang="zh-HK">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover, user-scalable=no">
  <title>香港旅蹤 · 雙語行程規劃</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Inter:opsz,wght@14..32,300;400;500;600;700&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">
  <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <style>
    * { font-family: 'Inter', sans-serif; }
    body { background: #f5f7fb; }
    .map-container { height: 220px; border-radius: 24px; overflow: hidden; }
    .card-shadow { box-shadow: 0 8px 20px rgba(0,0,0,0.03), 0 2px 6px rgba(0,0,0,0.05); }
    button, .clickable { cursor: pointer; }
    /* 圖片放大 modal 樣式 */
    .image-modal {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0,0,0,0.9);
      z-index: 1000;
      display: flex;
      align-items: center;
      justify-content: center;
      cursor: pointer;
    }
    .image-modal img {
      max-width: 90%;
      max-height: 90%;
      object-fit: contain;
      border-radius: 16px;
    }
  </style>
</head>
<body>
<div id="app" class="max-w-md mx-auto bg-gray-50 min-h-screen pb-24">
  <!-- 頂欄 + 語言切換 -->
  <div class="sticky top-0 z-20 bg-white/80 backdrop-blur-md border-b border-gray-100 px-5 py-3 flex justify-between items-center">
    <div class="flex items-center space-x-2">
      <i class="fa-solid fa-map-location-dot text-2xl text-red-500"></i>
      <h1 class="text-xl font-bold bg-gradient-to-r from-red-600 to-amber-600 bg-clip-text text-transparent">{{ t('app_name') }}</h1>
    </div>
    <div class="flex gap-2">
      <button @click="setLanguage('zh')" :class="locale === 'zh' ? 'bg-red-500 text-white' : 'bg-gray-100 text-gray-700'" class="text-sm px-3 py-1 rounded-full transition-all">繁</button>
      <button @click="setLanguage('en')" :class="locale === 'en' ? 'bg-red-500 text-white' : 'bg-gray-100 text-gray-700'" class="text-sm px-3 py-1 rounded-full transition-all">EN</button>
    </div>
  </div>

  <div class="p-5 space-y-5">
    <!-- 設定表單 -->
    <div class="bg-white rounded-2xl card-shadow p-5 space-y-5">
      <div class="flex justify-between">
        <span class="text-sm font-semibold text-gray-500"><i class="fa-regular fa-calendar-alt mr-1"></i> {{ t('plan_title') }}</span>
        <span class="text-xs text-gray-400">{{ t('collab_badge') }}</span>
      </div>
      <div>
        <div class="flex justify-between text-sm font-medium text-gray-700 mb-2">
          <span>{{ t('days_label') }}</span>
          <span class="bg-amber-100 px-3 py-0.5 rounded-full text-amber-800">{{ days }} {{ t('days_unit') }}</span>
        </div>
        <input type="range" min="1" max="5" step="1" v-model.number="days" class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer accent-red-500">
      </div>
      <div>
        <div class="text-sm font-medium text-gray-700 mb-2">💰 {{ t('budget_label') }}</div>
        <div class="flex gap-2 flex-wrap">
          <button v-for="budgetItem in budgets" :key="budgetItem.value" @click="selectedBudget = budgetItem.value" 
            class="px-4 py-2 rounded-full text-sm font-medium transition-all" 
            :class="selectedBudget === budgetItem.value ? 'bg-red-500 text-white shadow-md' : 'bg-gray-100 text-gray-700'">
            {{ t(budgetItem.labelKey) }}
          </button>
        </div>
      </div>
      <div>
        <div class="text-sm font-medium text-gray-700 mb-2">🏝️ {{ t('interests_label') }}</div>
        <div class="flex flex-wrap gap-2">
          <button v-for="interest in interestsList" :key="interest.value" @click="toggleInterest(interest.value)"
            class="px-4 py-2 rounded-full text-sm font-medium transition-all"
            :class="selectedInterests.includes(interest.value) ? 'bg-red-500 text-white' : 'bg-gray-100 text-gray-700'">
            <i :class="interest.icon" class="mr-1"></i> {{ t(interest.labelKey) }}
          </button>
        </div>
      </div>
      <button @click="generateItinerary" class="w-full bg-gradient-to-r from-red-500 to-orange-500 text-white font-semibold py-3 rounded-xl shadow-md hover:shadow-lg transition-all flex items-center justify-center gap-2">
        <i class="fa-solid fa-wand-magic"></i> {{ t('generate_btn') }}
      </button>
      <div class="text-center text-xs text-gray-400 flex items-center justify-center gap-1">
        <i class="fa-regular fa-users"></i> {{ t('collab_mode') }}
      </div>
    </div>

    <!-- 🌟 熱門推薦欄（根據讚好數推薦） -->
    <div v-if="topRecommendedSpots.length" class="bg-white rounded-2xl card-shadow p-4">
      <div class="flex items-center gap-2 mb-3">
        <i class="fa-solid fa-fire-flame text-orange-500 text-lg"></i>
        <span class="font-bold text-gray-800">{{ t('hot_recommend_title') }}</span>
        <span class="text-xs text-gray-400">{{ t('hot_recommend_sub') }}</span>
      </div>
      <div class="flex gap-3 overflow-x-auto pb-2">
        <div v-for="spot in topRecommendedSpots" :key="spot.id" @click="openImageModal(spot.image)" class="flex-shrink-0 w-28 bg-gray-50 rounded-xl p-2 cursor-pointer hover:bg-gray-100 transition">
          <div class="w-24 h-24 rounded-lg bg-cover bg-center mx-auto" :style="{ backgroundImage: `url(${spot.image})`, backgroundSize: 'cover' }"></div>
          <p class="text-xs font-semibold text-center mt-1 truncate">{{ locale === 'zh' ? spot.name : spot.name_en }}</p>
          <div class="flex justify-center items-center gap-1 text-xs text-amber-600">
            <i class="fa-regular fa-thumbs-up"></i> <span>{{ spot.likes }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 生成後內容 -->
    <div v-if="generatedPlan" class="space-y-4">
      <div class="bg-white rounded-2xl card-shadow overflow-hidden">
        <div class="bg-gradient-to-r from-red-50 to-amber-50 px-5 py-3 border-b border-amber-100 flex justify-between items-center">
          <div>
            <span class="font-bold text-gray-800"><i class="fa-regular fa-clock"></i> {{ t('your_itinerary') }}</span>
            <p class="text-xs text-gray-500 mt-0.5">{{ days }}{{ t('days_unit') }}｜{{ t(selectedBudgetLabelKey()) }}｜{{ t('interests_prefix') }}: {{ selectedInterests.map(i => t(interestLabelMap[i])).join(', ') || t('random_explore') }}</p>
          </div>
          <i class="fa-regular fa-heart text-red-400 text-xl"></i>
        </div>
        <div class="divide-y divide-gray-100">
          <div v-for="(dayPlan, idx) in generatedPlan.days" :key="idx" class="p-4">
            <div class="flex items-center gap-2 mb-3">
              <div class="w-8 h-8 rounded-full bg-red-100 text-red-600 flex items-center justify-center font-bold text-sm">D{{ idx+1 }}</div>
              <span class="font-semibold text-gray-800">{{ dayPlan.title }}</span>
              <span class="text-xs text-gray-400 ml-auto"><i class="fa-regular fa-compass"></i> {{ dayPlan.pois.length }} {{ t('attractions_unit') }}</span>
            </div>
            <div class="space-y-2">
              <div v-for="poi in dayPlan.pois" :key="poi.id" class="flex items-start gap-3 bg-gray-50 p-3 rounded-xl">
                <!-- 可點擊放大的圖片 -->
                <div @click="openImageModal(poi.image)" class="w-14 h-14 rounded-xl bg-cover bg-center flex-shrink-0 shadow-sm cursor-pointer hover:opacity-80 transition" :style="{ backgroundImage: `url(${poi.image})`, backgroundSize: 'cover' }"></div>
                <div class="flex-1">
                  <div class="flex justify-between items-start">
                    <h4 class="font-semibold text-gray-800">{{ locale === 'zh' ? poi.name : poi.name_en }}</h4>
                    <div class="flex items-center gap-1">
                      <button @click="likePoi(poi.id)" class="text-xs text-gray-400 hover:text-red-500"><i class="fa-regular fa-thumbs-up"></i> <span class="text-[10px]">{{ poi.likes || 0 }}</span></button>
                      <button @click="addComment(poi.id)" class="text-xs text-gray-400"><i class="fa-regular fa-comment"></i></button>
                    </div>
                  </div>
                  <p class="text-xs text-gray-500 mt-1"><i class="fa-regular fa-location-dot"></i> {{ locale === 'zh' ? poi.district : poi.district_en }} · 💰 {{ locale === 'zh' ? poi.costHint : poi.costHintEn }}</p>
                  <div class="flex gap-1 mt-1">
                    <span class="text-[10px] bg-white px-2 py-0.5 rounded-full shadow-sm">{{ locale === 'zh' ? poi.category : poi.category_en }}</span>
                    <span class="text-[10px] bg-amber-50 text-amber-700 px-2 py-0.5 rounded-full"><i class="fa-regular fa-clock"></i> {{ poi.duration }}h</span>
                  </div>
                </div>
              </div>
            </div>
            <div class="mt-3 flex gap-2">
              <button @click="navigateDay(idx)" class="text-xs flex-1 bg-gray-100 py-2 rounded-xl text-gray-700 flex items-center justify-center gap-1"><i class="fa-solid fa-route"></i> {{ t('nav_today') }}</button>
              <button @click="openMapForDay(idx)" class="text-xs flex-1 bg-red-50 text-red-600 py-2 rounded-xl flex items-center justify-center gap-1"><i class="fa-regular fa-map"></i> {{ t('view_map') }}</button>
            </div>
          </div>
        </div>
      </div>

      <div class="bg-white rounded-2xl card-shadow p-4 space-y-3">
        <div class="flex justify-between items-center">
          <span class="font-semibold text-gray-700"><i class="fa-regular fa-map"></i> {{ t('interactive_map') }}</span>
          <span class="text-xs bg-blue-50 px-2 py-1 rounded-full text-blue-600"><i class="fa-regular fa-users"></i> {{ t('collab_marker') }}</span>
        </div>
        <div id="miniMap" class="map-container w-full"></div>
        <p class="text-[11px] text-gray-400 text-center">{{ t('map_hint') }}</p>
        <div class="border-t border-gray-100 pt-3 mt-1">
          <div class="flex items-center gap-2">
            <i class="fa-regular fa-pen-to-square text-gray-400"></i>
            <input type="text" v-model="collabMessage" :placeholder="t('collab_placeholder')" class="flex-1 bg-gray-100 rounded-full py-2 px-4 text-sm">
            <button @click="postCollaboration" class="bg-red-100 text-red-600 px-4 py-2 rounded-full text-sm">{{ t('send_btn') }}</button>
          </div>
          <div class="mt-2 text-xs text-gray-500 max-h-24 overflow-y-auto space-y-1">
            <div v-for="msg in collaborationFeed" :key="msg.id" class="flex gap-2 items-start">
              <span class="font-medium text-gray-700 min-w-[60px]">{{ msg.user }}</span>
              <span>{{ msg.text }}</span>
            </div>
          </div>
        </div>
      </div>

      <div class="bg-white rounded-2xl card-shadow p-4">
        <div class="flex justify-between mb-2"><span class="font-semibold"><i class="fa-regular fa-images"></i> {{ t('gallery_title') }}</span><span class="text-[10px] text-gray-400">{{ t('gallery_sub') }}</span></div>
        <div class="flex gap-2 overflow-x-auto pb-2">
          <div v-for="insp in inspirationImages" :key="insp.id" @click="openImageModal(insp.url)" class="flex-shrink-0 w-24 h-24 rounded-xl bg-cover bg-center shadow-sm cursor-pointer hover:opacity-80 transition" :style="{ backgroundImage: `url(${insp.url})` }" :title="insp.title"></div>
        </div>
      </div>
    </div>

    <!-- 空狀態 -->
    <div v-else class="bg-white/60 rounded-2xl p-8 text-center text-gray-400 flex flex-col items-center gap-2">
      <i class="fa-regular fa-map-compass text-4xl"></i>
      <p class="text-sm">{{ t('empty_state') }}</p>
      <p class="text-xs">{{ t('empty_hint') }}</p>
    </div>
  </div>

  <!-- 圖片放大 Modal -->
  <div v-if="modalImage" class="image-modal" @click="closeImageModal">
    <img :src="modalImage" alt="Enlarged view">
  </div>
</div>

<script>
const { createApp, ref, onMounted, nextTick, computed } = Vue;

const translations = {
  zh: {
    app_name: "旅蹤香港", plan_title: "行程設定", collab_badge: "多人協作",
    days_label: "📅 旅遊日數", days_unit: "日", budget_label: "預算風格",
    interests_label: "興趣類型 (可多選)", generate_btn: "生成專屬行程", collab_mode: "多人協作模式 · 所有人可編輯及投票景點",
    your_itinerary: "你的個人化行程", interests_prefix: "興趣", random_explore: "隨機探索",
    attractions_unit: "個景點", nav_today: "當日導航", view_map: "查看地圖",
    interactive_map: "互動地圖 (香港主要景點)", collab_marker: "協作標記", map_hint: "點擊地圖標記可查看景點資訊｜多人協作時可「讚好」景點排序",
    collab_placeholder: "協作討論: 加個景點或建議...", send_btn: "發送", gallery_title: "香港瞬間 · 景點圖像廊", gallery_sub: "靈感參考",
    empty_state: "設定行程偏好，點擊「生成專屬行程」", empty_hint: "支援團隊協作，所有成員可即時更新景點喜好",
    budget_economy: "經濟型", budget_standard: "標準型", budget_luxury: "豪華型",
    interest_nature: "🌿 自然生態", interest_culture: "🎨 文化藝術", interest_food: "🍜 美食夜市", interest_family: "🎢 親子樂園", interest_urban: "🏙️ 都市地標",
    cost_economy: "經濟", cost_standard: "中等", cost_luxury: "高", cost_free: "免費",
    hot_recommend_title: "🔥 熱門推薦", hot_recommend_sub: "根據讚好數精選"
  },
  en: {
    app_name: "Wander Hong Kong", plan_title: "Trip Settings", collab_badge: "Collab Mode",
    days_label: "📅 Number of Days", days_unit: "D", budget_label: "Budget Style",
    interests_label: "Interests (multi-select)", generate_btn: "Generate My Trip", collab_mode: "Collaborative Mode · everyone can vote & edit",
    your_itinerary: "Your Personalized Itinerary", interests_prefix: "Interests", random_explore: "Random",
    attractions_unit: "attractions", nav_today: "Navigate Today", view_map: "View on Map",
    interactive_map: "Interactive Map (Hong Kong)", collab_marker: "collab pins", map_hint: "Click markers for details · Like attractions to influence ranking",
    collab_placeholder: "Discuss: add a spot or suggestion...", send_btn: "Post", gallery_title: "HK Moments · Photo Inspiration", gallery_sub: "inspiration",
    empty_state: "Set your preferences and generate a trip", empty_hint: "Team collaboration: real-time updates & likes",
    budget_economy: "Budget", budget_standard: "Standard", budget_luxury: "Luxury",
    interest_nature: "🌿 Nature", interest_culture: "🎨 Culture & Arts", interest_food: "🍜 Food & Night", interest_family: "🎢 Family Fun", interest_urban: "🏙️ City Landmarks",
    cost_economy: "Budget", cost_standard: "Mid-Range", cost_luxury: "Premium", cost_free: "Free",
    hot_recommend_title: "🔥 Hot Picks", hot_recommend_sub: "based on top likes"
  }
};

const HK_ATTRACTIONS = [
  { id: 1, name: "太平山頂", name_en: "Victoria Peak", district: "中西區", district_en: "Central & Western", category: "自然景觀", category_en: "Nature", costLevel: "中等", costHint: "中等", costHintEn: "Mid-Range", duration: 2, image: "https://i.postimg.cc/QtqJ31fV/IMG-2076.jpg", likes: 24 },
  { id: 2, name: "維多利亞港", name_en: "Victoria Harbour", district: "油尖旺", district_en: "Yau Tsim Mong", category: "都市地標", category_en: "Landmark", costLevel: "免費", costHint: "免費", costHintEn: "Free", duration: 1.5, image: "https://i.postimg.cc/L6Qv5k2R/IMG-2077.jpg", likes: 32 },
  { id: 3, name: "香港迪士尼樂園", name_en: "Hong Kong Disneyland", district: "大嶼山", district_en: "Lantau", category: "主題樂園", category_en: "Theme Park", costLevel: "高", costHint: "高", costHintEn: "Premium", duration: 8, image: "https://i.postimg.cc/m2g8TpfL/IMG-2078.webp", likes: 56 },
  { id: 4, name: "廟街夜市", name_en: "Temple Street Night Market", district: "油麻地", district_en: "Yau Ma Tei", category: "夜市/美食", category_en: "Night Market", costLevel: "經濟", costHint: "經濟", costHintEn: "Budget", duration: 2, image: "https://i.postimg.cc/YqCbtDKv/IMG-2079.webp", likes: 18 },
  { id: 5, name: "南丫島", name_en: "Lamma Island", district: "離島", district_en: "Outlying Islands", category: "自然景觀", category_en: "Nature", costLevel: "中等", costHint: "中等", costHintEn: "Mid-Range", duration: 4, image: "https://i.postimg.cc/tCvDJdp9/IMG-2080.webp", likes: 28 },
  { id: 6, name: "M+博物館", name_en: "M+ Museum", district: "西九龍", district_en: "West Kowloon", category: "藝術文化", category_en: "Art & Culture", costLevel: "中等", costHint: "中等", costHintEn: "Mid-Range", duration: 3, image: "https://i.postimg.cc/7YtXh1w0/IMG-2081.webp", likes: 21 },
  { id: 7, name: "大館", name_en: "Tai Kwun", district: "中環", district_en: "Central", category: "歷史建築", category_en: "Heritage", costLevel: "免費", costHint: "免費", costHintEn: "Free", duration: 2, image: "https://i.postimg.cc/yYpnxhsR/IMG-2082.webp", likes: 17 },
  { id: 8, name: "蘭桂坊", name_en: "Lan Kwai Fong", district: "中環", district_en: "Central", category: "夜生活", category_en: "Nightlife", costLevel: "中等", costHint: "中等", costHintEn: "Mid-Range", duration: 2, image: "https://i.postimg.cc/qMDQqc43/IMG-2083.webp", likes: 13 },
  { id: 9, name: "黃大仙祠", name_en: "Wong Tai Sin Temple", district: "黃大仙", district_en: "Wong Tai Sin", category: "宗教文化", category_en: "Religious", costLevel: "免費", costHint: "免費", costHintEn: "Free", duration: 1.5, image: "https://i.postimg.cc/hPZMv8cx/IMG-2084.webp", likes: 9 },
  { id: 10, name: "昂坪360", name_en: "Ngong Ping 360", district: "大嶼山", district_en: "Lantau", category: "纜車景點", category_en: "Cable Car", costLevel: "高", costHint: "高", costHintEn: "Premium", duration: 3, image: "https://i.postimg.cc/xTpg8Gn0/IMG-2085.webp", likes: 31 }
];

const interestMapping = { nature: [1,5,10], culture: [6,7,9,4], food_night: [4,8], family: [3], urban: [2,6,8] };
const budgetCostMap = { "經濟型": "經濟", "標準型": "中等", "豪華型": "高" };

createApp({
  setup() {
    const locale = ref('zh');
    const days = ref(2);
    const budgets = ref([
      { value: "經濟型", labelKey: "budget_economy" },
      { value: "標準型", labelKey: "budget_standard" },
      { value: "豪華型", labelKey: "budget_luxury" }
    ]);
    const selectedBudget = ref("標準型");
    const interestsList = ref([
      { value: "nature", labelKey: "interest_nature", icon: "fa-tree" },
      { value: "culture", labelKey: "interest_culture", icon: "fa-palette" },
      { value: "food_night", labelKey: "interest_food", icon: "fa-utensils" },
      { value: "family", labelKey: "interest_family", icon: "fa-child" },
      { value: "urban", labelKey: "interest_urban", icon: "fa-building" }
    ]);
    const selectedInterests = ref([]);
    const generatedPlan = ref(null);
    const collabMessage = ref("");
    const collaborationFeed = ref([
      { id: 1, user: "小美", text: "推薦南丫島海鮮～ / Recommend Lamma seafood!" },
      { id: 2, user: "阿強", text: "想多排一點夜景行程 / More night views plz" }
    ]);
    let feedId = 3;
    let mapInstance = null;
    let markersLayer = [];
    
    // 圖片放大 modal
    const modalImage = ref(null);
    const openImageModal = (imgUrl) => { modalImage.value = imgUrl; };
    const closeImageModal = () => { modalImage.value = null; };

    const inspirationImages = ref([
      { id: 1, url: "https://i.postimg.cc/L6Qv5k2R/IMG-2077.jpg", title: "維港 / Victoria Harbour" },
      { id: 2, url: "https://i.postimg.cc/QtqJ31fV/IMG-2076.jpg", title: "山頂 / The Peak" },
      { id: 3, url: "https://i.postimg.cc/YqCbtDKv/IMG-2079.webp", title: "廟街 / Temple Street" },
      { id: 4, url: "https://i.postimg.cc/hPZMv8cx/IMG-2084.webp", title: "黃大仙 / Wong Tai Sin" }
    ]);

    // 熱門推薦：根據讚好數取前4名（即時更新）
    const topRecommendedSpots = computed(() => {
      return [...HK_ATTRACTIONS].sort((a, b) => b.likes - a.likes).slice(0, 4);
    });

    const t = (key) => translations[locale.value]?.[key] || key;
    const setLanguage = (lang) => { 
      locale.value = lang; 
      if (generatedPlan.value) refreshLocalizedTitles();
    };
    
    const refreshLocalizedTitles = () => {
      if (!generatedPlan.value) return;
      generatedPlan.value.days.forEach((day, i) => {
        day.title = locale.value === 'zh' 
          ? `探索${i===0 ? "九龍半島" : (i===1 ? "港島風情" : "離島/新界")}之旅` 
          : `Day ${i+1} Adventure`;
      });
    };
    
    const selectedBudgetLabelKey = () => budgets.value.find(b => b.value === selectedBudget.value)?.labelKey || "budget_standard";
    const interestLabelMap = { nature: "interest_nature", culture: "interest_culture", food_night: "interest_food", family: "interest_family", urban: "interest_urban" };

    const toggleInterest = (val) => {
      const idx = selectedInterests.value.indexOf(val);
      idx === -1 ? selectedInterests.value.push(val) : selectedInterests.value.splice(idx, 1);
    };

    const generateItinerary = () => {
      let filtered = [...HK_ATTRACTIONS];
      if (selectedInterests.value.length) {
        let allowedIds = new Set();
        selectedInterests.value.forEach(interest => {
          (interestMapping[interest] || []).forEach(id => allowedIds.add(id));
        });
        if (allowedIds.size) filtered = filtered.filter(att => allowedIds.has(att.id));
      }
      const targetBudgetLevel = budgetCostMap[selectedBudget.value];
      if (targetBudgetLevel) {
        filtered = filtered.filter(att => att.costLevel === targetBudgetLevel || 
          (selectedBudget.value === "經濟型" && att.costLevel !== "高") || 
          (selectedBudget.value === "豪華型" && att.costLevel !== "經濟"));
      }
      if (filtered.length < days.value * 1.5) {
        const extra = HK_ATTRACTIONS.filter(a => !filtered.includes(a)).slice(0, days.value * 2);
        filtered = [...filtered, ...extra];
      }
      const shuffled = [...filtered].sort(() => 0.5 - Math.random());
      const dayCount = days.value;
      const daysPlan = [];
      let pointer = 0;
      for (let i = 0; i < dayCount; i++) {
        const poisPerDay = Math.min(3, Math.max(2, Math.ceil(shuffled.length / dayCount)));
        const dailyPois = shuffled.slice(pointer, pointer + poisPerDay);
        pointer += poisPerDay;
        if (dailyPois.length === 0) break;
        const poisWithExtras = dailyPois.map(poi => ({
          ...poi,
          costHint: poi.costHint,
          costHintEn: poi.costHintEn,
          likes: poi.likes,
          duration: poi.duration,
          image: poi.image,
          id: poi.id
        }));
        daysPlan.push({
          title: locale.value === 'zh' ? `探索${i===0 ? "九龍半島" : (i===1 ? "港島風情" : "離島/新界")}之旅` : `Day ${i+1} Adventure`,
          pois: poisWithExtras
        });
      }
      while (daysPlan.length < dayCount) {
        const fallback = HK_ATTRACTIONS.slice(0,2).map(p => ({...p, costHint: p.costHint, costHintEn: p.costHintEn, likes: p.likes, duration: p.duration, image: p.image}));
        daysPlan.push({ title: locale.value === 'zh' ? "深度慢活日" : "Leisurely Day", pois: fallback });
      }
      generatedPlan.value = { days: daysPlan };
      nextTick(() => initOrUpdateMap());
    };

    const initOrUpdateMap = () => {
      if (!generatedPlan.value) return;
      const allPois = [];
      generatedPlan.value.days.forEach(day => day.pois.forEach(poi => allPois.push(poi)));
      if (!mapInstance && document.getElementById("miniMap")) {
        mapInstance = L.map('miniMap').setView([22.3193, 114.169], 12);
        L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OSM</a>',
          subdomains: 'abcd',
          maxZoom: 18
        }).addTo(mapInstance);
      } else if (mapInstance) {
        markersLayer.forEach(marker => mapInstance.removeLayer(marker));
        markersLayer = [];
      }
      if (mapInstance) {
        const bounds = [];
        allPois.forEach(poi => {
          const displayName = locale.value === 'zh' ? poi.name : poi.name_en;
          const displayDistrict = locale.value === 'zh' ? poi.district : poi.district_en;
          const popupContent = `<b>${displayName}</b><br>${displayDistrict}<br>👍 ${poi.likes} ${locale.value === 'zh' ? '讚' : 'likes'}<br><img src="${poi.image}" width="80" style="border-radius:12px; margin-top:5px; object-fit:cover;">`;
          const marker = L.marker([poi.lat, poi.lng]).addTo(mapInstance).bindPopup(popupContent);
          markersLayer.push(marker);
          bounds.push([poi.lat, poi.lng]);
        });
        bounds.length ? mapInstance.fitBounds(bounds) : mapInstance.setView([22.3193, 114.169], 12);
      }
    };

    const navigateDay = (idx) => { 
      const firstPoi = generatedPlan.value.days[idx].pois[0];
      const msg = locale.value === 'zh' ? `導航開始: 前往 ${firstPoi.name}` : `Navigate to: ${firstPoi.name_en}`;
      alert(msg); 
    };
    
    const openMapForDay = (idx) => {
      const day = generatedPlan.value.days[idx];
      if (day?.pois.length && mapInstance) mapInstance.fitBounds(day.pois.map(p => [p.lat, p.lng]));
      else if (locale.value === 'zh') alert("地圖已聚焦當日景點");
      else alert("Map focused on today's attractions");
    };
    
    const likePoi = (poiId) => {
      for (let day of generatedPlan.value.days) {
        const found = day.pois.find(p => p.id === poiId);
        if (found) {
          found.likes = (found.likes || 0) + 1;
          // 同步更新主資料庫的讚數，讓熱門推薦即時更新
          const mainPoi = HK_ATTRACTIONS.find(p => p.id === poiId);
          if (mainPoi) mainPoi.likes = found.likes;
          const likeText = locale.value === 'zh' ? `讚好了 ${found.name} 👍` : `Liked ${found.name_en} 👍`;
          collaborationFeed.value.unshift({ id: feedId++, user: locale.value === 'zh' ? "旅伴" : "Traveller", text: likeText });
          if (collaborationFeed.value.length > 8) collaborationFeed.value.pop();
          break;
        }
      }
    };
    
    const addComment = (poiId) => {
      let poiName = "", poiNameEn = "";
      for (let day of generatedPlan.value.days) {
        const found = day.pois.find(p => p.id === poiId);
        if (found) { poiName = found.name; poiNameEn = found.name_en; break; }
      }
      const promptMsg = locale.value === 'zh' ? `對 ${poiName} 留下協作建議:` : `Leave a suggestion for ${poiNameEn}:`;
      const comment = prompt(promptMsg, locale.value === 'zh' ? "值得一訪!" : "Worth visiting!");
      if (comment?.trim()) {
        collaborationFeed.value.unshift({ id: feedId++, user: locale.value === 'zh' ? "協作者" : "Collaborator", text: `💬 ${locale.value === 'zh' ? poiName : poiNameEn}: ${comment}` });
      }
    };
    
    const postCollaboration = () => {
      if (collabMessage.value.trim()) {
        collaborationFeed.value.unshift({ id: feedId++, user: locale.value === 'zh' ? "我" : "Me", text: collabMessage.value });
        collabMessage.value = "";
      }
    };

    onMounted(() => {
      if (document.getElementById("miniMap")) {
        mapInstance = L.map('miniMap').setView([22.3193, 114.169], 12);
        L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', { subdomains: 'abcd', maxZoom: 18 }).addTo(mapInstance);
      }
    });

    return {
      locale, days, budgets, selectedBudget, interestsList, selectedInterests, generatedPlan,
      collabMessage, collaborationFeed, inspirationImages, modalImage, topRecommendedSpots,
      t, setLanguage, selectedBudgetLabelKey, interestLabelMap, toggleInterest, generateItinerary,
      navigateDay, openMapForDay, likePoi, addComment, postCollaboration,
      openImageModal, closeImageModal
    };
  }
}).mount('#app');
</script>
</body>
</html>
