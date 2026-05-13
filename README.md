<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>個案水量紀錄系統</title>

  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"></script>

  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      font-family: "Noto Sans TC", sans-serif;
    }

    body {
      background: #f4efe8;
      color: #4d443d;
      padding: 20px;
    }

    .container {
      max-width: 1200px;
      margin: auto;
    }

    .title {
      background: white;
      padding: 30px;
      border-radius: 25px;
      margin-bottom: 20px;
      box-shadow: 0 4px 15px rgba(0,0,0,0.05);
    }

    h1 {
      font-size: 36px;
      margin-bottom: 10px;
    }

    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
      gap: 20px;
      margin-bottom: 20px;
    }

    .card {
      background: white;
      padding: 25px;
      border-radius: 25px;
      box-shadow: 0 4px 15px rgba(0,0,0,0.05);
    }

    .card h2 {
      margin-bottom: 15px;
    }

    input {
      width: 100%;
      padding: 14px;
      border-radius: 15px;
      border: 1px solid #d6cec4;
      margin-bottom: 15px;
      font-size: 16px;
    }

    button {
      background: #7d6d5f;
      color: white;
      border: none;
      padding: 14px 20px;
      border-radius: 15px;
      cursor: pointer;
      font-size: 16px;
      transition: 0.3s;
    }

    button:hover {
      opacity: 0.9;
    }

    .upload-box {
      border: 2px dashed #c8beb3;
      border-radius: 25px;
      padding: 50px 20px;
      text-align: center;
      background: #faf7f3;
      cursor: pointer;
    }

    .upload-box:hover {
      background: #f1ebe4;
    }

    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 15px;
    }

    th {
      background: #f0ebe5;
      padding: 15px;
      text-align: left;
    }

    td {
      padding: 15px;
      border-top: 1px solid #eee;
    }

    .progress {
      width: 100%;
      height: 10px;
      background: #e5ddd4;
      border-radius: 999px;
      overflow: hidden;
    }

    .progress-bar {
      height: 100%;
      background: #7d6d5f;
    }

    .status {
      margin-top: 15px;
      padding: 15px;
      background: #f4efe8;
      border-radius: 15px;
    }

    canvas {
      margin-top: 20px;
    }

    .footer {
      margin-top: 20px;
      text-align: center;
      color: #8b8177;
      font-size: 14px;
    }
  </style>
</head>
<body>

<div class="container">

  <div class="title">
    <h1>個案水量紀錄系統</h1>
    <p>
      支援個案固定水量、OCR 紙本辨識、每月統計圖表、資料匯出與 GitHub Pages。
    </p>
  </div>

  <div class="grid">

    <div class="card">
      <h2>新增個案</h2>

      <input type="text" id="name" placeholder="個案姓名">
      <input type="number" id="water" placeholder="固定水量 (ml)">

      <button onclick="addClient()">儲存個案</button>
    </div>

    <div class="card">
      <h2>OCR 紙本辨識</h2>

      <label class="upload-box">
        <input type="file" id="imageUpload" hidden>
        <p>點擊上傳紙本照片</p>
        <p style="margin-top:10px;color:#888;">
          JPG / PNG
        </p>
      </label>

      <div class="status" id="ocrStatus">
        尚未辨識
      </div>

      <button style="margin-top:15px;" onclick="fakeOCR()">
        開始 OCR 辨識
      </button>
    </div>

  </div>

  <div class="card">
    <h2>個案今日水量</h2>

    <button onclick="exportCSV()" style="margin-bottom:20px;">
      匯出 CSV
    </button>

    <table>
      <thead>
        <tr>
          <th>個案</th>
          <th>固定水量</th>
          <th>目前水量</th>
          <th>達成率</th>
        </tr>
      </thead>

      <tbody id="clientTable"></tbody>
    </table>
  </div>

  <div class="card">
    <h2>OCR 歷史紀錄</h2>

    <table>
      <thead>
        <tr>
          <th>日期</th>
          <th>時間</th>
          <th>個案</th>
          <th>增加水量</th>
        </tr>
      </thead>

      <tbody id="historyTable"></tbody>
    </table>
  </div>

  <div class="card">
    <h2>每月統計圖表</h2>

    <canvas id="chart"></canvas>
  </div>

  <div class="footer">
    GitHub Pages 可直接部署｜手機與電腦皆可使用
  </div>

