<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import Chart from 'chart.js/auto'

// --- 1. Madras University UG Grade Conversion Rules ---
const GRADE_RULES = [
  { grade: 'O',  minMarks: 90, maxMarks: 100, point: 10.0, label: 'Outstanding' },
  { grade: 'A+', minMarks: 80, maxMarks: 89,  point: 9.0,  label: 'Excellent' },
  { grade: 'A',  minMarks: 75, maxMarks: 79,  point: 8.0,  label: 'Very Good' },
  { grade: 'B+', minMarks: 65, maxMarks: 74,  point: 7.0,  label: 'Good' },
  { grade: 'B',  minMarks: 50, maxMarks: 64,  point: 6.0,  label: 'Above Average' },
  { grade: 'C',  minMarks: 40, maxMarks: 49,  point: 5.0,  label: 'Average' },
  { grade: 'RA', minMarks: 0,  maxMarks: 39,  point: 0.0,  label: 'Re-appear / Fail' }
]

const CATEGORIES = ['Core', 'Elective', 'Allied', 'General/Skill']

// --- 2. Reactive State & LocalStorage (no pre-filled sample data) ---
function loadStoredData() {
  if (typeof window === 'undefined') return []
  try {
    const raw = localStorage.getItem('unimadras_cgpa_data')
    if (!raw) return []
    const parsed = JSON.parse(raw)
    return Array.isArray(parsed) ? parsed : []
  } catch (e) {
    console.warn('Could not load saved CGPA data, starting empty:', e)
    return []
  }
}

const semesters = ref(loadStoredData())
const activeSemIndex = ref(0)
const showInfoModal = ref(false)

// Target Simulator Inputs - left empty, no defaults
const targetCGPA = ref(null)
const remainingCredits = ref(null)

// Chart Instance
let chartInstance = null

// --- 3. Calculation Functions ---
function getGradeFromMarks(secured, max) {
  const s = Number(secured)
  const m = Number(max)

  if (!m || m <= 0 || isNaN(s) || isNaN(m)) {
    return { grade: 'RA', point: 0.0 }
  }

  let pct = (s / m) * 100
  pct = Math.min(100, Math.max(0, pct))

  for (const rule of GRADE_RULES) {
    if (pct >= rule.minMarks) return { grade: rule.grade, point: rule.point }
  }
  return { grade: 'RA', point: 0.0 }
}

function isRowInvalid(course) {
  const s = Number(course.securedMarks)
  const m = Number(course.maxMarks)
  const c = Number(course.credits)
  if (isNaN(s) || isNaN(m) || isNaN(c)) return true
  if (s < 0 || m <= 0 || c < 0) return true
  if (s > m) return true
  return false
}

function calculateSGPA(sem) {
  let totalPts = 0
  let totalCreds = 0
  sem.courses.forEach(c => {
    const creds = Math.max(0, Number(c.credits) || 0)
    const { point } = getGradeFromMarks(c.securedMarks, c.maxMarks)
    totalPts += creds * point
    totalCreds += creds
  })
  return totalCreds > 0 ? Number((totalPts / totalCreds).toFixed(2)) : 0
}

const computedSemesters = computed(() => {
  return semesters.value.map(sem => ({
    ...sem,
    sgpa: calculateSGPA(sem),
    totalCredits: sem.courses.reduce((acc, c) => acc + Math.max(0, Number(c.credits) || 0), 0)
  }))
})

const overallStats = computed(() => {
  let grandPts = 0
  let grandCreds = 0
  computedSemesters.value.forEach(sem => {
    sem.courses.forEach(c => {
      const creds = Math.max(0, Number(c.credits) || 0)
      const { point } = getGradeFromMarks(c.securedMarks, c.maxMarks)
      grandPts += creds * point
      grandCreds += creds
    })
  })
  const cgpa = grandCreds > 0 ? grandPts / grandCreds : 0
  const percentage = cgpa * 10.0 // Madras Univ Rule

  let classification = grandCreds > 0 ? 'Pass' : '—'
  if (grandCreds > 0) {
    if (cgpa >= 7.5) classification = 'First Class with Distinction'
    else if (cgpa >= 6.0) classification = 'First Class'
    else if (cgpa >= 5.0) classification = 'Second Class'
  }

  return {
    cgpa: cgpa.toFixed(2),
    totalCredits: grandCreds,
    percentage: percentage.toFixed(2),
    classification,
    hasData: grandCreds > 0
  }
})

