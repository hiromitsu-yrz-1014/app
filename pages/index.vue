<template>
  <div class="min-h-screen bg-gray-50 p-6">
    <div class="flex justify-between items-center mb-6 bg-white p-4 rounded-xl shadow-sm">
      <div>
        <h1 class="text-2xl font-bold text-gray-800">建設リソース配置カレンダー</h1>
        <p class="text-sm text-gray-500">人員・車両の稼働状況一覧</p>
      </div>
      <div class="text-sm bg-blue-50 text-blue-700 px-4 py-2 rounded-lg font-semibold">
        対象期間: {{ currentWeekStart }} 〜 {{ currentWeekEnd }}
      </div>
    </div>

    <div class="bg-white rounded-xl shadow-sm overflow-hidden border border-gray-200 mb-8">
      <div class="overflow-x-auto">
        <table class="w-full border-collapse text-left min-w-[1000px]">
          <thead>
            <tr class="bg-gray-100 border-b border-gray-200">
              <th class="p-4 font-bold text-gray-700 w-64 border-r border-gray-200 sticky left-0 bg-gray-100">リソース名 / 日付</th>
              <th v-for="date in dateList" :key="date" 
                  class="p-3 font-bold text-center border-r border-gray-200 min-w-[100px]"
                  :class="isToday(date) ? 'bg-yellow-50 text-yellow-800 font-black' : 'text-gray-700'">
                {{ formatDate(date) }}
              </th>
            </tr>
          </thead>
          
          <tbody>
            <tr class="bg-blue-50/50"><td :colspan="dateList.length + 1" class="p-2 font-bold text-blue-800 text-sm border-b border-gray-200">■ 人員 (Staff)</td></tr>
            <tr v-for="member in staffList" :key="member.id" class="border-b border-gray-200 hover:bg-gray-50">
              <td class="p-4 border-r border-gray-200 sticky left-0 bg-white shadow-[2px_0_5px_rgba(0,0,0,0.05)]">
                <div class="font-bold text-gray-800">{{ member.name }}</div>
                <div class="text-xs text-gray-500">{{ member.role || '一般' }}</div>
              </td>
              <td v-for="date in dateList" :key="date" 
                  class="p-2 border-r border-gray-200 text-center align-middle h-16"
                  :class="{ 'bg-yellow-50/30': isToday(date) }">
                <div v-if="getAssignment('staff', member.id, date)" 
                     class="text-xs p-2 rounded font-semibold text-white shadow-sm"
                     :style="{ backgroundColor: getAssignment('staff', member.id, date).color || '#3B82F6' }">
                  {{ getAssignment('staff', member.id, date).projectName }}
                  <span v-if="getAssignment('staff', member.id, date).note" class="block text-[10px] opacity-80 border-t border-white/20 mt-1">
                    {{ getAssignment('staff', member.id, date).note }}
                  </span>
                </div>
              </td>
            </tr>

            <tr class="bg-green-50/50"><td :colspan="dateList.length + 1" class="p-2 font-bold text-green-800 text-sm border-b border-gray-200">■ 車両・重機 (Vehicles)</td></tr>
            <tr v-for="vehicle in vehicleList" :key="vehicle.id" class="border-b border-gray-200 hover:bg-gray-50">
              <td class="p-4 border-r border-gray-200 sticky left-0 bg-white shadow-[2px_0_5px_rgba(0,0,0,0.05)]">
                <div class="font-bold text-gray-800">{{ vehicle.name }}</div>
                <div class="text-xs text-gray-500">{{ vehicle.type }} <span v-if="vehicle.plate_number">({{ vehicle.plate_number }})</span></div>
              </td>
              <td v-for="date in dateList" :key="date" 
                  class="p-2 border-r border-gray-200 text-center align-middle h-16"
                  :class="{ 'bg-yellow-50/30': isToday(date) }">
                <div v-if="getAssignment('vehicle', vehicle.id, date)" 
                     class="text-xs p-2 rounded font-semibold text-white shadow-sm"
                     :style="{ backgroundColor: getAssignment('vehicle', vehicle.id, date).color || '#10B981' }">
                  {{ getAssignment('vehicle', vehicle.id, date).projectName }}
                  <span v-if="getAssignment('vehicle', vehicle.id, date).note" class="block text-[10px] opacity-80 border-t border-white/20 mt-1">
                    {{ getAssignment('vehicle', vehicle.id, date).note }}
                  </span>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div class="bg-gray-800 text-gray-200 p-6 rounded-xl shadow-inner font-mono text-xs">
      <h3 class="text-sm font-bold text-yellow-400 mb-3">【デバッグエリア】schedules テーブルの生データ確認</h3>
      <div v-if="scheduleList.length === 0" class="text-gray-400">データがありません。</div>
      <div v-else class="space-y-3">
        <div v-for="(s, index) in scheduleList" :key="index" class="border-b border-gray-700 pb-2">
          <div><span class="text-blue-400">データ {{ index + 1 }}:</span></div>
          <div class="pl-4">・<span class="text-green-400">work_date (生データ):</span> "{{ s.work_date }}"</div>
          <div class="pl-4">・<span class="text-purple-400">project_id:</span> {{ s.project_id }}</div>
          <div class="pl-4">・<span class="text-yellow-400">staff_id:</span> {{ s.staff_id || 'null' }} / <span class="text-yellow-400">vehicle_id:</span> {{ s.vehicle_id || 'null' }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
const supabase = useSupabaseClient()

const staffList = ref([])
const vehicleList = ref([])
const projectMap = ref(new Map())
const scheduleList = ref([])
const dateList = ref([])

const currentWeekStart = ref('')
const currentWeekEnd = ref('')

// 今日かどうかを判定する関数
const isToday = (dateStr) => {
  const todayStr = new Date().toISOString().split('T')[0]
  return dateStr === todayStr
}

// カレンダーに表示する日付リストの生成（2026-05-11 〜 2026-05-24 の2週間分に固定）
const generateDateList = () => {
  const dates = []
  // 生データ(5/15)が含まれるように、5月11日（月）から14日間分を生成
  const baseDate = new Date('2026-05-11')
  for (let i = 0; i < 14; i++) {
    const d = new Date(baseDate)
    d.setDate(baseDate.getDate() + i)
    const yyyy = d.getFullYear()
    const mm = String(d.getMonth() + 1).padStart(2, '0')
    const dd = String(d.getDate()).padStart(2, '0')
    dates.push(`${yyyy}-${mm}-${dd}`)
  }
  dateList.value = dates
  currentWeekStart.value = dates[0]
  currentWeekEnd.value = dates[dates.length - 1]
}

const formatDate = (dateStr) => {
  const d = new Date(dateStr)
  const week = ['日', '月', '火', '水', '木', '金', '土']
  return `${d.getMonth() + 1}/${d.getDate()}(${week[d.getDay()]})`
}

// 特定の日の配置データを取得する関数
const getAssignment = (type, id, date) => {
  const sched = scheduleList.value.find(s => {
    if (!s.work_date) return false
    
    // YYYY-MM-DDの形を取り出す
    const dbDateStr = s.work_date.split('T')[0].split(' ')[0]
    const matchDate = dbDateStr === date
    
    // 指定されたリソースタイプ（人員か車両か）に応じてIDを比較
    const matchId = type === 'staff' ? s.staff_id === id : s.vehicle_id === id
    
    return matchDate && matchId
  })

  if (!sched || !sched.project_id) return null

  const project = projectMap.value.get(sched.project_id)
  return {
    projectName: project ? project.name : '不明な現場',
    color: project ? project.color_code : '#9CA3AF',
    note: sched.note
  }
}

onMounted(async () => {
  generateDateList()

  const { data: staffData } = await supabase.from('staff').select('*').order('name')
  staffList.value = staffData || []

  const { data: vehicleData } = await supabase.from('vehicles').select('*').order('name')
  vehicleList.value = vehicleData || []

  const { data: projectData } = await supabase.from('projects').select('*')
  const pMap = new Map()
  if (projectData) {
    projectData.forEach(p => pMap.set(p.id, p))
  }
  projectMap.value = pMap

  const { data: scheduleData } = await supabase.from('schedules').select('*')
  scheduleList.value = scheduleData || []
})
</script>