</div>

<script>

let clients = JSON.parse(localStorage.getItem('clients')) || [
  {
    name: '王奶奶',
    base: 1500,
    current: 1800
  },
  {
    name: '林爺爺',
    base: 1200,
    current: 900
  }
]

// 每筆 OCR 歷史紀錄
let historyRecords = JSON.parse(localStorage.getItem('historyRecords')) || []

function saveHistory() {
  localStorage.setItem('historyRecords', JSON.stringify(historyRecords))
}

function saveData() {
  localStorage.setItem('clients', JSON.stringify(clients))
}

function renderTable()
renderHistory()() {
  const table = document.getElementById('historyTable')

  if (!table) return

  table.innerHTML = ''

  historyRecords
    .slice()
    .reverse()
    .forEach(record => {
      table.innerHTML += `
        <tr>
          <td>${record.date}</td>
          <td>${record.time}</td>
          <td>${record.name}</td>
          <td>${record.amount} ml</td>
        </tr>
      `
    })
}

function renderTable() {
  const table = document.getElementById('clientTable')
  table.innerHTML = ''

  clients.forEach((client) => {
    const percent = Math.round((client.current / client.base) * 100)

    table.innerHTML += `
      <tr>
        <td>${client.name}</td>
        <td>${client.base} ml</td>
        <td>${client.current} ml</td>
        <td>
          <div class="progress">
            <div class="progress-bar" style="width:${percent}%"></div>
          </div>
          <div style="margin-top:8px;">${percent}%</div>
        </td>
      </tr>
    `
  })
}

function addClient() {
  const name = document.getElementById('name').value
  const water = document.getElementById('water').value

  if (!name || !water) {
    alert('請輸入完整資料')
    return
  }

  clients.push({
    name,
    base: Number(water),
    current: Number(water)
  })

  saveData()
  renderTable()
renderHistory()

  document.getElementById('name').value = ''
  document.getElementById('water').value = ''
}

async function fakeOCR() {
  const fileInput = document.getElementById('imageUpload')
  const file = fileInput.files[0]

  if (!file) {
    alert('請先上傳照片')
    return
  }

  document.getElementById('ocrStatus').innerHTML = 'OCR 辨識中，請稍候...'

  try {
    const result = await Tesseract.recognize(
      file,
      'chi_tra+eng',
      {
        tessedit_char_whitelist:
          '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ毫升MLml王奶奶林爺爺'
      }
    )

    const text = result.data.text

    const numbers = text.match(/\d+/g)

    if (!numbers || numbers.length === 0) {
      document.getElementById('ocrStatus').innerHTML = '找不到數字，請重新拍攝'
      return
    }

    // 一張照片多位個案自動辨識
    let updatedClients = []

    clients.forEach(client => {
      if (text.includes(client.name)) {
        const waterAmount = Number(numbers[0])

        client.current += waterAmount

        // 自動日期紀錄
        const now = new Date()

        historyRecords.push({
          name: client.name,
          amount: waterAmount,
          date: now.toLocaleDateString('zh-TW'),
          time: now.toLocaleTimeString('zh-TW')
        })

        updatedClients.push(`${client.name} +${waterAmount}ml`)
      }
    })

    saveHistory()

    document.getElementById('ocrStatus').innerHTML =
      `OCR 辨識成功：${updatedClients.join('、')}`

    saveData()
    renderTable()
renderHistory()
  }

  catch (error) {
    console.error(error)

    document.getElementById('ocrStatus').innerHTML = 'OCR 辨識失敗'
  }
}
}

function exportCSV() {
  let csv = '個案,固定水量,目前水量\n'

  clients.forEach(client => {
    csv += `${client.name},${client.base},${client.current}\n`
  })

  const blob = new Blob([csv], { type: 'text/csv' })
  const url = window.URL.createObjectURL(blob)

  const a = document.createElement('a')
  a.href = url
  a.download = '水量紀錄.csv'
  a.click()
}

renderTable()
renderHistory()

const ctx = document.getElementById('chart')

new Chart(ctx, {
  type: 'bar',
  data: {
    labels: ['1月', '2月', '3月', '4月', '5月'],
    datasets: [{
      label: '每月總水量',
      data: [42000, 38000, 46500, 52000, 49800],
      borderRadius: 10
    }]
  },
  options: {
    responsive: true
  }
})

</script>

</body>
</html>
