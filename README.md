# psjsjsksys171-max.github.io
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>我的菜谱记录</title>

<!-- PWA manifest -->
<link rel="manifest" href="manifest.json" />
<meta name="theme-color" content="#4f46e5" />

<script src="https://cdn.tailwindcss.com"></script>

<style>
  body {
    font-family: -apple-system, system-ui, "Segoe UI", Roboto, "Helvetica Neue", "Noto Sans", Arial;
    padding: 16px;
    max-width: 720px;
    margin: auto;
  }
  input, textarea { width: 100%; padding: 12px; font-size: 16px; border-radius: 10px; border: 1px solid #ddd; }
  button { padding: 12px 16px; font-size: 15px; border-radius: 10px; line-height: 1.2; }
  pre { font-size: 14px; padding: 8px; background: #f8fafc; border-radius: 8px; white-space: pre-wrap; }
  .action-group { display:flex; gap:8px; flex-wrap:wrap; }
  /* 安装按钮样式 */
  #installWrap { position: fixed; bottom: 18px; left: 50%; transform: translateX(-50%); z-index: 999; }
  #installBtn { display: inline-flex; align-items:center; gap:8px; padding: 12px 16px; background: #4f46e5; color: #fff; border-radius: 999px; box-shadow: 0 6px 18px rgba(79,70,229,0.18); border: none; }
</style>
</head>
<body>

<h2 class="text-2xl font-bold mb-3">🍳 我的菜谱本</h2>

<input id="search" placeholder="🔍 按食材搜索，例如：猪肉" class="mb-4">

<div id="list" class="grid grid-cols-1 gap-4"></div>

<hr class="my-6">

<div class="p-4 border rounded-xl space-y-3 bg-gray-50 shadow-sm">
  <h3 class="text-lg font-semibold">
    <span id="formTitle">➕ 添加菜谱</span>
  </h3>
  <input id="name" placeholder="菜名（必填）">
  <textarea id="ingredients" rows="5" placeholder="食材（每行一个）"></textarea>
  <input id="link" placeholder="小红书 / 下厨房链接">
  <button onclick="saveRecipe()" class="bg-indigo-600 text-white w-full">💾 保存</button>
  <button onclick="cancelEdit()" id="cancelBtn" class="bg-gray-300 w-full hidden">取消编辑</button>
</div>

<hr class="my-6">

<h3 class="text-lg font-semibold mb-2">📦 菜篮子</h3>
<div id="basket" class="bg-gray-50 p-3 rounded-lg border"></div>
<button onclick="clearBasket()" class="bg-red-500 text-white mt-3 w-full">🗑️ 清空菜篮子</button>

<hr class="my-6">

<h3 class="text-lg font-semibold mb-2">📤 数据备份（多设备迁移）</h3>
<div class="flex flex-col gap-3">
  <button onclick="exportJSON()" class="bg-green-600 text-white w-full">⬇️ 导出 JSON</button>
  <label for="importFile" class="bg-gray-200 text-center py-3 rounded cursor-pointer">⬆️ 导入 JSON 文件</label>
  <input type="file" id="importFile" accept=".json" class="hidden" onchange="importJSON(event)">
</div>
<small class="text-gray-500">💡电脑导出 → 发到手机 → 导入即可恢复菜谱</small>

<!-- 安装提示按钮（默认隐藏，只有当beforeinstallprompt触发时显示） -->
<div id="installWrap" style="display:none">
  <button id="installBtn">⬇️ 将页面添加到主屏</button>
</div>

<script>
/********** PWA: 注册 Service Worker **********/
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('./service-worker.js')
    .then(() => console.log('Service Worker registered'))
    .catch(err => console.warn('SW register failed', err));
}

/********** beforeinstallprompt 处理（Android/Chrome） **********/
let deferredPrompt = null;
window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();
  deferredPrompt = e;
  const wrap = document.getElementById('installWrap');
  wrap.style.display = 'block';
});

document.getElementById('installBtn').addEventListener('click', async () => {
  if (!deferredPrompt) return;
  deferredPrompt.prompt();
  const choice = await deferredPrompt.userChoice;
  if (choice.outcome === 'accepted') {
    console.log('User accepted the install prompt');
  } else {
    console.log('User dismissed the install prompt');
  }
  deferredPrompt = null;
  document.getElementById('installWrap').style.display = 'none';
});