const targetSimulationResult = computed(() => {
  const currentCGPA = Number(overallStats.value.cgpa)
  const currentCredits = Number(overallStats.value.totalCredits)
  const target = Number(targetCGPA.value)
  const rem = Number(remainingCredits.value)

  if (!target || !rem || rem <= 0) return null

  const totalFutureCredits = currentCredits + rem
  const neededTotalPts = target * totalFutureCredits
  const currentPts = currentCGPA * currentCredits
  const requiredPts = neededTotalPts - currentPts
  const requiredGPA = requiredPts / rem

  if (requiredGPA > 10.0) {
    return { status: 'impossible', value: requiredGPA.toFixed(2), msg: `Not achievable. Required GPA (${requiredGPA.toFixed(2)}) exceeds max 10.0.` }
  } else if (requiredGPA <= 0) {
    return { status: 'achieved', value: '0.00', msg: `Already achieved! Even with 0 GPA in future credits, your target is safe.` }
  } else {
    return { status: 'possible', value: requiredGPA.toFixed(2), msg: `You need a GPA of ${requiredGPA.toFixed(2)} in your remaining ${rem} credits.` }
  }
})

// --- 4. Actions & Management ---
function emptyCourse() {
  return {
    id: `c_${Date.now()}_${Math.random().toString(36).slice(2, 7)}`,
    name: '',
    category: 'Core',
    maxMarks: null,
    securedMarks: null,
    credits: null
  }
}

function addSemester() {
  const newSemNum = semesters.value.length + 1
  semesters.value.push({
    id: `sem_${Date.now()}`,
    name: `Semester ${newSemNum}`,
    courses: []
  })
  activeSemIndex.value = semesters.value.length - 1
}

function removeSemester(index) {
  if (semesters.value.length <= 1) return alert("Must have at least one semester!")
  semesters.value.splice(index, 1)
  activeSemIndex.value = Math.max(0, activeSemIndex.value - 1)
}

function addCourse(semIndex) {
  semesters.value[semIndex].courses.push(emptyCourse())
}

function removeCourse(semIndex, courseIndex) {
  semesters.value[semIndex].courses.splice(courseIndex, 1)
}

function resetData() {
  if (confirm("Reset all data? This will erase every semester, subject, and mark you've entered.")) {
    semesters.value = []
    activeSemIndex.value = 0
    targetCGPA.value = null
    remainingCredits.value = null
  }
}

// Watchers & Chart Rendering
watch(semesters, (val) => {
  try {
    localStorage.setItem('unimadras_cgpa_data', JSON.stringify(val))
  } catch (e) {
    console.warn('Could not save CGPA data:', e)
  }
  nextTick(updateChart)
}, { deep: true })

function updateChart() {
  const ctx = document.getElementById('sgpaTrendChart')
  if (!ctx) return
  const labels = computedSemesters.value.map(s => s.name)
  const data = computedSemesters.value.map(s => s.sgpa)

  if (chartInstance) chartInstance.destroy()

  chartInstance = new Chart(ctx, {
    type: 'line',
    data: {
      labels,
      datasets: [{
        label: 'SGPA Trend',
        data,
        borderColor: '#3b82f6',
        backgroundColor: 'rgba(59, 130, 246, 0.15)',
        fill: true,
        tension: 0.3,
        pointRadius: 6,
        pointBackgroundColor: '#3b82f6'
      }]
    },
    options: {
      responsive: true,
      scales: {
        y: { min: 0, max: 10, grid: { color: '#334155' }, ticks: { color: '#94a3b8' } },
        x: { grid: { color: '#334155' }, ticks: { color: '#94a3b8' } }
      },
      plugins: { legend: { display: false } }
    }
  })
}

