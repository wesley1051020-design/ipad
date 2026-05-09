<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Wesley 雲端平板同步系統</title>
    <style>
        :root {
            --primary: #1a73e8;
            --bg: #f1f3f4;
            --border: #dadce0;
            --avail-green: #e6f4ea;
            --sold-red: #fce8e6;
        }

        body { font-family: "Microsoft JhengHei", sans-serif; background: var(--bg); margin: 0; padding: 10px; }
        .container { max-width: 1200px; margin: 0 auto; background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
        
        .header { display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid var(--primary); padding-bottom: 10px; margin-bottom: 20px; }
        h2 { margin: 0; color: var(--primary); }

        /* 響應式表格 */
        .table-wrap { overflow-x: auto; }
        table { width: 100%; border-collapse: collapse; min-width: 800px; table-layout: fixed; }
        th, td { border: 1px solid var(--border); padding: 12px 5px; text-align: center; }
        th { background: #f8f9fa; font-size: 0.9rem; }
        
        .cell { cursor: pointer; height: 80px; transition: 0.2s; position: relative; }
        .cell:hover { opacity: 0.8; }
        
        /* 狀態顏色 */
        .avail { background: var(--avail-green); color: #137333; }
        .borrowed { background: var(--sold-red); color: #d93025; font-weight: bold; }
        
        .user-name { display: block; font-size: 0.95rem; }
        .course-name { display: block; font-size: 0.75rem; background: white; border-radius: 4px; padding: 2px; margin-top: 5px; border: 1px solid #ffccc7; }
        .status-tag { font-size: 0.7rem; color: #666; }

        /* 表單區 */
        .booking-form { margin-top: 30px; padding: 20px; background: #f8f9fa; border-radius: 8px; border: 1px solid var(--border); }
        .form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; }
        .input-group { display: flex; flex-direction: column; }
        label { font-weight: bold; margin-bottom: 5px; font-size: 0.9rem; }
        input, select { padding: 12px; border: 1px solid var(--border); border-radius: 8px; }

        .btn-confirm { 
            width: 100%; padding: 16px; background: var(--primary); color: white; 
            border: none; border-radius: 8px; font-size: 1.1rem; font-weight: bold; cursor: pointer; 
            margin-top: 15px;
        }

        /* 登入視窗 */
        #login-modal {
            display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.8);
            z-index: 999; justify-content: center; align-items: center;
        }
        .modal-card { background: white; padding: 30px; border-radius: 12px; width: 300px; text-align: center; }
    </style>
</head>
<body>

<div class="container">
    <div class="header">
        <h2>💻 平板雲端借用看板</h2>
        <button onclick="clearData()" style="font-size: 0.7rem; background:none; border:1px solid #ccc; cursor:pointer; color:#999;">重設數據</button>
    </div>

    <div class="table-wrap">
        <table>
            <thead>
                <tr>
                    <th style="width: 100px;">節次</th>
                    <th>週一</th><th>週二</th><th>週三</th><th>週四</th><th>週五</th><th>週六</th><th>週日</th>
                </tr>
            </thead>
            <tbody id="schedule-body">
                </tbody>
        </table>
    </div>

    <div class="booking-form">
        <h3 style="margin-top:0;">新增借用申請</h3>
        <div class="form-grid">
            <div class="input-group">
                <label>借用人</label>
                <input type="text" id="input-name" placeholder="輸入姓名">
            </div>
            <div class="input-group">
                <label>選擇節次 (點擊上方表格)</label>
                <input type="text" id="input-course" readonly style="background:#eee;">
            </div>
            <div class="input-group">
                <label>借用用途</label>
                <input type="text" id="input-desc" placeholder="例如：資訊課">
            </div>
        </div>
        <button class="btn-confirm" onclick="openLogin()">確認提交並儲存至雲端</button>
    </div>
</div>

<div id="login-modal">
    <div class="modal-card">
        <h3>身份驗證</h3>
        <input type="text" id="acc" placeholder="帳號" style="width:100%; margin-bottom:10px; padding:10px; box-sizing:border-box;">
        <input type="password" id="pass" placeholder="密碼" style="width:100%; margin-bottom:15px; padding:10px; box-sizing:border-box;">
        <button class="btn-confirm" style="padding:10px;" onclick="doSubmit()">登入並同步</button>
        <button onclick="document.getElementById('login-modal').style.display='none'" style="background:none; border:none; color:gray; cursor:pointer; margin-top:10px;">取消</button>
    </div>
</div>

<script>
    const periods = ["第一節", "第二節", "第三節", "第四節", "第五節", "第六節", "第七節"];
    let selectedKey = null;

    // 讀取「真實存檔」：從瀏覽器記憶體抓資料
    let db = JSON.parse(localStorage.getItem('tablet_db')) || {
        "1-4": { u: "Wesley", c: "資訊教育" }
    };

    function renderTable() {
        const body = document.getElementById('schedule-body');
        body.innerHTML = "";
        periods.forEach((p, pIdx) => {
            let row = `<tr><td style="background:#f8f9fa; font-weight:bold;">${p}</td>`;
            for (let dIdx = 0; dIdx < 7; dIdx++) {
                const key = `${dIdx}-${pIdx}`;
                if (db[key]) {
                    row += `<td class="cell borrowed">
                                <span class="user-name">${db[key].u}</span>
                                <span class="course-name">${db[key].c}</span>
                            </td>`;
                } else {
                    row += `<td class="cell avail" onclick="selectSlot('${key}', '${p}', ${dIdx})">
                                <span style="font-weight:bold;">✓</span><br>
                                <span class="status-tag">29台</span>
                            </td>`;
                }
            }
            row += `</tr>`;
            body.innerHTML += row;
        });
    }

    function selectSlot(key, pName, dIdx) {
        const weekDays = ["週一", "週二", "週三", "週四", "週五", "週六", "週日"];
        selectedKey = key;
        document.getElementById('input-course').value = weekDays[dIdx] + " " + pName;
        document.getElementById('input-desc').focus();
    }

    function openLogin() {
        if (!selectedKey || !document.getElementById('input-name').value) {
            alert('請先填寫姓名並點擊表格選擇時段！'); return;
        }
        document.getElementById('login-modal').style.display = 'flex';
    }

    function doSubmit() {
        const a = document.getElementById('acc').value;
        const p = document.getElementById('pass').value;

        if (a === "Wesley" && p === "ab1051020") {
            // 寫入資料
            db[selectedKey] = {
                u: document.getElementById('input-name').value,
                c: document.getElementById('input-desc').value
            };
            // 儲存至本地記憶體 (真實存檔)
            localStorage.setItem('tablet_db', JSON.stringify(db));
            
            alert('同步成功！數據已儲存。');
            document.getElementById('login-modal').style.display = 'none';
            renderTable();
        } else {
            alert('驗證失敗！');
        }
    }

    function clearData() {
        if(confirm('確定要清除所有存檔嗎？')) {
            localStorage.removeItem('tablet_db');
            location.reload();
        }
    }

    window.onload = renderTable;
</script>

</body>
</html>