/********** 应用现有功能（从你当前逻辑复制） **********/
const KEY = "recipes";
const BASKET_KEY = "basket";
let editingId = null;

const loadRecipes = () => JSON.parse(localStorage.getItem(KEY) || "[]");
const saveRecipes = (data) => localStorage.setItem(KEY, JSON.stringify(data));
const loadBasket = () => JSON.parse(localStorage.getItem(BASKET_KEY) || "[]");
const saveBasket = (data) => localStorage.setItem(BASKET_KEY, JSON.stringify(data));

function render() {
  const recipes = loadRecipes();
  const basket = loadBasket();
  const searchKey = document.getElementById("search").value.trim();
  const listEl = document.getElementById("list");

  const filtered = searchKey
    ? recipes.filter(r => r.ingredients.some(i => i.includes(searchKey)))
    : recipes;

  listEl.innerHTML = filtered.map(r => `
    <div class="recipe p-4 border rounded-xl shadow-sm bg-white flex flex-col gap-2">
      <div class="flex justify-between items-center">
        <div>
          <div class="text-xl font-bold">${r.name}</div>
          ${r.link ? `<a target="_blank" href="${r.link}" class="text-blue-600 text-xs">${r.link}</a>` : ""}
        </div>
      </div>
      <pre>${r.ingredients.join("\n")}</pre>
      <div class="action-group">
        <button class="bg-indigo-600 text-white flex-1" onclick="addToBasket(${r.id})">🛒 加入菜篮</button>
        <button class="bg-yellow-500 text-white flex-1" onclick="editRecipe(${r.id})">✏️ 编辑</button>
        <button class="bg-red-500 text-white flex-1" onclick="deleteRecipe(${r.id})">🗑️ 删除</button>
      </div>
    </div>
  `).join("");

  document.getElementById("basket").innerHTML = basket.join(", ");
}

document.getElementById("search").oninput = render;

function saveRecipe() {
  const name = document.getElementById("name").value.trim();
  const rawIng = document.getElementById("ingredients").value;
  const link = document.getElementById("link").value.trim();
  if (!name) return alert("菜名不能为空");
  const ingredients = rawIng.split("\n").map(i => i.trim()).filter(i=>i);
  const list = loadRecipes();
  if (editingId) {
    const idx = list.findIndex(r => r.id === editingId);
    list[idx] = { id: editingId, name, ingredients, link };
    saveRecipes(list);
    editingId = null;
  } else {
    list.push({ id: Date.now(), name, ingredients, link });
    saveRecipes(list);
  }
  clearFields();
  render();
  updateFormUI(false);
}

function clearFields() { name.value=""; ingredients.value=""; link.value=""; }

function editRecipe(id) {
  const list = loadRecipes();
  const r = list.find(x => x.id === id);
  document.getElementById("name").value = r.name;
  document.getElementById("ingredients").value = r.ingredients.join("\n");
  document.getElementById("link").value = r.link;
  editingId = id;
  updateFormUI(true);
}
function cancelEdit(){ clearFields(); editingId = null; updateFormUI(false); }
function updateFormUI(isEdit){ document.getElementById("formTitle").innerText = isEdit ? "✏️ 编辑菜谱" : "➕ 添加菜谱"; document.getElementById("cancelBtn").classList.toggle("hidden", !isEdit); }
function deleteRecipe(id){ saveRecipes(loadRecipes().filter(x => x.id !== id)); render(); }
function addToBasket(id){ const recipes = loadRecipes(); const r = recipes.find(x => x.id===id); const basket = loadBasket(); saveBasket([...basket, ...r.ingredients]); render(); }
function clearBasket(){ saveBasket([]); render(); }
function exportJSON(){ const data={recipes:loadRecipes(), basket:loadBasket()}; const blob=new Blob([JSON.stringify(data,null,2)],{type:"application/json"}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download="recipes_backup.json"; a.click(); URL.revokeObjectURL(url); }
function importJSON(event){ const file = event.target.files[0]; if(!file) return; const reader = new FileReader(); reader.onload = e => { try { const data = JSON.parse(e.target.result); saveRecipes(data.recipes||[]); saveBasket(data.basket||[]); alert("导入成功！🎉"); render(); } catch { alert("文件格式不正确"); } }; reader.readAsText(file); }

render();
</script>

</body>
</html>
