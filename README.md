# 🎓 University of Madras UG CGPA Calculator & Goal Simulator

An interactive, privacy-focused web application designed for undergraduate (UG) students under the **University of Madras** 10-point grading system. Calculate semester SGPAs, overall degree CGPA, equivalent percentage, track academic progression visually, and simulate target GPA requirements.

![Vue 3](https://img.shields.io/badge/Vue.js-3.x-4fc08d?style=for-the-badge&logo=vuedotjs&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-6.x-646cff?style=for-the-badge&logo=vite&logoColor=white)
![Chart.js](https://img.shields.io/badge/Chart.js-4.x-ff6384?style=for-the-badge&logo=chartdotjs&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)

---

## 🚀 Key Features

* **Madras University Grading Standard:** Pre-configured with official UG grade point mappings (O, A+, A, B+, B, C, RA) and automatic class classification (*First Class with Distinction*, *First Class*, *Second Class*).
* **Dynamic Semester & Subject Manager:** Add, edit, or remove semesters and subjects on the fly with category tagging (*Core, Elective, Allied, General/Skill*).
* **Target CGPA Goal Simulator:** Calculate the exact average GPA needed over remaining credits to achieve a target overall CGPA, complete with feasibility detection ($> 10.0$ or already achieved).
* **Visual Analytics:** Real-time Chart.js line graph mapping SGPA performance progression across semesters.
* **100% Client-Side Privacy:** Automatic session syncing using browser `localStorage`—your academic data never leaves your device.
* **Responsive Dark / Light Interface:** System-aware theme adaptation built with custom CSS design tokens.

---

## 📊 Academic Grading Mappings

Calculations adhere strictly to the University of Madras Undergraduate 10-Point Scale:

| Percentage Range | Letter Grade | Grade Point | Performance Classification |
| :---: | :---: | :---: | :--- |
| **$90\% - 100\%$** | **O** | **10.0** | Outstanding |
| **$80\% - 89\%$** | **A+** | **9.0** | Excellent |
| **$75\% - 79\%$** | **A** | **8.0** | Very Good |
| **$65\% - 74\%$** | **B+** | **7.0** | Good |
| **$50\% - 64\%$** | **B** | **6.0** | Above Average |
| **$40\% - 49\%$** | **C** | **5.0** | Average |
| **$< 40\%$** | **RA** | **0.0** | Re-appear / Fail |

### Core Formulas Used
* **Semester SGPA:**
  $$\text{SGPA} = \frac{\sum (\text{Credits}_i \times \text{Grade Point}_i)}{\sum \text{Credits}_i}$$
* **Equivalent Percentage:**
  $$\text{Percentage} = \text{CGPA} \times 10.0$$

---

## 🛠️ Tech Stack

* **Frontend Framework:** Vue 3 Composition API (`<script setup>`)
* **Build Tool:** Vite
* **Data Visualization:** Chart.js
* **Styling:** Modular CSS3 with Custom Properties (Design Tokens)
* **Storage:** Browser `localStorage`

---

## 💻 Local Development & Setup

### Prerequisites
Ensure you have **Node.js** (v18 or higher) and **npm** installed on your machine.

### 1. Clone the Repository
```bash
git clone [https://github.com/Pranav-MSK/CGPACalculator.git](https://github.com/Pranav-MSK/CGPACalculator.git)
cd CGPACalculator
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Run Development Server
```bash
npm run dev
```

Open http://localhost:5173/ in your browser to view the application.

### 4. Build for Production
```bash
npm run build
```

## 📁 Project Directory Structure
```plaintext
CGPACalculator/
├── public/
│   └── favicon.png
├── src/
│   ├── App.vue          # Primary reactive dashboard component & logic
│   ├── main.js          # Vue app entry point
│   └── style.css        # Global CSS variables & layout tokens
├── index.html           # Main HTML document template
├── package.json         # Project dependencies & npm scripts
└── vite.config.js       # Vite bundler configuration
```

## 👤 Author
- Pranav M S Krishnan
- GitHub: [@Pranav-MSK](https://github.com/Pranav-MSK)

## 📄 License
This project is open-source and available under the MIT License.