onMounted(() => {
  nextTick(updateChart)
})
</script>

<template>
  <div class="app-container">
    <!-- Zone A: Global Header -->
    <header class="header">
      <div>
        <h1>University of Madras CGPA Calculator</h1>
        <p class="subtitle">Undergraduate (UG) 10-Point Scale Dashboard</p>
      </div>
      <button class="btn-info" @click="showInfoModal = true">
        <svg class="icon-info" viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <circle cx="12" cy="12" r="10"></circle>
          <line x1="12" y1="16" x2="12" y2="12"></line>
          <line x1="12" y1="8" x2="12.01" y2="8"></line>
        </svg>
        <span>Grading Rules</span>
      </button>
    </header>

    <!-- Zone A: Key Metrics Bar -->
    <section class="metrics-grid">
      <div class="metric-card primary">
        <span class="label">Cumulative CGPA</span>
        <span class="val">{{ overallStats.cgpa }} <small>/ 10.0</small></span>
      </div>
      <div class="metric-card">
        <span class="label">Total Credits Earned</span>
        <span class="val">{{ overallStats.totalCredits }}</span>
      </div>
      <div class="metric-card">
        <span class="label">Equivalent Percentage</span>
        <span class="val">{{ overallStats.percentage }}%</span>
      </div>
      <div class="metric-card">
        <span class="label">Class Classification</span>
        <span class="val badge">{{ overallStats.classification }}</span>
      </div>
    </section>

    <!-- Zone B: Main Workstation -->
    <div class="main-grid">
      <div class="card">
        <div v-if="computedSemesters.length === 0" class="empty-state">
          <p>No semesters yet.</p>
          <button class="btn" @click="addSemester">+ Add Your First Semester</button>
        </div>

        <template v-else>
          <div class="sem-tabs-header">
            <div class="tabs">
              <button 
                v-for="(sem, idx) in computedSemesters" 
                :key="sem.id"
                :class="['tab-btn', { active: activeSemIndex === idx }]"
                @click="activeSemIndex = idx">
                {{ sem.name }} <span class="tab-badge">SGPA: {{ sem.sgpa }}</span>
              </button>
            </div>
            <button class="btn btn-sm" @click="addSemester">+ Add Semester</button>
          </div>

          <div v-if="computedSemesters[activeSemIndex]" class="sem-content">
            <div v-if="semesters[activeSemIndex].courses.length === 0" class="empty-state small">
              <p>No subjects added to this semester yet.</p>
              <button class="btn btn-secondary" @click="addCourse(activeSemIndex)">+ Add Subject</button>
            </div>

            <template v-else>
              <div class="table-container">
                <table class="course-table">
                  <thead>
                    <tr>
                      <th>Subject Title</th>
                      <th>Category</th>
                      <th>Max Marks</th>
                      <th>Secured</th>
                      <th>Credits</th>
                      <th>Grade</th>
                      <th></th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="(course, cIdx) in semesters[activeSemIndex].courses"
                      :key="course.id"
                      :class="{ 'row-invalid': isRowInvalid(course) }">
                      <td><input v-model="course.name" type="text" placeholder="Course Name" /></td>
                      <td>
                        <select v-model="course.category">
                          <option v-for="cat in CATEGORIES" :key="cat" :value="cat">{{ cat }}</option>
                        </select>
                      </td>
                      <td><input v-model.number="course.maxMarks" type="number" min="1" placeholder="100" value=100 style="width: 65px;" /></td>
                      <td><input v-model.number="course.securedMarks" type="number" min="0" placeholder="0" style="width: 65px;" /></td>
                      <td><input v-model.number="course.credits" type="number" min="0" placeholder="0" style="width: 55px;" /></td>
                      <td>
                        <span class="grade-pill">
                          {{ getGradeFromMarks(course.securedMarks, course.maxMarks).grade }}
                          ({{ getGradeFromMarks(course.securedMarks, course.maxMarks).point.toFixed(1) }})
                        </span>
                        <span v-if="isRowInvalid(course)" class="row-warning" title="Check this row: marks/credits look invalid or incomplete">⚠️</span>
                      </td>
                      <td>
                        <button class="btn-del" @click="removeCourse(activeSemIndex, cIdx)">✕</button>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>

              <div class="sem-footer">
                <button class="btn btn-secondary" @click="addCourse(activeSemIndex)">+ Add Subject</button>
                <button class="btn btn-danger btn-sm" @click="removeSemester(activeSemIndex)">Delete This Semester</button>
              </div>
            </template>
          </div>
        </template>
      </div>

      <!-- Right Column: Simulator & Analytics -->
      <div class="sidebar">
        <!-- Zone C: Target CGPA Simulator -->
        <div class="card">
          <h3>🎯 Target CGPA Simulator</h3>
          <div class="sim-group">
            <label>Desired Overall CGPA</label>
            <input v-model.number="targetCGPA" type="number" step="0.1" min="0" max="10" placeholder="e.g. 8.5" />
          </div>
          <div class="sim-group">
            <label>Remaining Credits to Complete</label>
            <input v-model.number="remainingCredits" type="number" min="1" placeholder="e.g. 24" />
          </div>

          <div v-if="targetSimulationResult" :class="['sim-res', targetSimulationResult.status]">
            <strong>Result:</strong> {{ targetSimulationResult.msg }}
          </div>
        </div>

        <!-- Zone D: Visual Analytics Chart -->
        <div class="card">
          <h3>📈 SGPA Progression Trend</h3>
          <p v-if="computedSemesters.length === 0" class="empty-hint">Add a semester to see your progression chart.</p>
          <canvas v-show="computedSemesters.length > 0" id="sgpaTrendChart" height="180"></canvas>
        </div>

        <div class="card data-actions">
          <button class="btn btn-danger" @click="resetData">Reset All Data</button>
        </div>
      </div>
    </div>

    <!-- Info Modal -->
    <div v-if="showInfoModal" class="modal-overlay" @click.self="showInfoModal = false">
      <div class="modal-card">
        <h2>University of Madras UG Grading System</h2>
        <p>Conversion rules applied automatically based on percentage marks:</p>
        <table class="info-table">
          <thead>
            <tr><th>Marks Range</th><th>Grade</th><th>Points</th><th>Performance</th></tr>
          </thead>
          <tbody>
            <tr v-for="rule in GRADE_RULES" :key="rule.grade">
              <td>{{ rule.minMarks }}% - {{ rule.maxMarks }}%</td>
              <td><strong>{{ rule.grade }}</strong></td>
              <td>{{ rule.point.toFixed(1) }}</td>
              <td>{{ rule.label }}</td>
            </tr>
          </tbody>
        </table>
        <p class="modal-note">
          <strong>Formulas:</strong><br />
          • <strong>SGPA</strong> = Total Grade Points Earned ÷ Total Semester Credits<br />
          • <strong>Percentage</strong> = CGPA × 10.0
        </p>
        <button class="btn" @click="showInfoModal = false">Close</button>
      </div>
    </div>

    <!-- Zone E: Global Footer -->
    <footer class="app-footer">
      <div class="footer-content">
        <p>
          Built for <strong>University of Madras UG Students</strong> • Designed & Developed by 
          <a href="https://github.com/Pranav-MSK" target="_blank" rel="noopener noreferrer" class="author-link">
            Pranav M S Krishnan
          </a>
        </p>
        <div class="footer-links">
          <a href="https://github.com/Pranav-MSK/CGPACalculator" target="_blank" rel="noopener noreferrer" class="github-link">
            <svg class="icon-github" viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
              <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/>
            </svg>
            <span>GitHub Repo</span>
          </a>
        </div>
      </div>
    </footer>
  </div>
</template>

<style src="./style.css"></style>