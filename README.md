    <!DOCTYPE html>
<html lang="tr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>🏭 Üretim Takip Sistemi - Supabase</title>
  
  <!-- PWA Meta Tags -->
  <meta name="theme-color" content="#667eea">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <meta name="apple-mobile-web-app-title" content="İş Takip">
  <link rel="manifest" href="manifest.json">
  <link rel="apple-touch-icon" href="icon-192.png">
  
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
  <style> *{box-sizing:border-box} body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,Helvetica,Arial,sans-serif} </style>
</head>
<body>
<div id="root"></div>

<script type="text/babel">
const { useState, useEffect, useRef, useMemo } = React;

/* ---------- Supabase Config ---------- */
const SUPABASE_URL = 'https://myyjfwqdsfohgjirgoqz.supabase.co';
const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Im15eWpmd3Fkc2ZvaGdqaXJnb3F6Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NjA0NjU0MzMsImV4cCI6MjA3NjA0MTQzM30.kQ4CLh9-ngs-MFcX8V-mNsOwfSZfGcV0nSn_dwZi-8s';

/* ---------- Supabase Helper ---------- */
async function supabaseFetch(endpoint, options = {}) {
  const url = `${SUPABASE_URL}/rest/v1/${endpoint}`;
  const headers = {
    'apikey': SUPABASE_KEY,
    'Authorization': `Bearer ${SUPABASE_KEY}`,
    'Content-Type': 'application/json',
    'Prefer': 'return=representation',
    ...options.headers
  };
  const response = await fetch(url, { ...options, headers });
  if (!response.ok) throw new Error(`Supabase error: ${response.statusText}`);
  return response.json();
}

// ✅ Ayarlar için Supabase helper fonksiyonları
async function loadFromSupabase(tableName) {
  try {
    const data = await supabaseFetch(`${tableName}?order=name.asc`);
    return (data || []).map(item => item.name);
  } catch(e) {
    console.error(`${tableName} yüklenemedi:`, e);
    return [];
  }
}

async function saveToSupabase(tableName, name, showDialog = null) {
  try {
    await supabaseFetch(tableName, {
      method: 'POST',
      body: JSON.stringify({ name: name.trim() })
    });
    return true;
  } catch(e) {
    console.error(`${tableName} kaydedilemedi:`, e);
    if(e.message.includes('duplicate') && showDialog) {
      showDialog({ message: 'Bu isim zaten mevcut!', type: 'warning', showCancel: false });
    }
    return false;
  }
}

async function deleteFromSupabase(tableName, name) {
  try {
    await supabaseFetch(`${tableName}?name=eq.${encodeURIComponent(name)}`, {
      method: 'DELETE'
    });
    return true;
  } catch(e) {
    console.error(`${tableName} silinemedi:`, e);
    return false;
  }
}

// ✅ Stok güncelleme ve log kaydetme
async function updateStock(productId, newQuantity, oldQuantity, jobName, action = 'azaltma') {
  try {
    const product = await supabaseFetch(`products?id=eq.${productId}`);
    if(!product || product.length === 0) throw new Error('Ürün bulunamadı');
    
    const prod = product[0];
    
    await supabaseFetch(`products?id=eq.${productId}`, {
      method: 'PATCH',
      body: JSON.stringify({ quantity: newQuantity })
    });
    
    await supabaseFetch('stock_logs', {
      method: 'POST',
      body: JSON.stringify({
        product_id: productId,
        barcode: prod.barcode,
        product_name: prod.product_name,
        old_quantity: oldQuantity,
        new_quantity: newQuantity,
        change: newQuantity - oldQuantity,
        action: action,
        user: 'Web',
        note: `İş: ${jobName}`
      })
    });
    
    return true;
  } catch(e) {
    console.error('Stok güncellenemedi:', e);
    return false;
  }
}

// ✅ İş kaydetme - TÜM VERİLER DAHİL
async function saveJobToSupabase(job) {
  try {
    const jobData = {
      job_id: String(job.id),
      customer_name: job.customerName || '',
      customer_phone: job.customerPhone || '',
      customer_address: job.customerAddress || '',
      product_type: job.productType || '',
      order_number: job.orderNumber || '',
      start_date: job.startDate || null,
      target_date: job.targetDate || null,
      status: job.status || 'active',
      completed_at: job.completedAt || null,
      cabinet_data: job.cabinet || null,
      cabinet_notes: job.cabinetNotes || '',
      cabinet_extras: job.cabinetExtras || [],
      supplier_orders: job.supplierOrders || {},
      lazer_orders: job.lazerOrders || {},
      tabs: job.tabs || [],
      created_by: 'Web',
      updated_at: new Date().toISOString()
    };

    const existing = await supabaseFetch(`jobs?job_id=eq.${job.id}`);
    
    if(existing && existing.length > 0) {
      await supabaseFetch(`jobs?job_id=eq.${job.id}`, {
        method: 'PATCH',
        body: JSON.stringify(jobData)
      });
    } else {
      await supabaseFetch('jobs', {
        method: 'POST',
        body: JSON.stringify(jobData)
      });
    }
    return true;
  } catch(e) {
    console.error('İş kaydedilemedi:', e);
    return false;
  }
}

// ✅ Ürün Ağacı - Supabase'e Kaydetme
async function saveProductTreeToSupabase(productType, items) {
  try {
    // Önce mevcut kayıtları sil
    await supabaseFetch(`product_tree_items?product_type=eq.${encodeURIComponent(productType)}`, {
      method: 'DELETE'
    });
    
    // Yeni kayıtları ekle
    if(items && items.length > 0) {
      for(const item of items) {
        await supabaseFetch('product_tree_items', {
          method: 'POST',
          body: JSON.stringify({
            product_type: productType,
            material_name: item.product,
            description: item.description || '',
            quantity: item.qty || 1,
            supplier: item.supplier,
            supplier_type: item.type || 'material'
          })
        });
      }
    }
    return true;
  } catch(e) {
    console.error('Ürün ağacı kaydedilemedi:', e);
    return false;
  }
}

// ✅ Ürün Ağacı - Supabase'den Yükleme
async function loadProductTreeFromSupabase(productType) {
  try {
    const data = await supabaseFetch(`product_tree_items?product_type=eq.${encodeURIComponent(productType)}`);
    return (data || []).map(item => ({
      id: item.id,
      product: item.material_name,
      description: item.description,
      qty: item.quantity,
      supplier: item.supplier,
      type: item.supplier_type
    }));
  } catch(e) {
    console.error('Ürün ağacı yüklenemedi:', e);
    return [];
  }
}

// ✅ Tüm Ürün Ağaçlarını Yükle
async function loadAllProductTreesFromSupabase() {
  try {
    const data = await supabaseFetch('product_tree_items?order=product_type.asc');
    const trees = {};
    
    (data || []).forEach(item => {
      if(!trees[item.product_type]) {
        trees[item.product_type] = [];
      }
      trees[item.product_type].push({
        id: item.id,
        product: item.material_name,
        description: item.description,
        qty: item.quantity,
        supplier: item.supplier,
        type: item.supplier_type
      });
    });
    
    return trees;
  } catch(e) {
    console.error('Ürün ağaçları yüklenemedi:', e);
    return {};
  }
}

// ✅ İş Akış Şablonları - Supabase'e Kaydetme
async function saveWorkflowTemplateToSupabase(productType, tasks) {
  try {
    // Önce mevcut kayıtları sil
    await supabaseFetch(`workflow_templates?product_type=eq.${encodeURIComponent(productType)}`, {
      method: 'DELETE'
    });
    
    // Yeni kayıtları ekle ve Supabase'den dönen id'leri al
    const savedTasks = [];
    if(tasks && tasks.length > 0) {
      for(const task of tasks) {
        const result = await supabaseFetch('workflow_templates', {
          method: 'POST',
          body: JSON.stringify({
            product_type: productType,
            task_text: task.text || task,
            task_order: task.order || 0
          })
        });
        // Supabase'den dönen id'yi kullan
        if(result && Array.isArray(result) && result.length > 0) {
          savedTasks.push({
            id: result[0].id,
            text: result[0].task_text || task.text || task,
            order: result[0].task_order || task.order || 0
          });
        } else if(result && !Array.isArray(result) && result.id) {
          // Tek bir obje döndüyse
          savedTasks.push({
            id: result.id,
            text: result.task_text || task.text || task,
            order: result.task_order || task.order || 0
          });
        } else {
          // Eğer result dönmezse, mevcut task'ı kullan
          savedTasks.push({
            id: task.id || Date.now(),
            text: task.text || task,
            order: task.order || 0
          });
        }
      }
    }
    return savedTasks.length > 0 ? savedTasks : true;
  } catch(e) {
    console.error('İş akış şablonu kaydedilemedi:', e);
    return false;
  }
}

// ✅ İş Akış Şablonları - Supabase'den Yükleme
async function loadWorkflowTemplateFromSupabase(productType) {
  try {
    const data = await supabaseFetch(`workflow_templates?product_type=eq.${encodeURIComponent(productType)}&order=task_order.asc`);
    return (data || []).map(item => ({
      id: item.id,
      text: item.task_text,
      order: item.task_order || 0
    }));
  } catch(e) {
    console.error('İş akış şablonu yüklenemedi:', e);
    return [];
  }
}

// ✅ Tüm İş Akış Şablonlarını Yükle
async function loadAllWorkflowTemplatesFromSupabase() {
  try {
    const data = await supabaseFetch('workflow_templates?order=product_type.asc,task_order.asc');
    const templates = {};
    
    (data || []).forEach(item => {
      if(!templates[item.product_type]) {
        templates[item.product_type] = [];
      }
      templates[item.product_type].push({
        id: item.id,
        text: item.task_text,
        order: item.task_order || 0
      });
    });
    
    return templates;
  } catch(e) {
    console.error('İş akış şablonları yüklenemedi:', e);
    return {};
  }
}

// İşleri Supabase'den yükle
async function loadJobsFromSupabase() {
  try {
    const data = await supabaseFetch('jobs?order=created_at.desc');
    if(!data) return [];
    
return data.map(j => ({
  id: j.job_id,  // ✅ String olarak bırak, parseInt yapma
      customerName: j.customer_name || '',
      customerPhone: j.customer_phone || '',
      customerAddress: j.customer_address || '',
      productType: j.product_type || '',
      orderNumber: j.order_number || '',
      startDate: j.start_date || '',
      targetDate: j.target_date || '',
      status: j.status || 'active',
      completedAt: j.completed_at || null,
      cabinet: j.cabinet_data || null,
      cabinetNotes: j.cabinet_notes || '',
      cabinetExtras: j.cabinet_extras || [],
      supplierOrders: j.supplier_orders || {},
      lazerOrders: j.lazer_orders || {},
      tabs: j.tabs || []
    }));
  } catch(e) {
    console.error('İşler yüklenemedi:', e);
    return [];
  }
}

/* ---------- Sabitler ---------- */
/* YENİ KOD */
const DEFAULT_PRODUCTS = ['Meze Dolabı','Tatlı Teşhir Dolabı','Kasap Dolabı','Dry Age Dolabı'];
const DEFAULT_TEAM = ['Ahmet','Mehmet','Ayşe'];
const DEFAULT_LAZER = ['Profesyonel','Özmetsan'];
const DEFAULT_SUPPLIERS = ['Cantaş','Karataş','Buzçelik'];

const DEFAULT_TABS = [
  { id: 'tab-genel', title: '📊 Genel Bilgiler', tasks: [], deletable: false },
  { id: 'tab-lazer', title: '⚡ Lazer & Kesim', tasks: [], deletable: true },
  { id: 'tab-malzeme', title: '📦 Malzeme Tedarik', tasks: [], deletable: true },
  { id: 'tab-boya', title: '🎨 Boya & Kaplama', tasks: [], deletable: true },
  { id: 'tab-planlanan', title: '📋 Yapılan ve Planlanan İşler', tasks: [], deletable: false }
];

/* ---------- İkon Helper ---------- */
function Icon({ type, className="w-5 h-5" }){
  const icons = {
    plus: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 4v16m8-8H4"/></svg>,
    trash: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/></svg>,
    edit: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"/></svg>,
    info: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M13 16h-1v-4h-1m1-4h.01M12 2a10 10 0 100 20 10 10 0 000-20z"/></svg>,
    right: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 5l7 7-7 7"/></svg>,
    users: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M17 20h5v-2a4 4 0 00-3-3.87M9 20H4v-2a4 4 0 013-3.87M12 11a4 4 0 100-8 4 4 0 000 8z"/></svg>,
    download: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 3v12m0 0l4-4m-4 4l-4-4M21 21H3" /></svg>,
    search: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/></svg>,
    box: <svg className={className} fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10M4 7v10l8 4"/></svg>
  };
  return icons[type] || null;
}

/* ---------- Custom Dialog Component ---------- */
function CustomDialog({ message, onConfirm, onCancel, showCancel = true, type = 'info' }) {
  const typeConfig = {
    info: { bg: 'bg-blue-100', icon: 'text-blue-600', button: 'bg-blue-600 hover:bg-blue-700', iconPath: 'M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z' },
    success: { bg: 'bg-green-100', icon: 'text-green-600', button: 'bg-green-600 hover:bg-green-700', iconPath: 'M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z' },
    error: { bg: 'bg-red-100', icon: 'text-red-600', button: 'bg-red-600 hover:bg-red-700', iconPath: 'M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z' },
    warning: { bg: 'bg-yellow-100', icon: 'text-yellow-600', button: 'bg-yellow-600 hover:bg-yellow-700', iconPath: 'M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z' }
  };
  
  const config = typeConfig[type] || typeConfig.info;
  
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-[9999] p-4">
      <div className="bg-white rounded-lg shadow-2xl max-w-md w-full animate-fadeIn">
        <div className="p-6">
          <div className="flex items-start gap-3 mb-4">
            <div className={`w-10 h-10 rounded-full ${config.bg} flex items-center justify-center flex-shrink-0`}>
              <svg className={`w-6 h-6 ${config.icon}`} fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d={config.iconPath} />
              </svg>
            </div>
            <div className="flex-1">
              <p className="text-gray-800 whitespace-pre-line leading-relaxed">{message}</p>
            </div>
          </div>
        </div>
        <div className="bg-gray-50 px-6 py-3 rounded-b-lg flex gap-3 justify-end">
          {showCancel && (
            <button 
              onClick={onCancel}
              className="px-4 py-2 bg-white border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-100 transition"
            >
              İptal
            </button>
          )}
          <button 
            onClick={() => {
              if(onConfirm) onConfirm();
              if(onCancel && !showCancel) onCancel();
            }}
            className={`px-4 py-2 ${config.button} text-white rounded-lg transition`}
          >
            {showCancel ? 'Tamam' : 'Kapat'}
          </button>
        </div>
      </div>
      <style jsx>{`
        @keyframes fadeIn {
          from { opacity: 0; transform: scale(0.95); }
          to { opacity: 1; transform: scale(1); }
        }
        .animate-fadeIn {
          animation: fadeIn 0.15s ease-out;
        }
      `}</style>
    </div>
  );
}

/* ---------- LocalStorage helpers ---------- */
const LS_KEYS = { 
  jobs: 'pts_jobs_sup_v2', 
  products: 'pts_products_sup_v2', 
  team: 'pts_team_sup_v2', 
  orderSeq: 'pts_orderseq_sup_v2',
  lazerSuppliers: 'pts_lazer_sup_v2'
};
function lsLoad(key, fallback) { try { const raw = localStorage.getItem(key); return raw ? JSON.parse(raw) : fallback; } catch { return fallback; } }
function lsSave(key, data) { try { localStorage.setItem(key, JSON.stringify(data)); } catch(e){ console.warn('LS save failed', e); } }

/* ---------- Order no generator ---------- */
function nextOrderNumber(jobs) {
  const seqStored = lsLoad(LS_KEYS.orderSeq, null);
  if(typeof seqStored === 'number') {
    const next = seqStored + 1;
    lsSave(LS_KEYS.orderSeq, next);
    return 'BEGA' + String(next).padStart(3, '0');
  }
  let max = 0;
  (jobs || []).forEach(j => {
    const m = (j.orderNumber || '').match(/BEGA0*([0-9]+)/i);
    if(m && m[1]) {
      const n = parseInt(m[1],10);
      if(n > max) max = n;
    }
  });
  const next = max + 1 || 1;
  lsSave(LS_KEYS.orderSeq, next);
  return 'BEGA' + String(next).padStart(3, '0');
}

/* ---------- Excel Styling Helper ---------- */
function applyExcelStyles(ws, data) {
  // Header satırını kalın yap ve merkeze hizala
  const headerRow = 0;
  const lastRow = data.length - 1;
  const lastCol = data[0] ? data[0].length - 1 : 0;
  
  // Header için stil
  for(let col = 0; col <= lastCol; col++) {
    const cellRef = XLSX.utils.encode_cell({ r: headerRow, c: col });
    if(!ws[cellRef]) continue;
    
    ws[cellRef].s = {
      font: { bold: true, sz: 12, color: { rgb: "FFFFFF" } },
      fill: { fgColor: { rgb: "4472C4" } }, // Mavi arka plan
      alignment: { horizontal: "center", vertical: "center", wrapText: true },
      border: {
        top: { style: "thin", color: { rgb: "000000" } },
        bottom: { style: "thin", color: { rgb: "000000" } },
        left: { style: "thin", color: { rgb: "000000" } },
        right: { style: "thin", color: { rgb: "000000" } }
      }
    };
  }
  
  // Veri satırlarına border ekle
  for(let row = 1; row <= lastRow; row++) {
    for(let col = 0; col <= lastCol; col++) {
      const cellRef = XLSX.utils.encode_cell({ r: row, c: col });
      if(!ws[cellRef]) continue;
      
      if(!ws[cellRef].s) ws[cellRef].s = {};
      ws[cellRef].s.border = {
        top: { style: "thin", color: { rgb: "CCCCCC" } },
        bottom: { style: "thin", color: { rgb: "CCCCCC" } },
        left: { style: "thin", color: { rgb: "CCCCCC" } },
        right: { style: "thin", color: { rgb: "CCCCCC" } }
      };
      ws[cellRef].s.alignment = { vertical: "center", wrapText: true };
      
      // Sayı sütunlarını sağa hizala
      if(col === 3) { // Adet sütunu
        ws[cellRef].s.alignment.horizontal = "right";
      }
    }
  }
  
  // Satır yükseklikleri
  ws['!rows'] = [];
  ws['!rows'][0] = { hpt: 25 }; // Header yüksekliği
  for(let i = 1; i <= lastRow; i++) {
    ws['!rows'][i] = { hpt: 20 };
  }
}

/* ---------- XLSX export helpers ---------- */
function exportSupplierXlsx(supplierName, items, job) {
  const wb = XLSX.utils.book_new();
  
  // Başlık ve bilgi satırları
  const aoa = [
    [`${supplierName} - Sipariş Listesi`],
    [''],
    ['Sipariş No:', job.orderNumber || '-'],
    ['Müşteri:', job.customerName || '-'],
    ['Tarih:', new Date().toLocaleDateString('tr-TR')],
    [''],
    ['Tedarikçi','Ürün','Açıklama','Adet','Sipariş No','Müşteri']
  ];
  
  items.forEach(it => {
    aoa.push([supplierName, it.product || '', it.description || '', it.qty || 1, job.orderNumber || '', job.customerName || '']);
  });
  
  const ws = XLSX.utils.aoa_to_sheet(aoa);
  ws['!cols'] = [{wch:18},{wch:30},{wch:35},{wch:10},{wch:15},{wch:25}];
  
  // Stil uygula (sadece tablo kısmına)
  const headerRowIndex = 6; // Header satırı
  const lastRow = aoa.length - 1;
  const lastCol = 5;
  
  // Header stil
  for(let col = 0; col <= lastCol; col++) {
    const cellRef = XLSX.utils.encode_cell({ r: headerRowIndex, c: col });
    if(ws[cellRef]) {
      ws[cellRef].s = {
        font: { bold: true, sz: 11, color: { rgb: "FFFFFF" } },
        fill: { fgColor: { rgb: "4472C4" } },
        alignment: { horizontal: "center", vertical: "center", wrapText: true },
        border: {
          top: { style: "thin" }, bottom: { style: "thin" },
          left: { style: "thin" }, right: { style: "thin" }
        }
      };
    }
  }
  
  // Veri satırları border
  for(let row = headerRowIndex + 1; row <= lastRow; row++) {
    for(let col = 0; col <= lastCol; col++) {
      const cellRef = XLSX.utils.encode_cell({ r: row, c: col });
      if(ws[cellRef]) {
        if(!ws[cellRef].s) ws[cellRef].s = {};
        ws[cellRef].s.border = {
          top: { style: "thin" }, bottom: { style: "thin" },
          left: { style: "thin" }, right: { style: "thin" }
        };
        ws[cellRef].s.alignment = { vertical: "center", wrapText: true };
        if(col === 3) ws[cellRef].s.alignment.horizontal = "right";
      }
    }
  }
  
  // Başlık stil
  const titleRef = XLSX.utils.encode_cell({ r: 0, c: 0 });
  if(ws[titleRef]) {
    ws[titleRef].s = {
      font: { bold: true, sz: 14, color: { rgb: "1F4E78" } },
      alignment: { horizontal: "left", vertical: "center" }
    };
  }
  
  ws['!rows'] = [{ hpt: 25 }, { hpt: 15 }, { hpt: 18 }, { hpt: 18 }, { hpt: 18 }, { hpt: 15 }, { hpt: 25 }];
  for(let i = 7; i <= lastRow; i++) {
    ws['!rows'][i] = { hpt: 20 };
  }
  
  XLSX.utils.book_append_sheet(wb, ws, supplierName.slice(0, 31));
  const filename = `${supplierName.replace(/\s+/g,'_')}_${job.orderNumber || job.id}.xlsx`;
  XLSX.writeFile(wb, filename);
}

function exportAllSuppliersXlsx(ordersBySupplier, job, sheetName = 'Malzeme') {
  const wb = XLSX.utils.book_new();
  
  for(const s of Object.keys(ordersBySupplier)) {
    const items = ordersBySupplier[s];
    const aoa = [
      [`${s} - Sipariş Listesi`],
      [''],
      ['Sipariş No:', job.orderNumber || '-'],
      ['Müşteri:', job.customerName || '-'],
      ['Tarih:', new Date().toLocaleDateString('tr-TR')],
      [''],
      ['Tedarikçi','Ürün','Açıklama','Adet','Sipariş No','Müşteri']
    ];
    
    items.forEach(it => aoa.push([s, it.product || '', it.description || '', it.qty || 1, job.orderNumber || '', job.customerName || '']));
    
    const ws = XLSX.utils.aoa_to_sheet(aoa);
    ws['!cols'] = [{wch:18},{wch:30},{wch:35},{wch:10},{wch:15},{wch:25}];
    
    // Stil uygula
    const headerRowIndex = 6;
    const lastRow = aoa.length - 1;
    const lastCol = 5;
    
    // Başlık
    const titleRef = XLSX.utils.encode_cell({ r: 0, c: 0 });
    if(ws[titleRef]) {
      ws[titleRef].s = {
        font: { bold: true, sz: 14, color: { rgb: "1F4E78" } },
        alignment: { horizontal: "left", vertical: "center" }
      };
    }
    
    // Header
    for(let col = 0; col <= lastCol; col++) {
      const cellRef = XLSX.utils.encode_cell({ r: headerRowIndex, c: col });
      if(ws[cellRef]) {
        ws[cellRef].s = {
          font: { bold: true, sz: 11, color: { rgb: "FFFFFF" } },
          fill: { fgColor: { rgb: "4472C4" } },
          alignment: { horizontal: "center", vertical: "center", wrapText: true },
          border: {
            top: { style: "thin" }, bottom: { style: "thin" },
            left: { style: "thin" }, right: { style: "thin" }
          }
        };
      }
    }
    
    // Veri satırları
    for(let row = headerRowIndex + 1; row <= lastRow; row++) {
      for(let col = 0; col <= lastCol; col++) {
        const cellRef = XLSX.utils.encode_cell({ r: row, c: col });
        if(ws[cellRef]) {
          if(!ws[cellRef].s) ws[cellRef].s = {};
          ws[cellRef].s.border = {
            top: { style: "thin" }, bottom: { style: "thin" },
            left: { style: "thin" }, right: { style: "thin" }
          };
          ws[cellRef].s.alignment = { vertical: "center", wrapText: true };
          if(col === 3) ws[cellRef].s.alignment.horizontal = "right";
        }
      }
    }
    
    // Satır yükseklikleri
    ws['!rows'] = [{ hpt: 25 }, { hpt: 15 }, { hpt: 18 }, { hpt: 18 }, { hpt: 18 }, { hpt: 15 }, { hpt: 25 }];
    for(let i = 7; i <= lastRow; i++) {
      ws['!rows'][i] = { hpt: 20 };
    }
    
    XLSX.utils.book_append_sheet(wb, ws, s.slice(0,31));
  }
  
  const filename = `${sheetName}_${job.orderNumber || job.id}.xlsx`;
  XLSX.writeFile(wb, filename);
}

/* ---------- Application ---------- */
function ProductionTracker(){
  // ✅ TÜM STATE TANIMLARI BURADA
  const [products, setProducts] = useState([]);
  const [teamMembers, setTeamMembers] = useState([]);
  const [lazerSuppliers, setLazerSuppliers] = useState([]);
  const [suppliers, setSuppliers] = useState([]);
  const [productTrees, setProductTrees] = useState({}); // ✅ YENİ EKLEME
  const [workflowTemplates, setWorkflowTemplates] = useState({}); // ✅ İş akış şablonları
  const [loadingSettings, setLoadingSettings] = useState(false);
  const [jobs, setJobs] = useState([]);
  const [view, setView] = useState('jobs');
  const [expandedCustomers, setExpandedCustomers] = useState({}); // Müşteri açık/kapalı durumu
  const [selectedJobId, setSelectedJobId] = useState(null);
  const [newProduct, setNewProduct] = useState('');
  const [newMember, setNewMember] = useState('');
  const [newLazerSupplier, setNewLazerSupplier] = useState('');
  const [newSupplier, setNewSupplier] = useState(''); // ✅ EKLE
  const [jobFilter, setJobFilter] = useState('');
  const [showCreateModal, setShowCreateModal] = useState(false);
  const [formData, setFormData] = useState({
    customerName: '',
    customerPhone: '',
    customerAddress: '',
    productType: products[0] || '',
    productCustom: '',
    startDate: new Date().toISOString().split('T')[0],
    targetDate: ''
  });
  const [loadingJobs, setLoadingJobs] = useState(false);
  const [showDeleteConfirm, setShowDeleteConfirm] = useState(null);
  const [dialog, setDialog] = useState(null); // ✅ Dialog state: { message, type, onConfirm, onCancel, showCancel }
  // ✅ EKİP ÜYESİ FONKSİYONLARI
  const addTeamMember = async (name) => {
    if(!name.trim()) return;
    const success = await saveToSupabase('team_members', name, setDialog);
    if(success) {
      setTeamMembers(prev => [...prev, name.trim()]);
    }
  };

  const deleteTeamMember = async (name) => {
    setDialog({
      message: `${name} silinsin mi?`,
      type: 'warning',
      showCancel: true,
      onConfirm: async () => {
        const success = await deleteFromSupabase('team_members', name);
        if(success) {
          setTeamMembers(prev => prev.filter(m => m !== name));
          setDialog(null);
        }
      },
      onCancel: () => setDialog(null)
    });
  };

  // ✅ ÜRÜN TİPİ FONKSİYONLARI
  const addProduct = async (name) => {
    if(!name.trim()) return;
    const success = await saveToSupabase('product_types', name, setDialog);
    if(success) {
      setProducts(prev => [...prev, name.trim()]);
    }
  };

  const deleteProduct = async (name) => {
    setDialog({
      message: `${name} silinsin mi?`,
      type: 'warning',
      showCancel: true,
      onConfirm: async () => {
        const success = await deleteFromSupabase('product_types', name);
        if(success) {
          setProducts(prev => prev.filter(p => p !== name));
          setDialog(null);
        }
      },
      onCancel: () => setDialog(null)
    });
  };

  // ✅ LAZER FİRMASI FONKSİYONLARI
  const addLazerSupplier = async (name) => {
    if(!name.trim()) return;
    const success = await saveToSupabase('lazer_suppliers', name, setDialog);
    if(success) {
      setLazerSuppliers(prev => [...prev, name.trim()]);
    }
  };

  const deleteLazerSupplier = async (name) => {
    setDialog({
      message: `${name} silinsin mi?`,
      type: 'warning',
      showCancel: true,
      onConfirm: async () => {
        const success = await deleteFromSupabase('lazer_suppliers', name);
        if(success) {
          setLazerSuppliers(prev => prev.filter(s => s !== name));
          setDialog(null);
        }
      },
      onCancel: () => setDialog(null)
    });
  };

  // ✅ MALZEME TEDARİKÇİSİ FONKSİYONLARI
  const addSupplier = async (name) => {
    if(!name.trim()) return;
    const success = await saveToSupabase('material_suppliers', name, setDialog);
    if(success) {
      setSuppliers(prev => [...prev, name.trim()]);
    }
  };

  const deleteSupplier = async (name) => {
    setDialog({
      message: `${name} silinsin mi?`,
      type: 'warning',
      showCancel: true,
      onConfirm: async () => {
        const success = await deleteFromSupabase('material_suppliers', name);
        if(success) {
          setSuppliers(prev => prev.filter(s => s !== name));
          setDialog(null);
        }
      },
      onCancel: () => setDialog(null)
    });
  };
// ✅ İŞLERİ YÜKLE
useEffect(() => {
  let isMounted = true;  // ✅ Cleanup için bayrak
  
  const loadJobs = async () => {
    setLoadingJobs(true);
    const supabaseJobs = await loadJobsFromSupabase();
    
    // ✅ Component hala mount'ta mı kontrol et
    if(isMounted && supabaseJobs.length > 0) {
      const localJobs = lsLoad(LS_KEYS.jobs, []);
      const mergedJobs = [...supabaseJobs];
      
      localJobs.forEach(lj => {
        if(!supabaseJobs.find(sj => sj.id === lj.id)) {
          mergedJobs.push(lj);
        }
      });
      
      setJobs(mergedJobs);
    }
    
    if(isMounted) {
      setLoadingJobs(false);
    }
  };
  
  loadJobs();
  
  // ✅ Cleanup fonksiyonu
  return () => {
    isMounted = false;
  };
}, []);

// ✅ AYARLARI YÜKLE
useEffect(() => {
  let isMounted = true;
  
  const loadSettings = async () => {
    setLoadingSettings(true);
    
    const [teamData, productsData, lazerData, suppliersData, treesData, workflowData] = await Promise.all([
      loadFromSupabase('team_members'),
      loadFromSupabase('product_types'),
      loadFromSupabase('lazer_suppliers'),
      loadFromSupabase('material_suppliers'),
      loadAllProductTreesFromSupabase(),  // ✅ YENİ: Supabase'den ürün ağaçlarını yükle
      loadAllWorkflowTemplatesFromSupabase()  // ✅ İş akış şablonlarını yükle
    ]);
    
    if(isMounted) {
      setTeamMembers(teamData.length > 0 ? teamData : lsLoad(LS_KEYS.team, DEFAULT_TEAM));
      setProducts(productsData.length > 0 ? productsData : lsLoad(LS_KEYS.products, DEFAULT_PRODUCTS));
      setLazerSuppliers(lazerData.length > 0 ? lazerData : lsLoad(LS_KEYS.lazerSuppliers, DEFAULT_LAZER));
      setSuppliers(suppliersData.length > 0 ? suppliersData : DEFAULT_SUPPLIERS);
      
      // ✅ Supabase'den gelen ağaçlar varsa onları kullan, yoksa localStorage'dan yükle
      const finalTrees = Object.keys(treesData).length > 0 ? treesData : lsLoad('pts_product_trees', {});
      setProductTrees(finalTrees);
      
      // ✅ İş akış şablonlarını yükle
      setWorkflowTemplates(workflowData);
      
      setLoadingSettings(false);
    }
  };
  
  loadSettings();
  
  return () => {
    isMounted = false;
  };
}, []);


const updateJobById = (id, updater) => {
  setJobs(prev => {
    const updatedJobs = prev.map(j => {
      if(j.id !== id) return j;
      const updated = typeof updater === 'function' ? updater(j) : { ...j, ...updater };
      
      // ✅ setTimeout yerine direkt kaydet
      saveJobToSupabase(updated).catch(e => {
        console.error('Supabase sync error:', e);
        setDialog({
          message: 'Değişiklik kaydedilemedi! İnternet bağlantınızı kontrol edin.',
          type: 'error',
          showCancel: false,
          onConfirm: () => setDialog(null)
        });
      });
      
      return updated;
    });
    return updatedJobs;
  });
};

const deleteJob = (id) => {
  setShowDeleteConfirm(id);
};

  const completeJob = (id) => {
    updateJobById(id, j => ({ ...j, status:'completed', completedAt: new Date().toISOString() }));
    setView('completed');
    setSelectedJobId(null);
  };
const exportAllJobsReport = () => {
    const activeJobs = jobs.filter(j => j.status === 'active');
    
    if(activeJobs.length === 0) {
      setDialog({
        message: 'Aktif iş bulunmuyor!',
        type: 'info',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
      return;
    }
    
    const wb = XLSX.utils.book_new();
    
    activeJobs.forEach((job, jobIndex) => {
      const data = [
        [`İŞ #${jobIndex + 1}: ${job.customerName || job.productType}`],
        [''],
        ['Müşteri:', job.customerName || '-'],
        ['Ürün:', job.productType || '-'],
        ['Sipariş No:', job.orderNumber || '-'],
        ['Başlangıç:', job.startDate || '-'],
        ['Hedef:', job.targetDate || '-'],
        [''],
        ['─'.repeat(80)],
        ['']
      ];
      
      // Siparişler
      const ordersBySupplier = {};
      
      if(job.supplierOrders) {
        Object.keys(job.supplierOrders).forEach(supplier => {
          const items = job.supplierOrders[supplier];
          if(items.length > 0) {
            if(!ordersBySupplier[supplier]) ordersBySupplier[supplier] = [];
            items.forEach(item => {
              ordersBySupplier[supplier].push({
                product: item.product,
                description: item.description,
                qty: item.qty,
                type: 'Malzeme'
              });
            });
          }
        });
      }
      
      if(job.lazerOrders) {
        Object.keys(job.lazerOrders).forEach(supplier => {
          const items = job.lazerOrders[supplier];
          if(items.length > 0) {
            if(!ordersBySupplier[supplier]) ordersBySupplier[supplier] = [];
            items.forEach(item => {
              ordersBySupplier[supplier].push({
                product: item.product,
                description: item.description,
                qty: item.qty,
                type: 'Lazer'
              });
            });
          }
        });
      }
      
      if(Object.keys(ordersBySupplier).length > 0) {
        data.push(['📦 SİPARİŞLER']);
        data.push(['']);
        
        Object.keys(ordersBySupplier).forEach(supplier => {
          data.push([supplier, '', '', '']);
          data.push(['Ürün', 'Açıklama', 'Adet', 'Tip']);
          
          ordersBySupplier[supplier].forEach(item => {
            data.push([
              item.product || '-',
              item.description || '-',
              item.qty || 1,
              item.type || '-'
            ]);
          });
          data.push(['']);
        });
      }
      
      const ws = XLSX.utils.aoa_to_sheet(data);
      ws['!cols'] = [{wch: 15}, {wch: 40}, {wch: 30}, {wch: 10}];
      
      // Başlık stil
      const titleRef = XLSX.utils.encode_cell({ r: 0, c: 0 });
      if(ws[titleRef]) {
        ws[titleRef].s = {
          font: { bold: true, sz: 16, color: { rgb: "1F4E78" } },
          alignment: { horizontal: "left", vertical: "center" }
        };
      }
      
      // Tablo başlıklarını bul ve stil uygula
      let currentRow = 0;
      for(let i = 0; i < data.length; i++) {
        if(Array.isArray(data[i]) && data[i].length > 0) {
          // Siparişler başlığı
          if(data[i][0] && data[i][0].includes('SİPARİŞLER')) {
            const headerRef = XLSX.utils.encode_cell({ r: i, c: 0 });
            if(ws[headerRef]) {
              ws[headerRef].s = {
                font: { bold: true, sz: 12, color: { rgb: "1F4E78" } },
                alignment: { horizontal: "left", vertical: "center" }
              };
            }
          }
          // Tablo header'ları (Ürün, Açıklama, Adet, Tip)
          if(data[i][0] === 'Ürün' && data[i][1] === 'Açıklama') {
            for(let col = 0; col < 4; col++) {
              const cellRef = XLSX.utils.encode_cell({ r: i, c: col });
              if(ws[cellRef]) {
                ws[cellRef].s = {
                  font: { bold: true, sz: 11, color: { rgb: "FFFFFF" } },
                  fill: { fgColor: { rgb: "4472C4" } },
                  alignment: { horizontal: "center", vertical: "center", wrapText: true },
                  border: {
                    top: { style: "thin" }, bottom: { style: "thin" },
                    left: { style: "thin" }, right: { style: "thin" }
                  }
                };
              }
            }
            // Bu header'dan sonraki veri satırlarına border ekle
            for(let row = i + 1; row < data.length; row++) {
              if(Array.isArray(data[row]) && data[row].length >= 4 && data[row][0] !== '' && !data[row][0].includes('─')) {
                for(let col = 0; col < 4; col++) {
                  const cellRef = XLSX.utils.encode_cell({ r: row, c: col });
                  if(ws[cellRef]) {
                    if(!ws[cellRef].s) ws[cellRef].s = {};
                    ws[cellRef].s.border = {
                      top: { style: "thin" }, bottom: { style: "thin" },
                      left: { style: "thin" }, right: { style: "thin" }
                    };
                    ws[cellRef].s.alignment = { vertical: "center", wrapText: true };
                    if(col === 2) ws[cellRef].s.alignment.horizontal = "right";
                  }
                }
              } else if(data[row][0] === '') {
                break; // Boş satır geldiğinde dur
              }
            }
          }
        }
      }
      
      // Satır yükseklikleri
      ws['!rows'] = [];
      for(let i = 0; i < data.length; i++) {
        ws['!rows'][i] = { hpt: i === 0 ? 25 : 20 };
      }
      
      const sheetName = `${job.orderNumber || job.id}`.substring(0, 31);
      XLSX.utils.book_append_sheet(wb, ws, sheetName);
    });
    
    XLSX.writeFile(wb, `Tum_Aktif_Isler_Raporu_${new Date().toISOString().split('T')[0]}.xlsx`);
  };
const visibleJobs = React.useMemo(() => {
  return jobs.filter(j => {
    const statusFilter = view === 'completed' ? 'completed' : 'active';
    if(jobFilter.trim() === '') return j.status === statusFilter;
    const f = jobFilter.toLowerCase();
    return j.status === statusFilter && (
      (j.productType || '').toLowerCase().includes(f) ||
      (j.customerName || '').toLowerCase().includes(f) ||
      (j.orderNumber || '').toLowerCase().includes(f)
    );
  });
}, [jobs, view, jobFilter]);  // ✅ Sadece bunlar değişince yeniden hesapla

  return (
    <div className="min-h-screen bg-gray-50 p-4">
      <div className="max-w-7xl mx-auto">
        <div className="bg-white rounded-lg shadow-sm p-6 mb-6">  
          <div className="flex items-start justify-between">
            <div>
              <h1 className="text-3xl font-bold">🏭 BEGA Üretim Takip Sistemi</h1>
              <p className="text-gray-600">İş Takibi</p>
            </div>
<div className="flex gap-2">
  <button onClick={() => setView('jobs')} className={`px-4 py-2 rounded-lg font-medium ${view==='jobs'? 'bg-blue-500 text-white':'bg-gray-100'}`}>📋 Aktif İşler ({jobs.filter(j=>j.status==='active').length})</button>
  <button onClick={() => setView('completed')} className={`px-4 py-2 rounded-lg font-medium ${view==='completed'? 'bg-blue-500 text-white':'bg-gray-100'}`}>✅ Tamamlanan ({jobs.filter(j=>j.status==='completed').length})</button>
  <button onClick={() => setView('productTree')} className={`px-4 py-2 rounded-lg font-medium ${view==='productTree'? 'bg-blue-500 text-white':'bg-gray-100'}`}>🌳 Ürün Ağacı</button>
  <button onClick={() => setView('workflowTemplates')} className={`px-4 py-2 rounded-lg font-medium ${view==='workflowTemplates'? 'bg-blue-500 text-white':'bg-gray-100'}`}>📋 İş Akış Şablonları</button>
  <button onClick={() => setView('settings')} className={`px-4 py-2 rounded-lg font-medium ${view==='settings'? 'bg-blue-500 text-white':'bg-gray-100'}`}>⚙️ Ayarlar</button>
  <button onClick={() => window.open('https://bega-stok-takip.netlify.app/', '_blank')} className="px-4 py-2 rounded-lg font-medium bg-green-100 hover:bg-green-200 flex items-center gap-2">
    <Icon type="box" className="w-5 h-5" /> Stok
  </button>
</div>
          </div>
        </div>
{view === 'productTree' && (
  <ProductTreePage 
    products={products}
    productTrees={productTrees}
    setProductTrees={setProductTrees}
    suppliers={suppliers}
    lazerSuppliers={lazerSuppliers}
    back={() => setView('settings')}
  />
)}
{view === 'workflowTemplates' && (
  <WorkflowTemplatesPage 
    products={products}
    workflowTemplates={workflowTemplates}
    setWorkflowTemplates={setWorkflowTemplates}
    back={() => setView('settings')}
  />
)}
{view === 'settings' && (
  <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mb-6">
    {/* ✅ 1. KART - EKİP ÜYELERİ */}
    <div className="bg-white rounded-lg shadow-sm p-6">
      <h2 className="text-xl font-bold mb-4 flex items-center gap-2"><Icon type="users" /> Ekip Üyeleri</h2>
      <div className="flex gap-2 mb-4">
        <input 
          value={newMember} 
          onChange={e=>setNewMember(e.target.value)} 
          placeholder="Yeni ekip üyesi..." 
          className="flex-1 px-3 py-2 border rounded-lg" 
          onKeyDown={async (e)=>{ if(e.key==='Enter' && newMember.trim()){ await addTeamMember(newMember); setNewMember(''); }}} 
        />
        <button 
          onClick={async ()=>{ if(newMember.trim()){ await addTeamMember(newMember); setNewMember(''); }}} 
          className="px-4 py-2 bg-green-500 text-white rounded-lg"
        >
          <Icon type="plus" />
        </button>
      </div>
      <div className="space-y-2 max-h-96 overflow-auto">
{teamMembers.map((m)=>(
  <div key={m}>  {/* ✅ m (isim) unique */}
            <span>{m}</span>
            <button onClick={()=> deleteTeamMember(m)} className="text-red-500"><Icon type="trash" className="w-4 h-4"/></button>
          </div>
        ))}
      </div>
    </div>

    {/* ✅ 2. KART - ÜRÜN TİPLERİ */}
    <div className="bg-white rounded-lg shadow-sm p-6">
      <h2 className="text-xl font-bold mb-4">📦 Ürün Tipleri</h2>
      <div className="flex gap-2 mb-4">
        <input 
          value={newProduct} 
          onChange={e=>setNewProduct(e.target.value)} 
          placeholder="Yeni ürün tipi..." 
          className="flex-1 px-3 py-2 border rounded-lg" 
          onKeyDown={async (e)=>{ if(e.key==='Enter' && newProduct.trim()){ await addProduct(newProduct); setNewProduct(''); }}} 
        />
        <button 
          onClick={async ()=>{ if(newProduct.trim()){ await addProduct(newProduct); setNewProduct(''); }}} 
          className="px-4 py-2 bg-blue-500 text-white rounded-lg"
        >
          <Icon type="plus" />
        </button>
      </div>
      <div className="space-y-2 max-h-96 overflow-auto">
{products.map((p)=>(
  <div key={p}>  {/* ✅ p (ürün adı) unique */}
            <span>{p}</span>
            <button onClick={()=> deleteProduct(p)} className="text-red-500"><Icon type="trash" className="w-4 h-4"/></button>
          </div>
        ))}
      </div>
    </div>

    {/* ✅ 3. KART - LAZER FİRMALARI */}
    <div className="bg-white rounded-lg shadow-sm p-6">
      <h2 className="text-xl font-bold mb-4">⚡ Lazer Kesim Firmaları</h2>
      <div className="flex gap-2 mb-4">
        <input 
          value={newLazerSupplier} 
          onChange={e=>setNewLazerSupplier(e.target.value)} 
          placeholder="Yeni firma..." 
          className="flex-1 px-3 py-2 border rounded-lg" 
          onKeyDown={async (e)=>{ if(e.key==='Enter' && newLazerSupplier.trim()){ await addLazerSupplier(newLazerSupplier); setNewLazerSupplier(''); }}} 
        />
        <button 
          onClick={async ()=>{ if(newLazerSupplier.trim()){ await addLazerSupplier(newLazerSupplier); setNewLazerSupplier(''); }}} 
          className="px-4 py-2 bg-purple-500 text-white rounded-lg"
        >
          <Icon type="plus" />
        </button>
      </div>
      <div className="space-y-2 max-h-96 overflow-auto">
{lazerSuppliers.map((m)=>(
  <div key={m}>  {/* ✅ m (firma adı) unique */}
            <span>{m}</span>
            <button onClick={()=> deleteLazerSupplier(m)} className="text-red-500"><Icon type="trash" className="w-4 h-4"/></button>
          </div>
        ))}
      </div>
    </div>

    {/* ✅ 4. KART - MALZEME TEDARİKÇİLERİ (YENİ) */}
    <div className="bg-white rounded-lg shadow-sm p-6">
      <h2 className="text-xl font-bold mb-4">🏢 Malzeme Tedarikçileri</h2>
      <div className="flex gap-2 mb-4">
        <input 
          value={newSupplier} 
          onChange={e=>setNewSupplier(e.target.value)} 
          placeholder="Yeni tedarikçi..." 
          className="flex-1 px-3 py-2 border rounded-lg" 
          onKeyDown={async (e)=>{ if(e.key==='Enter' && newSupplier.trim()){ await addSupplier(newSupplier); setNewSupplier(''); }}} 
        />
        <button 
          onClick={async ()=>{ if(newSupplier.trim()){ await addSupplier(newSupplier); setNewSupplier(''); }}} 
          className="px-4 py-2 bg-orange-500 text-white rounded-lg"
        >
          <Icon type="plus" />
        </button>
      </div>
      <div className="space-y-2 max-h-96 overflow-auto">
{suppliers.map((s)=>(
  <div key={s} className="flex items-center justify-between">
    <span>{s}</span>
    <button onClick={()=> deleteSupplier(s)} className="text-red-500"><Icon type="trash" className="w-4 h-4"/></button>
  </div>
))}
      </div>
    </div>

  </div>
)}

        {view === 'jobs' && (
          <>
<div className="flex flex-col md:flex-row md:items-center gap-4 mb-6">
              <div className="flex gap-2">
                <button onClick={() => setShowCreateModal(true)} className="px-6 py-3 bg-green-500 text-white rounded-lg flex items-center gap-2"><Icon type="plus" /> Yeni İş Oluştur</button>
                <button onClick={exportAllJobsReport} className="px-6 py-3 bg-purple-600 text-white rounded-lg flex items-center gap-2"><Icon type="download" /> Tüm İşlerin Raporu</button>
              </div>
              <div className="ml-auto flex items-center gap-2">
                <input value={jobFilter} onChange={e=>setJobFilter(e.target.value)} placeholder="Ara: müşteri, ürün, sipariş no..." className="px-3 py-2 border rounded" />
                <button onClick={()=> setJobFilter('')} className="px-3 py-2 bg-gray-100 rounded">Temizle</button>
              </div>
            </div>

            {loadingJobs && <div className="text-center py-8 text-gray-500">Supabase'den işler yükleniyor...</div>}

            <div className="space-y-6">
              {visibleJobs.length === 0 ? (
                <div className="bg-white rounded-lg shadow-sm p-12 text-center text-gray-500">Henüz aktif iş yok veya filtre sonucu yok.</div>
              ) : (() => {
                // Müşterilere göre grupla
                const jobsByCustomer = {};
                visibleJobs.forEach(job => {
                  const customer = job.customerName || 'Müşteri Belirtilmemiş';
                  if(!jobsByCustomer[customer]) {
                    jobsByCustomer[customer] = [];
                  }
                  jobsByCustomer[customer].push(job);
                });

                return Object.entries(jobsByCustomer).map(([customer, customerJobs]) => {
                  const isExpanded = expandedCustomers[customer] === true; // Varsayılan olarak kapalı
                  return (
                  <div key={customer} className="bg-white rounded-lg shadow-sm overflow-hidden">
                    <div 
                      className="bg-blue-50 border-b p-4 cursor-pointer hover:bg-blue-100 transition"
                      onClick={() => setExpandedCustomers(prev => ({ ...prev, [customer]: !isExpanded }))}
                    >
                      <div className="flex items-center justify-between">
                        <div className="flex items-center gap-3">
                          <Icon type={isExpanded ? "chevron-down" : "chevron-right"} className="w-5 h-5 text-blue-700" />
                          <div>
                            <h2 className="text-xl font-bold text-blue-900">{customer}</h2>
                            <p className="text-sm text-blue-700">{customerJobs.length} iş</p>
                          </div>
                        </div>
                        <button 
                          onClick={(e) => {
                            e.stopPropagation(); // Müşteri açma/kapama işlemini engelle
                            // Bu müşteri için yeni iş oluştur
                            const firstJob = customerJobs[0];
                            setFormData({
                              customerName: customer, 
                              customerPhone: firstJob?.customerPhone || '', 
                              customerAddress: firstJob?.customerAddress || '',
                              productType: products[0] || '',
                              productCustom: '',
                              startDate: new Date().toISOString().split('T')[0],
                              targetDate: ''
                            });
                            setShowCreateModal(true);
                          }}
                          className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600"
                        >
                          + Yeni İş Ekle
                        </button>
                      </div>
                    </div>
                    {isExpanded && (
                    <div className="divide-y">
                      {customerJobs.map(job => (
                        <div key={job.id} className="p-4 hover:bg-gray-50">
                          <div className="flex justify-between items-start">
                            <div className="flex-1">
                              <h3 className="text-lg font-semibold">{job.productType || 'Ürün belirtilmemiş'}</h3>
                              <p className="text-sm text-gray-600">Sipariş No: {job.orderNumber || 'Belirtilmemiş'}</p>
                              <div className="flex gap-4 mt-2 text-sm">
                                <span>📅 Başlangıç: {job.startDate}</span>
                                {job.targetDate && <span>🎯 Hedef: {job.targetDate}</span>}
                              </div>
                            </div>
                            <div className="flex flex-col items-end gap-2">
                              <div className="flex gap-2">
                                <button onClick={() => { setSelectedJobId(job.id); setView('jobDetail'); }} className="px-3 py-2 bg-blue-500 text-white rounded-lg flex items-center gap-2"><Icon type="right"/> Aç</button>
                                <button onClick={() => {
                                  setDialog({
                                    message: 'Bu işi tamamla?',
                                    type: 'info',
                                    showCancel: true,
                                    onConfirm: () => {
                                      completeJob(job.id);
                                      setDialog(null);
                                    },
                                    onCancel: () => setDialog(null)
                                  });
                                }} className="px-3 py-2 bg-green-500 text-white rounded-lg">Tamamla</button>
                              </div>
                              <div className="mt-2 flex gap-2">
                                <button onClick={() => {
                                  setDialog({
                                    message: 'Silinsin mi?',
                                    type: 'warning',
                                    showCancel: true,
                                    onConfirm: () => {
                                      deleteJob(job.id);
                                      setDialog(null);
                                    },
                                    onCancel: () => setDialog(null)
                                  });
                                }} className="px-3 py-2 bg-red-100 text-red-600 rounded"><Icon type="trash" /></button>
                              </div>
                            </div>
                          </div>
                        </div>
                      ))}
                    </div>
                    )}
                  </div>
                  );
                });
              })()}
            </div>
          </>
        )}

        {view === 'completed' && (
          <>
            <div className="flex flex-col md:flex-row md:items-center gap-4 mb-6">
              <h2 className="text-xl font-semibold">✅ Tamamlanan İşler</h2>
              <div className="ml-auto flex items-center gap-2">
                <input value={jobFilter} onChange={e=>setJobFilter(e.target.value)} placeholder="Ara..." className="px-3 py-2 border rounded" />
                <button onClick={()=> setJobFilter('')} className="px-3 py-2 bg-gray-100 rounded">Temizle</button>
              </div>
            </div>

            <div className="space-y-6">
              {visibleJobs.length === 0 ? (
                <div className="bg-white rounded-lg shadow-sm p-12 text-center text-gray-500">Henüz tamamlanmış iş yok.</div>
              ) : (() => {
                // Müşterilere göre grupla
                const jobsByCustomer = {};
                visibleJobs.forEach(job => {
                  const customer = job.customerName || 'Müşteri Belirtilmemiş';
                  if(!jobsByCustomer[customer]) {
                    jobsByCustomer[customer] = [];
                  }
                  jobsByCustomer[customer].push(job);
                });

                return Object.entries(jobsByCustomer).map(([customer, customerJobs]) => {
                  const isExpanded = expandedCustomers[customer] === true; // Varsayılan olarak kapalı
                  return (
                  <div key={customer} className="bg-white rounded-lg shadow-sm overflow-hidden opacity-75">
                    <div 
                      className="bg-green-50 border-b p-4 cursor-pointer hover:bg-green-100 transition"
                      onClick={() => setExpandedCustomers(prev => ({ ...prev, [customer]: !isExpanded }))}
                    >
                      <div className="flex items-center justify-between">
                        <div className="flex items-center gap-3">
                          <Icon type={isExpanded ? "chevron-down" : "chevron-right"} className="w-5 h-5 text-green-700" />
                          <div>
                            <h2 className="text-xl font-bold text-green-900">{customer}</h2>
                            <p className="text-sm text-green-700">{customerJobs.length} iş</p>
                          </div>
                        </div>
                      </div>
                    </div>
                    {isExpanded && (
                    <div className="divide-y">
                      {customerJobs.map(job => (
                        <div key={job.id} className="p-6 hover:bg-gray-50">
                          <div className="flex justify-between items-start">
                            <div className="flex-1">
                              <div className="flex items-center gap-2">
                                <h3 className="text-lg font-semibold">{job.productType || 'Ürün belirtilmemiş'}</h3>
                                <span className="px-2 py-1 bg-green-100 text-green-700 text-xs rounded">Tamamlandı</span>
                              </div>
                              <p className="text-sm text-gray-600">Sipariş No: {job.orderNumber || 'Belirtilmemiş'}</p>
                              <div className="flex gap-4 mt-2 text-sm">
                                <span>📅 Başlangıç: {job.startDate}</span>
                                {job.completedAt && <span>✅ Tamamlanma: {new Date(job.completedAt).toLocaleDateString('tr-TR')}</span>}
                              </div>
                            </div>

                            <div className="flex flex-col items-end gap-2">
                              <div className="flex gap-2">
                                <button onClick={() => { setSelectedJobId(job.id); setView('jobDetail'); }} className="px-3 py-2 bg-blue-500 text-white rounded-lg flex items-center gap-2"><Icon type="right"/> Görüntüle</button>
                              </div>
                              <div className="mt-2 flex gap-2">
                                <button onClick={() => {
                                  setDialog({
                                    message: 'Aktif işlere geri döndür?',
                                    type: 'info',
                                    showCancel: true,
                                    onConfirm: () => {
                                      updateJobById(job.id, {status: 'active'});
                                      setDialog(null);
                                    },
                                    onCancel: () => setDialog(null)
                                  });
                                }} className="px-3 py-2 bg-yellow-100 text-yellow-700 rounded text-sm">Aktife Döndür</button>
                                <button onClick={() => {
                                  setDialog({
                                    message: 'Kalıcı olarak silinsin mi?',
                                    type: 'warning',
                                    showCancel: true,
                                    onConfirm: () => {
                                      deleteJob(job.id);
                                      setDialog(null);
                                    },
                                    onCancel: () => setDialog(null)
                                  });
                                }} className="px-3 py-2 bg-red-100 text-red-600 rounded"><Icon type="trash" /></button>
                              </div>
                            </div>
                          </div>
                        </div>
                      ))}
                    </div>
                    )}
                  </div>
                  );
                });
              })()}
            </div>
          </>
        )}

        {view === 'jobDetail' && selectedJobId && (
          <JobDetail
            key={selectedJobId}
            job={jobs.find(j => j.id === selectedJobId)}
            back={() => { setView('jobs'); setSelectedJobId(null); }}
            updateJob={(updater) => updateJobById(selectedJobId, updater)}
            teamMembers={teamMembers}
            products={products}
            lazerSuppliers={lazerSuppliers}
            suppliers={suppliers}
            deleteJob={() => deleteJob(selectedJobId)}
            completeJob={() => completeJob(selectedJobId)}
            onExportSupplier={exportSupplierXlsx}
            onExportAll={exportAllSuppliersXlsx}
          />
        )}
      </div>

      {showCreateModal && (
        <CreateJobModal
          products={products}
          initialData={formData}
          onClose={() => { setShowCreateModal(false); setFormData({customerName: '', customerPhone: '', customerAddress: '', productType: products[0] || '', productCustom: '', startDate: new Date().toISOString().split('T')[0], targetDate: ''}); }}
          onCreate={async (jobData) => {
  // ✅ Unique ID oluştur
  const now = Date.now();
  const randomPart = Math.random().toString(36).substr(2, 9);
  const uniqueId = `${now}-${randomPart}`;
  
  // ✅ tabsCopy'yi oluştur
  const tabsCopy = DEFAULT_TABS.map(t => ({
    ...t,
    id: t.id + '-' + now,
    tasks: []
  }));
  
  // ✅ YENİ: Seçilen ürün tipinin malzeme listesini Supabase'den yükle
  const productType = jobData.productType === '__custom__' ? jobData.productCustom : jobData.productType;
  const productMaterials = await loadProductTreeFromSupabase(productType);
  
  // ✅ YENİ: İş akış şablonlarını yükle ve "Yapılan ve Planlanan İşler" sekmesine ekle
  const workflowTasks = await loadWorkflowTemplateFromSupabase(productType);
  const planlananTab = tabsCopy.find(t => t.id.startsWith('tab-planlanan'));
  if(planlananTab && workflowTasks.length > 0) {
    planlananTab.tasks = workflowTasks.map((task, idx) => ({
      id: `task-${now}-${idx}`,
      text: task.text,
      done: false,
      assignedTo: [],
      notes: ''
    }));
  }
  
  // ✅ Malzemeleri tedarikçilere göre ayır
  const supplierOrders = {};
  const lazerOrders = {};
  
  productMaterials.forEach(material => {
    const orderData = {
      product: material.product,
      description: material.description || '',
      qty: material.qty || 1
    };
    
    if(material.type === 'lazer') {
      if(!lazerOrders[material.supplier]) lazerOrders[material.supplier] = [];
      lazerOrders[material.supplier].push(orderData);
    } else {
      if(!supplierOrders[material.supplier]) supplierOrders[material.supplier] = [];
      supplierOrders[material.supplier].push(orderData);
    }
  });
  
  const job = {
    id: uniqueId,
    ...jobData,
    orderNumber: jobData.orderNumber || nextOrderNumber(jobs),
    status: 'active',
    tabs: tabsCopy,
    cabinet: jobData.cabinet || null,
    cabinetCompleted: !!jobData.cabinet,
    cabinetNotes: '',
    cabinetExtras: [],
    supplierOrders: supplierOrders,  // ✅ Otomatik dolduruldu
    lazerOrders: lazerOrders  // ✅ Otomatik dolduruldu
  };
  
  setJobs(prev => [...prev, job]);
  setShowCreateModal(false);
  setSelectedJobId(uniqueId);
  setView('jobDetail');
  
  saveJobToSupabase(job).then(success => {
    if(success) {
      console.log('İş Supabase\'e kaydedildi');
      if(productMaterials.length > 0) {
        setDialog({
          message: `✅ İş oluşturuldu!\n\n${productMaterials.length} malzeme otomatik olarak eklendi.`,
          type: 'success',
          showCancel: false,
          onConfirm: () => setDialog(null)
        });
      }
    }
  }).catch(e => {
    console.error('Supabase kayıt hatası:', e);
    setDialog({
      message: 'İş oluşturuldu ama Supabase\'e kaydedilemedi!',
      type: 'warning',
      showCancel: false,
      onConfirm: () => setDialog(null)
    });
  });
}}
          nextOrderNumber={() => nextOrderNumber(jobs)}
        />
      )}
      {showDeleteConfirm && (
        <CustomDialog
          message={`Bu işi silmek istediğinize emin misiniz?\n\nSipariş: ${jobs.find(j => j.id === showDeleteConfirm)?.orderNumber}`}
          onConfirm={async () => {
            const jobIdStr = String(showDeleteConfirm);
            try {
              await supabaseFetch(`jobs?job_id=eq.${encodeURIComponent(jobIdStr)}`, { method: 'DELETE' });
              setJobs(prev => prev.filter(j => j.id !== showDeleteConfirm));
              if(selectedJobId === showDeleteConfirm) { 
                setSelectedJobId(null); 
                setView('jobs'); 
              }
              setShowDeleteConfirm(null);
            } catch(e) {
              console.error('Silme hatası:', e);
              setDialog({
                message: '❌ İş silinemedi!',
                type: 'error',
                showCancel: false,
                onConfirm: () => setDialog(null)
              });
            }
          }}
          onCancel={() => setShowDeleteConfirm(null)}
        />
      )}
      {dialog && (
        <CustomDialog
          message={dialog.message}
          type={dialog.type}
          showCancel={dialog.showCancel}
          onConfirm={dialog.onConfirm}
          onCancel={dialog.onCancel || (() => setDialog(null))}
        />
      )}
    </div>  // ✅ ProductionTracker'ın div'i kapanıyor
  );
}

/* ---------- CreateJobModal ---------- */  // ✅ Buradan CreateJobModal başlıyor
function CreateJobModal({ products, onClose, onCreate, nextOrderNumber, initialData = null }) {
  const [step, setStep] = useState(0);
  const [formData, setFormData] = useState(initialData || {
    customerName: '',
    customerPhone: '',
    customerAddress: '',
    productType: products[0] || '',
    productCustom: '',
    startDate: new Date().toISOString().split('T')[0],
    targetDate: ''
  });
  const [cabinet, setCabinet] = useState({
    coolingType: '',
    doorType: '',
    glassType: '',
    coolingGroup: '',
    cabinetSize: '',
    caseType: '',
    caseColor: '',
    extras: []
  });
  const [extraInput, setExtraInput] = useState('');
  const [orderNum, setOrderNum] = useState('');

  useEffect(() => {
    setOrderNum(nextOrderNumber());
  }, []);

  const wizardSteps = [
    { id: 'info', title: 'İş Bilgileri' },
    { id: 'cooling', title: 'Soğutma Tipi' },  
    { id: 'door', title: 'Kapı Türü' },
    { id: 'glass', title: 'Cam Tipi' },
    { id: 'coolingGroup', title: 'Soğutma Grubu' },
    { id: 'cabinetSize', title: 'Dolap Ölçüsü' },
    { id: 'case', title: 'Kasa Tipi' },
    { id: 'extras', title: 'Ek Özellikler' },
    { id: 'confirm', title: 'Onay' }
  ];

  const next = () => {
    if (step < wizardSteps.length - 1) setStep(step + 1);
  };

  const back = () => {
    if (step > 0) setStep(step - 1);
  };

  const skip = () => {
    if (step < wizardSteps.length - 1) setStep(step + 1);
  };

  const handleCreate = () => {
    const finalData = {
      ...formData,
      orderNumber: orderNum,
      cabinet: cabinet
    };
    onCreate(finalData);
  };

  return (
    <div className="fixed inset-0 bg-black/40 flex items-center justify-center z-50 p-4">
      <div className="bg-white rounded-lg p-6 w-full max-w-3xl max-h-[90vh] overflow-y-auto">
        <div className="flex items-center justify-between mb-4">
          <h3 className="text-xl font-bold">Yeni İş Oluştur - {wizardSteps[step].title}</h3>
          <button onClick={onClose} className="text-gray-500 hover:text-gray-700">✕</button>
        </div>

        <div className="mb-6">
          <div className="flex items-center justify-between text-xs mb-2">
            <span>Adım {step + 1} / {wizardSteps.length}</span>
            <span className="text-gray-500">{wizardSteps[step].title}</span>
          </div>
          <div className="w-full bg-gray-200 h-2 rounded">
            <div className="bg-blue-500 h-2 rounded transition-all" style={{width: `${((step + 1) / wizardSteps.length) * 100}%`}}></div>
          </div>
        </div>

        <div className="min-h-[300px]">
          {step === 0 && (
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label className="text-sm text-gray-600 block mb-1">Müşteri Adı *</label>
                <input value={formData.customerName} onChange={e=>setFormData({...formData, customerName:e.target.value})} className="w-full px-3 py-2 border rounded" placeholder="Müşteri adı"/>
              </div>
              <div>
                <label className="text-sm text-gray-600 block mb-1">Telefon</label>
                <input value={formData.customerPhone} onChange={e=>setFormData({...formData, customerPhone:e.target.value})} className="w-full px-3 py-2 border rounded" placeholder="Telefon"/>
              </div>
              <div className="md:col-span-2">
                <label className="text-sm text-gray-600 block mb-1">Adres</label>
                <input value={formData.customerAddress} onChange={e=>setFormData({...formData, customerAddress:e.target.value})} className="w-full px-3 py-2 border rounded" placeholder="Adres"/>
              </div>
              <div>
                <label className="text-sm text-gray-600 block mb-1">Ürün Tipi *</label>
                <select value={formData.productType} onChange={e=>setFormData({...formData, productType:e.target.value})} className="w-full px-3 py-2 border rounded">
                  {products.map(p => <option key={p} value={p}>{p}</option>)}
                  <option value="__custom__">-- Özel Ürün --</option>
                </select>
                {formData.productType === '__custom__' && (
                  <input placeholder="Özel ürün adı" value={formData.productCustom} onChange={e=>setFormData({...formData, productCustom:e.target.value})} className="mt-2 w-full px-3 py-2 border rounded" />
                )}
              </div>
              <div>
                <label className="text-sm text-gray-600 block mb-1">Sipariş No</label>
                <input value={orderNum} readOnly className="w-full px-3 py-2 border rounded bg-gray-50" />
              </div>
              <div>
                <label className="text-sm text-gray-600 block mb-1">Başlangıç Tarihi</label>
                <input type="date" value={formData.startDate} onChange={e=>setFormData({...formData, startDate:e.target.value})} className="w-full px-3 py-2 border rounded" />
              </div>
              <div>
                <label className="text-sm text-gray-600 block mb-1">Hedef Tarih</label>
                <input type="date" value={formData.targetDate} onChange={e=>setFormData({...formData, targetDate:e.target.value})} className="w-full px-3 py-2 border rounded" />
              </div>
            </div>
          )}

          {step === 1 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Soğutma tipini seçin:</p>
              <div className="flex gap-3">
                <label className={`flex-1 px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.coolingType==='üfleme' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="cool" value="üfleme" checked={cabinet.coolingType==='üfleme'} onChange={e=>setCabinet({...cabinet, coolingType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Üfleme</span>
                </label>
                <label className={`flex-1 px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.coolingType==='statik' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="cool" value="statik" checked={cabinet.coolingType==='statik'} onChange={e=>setCabinet({...cabinet, coolingType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Statik</span>
                </label>
              </div>
            </div>
          )}

          {step === 2 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Kapı türünü seçin:</p>
              <div className="flex gap-3">
                <label className={`flex-1 px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.doorType==='çarpma' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="door" value="çarpma" checked={cabinet.doorType==='çarpma'} onChange={e=>setCabinet({...cabinet, doorType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Çarpma Kapı</span>
                </label>
                <label className={`flex-1 px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.doorType==='sürgü' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="door" value="sürgü" checked={cabinet.doorType==='sürgü'} onChange={e=>setCabinet({...cabinet, doorType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Sürgü Kapı</span>
                </label>
              </div>
            </div>
          )}

          {step === 3 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Cam tipini seçin:</p>
              <div className="flex gap-3 flex-wrap">
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.glassType==='dik' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="glass" value="dik" checked={cabinet.glassType==='dik'} onChange={e=>setCabinet({...cabinet, glassType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Dik Cam</span>
                </label>
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.glassType==='oval' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="glass" value="oval" checked={cabinet.glassType==='oval'} onChange={e=>setCabinet({...cabinet, glassType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Oval Cam</span>
                </label>
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.glassType==='eğik' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="glass" value="eğik" checked={cabinet.glassType==='eğik'} onChange={e=>setCabinet({...cabinet, glassType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Eğik Cam</span>
                </label>
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.glassType==='akvaryum-düz' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="glass" value="akvaryum-düz" checked={cabinet.glassType==='akvaryum-düz'} onChange={e=>setCabinet({...cabinet, glassType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Akvaryum Tipi - Düz</span>
                </label>
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.glassType==='akvaryum-eğimli' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="glass" value="akvaryum-eğimli" checked={cabinet.glassType==='akvaryum-eğimli'} onChange={e=>setCabinet({...cabinet, glassType:e.target.value})} className="mr-2" />
                  <span className="font-medium">Akvaryum Tipi - Eğimli</span>
                </label>
              </div>
            </div>
          )}

          {step === 4 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Soğutma grubu nerede olacak?</p>
              <div className="flex gap-3">
                <label className={`flex-1 px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.coolingGroup==='içeride' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="coolingGroup" value="içeride" checked={cabinet.coolingGroup==='içeride'} onChange={e=>setCabinet({...cabinet, coolingGroup:e.target.value})} className="mr-2" />
                  <span className="font-medium">1 - İçeride</span>
                </label>
                <label className={`flex-1 px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.coolingGroup==='dışarıda' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="coolingGroup" value="dışarıda" checked={cabinet.coolingGroup==='dışarıda'} onChange={e=>setCabinet({...cabinet, coolingGroup:e.target.value})} className="mr-2" />
                  <span className="font-medium">2 - Dışarıda</span>
                </label>
              </div>
            </div>
          )}

          {step === 5 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Dolap ölçüsü ne olacak?</p>
              <input 
                value={cabinet.cabinetSize || ''} 
                onChange={e=>setCabinet({...cabinet, cabinetSize:e.target.value})} 
                className="w-full px-3 py-2 border rounded" 
                placeholder="Ölçü Giriniz..."
              />
            </div>
          )}

          {step === 6 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Kasa tipini seçin:</p>
              <div className="flex gap-3 flex-wrap mb-4">
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.caseType==='boyalı' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="case" value="boyalı" checked={cabinet.caseType==='boyalı'} onChange={e=>setCabinet({...cabinet, caseType:e.target.value, caseColor:''})} className="mr-2" />
                  <span className="font-medium">Boyalı</span>
                </label>
                <label className={`px-4 py-3 border-2 rounded-lg cursor-pointer transition ${cabinet.caseType==='paslanmaz' ? 'bg-blue-50 border-blue-400':'border-gray-200 hover:border-gray-300'}`}>
                  <input type="radio" name="case" value="paslanmaz" checked={cabinet.caseType==='paslanmaz'} onChange={e=>setCabinet({...cabinet, caseType:e.target.value, caseColor:''})} className="mr-2" />
                  <span className="font-medium">Paslanmaz</span>
                </label>
              </div>
              {cabinet.caseType === 'boyalı' && (
                <input value={cabinet.caseColor || ''} onChange={e=>setCabinet({...cabinet, caseColor: e.target.value})} className="w-full px-3 py-2 border rounded" placeholder="Boya rengi (örn: Beyaz, Gri, Siyah...)"/>
              )}
            </div>
          )}

          {step === 7 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Ekstra özellikler ekleyin:</p>
              <div className="flex gap-2 mb-4">
                <input value={extraInput} onChange={e=>setExtraInput(e.target.value)} className="flex-1 px-3 py-2 border rounded" placeholder="Ör: LED aydınlatma, özel kilit..." onKeyDown={(e)=>{ if(e.key==='Enter' && extraInput.trim()){ setCabinet({...cabinet, extras:[...cabinet.extras, extraInput.trim()]}); setExtraInput(''); }}} />
                <button onClick={()=>{ if(extraInput.trim()){ setCabinet({...cabinet, extras:[...cabinet.extras, extraInput.trim()]}); setExtraInput(''); }}} className="px-4 py-2 bg-green-500 text-white rounded">Ekle</button>
              </div>
              <div className="space-y-2">
                {cabinet.extras.map((ex, i) => (
                  <div key={i} className="flex items-center justify-between bg-gray-50 p-3 rounded">
                    <span>{i+1}. {ex}</span>
                    <button onClick={() => setCabinet({...cabinet, extras:cabinet.extras.filter((_,idx)=>idx!==i)})} className="text-red-500 px-2">Sil</button>
                  </div>
                ))}
              </div>
            </div>
          )}

          {step === 8 && (
            <div>
              <p className="text-sm text-gray-600 mb-4">Bilgileri kontrol edin ve onaylayın:</p>
              <div className="bg-gray-50 p-4 rounded space-y-2 text-sm">
                <div><strong>Müşteri:</strong> {formData.customerName || '-'}</div>
                <div><strong>Telefon:</strong> {formData.customerPhone || '-'}</div>
                <div><strong>Ürün:</strong> {formData.productType === '__custom__' ? formData.productCustom : formData.productType}</div>
                <div><strong>Sipariş No:</strong> {orderNum}</div>
                <div><strong>Başlangıç:</strong> {formData.startDate}</div>
                {formData.targetDate && <div><strong>Hedef:</strong> {formData.targetDate}</div>}
                <hr className="my-2"/>
                <div className="font-semibold">Dolap Özeti:</div>
                <div><strong>Soğutma:</strong> {cabinet.coolingType || 'Belirtilmedi'}</div>
                <div><strong>Kapı:</strong> {cabinet.doorType || 'Belirtilmedi'}</div>
                <div><strong>Cam:</strong> {cabinet.glassType ? (cabinet.glassType.includes('akvaryum') ? `Akvaryum Tipi - ${cabinet.glassType.includes('düz') ? 'Düz' : 'Eğimli'}` : cabinet.glassType) : 'Belirtilmedi'}</div>
                <div><strong>Soğutma Grubu:</strong> {cabinet.coolingGroup === 'içeride' ? '1 - İçeride' : cabinet.coolingGroup === 'dışarıda' ? '2 - Dışarıda' : 'Belirtilmedi'}</div>
                <div><strong>Dolap Ölçüsü:</strong> {cabinet.cabinetSize || 'Belirtilmedi'}</div>
                <div><strong>Kasa:</strong> {cabinet.caseType === 'boyalı' ? `Boyalı${cabinet.caseColor ? ' - ' + cabinet.caseColor : ''}` : (cabinet.caseType || 'Belirtilmedi')}</div>
                {cabinet.extras.length > 0 && (
                  <div>
                    <strong>Ekstra Özellikler:</strong>
                    <ul className="list-disc ml-5 mt-1">
                      {cabinet.extras.map((ex,i) => <li key={i}>{ex}</li>)}
                    </ul>
                  </div>
                )}
              </div>
            </div>
          )}
        </div>

        <div className="mt-6 flex justify-between items-center">
          <button onClick={onClose} className="px-4 py-2 bg-gray-100 rounded hover:bg-gray-200">İptal</button>
          <div className="flex gap-2">
            {step > 0 && <button onClick={back} className="px-4 py-2 bg-gray-200 rounded hover:bg-gray-300">Geri</button>}
            {step < wizardSteps.length - 1 && step > 0 && <button onClick={skip} className="px-4 py-2 bg-yellow-100 text-yellow-700 rounded hover:bg-yellow-200">Atla</button>}
            {step < wizardSteps.length - 1 ? (
              <button onClick={next} className="px-6 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">İleri</button>
            ) : (
              <button onClick={handleCreate} className="px-6 py-2 bg-green-500 text-white rounded hover:bg-green-600">İş Oluştur</button>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}

/* ---------- JobDetail Component ---------- */
function JobDetail(props) {
  const { job, back, updateJob, teamMembers = [], products = [], lazerSuppliers = [], suppliers = [], deleteJob, completeJob, onExportSupplier, onExportAll } = props;

  const [showInfoPanel, setShowInfoPanel] = useState(false);
  const [sidebarOpen, setSidebarOpen] = useState(true);
  const [activeTabId, setActiveTabId] = useState(job.tabs?.[0]?.id || null);
  const [newTabTitle, setNewTabTitle] = useState('');
  const [newTaskText, setNewTaskText] = useState('');
  const [expandedTaskId, setExpandedTaskId] = useState(null);
  const [editingTitleTaskId, setEditingTitleTaskId] = useState(null);
  const [dialog, setDialog] = useState(null); // ✅ Dialog state

  useEffect(() => {
    if(job.tabs && job.tabs.length && !activeTabId) setActiveTabId(job.tabs[0].id);
  }, [job]);

  if(!job) return <div className="bg-white rounded-lg p-6">İş bulunamadı.</div>;

  const setField = (field, value) => updateJob(prev => ({ ...prev, [field]: value }));

  const addTab = () => {
    if(!newTabTitle.trim()) return;
    const id = 'tab-' + Date.now() + '-' + Math.round(Math.random()*1000);
    const newTab = { id, title: newTabTitle.trim(), tasks: [], deletable: true };
    updateJob(prev => ({ ...prev, tabs: [...(prev.tabs||[]), newTab] }));
    setNewTabTitle('');
    setActiveTabId(id);
  };

  const removeTab = (tabId) => {
    const tab = job.tabs.find(t => t.id === tabId);
    if(tab && !tab.deletable) {
      setDialog({
        message: 'Bu sekme silinemez!',
        type: 'warning',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
      return;
    }
    setDialog({
      message: 'Bu sekmeyi silmek istediğinize emin misiniz?',
      type: 'warning',
      showCancel: true,
      onConfirm: () => {
        updateJob(prev => ({ ...prev, tabs: (prev.tabs||[]).filter(t => t.id !== tabId) }));
        if(activeTabId === tabId) setActiveTabId((job.tabs||[]).filter(t => t.id !== tabId)[0]?.id || null);
        setDialog(null);
      },
      onCancel: () => setDialog(null)
    });
  };

  const addTaskToTab = (tabId, text) => {
    if(!text.trim()) return;
    const task = { id: 'task-' + Date.now() + '-' + Math.round(Math.random()*1000), text: text.trim(), done:false, assignedTo:[], description:'', priority:'normal' };
    updateJob(prev => ({ ...prev, tabs: prev.tabs.map(t => t.id === tabId ? { ...t, tasks: [...t.tasks, task] } : t) }));
    setNewTaskText('');
    setExpandedTaskId(task.id);
  };

  const toggleTaskDone = (tabId, taskId) => {
    updateJob(prev => ({ ...prev, tabs: prev.tabs.map(t => t.id === tabId ? { ...t, tasks: t.tasks.map(ts => ts.id === taskId ? { ...ts, done: !ts.done } : ts) } : t) }));
  };

  const deleteTask = (tabId, taskId) => {
    updateJob(prev => ({ ...prev, tabs: prev.tabs.map(t => t.id === tabId ? { ...t, tasks: t.tasks.filter(ts => ts.id !== taskId) } : t) }));
  };

  const updateTask = (tabId, taskId, patch) => {
    updateJob(prev => ({ ...prev, tabs: prev.tabs.map(t => t.id === tabId ? { ...t, tasks: t.tasks.map(ts => ts.id === taskId ? { ...ts, ...patch } : ts) } : t) }));
  };

  const quickAssign = (tabId, taskId, member) => {
    if(!member) return;
    updateJob(prev => ({ ...prev, tabs: prev.tabs.map(t => {
      if(t.id !== tabId) return t;
      return { ...t, tasks: t.tasks.map(ts => {
        if(ts.id !== taskId) return ts;
        const set = new Set([...(ts.assignedTo||[]), member]);
        return { ...ts, assignedTo: Array.from(set) };
      })};
    })}));
  };

  const activeTab = (job.tabs || []).find(t => t.id === activeTabId) || null;

  const isGenelBilgiler = activeTab && activeTab.id.startsWith('tab-genel');
  const isMalzemeTab = activeTab && activeTab.id.startsWith('tab-malzeme');
  const isLazerTab = activeTab && activeTab.id.startsWith('tab-lazer');
  const isPlanlananTab = activeTab && activeTab.id.startsWith('tab-planlanan');


  return (
    <div className="bg-white rounded-lg shadow-sm overflow-hidden">
      <div className="p-4 border-b flex items-center justify-between">
        <div className="flex items-center gap-3">
          <button onClick={back} className="px-3 py-2 bg-gray-100 rounded-lg">← Geri</button>
          <h2 className="text-xl font-bold">{job.customerName || job.productType}</h2>
          <button onClick={() => setShowInfoPanel(true)} title="İş bilgilerini göster / düzenle" className="ml-2 px-2 py-1 bg-blue-50 rounded text-blue-600 flex items-center gap-1"><Icon type="info" /> Bilgi</button>
        </div>

        <div className="flex items-center gap-2">
          <div className="text-sm mr-4">Sipariş: <strong>{job.orderNumber}</strong></div>
          <button onClick={() => {
            setDialog({
              message: 'Bu işi tamamla?',
              type: 'info',
              showCancel: true,
              onConfirm: () => {
                completeJob();
                setDialog(null);
              },
              onCancel: () => setDialog(null)
            });
          }} className="px-3 py-2 bg-green-500 text-white rounded-lg">Tamamla</button>
          <button onClick={() => {
            setDialog({
              message: 'Bu işi silmek istediğinize emin misiniz?',
              type: 'warning',
              showCancel: true,
              onConfirm: () => {
                deleteJob();
                setDialog(null);
              },
              onCancel: () => setDialog(null)
            });
          }} className="px-3 py-2 bg-red-100 text-red-600 rounded-lg"><Icon type="trash" /></button>
        </div>
      </div>

      <div className="flex">
        <div className={`transition-all duration-200 ${sidebarOpen ? 'w-64' : 'w-12'} bg-gray-50 border-r p-3 flex flex-col`}>
          <div className="flex items-center justify-between mb-3">
            {sidebarOpen ? <h3 className="font-semibold">Sekmeler</h3> : <div className="text-center text-xs">📑</div>}
            <button onClick={() => setSidebarOpen(s => !s)} className="text-xs px-2 py-1 rounded bg-white/60">{sidebarOpen ? '◀' : '▶'}</button>
          </div>

          <div className="flex-1 overflow-auto space-y-2">
            {(job.tabs || []).map(tab => (
              <div key={tab.id} className={`flex items-center gap-2 p-2 rounded cursor-pointer ${activeTabId===tab.id? 'bg-white shadow':'hover:bg-gray-100'}`}>
                <button onClick={() => setActiveTabId(tab.id)} className="flex-1 text-left">
<div className="flex items-center justify-between">
                    <div className="text-sm truncate">{sidebarOpen ? tab.title : tab.title.split(' ')[0]}</div>
                  </div>
                </button>
                {sidebarOpen && tab.deletable !== false && (
                  <button title="Sil" onClick={() => removeTab(tab.id)} className="text-red-500 hover:text-red-700"><Icon type="trash" className="w-4 h-4"/></button>
                )}
              </div>
            ))}
          </div>

          {sidebarOpen && (
            <div className="mt-3">
              <input value={newTabTitle} onChange={e=>setNewTabTitle(e.target.value)} placeholder="Yeni sekme..." className="w-full px-2 py-1 text-sm border rounded mb-2" onKeyDown={(e)=>{if(e.key==='Enter')addTab()}}/>
              <button onClick={addTab} className="w-full px-3 py-2 bg-blue-500 text-white text-sm rounded">+ Sekme Ekle</button>
            </div>
          )}
        </div>

        <div className="flex-1 p-6 overflow-auto">
          {showInfoPanel && (
            <div className="fixed inset-0 z-50 flex items-center justify-center">
              <div className="absolute inset-0 bg-black/40" onClick={() => setShowInfoPanel(false)}></div>
              <div className="relative bg-white rounded-lg p-6 w-full max-w-3xl z-50 max-h-[80vh] overflow-auto">
                <div className="flex items-center justify-between mb-4">
                  <h3 className="font-semibold text-lg">İş Bilgileri</h3>
                  <button onClick={()=>setShowInfoPanel(false)} className="text-sm px-3 py-1 bg-gray-100 rounded">Kapat</button>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  <div>
                    <label className="text-xs text-gray-600 block mb-1">Müşteri</label>
                    <input value={job.customerName || ''} onChange={(e)=> setField('customerName', e.target.value)} className="w-full px-3 py-2 border rounded" />
                  </div>
                  <div>
                    <label className="text-xs text-gray-600 block mb-1">Telefon</label>
                    <input value={job.customerPhone || ''} onChange={(e)=> setField('customerPhone', e.target.value)} className="w-full px-3 py-2 border rounded" />
                  </div>
                  <div>
                    <label className="text-xs text-gray-600 block mb-1">Ürün Tipi</label>
                    <input value={job.productType || ''} onChange={(e)=> setField('productType', e.target.value)} className="w-full px-3 py-2 border rounded" />
                  </div>
                  <div>
                    <label className="text-xs text-gray-600 block mb-1">Sipariş No</label>
                    <input value={job.orderNumber || ''} readOnly className="w-full px-3 py-2 border rounded bg-gray-50" />
                  </div>
                  <div className="md:col-span-2">
                    <label className="text-xs text-gray-600 block mb-1">Adres</label>
                    <textarea value={job.customerAddress || ''} onChange={(e)=> setField('customerAddress', e.target.value)} className="w-full px-3 py-2 border rounded" rows="2"/>
                  </div>
                </div>

                <div className="mt-4 flex justify-end">
                  <button onClick={() => setShowInfoPanel(false)} className="px-4 py-2 bg-blue-500 text-white rounded">Tamam</button>
                </div>
              </div>
            </div>
          )}

          {activeTab ? (
            <>
              <div className="flex items-center justify-between mb-4">
                <h3 className="text-lg font-semibold">{activeTab.title}</h3>
              </div>

{isGenelBilgiler ? (
  <GenelBilgilerTab job={job} updateJob={updateJob} setDialog={setDialog} />
) : isMalzemeTab ? (
  <MalzemeTab job={job} updateJob={updateJob} suppliers={suppliers} onExportSupplier={onExportSupplier} onExportAll={onExportAll} />
) : isLazerTab ? (
  <LazerTab job={job} updateJob={updateJob} lazerSuppliers={lazerSuppliers} onExportSupplier={onExportSupplier} onExportAll={onExportAll} />
) : isPlanlananTab ? (
  <PlanlananIslerTab job={job} updateJob={updateJob} activeTab={activeTab} />
) : (
  <TasksTab
                  activeTab={activeTab} 
                  newTaskText={newTaskText} 
                  setNewTaskText={setNewTaskText} 
                  addTaskToTab={addTaskToTab} 
                  toggleTaskDone={toggleTaskDone} 
                  deleteTask={deleteTask} 
                  updateTask={updateTask} 
                  quickAssign={quickAssign} 
                  teamMembers={teamMembers}
                  expandedTaskId={expandedTaskId}
                  setExpandedTaskId={setExpandedTaskId}
                  editingTitleTaskId={editingTitleTaskId}
                  setEditingTitleTaskId={setEditingTitleTaskId}
                />
              )}
            </>
          ) : (
            <div className="text-gray-500 text-center py-12">Bir sekme seçin</div>
          )}
        </div>
      </div>
      {dialog && (
        <CustomDialog
          message={dialog.message}
          type={dialog.type}
          showCancel={dialog.showCancel}
          onConfirm={dialog.onConfirm}
          onCancel={dialog.onCancel || (() => setDialog(null))}
        />
      )}
    </div>
  );
}

/* ---------- Product Tree Manager ---------- */
function ProductTreeManager({ products, productTrees, setProductTrees }) {
  return (
    <div className="bg-white rounded-lg shadow-sm p-6">
      <h2 className="text-xl font-bold mb-4">🌳 Ürün Ağacı</h2>
      <p className="text-sm text-gray-600 mb-4">
        Her ürün tipi için gerekli malzemeleri tanımlayın.
      </p>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {products.map(p => (
          <div key={p} className="border rounded-lg p-4 hover:border-blue-400 transition">
            <div className="font-semibold text-lg mb-2">{p}</div>
            <div className="text-sm text-gray-500">
              {(productTrees[p] || []).length} malzeme tanımlı
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
/* ---------- Product Tree Page ---------- */
function ProductTreePage({ products, productTrees, setProductTrees, suppliers, lazerSuppliers, back }) {
  const [selectedProduct, setSelectedProduct] = useState('');
  const [items, setItems] = useState([]);
  const [newItem, setNewItem] = useState({ product: '', description: '', qty: 1, supplier: '', type: 'material' });
  const [showSupabaseModal, setShowSupabaseModal] = useState(false);
  const [supabaseProducts, setSupabaseProducts] = useState([]);
  const [supabaseSearch, setSupabaseSearch] = useState('');
  const [loadingSupabase, setLoadingSupabase] = useState(false);
  const [dialog, setDialog] = useState(null); // ✅ Dialog state
  
  const allSuppliers = [...suppliers.map(s => ({ name: s, type: 'material' })), ...lazerSuppliers.map(s => ({ name: s, type: 'lazer' }))];
  
  const loadProduct = async (productName) => {
    setSelectedProduct(productName);
    const supabaseItems = await loadProductTreeFromSupabase(productName);
    
    if(supabaseItems.length > 0) {
      setItems(supabaseItems);
    } else {
      setItems(productTrees[productName] || []);
    }
  };

  const loadSupabaseProducts = async () => {
    setLoadingSupabase(true);
    try {
      const data = await supabaseFetch('products?order=product_name.asc&limit=1000');
      setSupabaseProducts(data || []);
      setShowSupabaseModal(true);
    } catch(e) {
      console.error('Supabase ürünler yüklenemedi:', e);
      setDialog({
        message: '❌ Ürünler yüklenemedi!',
        type: 'error',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    } finally {
      setLoadingSupabase(false);
    }
  };

  const addFromSupabase = async (prod) => {
    if(!newItem.supplier) {
      setDialog({
        message: '❌ Lütfen önce tedarikçi seçin!',
        type: 'warning',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
      return;
    }
    
    const itemToAdd = {
      ...newItem,
      product: prod.product_name,
      description: newItem.description || `Stok: ${prod.quantity}`,
      id: Date.now()
    };
    
    const newItems = [...items, itemToAdd];
    setItems(newItems);
    
    const success = await saveProductTreeToSupabase(selectedProduct, newItems);
    
    if(success) {
      const newTrees = { ...productTrees, [selectedProduct]: newItems };
      setProductTrees(newTrees);
      lsSave('pts_product_trees', newTrees);
      setDialog({
        message: `✅ ${prod.product_name} eklendi!`,
        type: 'success',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    } else {
      setDialog({
        message: '⚠️ Ürün eklendi ama Supabase\'e kaydedilemedi!',
        type: 'warning',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    }
    
    setShowSupabaseModal(false);
    setSupabaseProducts([]);
    setSupabaseSearch('');
  };

  const addItem = async () => {
    if(!newItem.product.trim() || !newItem.supplier) {
      setDialog({
        message: '❌ Lütfen ürün adı ve tedarikçi seçin!',
        type: 'warning',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
      return;
    }
    
    const itemToAdd = { ...newItem, id: Date.now() };
    const newItems = [...items, itemToAdd];
    setItems(newItems);
    
    const success = await saveProductTreeToSupabase(selectedProduct, newItems);
    
    if(success) {
      const newTrees = { ...productTrees, [selectedProduct]: newItems };
      setProductTrees(newTrees);
      lsSave('pts_product_trees', newTrees);
    }
    
    setNewItem({ product: '', description: '', qty: 1, supplier: '', type: 'material' });
  };
  
  const deleteItem = async (itemId) => {
    setDialog({
      message: 'Bu malzemeyi sil?',
      type: 'warning',
      showCancel: true,
      onConfirm: async () => {
        const newItems = items.filter(i => i.id !== itemId);
        setItems(newItems);
        await saveProductTreeToSupabase(selectedProduct, newItems);
        const newTrees = { ...productTrees, [selectedProduct]: newItems };
        setProductTrees(newTrees);
        lsSave('pts_product_trees', newTrees);
        setDialog(null);
      },
      onCancel: () => setDialog(null)
    });
  };
  
  const updateItem = async (itemId, updates) => {
    const newItems = items.map(i => i.id === itemId ? { ...i, ...updates } : i);
    setItems(newItems);
    
    await saveProductTreeToSupabase(selectedProduct, newItems);
    
    const newTrees = { ...productTrees, [selectedProduct]: newItems };
    setProductTrees(newTrees);
    lsSave('pts_product_trees', newTrees);
  };
  
  return (
    <div className="space-y-6">
      <div className="flex items-center gap-4">
        <button onClick={back} className="px-4 py-2 bg-gray-100 rounded-lg">← Geri</button>
        <h2 className="text-2xl font-bold">🌳 Ürün Ağacı Yönetimi</h2>
      </div>
      
      {!selectedProduct ? (
        <div className="bg-white rounded-lg shadow-sm p-6">
          <h3 className="text-xl font-semibold mb-4">Ürün Seçin</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {products.map(p => (
              <button
                key={p}
                onClick={() => loadProduct(p)}
                className="p-6 border-2 border-gray-200 rounded-lg hover:border-blue-400 hover:bg-blue-50 transition text-left"
              >
                <div className="font-semibold text-lg">{p}</div>
                <div className="text-sm text-gray-500 mt-1">
                  {(productTrees[p] || []).length} malzeme tanımlı
                </div>
              </button>
            ))}
          </div>
        </div>
      ) : (
        <div className="space-y-6">
          <div className="bg-white rounded-lg shadow-sm p-6">
            <div className="flex items-center justify-between">
              <div>
                <button onClick={() => setSelectedProduct('')} className="text-blue-500 text-sm mb-2">← Ürün Listesine Dön</button>
                <h3 className="text-2xl font-bold">{selectedProduct}</h3>
                <p className="text-gray-500">{items.length} malzeme tanımlı</p>
              </div>
            </div>
          </div>
          
          <div className="bg-white rounded-lg shadow-sm p-6">
            <div className="flex items-center justify-between mb-4">
              <h4 className="font-semibold">Yeni Malzeme Ekle</h4>
              <button 
                onClick={loadSupabaseProducts}
                disabled={loadingSupabase}
                className="px-4 py-2 bg-purple-500 text-white rounded hover:bg-purple-600 disabled:opacity-50"
              >
                {loadingSupabase ? 'Yükleniyor...' : '📋 Supabase\'den Ekle'}
              </button>
            </div>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label className="text-sm font-medium text-gray-700 block mb-1">Malzeme Adı *</label>
                <input 
                  value={newItem.product}
                  onChange={(e) => setNewItem({...newItem, product: e.target.value})}
                  placeholder="Örn: Cam Panel"
                  className="w-full px-3 py-2 border rounded-lg"
                />
              </div>
              
              <div>
                <label className="text-sm font-medium text-gray-700 block mb-1">Açıklama</label>
                <input 
                  value={newItem.description}
                  onChange={(e) => setNewItem({...newItem, description: e.target.value})}
                  placeholder="Örn: 5mm Temperli Cam"
                  className="w-full px-3 py-2 border rounded-lg"
                />
              </div>
              
              <div>
                <label className="text-sm font-medium text-gray-700 block mb-1">Adet</label>
                <input 
                  type="number"
                  value={newItem.qty}
                  onChange={(e) => setNewItem({...newItem, qty: parseInt(e.target.value) || 1})}
                  min="1"
                  className="w-full px-3 py-2 border rounded-lg"
                />
              </div>
              
              <div>
                <label className="text-sm font-medium text-gray-700 block mb-1">Tedarikçi *</label>
                <select 
                  value={newItem.supplier}
                  onChange={(e) => {
                    const selected = allSuppliers.find(s => s.name === e.target.value);
                    setNewItem({...newItem, supplier: e.target.value, type: selected?.type || 'material'});
                  }}
                  className="w-full px-3 py-2 border rounded-lg"
                >
                  <option value="">Seçiniz...</option>
                  <optgroup label="Malzeme Tedarikçileri">
                    {suppliers.map(s => <option key={s} value={s}>{s}</option>)}
                  </optgroup>
                  <optgroup label="Lazer Kesim Firmaları">
                    {lazerSuppliers.map(s => <option key={s} value={s}>{s} (Lazer)</option>)}
                  </optgroup>
                </select>
              </div>
            </div>
            
            <button 
              onClick={addItem}
              className="mt-4 px-6 py-2 bg-green-500 text-white rounded-lg hover:bg-green-600"
            >
              + Ekle
            </button>
          </div>
          
          <div className="bg-white rounded-lg shadow-sm p-6">
            <h4 className="font-semibold mb-4">Malzeme Listesi</h4>
            {items.length === 0 ? (
              <p className="text-gray-500">Henüz malzeme eklenmemiş</p>
            ) : (
              <div className="space-y-3">
                {items.map(item => (
                  <ProductTreeItem 
                    key={item.id}
                    item={item}
                    onUpdate={(updates) => updateItem(item.id, updates)}
                    onDelete={() => deleteItem(item.id)}
                    suppliers={allSuppliers}
                  />
                ))}
              </div>
            )}
          </div>
        </div>
      )}
      
      {showSupabaseModal && (
        <div className="fixed inset-0 bg-black/40 flex items-center justify-center z-50 p-4">
          <div className="bg-white rounded-lg shadow-xl max-w-4xl w-full max-h-[80vh] overflow-hidden flex flex-col">
            <div className="p-4 border-b">
              <div className="flex items-center justify-between mb-3">
                <h3 className="font-semibold text-lg">Supabase Ürün Listesi ({supabaseProducts.length})</h3>
                <button 
                  onClick={() => {setShowSupabaseModal(false); setSupabaseProducts([]); setSupabaseSearch('');}} 
                  className="text-gray-500 hover:text-gray-700"
                >
                  ✕
                </button>
              </div>
              <div className="relative mb-3">
                <input 
                  value={supabaseSearch} 
                  onChange={e => setSupabaseSearch(e.target.value)} 
                  placeholder="Ürün ara..." 
                  className="w-full px-3 py-2 border rounded pl-10"
                />
                <Icon type="search" className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-gray-400"/>
              </div>
              
              <div className="bg-yellow-50 border border-yellow-200 rounded p-3 text-sm">
                <strong>⚠️ Dikkat:</strong> Ürün eklemeden önce aşağıdan <strong>Tedarikçi</strong> seçmeyi unutmayın!
              </div>
            </div>
            
            <div className="flex-1 overflow-auto p-4">
              <div className="grid grid-cols-1 gap-2">
                {supabaseProducts
                  .filter(prod => !supabaseSearch || prod.product_name.toLowerCase().includes(supabaseSearch.toLowerCase()))
                  .map(prod => (
                    <div key={prod.id} className="border rounded p-3 hover:bg-gray-50">
                      <div className="flex items-start justify-between gap-3">
                        <div className="flex-1">
                          <div className="font-medium">{prod.product_name}</div>
                          <div className="text-xs text-gray-500 mt-1">
                            Barkod: {prod.barcode}
                          </div>
                          <div className="text-xs text-gray-500 mt-1">
                            Stok: <span className={prod.quantity < 5 ? 'text-red-600 font-semibold' : 'text-green-600'}>{prod.quantity}</span> adet
                          </div>
                        </div>
                        <button 
                          onClick={() => addFromSupabase(prod)} 
                          className="px-4 py-2 bg-green-500 text-white rounded hover:bg-green-600"
                        >
                          Ekle
                        </button>
                      </div>
                    </div>
                  ))}
              </div>
            </div>
          </div>
        </div>
      )}
      {dialog && (
        <CustomDialog
          message={dialog.message}
          type={dialog.type}
          showCancel={dialog.showCancel}
          onConfirm={dialog.onConfirm}
          onCancel={dialog.onCancel || (() => setDialog(null))}
        />
      )}
    </div>
  );
}

function ProductTreeItem({ item, onUpdate, onDelete, suppliers }) {
  const [editing, setEditing] = useState(false);
  const [local, setLocal] = useState(item);
  
  const save = () => {
    onUpdate(local);
    setEditing(false);
  };
  
  return (
    <div className="border rounded-lg p-4">
      {editing ? (
        <div className="space-y-3">
          <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
            <input 
              value={local.product}
              onChange={(e) => setLocal({...local, product: e.target.value})}
              className="px-3 py-2 border rounded"
              placeholder="Malzeme adı"
            />
            <input 
              value={local.description}
              onChange={(e) => setLocal({...local, description: e.target.value})}
              className="px-3 py-2 border rounded"
              placeholder="Açıklama"
            />
            <input 
              type="number"
              value={local.qty}
              onChange={(e) => setLocal({...local, qty: parseInt(e.target.value) || 1})}
              className="px-3 py-2 border rounded"
              placeholder="Adet"
            />
            <select 
              value={local.supplier}
              onChange={(e) => {
                const selected = suppliers.find(s => s.name === e.target.value);
                setLocal({...local, supplier: e.target.value, type: selected?.type || 'material'});
              }}
              className="px-3 py-2 border rounded"
            >
              {suppliers.map(s => (
                <option key={s.name} value={s.name}>
                  {s.name} {s.type === 'lazer' ? '(Lazer)' : ''}
                </option>
              ))}
            </select>
          </div>
          <div className="flex gap-2">
            <button onClick={save} className="px-4 py-2 bg-green-500 text-white rounded">Kaydet</button>
            <button onClick={() => {setEditing(false); setLocal(item);}} className="px-4 py-2 bg-gray-200 rounded">İptal</button>
          </div>
        </div>
      ) : (
        <div className="flex items-center justify-between">
          <div className="flex-1">
            <div className="font-semibold">{item.product}</div>
            <div className="text-sm text-gray-600">{item.description || 'Açıklama yok'}</div>
            <div className="text-sm text-gray-500 mt-1">
              Adet: {item.qty} • Tedarikçi: {item.supplier} {item.type === 'lazer' ? '(Lazer)' : ''}
            </div>
          </div>
          <div className="flex gap-2">
            <button onClick={() => setEditing(true)} className="px-3 py-2 bg-blue-100 text-blue-600 rounded">
              <Icon type="edit" className="w-4 h-4" />
            </button>
            <button onClick={onDelete} className="px-3 py-2 bg-red-100 text-red-600 rounded">
              <Icon type="trash" className="w-4 h-4" />
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

/* ---------- Workflow Templates Page ---------- */
function WorkflowTemplatesPage({ products, workflowTemplates, setWorkflowTemplates, back }) {
  const [selectedProduct, setSelectedProduct] = useState('');
  const [tasks, setTasks] = useState([]);
  const [newTask, setNewTask] = useState('');
  const [dialog, setDialog] = useState(null);

  const loadProduct = async (productName) => {
    setSelectedProduct(productName);
    const supabaseTasks = await loadWorkflowTemplateFromSupabase(productName);
    
    if(supabaseTasks.length > 0) {
      setTasks(supabaseTasks);
      // Supabase'den gelen verileri workflowTemplates state'ine de kaydet
      const newTemplates = { ...workflowTemplates, [productName]: supabaseTasks };
      setWorkflowTemplates(newTemplates);
    } else {
      setTasks(workflowTemplates[productName] || []);
    }
  };

  const addTask = async () => {
    if(!newTask.trim()) {
      setDialog({
        message: '❌ Lütfen görev metni girin!',
        type: 'warning',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
      return;
    }
    
    const taskToAdd = {
      id: Date.now(),
      text: newTask.trim(),
      order: tasks.length
    };
    
    const newTasks = [...tasks, taskToAdd];
    setTasks(newTasks);
    
    const result = await saveWorkflowTemplateToSupabase(selectedProduct, newTasks);
    
    if(result) {
      // Supabase'den dönen güncel id'leri kullan
      const updatedTasks = Array.isArray(result) ? result : newTasks;
      setTasks(updatedTasks);
      const newTemplates = { ...workflowTemplates, [selectedProduct]: updatedTasks };
      setWorkflowTemplates(newTemplates);
      setDialog({
        message: `✅ Görev eklendi ve Supabase'ye kaydedildi!`,
        type: 'success',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    } else {
      setDialog({
        message: '⚠️ Görev eklendi ama Supabase\'e kaydedilemedi!',
        type: 'warning',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    }
    
    setNewTask('');
  };

  const deleteTask = async (taskId) => {
    setDialog({
      message: 'Bu görevi sil?',
      type: 'warning',
      showCancel: true,
      onConfirm: async () => {
        const newTasks = tasks.filter(t => t.id !== taskId);
        setTasks(newTasks);
        const result = await saveWorkflowTemplateToSupabase(selectedProduct, newTasks);
        if(result) {
          const updatedTasks = Array.isArray(result) ? result : newTasks;
          setTasks(updatedTasks);
          const newTemplates = { ...workflowTemplates, [selectedProduct]: updatedTasks };
          setWorkflowTemplates(newTemplates);
        }
        setDialog(null);
      },
      onCancel: () => setDialog(null)
    });
  };

  const updateTask = async (taskId, newText) => {
    if(!newText.trim()) return;
    
    const newTasks = tasks.map(t => t.id === taskId ? { ...t, text: newText.trim() } : t);
    setTasks(newTasks);
    
    const result = await saveWorkflowTemplateToSupabase(selectedProduct, newTasks);
    if(result) {
      const updatedTasks = Array.isArray(result) ? result : newTasks;
      setTasks(updatedTasks);
      const newTemplates = { ...workflowTemplates, [selectedProduct]: updatedTasks };
      setWorkflowTemplates(newTemplates);
    }
  };

  const moveTask = async (taskId, direction) => {
    const index = tasks.findIndex(t => t.id === taskId);
    if((direction === 'up' && index === 0) || (direction === 'down' && index === tasks.length - 1)) return;
    
    const newTasks = [...tasks];
    const targetIndex = direction === 'up' ? index - 1 : index + 1;
    [newTasks[index], newTasks[targetIndex]] = [newTasks[targetIndex], newTasks[index]];
    
    // Sıralamayı güncelle
    newTasks.forEach((t, i) => { t.order = i; });
    
    setTasks(newTasks);
    const result = await saveWorkflowTemplateToSupabase(selectedProduct, newTasks);
    if(result) {
      const updatedTasks = Array.isArray(result) ? result : newTasks;
      setTasks(updatedTasks);
      const newTemplates = { ...workflowTemplates, [selectedProduct]: updatedTasks };
      setWorkflowTemplates(newTemplates);
    }
  };

  return (
    <div className="space-y-6">
      <div className="flex items-center gap-4">
        <button onClick={back} className="px-4 py-2 bg-gray-100 rounded-lg">← Geri</button>
        <h2 className="text-2xl font-bold">📋 İş Akış Şablonları Yönetimi</h2>
      </div>
      
      {!selectedProduct ? (
        <div className="bg-white rounded-lg shadow-sm p-6">
          <h3 className="text-xl font-semibold mb-4">Ürün Seçin</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {products && products.length > 0 ? products.map(p => (
              <button
                key={p}
                onClick={() => loadProduct(p)}
                className="p-6 border-2 border-gray-200 rounded-lg hover:border-blue-400 hover:bg-blue-50 transition text-left"
              >
                <div className="font-semibold text-lg">{p}</div>
                <div className="text-sm text-gray-500 mt-1">
                  {(workflowTemplates[p] || []).length} görev tanımlı
                </div>
              </button>
            )) : (
              <div className="col-span-full text-center py-8 text-gray-500">
                Henüz ürün tipi tanımlanmamış. Lütfen önce Ayarlar'dan ürün tipi ekleyin.
              </div>
            )}
          </div>
        </div>
      ) : (
        <div className="space-y-6">
          <div className="bg-white rounded-lg shadow-sm p-6">
            <div className="flex items-center justify-between">
              <div>
                <button onClick={() => setSelectedProduct('')} className="text-blue-500 text-sm mb-2">← Ürün Listesine Dön</button>
                <h3 className="text-2xl font-bold">{selectedProduct}</h3>
                <p className="text-gray-500">{tasks.length} görev tanımlı</p>
              </div>
            </div>
          </div>
          
          <div className="bg-white rounded-lg shadow-sm p-6">
            <h4 className="font-semibold mb-4">Yeni Görev Ekle</h4>
            <div className="flex gap-2">
              <input 
                value={newTask}
                onChange={(e) => setNewTask(e.target.value)}
                placeholder="Görev metni..."
                className="flex-1 px-3 py-2 border rounded-lg"
                onKeyDown={(e) => { if(e.key === 'Enter') addTask(); }}
              />
              <button onClick={addTask} className="px-4 py-2 bg-green-500 text-white rounded-lg">
                <Icon type="plus" className="w-5 h-5" />
              </button>
            </div>
          </div>

          <div className="bg-white rounded-lg shadow-sm p-6">
            <h4 className="font-semibold mb-4">Görevler</h4>
            {tasks.length === 0 ? (
              <div className="text-center py-8 text-gray-500">Henüz görev eklenmemiş.</div>
            ) : (
              <div className="space-y-2">
                {tasks.map((task, index) => (
                  <WorkflowTaskItem
                    key={task.id}
                    task={task}
                    index={index}
                    total={tasks.length}
                    onUpdate={(newText) => updateTask(task.id, newText)}
                    onDelete={() => deleteTask(task.id)}
                    onMove={(dir) => moveTask(task.id, dir)}
                  />
                ))}
              </div>
            )}
          </div>
        </div>
      )}

      {dialog && (
        <CustomDialog
          message={dialog.message}
          type={dialog.type}
          showCancel={dialog.showCancel}
          onConfirm={dialog.onConfirm}
          onCancel={dialog.onCancel || (() => setDialog(null))}
        />
      )}
    </div>
  );
}

/* ---------- Workflow Task Item ---------- */
function WorkflowTaskItem({ task, index, total, onUpdate, onDelete, onMove }) {
  const [editing, setEditing] = useState(false);
  const [localText, setLocalText] = useState(task.text);

  useEffect(() => {
    setLocalText(task.text);
  }, [task.text]);

  const save = () => {
    if(localText.trim()) {
      onUpdate(localText);
      setEditing(false);
    }
  };

  return (
    <div className="flex items-center gap-2 p-3 border rounded-lg hover:bg-gray-50">
      <div className="flex items-center gap-1">
        <button 
          onClick={() => onMove('up')} 
          disabled={index === 0}
          className="px-2 py-1 bg-gray-100 rounded disabled:opacity-50"
          title="Yukarı taşı"
        >
          <Icon type="chevron-up" className="w-4 h-4" />
        </button>
        <button 
          onClick={() => onMove('down')} 
          disabled={index === total - 1}
          className="px-2 py-1 bg-gray-100 rounded disabled:opacity-50"
          title="Aşağı taşı"
        >
          <Icon type="chevron-down" className="w-4 h-4" />
        </button>
      </div>
      
      <div className="flex-1">
        {editing ? (
          <input 
            value={localText}
            onChange={(e) => setLocalText(e.target.value)}
            onBlur={save}
            onKeyDown={(e) => { if(e.key === 'Enter') save(); else if(e.key === 'Escape') { setEditing(false); setLocalText(task.text); } }}
            className="w-full px-3 py-2 border rounded"
            autoFocus
          />
        ) : (
          <div className="px-3 py-2" onClick={() => setEditing(true)}>
            <span className="text-gray-500 mr-2">#{index + 1}</span>
            <span className="cursor-pointer hover:text-blue-600">{task.text}</span>
          </div>
        )}
      </div>
      
      <div className="flex gap-2">
        {!editing && (
          <button onClick={() => setEditing(true)} className="px-3 py-2 bg-blue-100 text-blue-600 rounded">
            <Icon type="edit" className="w-4 h-4" />
          </button>
        )}
        <button onClick={onDelete} className="px-3 py-2 bg-red-100 text-red-600 rounded">
          <Icon type="trash" className="w-4 h-4" />
        </button>
      </div>
    </div>
  );
}

/* ---------- Genel Bilgiler Tab ---------- */
function GenelBilgilerTab({ job, updateJob, setDialog }) {
  const [newExtra, setNewExtra] = useState('');
  const [editingWizard, setEditingWizard] = useState(false);
  const [wizardDraft, setWizardDraft] = useState(job.cabinet || {});
  const [extraInput, setExtraInput] = useState('');
  
  useEffect(() => {
    setWizardDraft(job.cabinet || {});
  }, [job.cabinet]);
  
  const addExtra = () => {
    if(!newExtra.trim()) return;
    updateJob(prev => ({ ...prev, cabinetExtras: [...(prev.cabinetExtras||[]), newExtra.trim()] }));
    setNewExtra('');
  };

  const removeExtra = (idx) => {
    updateJob(prev => ({ ...prev, cabinetExtras: (prev.cabinetExtras||[]).filter((_,i)=>i!==idx) }));
  };

  const saveWizard = () => {
    updateJob(prev => ({ ...prev, cabinet: wizardDraft }));
    setEditingWizard(false);
    if(setDialog) {
      setDialog({
        message: 'Dolap bilgileri güncellendi!',
        type: 'success',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    }
  };

  const addWizardExtra = () => {
    if(!extraInput.trim()) return;
    setWizardDraft(prev => ({ ...prev, extras: [...(prev.extras||[]), extraInput.trim()] }));
    setExtraInput('');
  };

  const removeWizardExtra = (idx) => {
    setWizardDraft(prev => ({ ...prev, extras: (prev.extras||[]).filter((_,i)=>i!==idx) }));
  };

  return (
    <div className="space-y-6">
      <div>
        <div className="flex items-center justify-between mb-3">
          <h4 className="font-semibold">Dolap Özeti</h4>
          {!editingWizard && (
            <button onClick={() => setEditingWizard(true)} className="px-3 py-2 bg-blue-500 text-white rounded text-sm flex items-center gap-2">
              <Icon type="edit" className="w-4 h-4"/> Düzenle
            </button>
          )}
        </div>

        {editingWizard ? (
          <div className="bg-blue-50 p-4 rounded space-y-4">
            <div>
              <label className="text-sm font-medium mb-2 block">Soğutma Tipi</label>
              <div className="flex gap-2">
                <label className={`flex-1 px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.coolingType==='üfleme'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="cool-edit" value="üfleme" checked={wizardDraft.coolingType==='üfleme'} onChange={e=>setWizardDraft({...wizardDraft, coolingType:e.target.value})} className="mr-2"/>
                  Üfleme
                </label>
                <label className={`flex-1 px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.coolingType==='statik'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="cool-edit" value="statik" checked={wizardDraft.coolingType==='statik'} onChange={e=>setWizardDraft({...wizardDraft, coolingType:e.target.value})} className="mr-2"/>
                  Statik
                </label>
                <button onClick={()=>setWizardDraft({...wizardDraft, coolingType:''})} className="px-3 py-2 bg-gray-200 rounded text-sm">Temizle</button>
              </div>
            </div>

            <div>
              <label className="text-sm font-medium mb-2 block">Kapı Türü</label>
              <div className="flex gap-2">
                <label className={`flex-1 px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.doorType==='çarpma'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="door-edit" value="çarpma" checked={wizardDraft.doorType==='çarpma'} onChange={e=>setWizardDraft({...wizardDraft, doorType:e.target.value})} className="mr-2"/>
                  Çarpma
                </label>
                <label className={`flex-1 px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.doorType==='sürgü'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="door-edit" value="sürgü" checked={wizardDraft.doorType==='sürgü'} onChange={e=>setWizardDraft({...wizardDraft, doorType:e.target.value})} className="mr-2"/>
                  Sürgü
                </label>
                <button onClick={()=>setWizardDraft({...wizardDraft, doorType:''})} className="px-3 py-2 bg-gray-200 rounded text-sm">Temizle</button>
              </div>
            </div>

            <div>
              <label className="text-sm font-medium mb-2 block">Cam Tipi</label>
              <div className="flex gap-2">
                <label className={`px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.glassType==='dik'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="glass-edit" value="dik" checked={wizardDraft.glassType==='dik'} onChange={e=>setWizardDraft({...wizardDraft, glassType:e.target.value})} className="mr-2"/>
                  Dik
                </label>
                <label className={`px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.glassType==='oval'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="glass-edit" value="oval" checked={wizardDraft.glassType==='oval'} onChange={e=>setWizardDraft({...wizardDraft, glassType:e.target.value})} className="mr-2"/>
                  Oval
                </label>
                <label className={`px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.glassType==='eğik'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="glass-edit" value="eğik" checked={wizardDraft.glassType==='eğik'} onChange={e=>setWizardDraft({...wizardDraft, glassType:e.target.value})} className="mr-2"/>
                  Eğik
                </label>
                <button onClick={()=>setWizardDraft({...wizardDraft, glassType:''})} className="px-3 py-2 bg-gray-200 rounded text-sm">Temizle</button>
              </div>
            </div>

            <div>
              <label className="text-sm font-medium mb-2 block">Kasa Tipi</label>
              <div className="flex gap-2 mb-2">
                <label className={`px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.caseType==='boyalı'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="case-edit" value="boyalı" checked={wizardDraft.caseType==='boyalı'} onChange={e=>setWizardDraft({...wizardDraft, caseType:e.target.value, caseTypeCustom:''})} className="mr-2"/>
                  Boyalı
                </label>
                <label className={`px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.caseType==='paslanmaz'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="case-edit" value="paslanmaz" checked={wizardDraft.caseType==='paslanmaz'} onChange={e=>setWizardDraft({...wizardDraft, caseType:e.target.value, caseTypeCustom:''})} className="mr-2"/>
                  Paslanmaz
                </label>
                <label className={`px-3 py-2 border-2 rounded cursor-pointer ${wizardDraft.caseType==='özel'?'bg-white border-blue-400':'border-gray-300'}`}>
                  <input type="radio" name="case-edit" value="özel" checked={wizardDraft.caseType==='özel'} onChange={e=>setWizardDraft({...wizardDraft, caseType:e.target.value})} className="mr-2"/>
                  Özel
                </label>
                <button onClick={()=>setWizardDraft({...wizardDraft, caseType:'', caseTypeCustom:''})} className="px-3 py-2 bg-gray-200 rounded text-sm">Temizle</button>
              </div>
              {wizardDraft.caseType === 'özel' && (
                <input value={wizardDraft.caseTypeCustom || ''} onChange={e=>setWizardDraft({...wizardDraft, caseTypeCustom:e.target.value})} placeholder="Özel kasa açıklaması" className="w-full px-3 py-2 border rounded"/>
              )}
            </div>

            <div>
              <label className="text-sm font-medium mb-2 block">Wizard Ekstraları</label>
              <div className="flex gap-2 mb-2">
                <input value={extraInput} onChange={e=>setExtraInput(e.target.value)} placeholder="Ekstra özellik..." className="flex-1 px-3 py-2 border rounded" onKeyDown={(e)=>{if(e.key==='Enter')addWizardExtra()}}/>
                <button onClick={addWizardExtra} className="px-3 py-2 bg-green-500 text-white rounded">Ekle</button>
              </div>
              <div className="space-y-1">
                {(wizardDraft.extras||[]).map((ex,i)=>(
                  <div key={i} className="flex items-center justify-between bg-white p-2 rounded text-sm">
                    <span>{ex}</span>
                    <button onClick={()=>removeWizardExtra(i)} className="text-red-500 text-xs">Sil</button>
                  </div>
                ))}
              </div>
            </div>

            <div className="flex gap-2 justify-end pt-2">
              <button onClick={()=>{setEditingWizard(false); setWizardDraft(job.cabinet||{});}} className="px-4 py-2 bg-gray-200 rounded">İptal</button>
              <button onClick={saveWizard} className="px-4 py-2 bg-green-500 text-white rounded">Kaydet</button>
            </div>
          </div>
        ) : (
          job.cabinet ? (
            <div className="bg-gray-50 p-4 rounded space-y-2 text-sm">
              <div><strong>Soğutma:</strong> {job.cabinet.coolingType || 'Belirtilmedi'}</div>
              <div><strong>Kapı:</strong> {job.cabinet.doorType || 'Belirtilmedi'}</div>
              <div><strong>Cam:</strong> {job.cabinet.glassType || 'Belirtilmedi'}</div>
              <div><strong>Kasa:</strong> {job.cabinet.caseType === 'özel' ? job.cabinet.caseTypeCustom || 'Özel' : (job.cabinet.caseType || 'Belirtilmedi')}</div>
              {job.cabinet.extras && job.cabinet.extras.length > 0 && (
                <div>
                  <strong>Wizard Ekstraları:</strong>
                  <ul className="list-disc ml-5 mt-1">
                    {job.cabinet.extras.map((ex,i) => <li key={i}>{ex}</li>)}
                  </ul>
                </div>
              )}
            </div>
          ) : (
            <div className="text-sm text-gray-500">Dolap bilgisi henüz girilmedi.</div>
          )
        )}
      </div>

      <div>
        <h4 className="font-semibold mb-3">Ekstra Özellikler & Notlar</h4>
        <div className="flex gap-2 mb-3">
          <input value={newExtra} onChange={e=>setNewExtra(e.target.value)} placeholder="Ekstra özellik veya not ekle..." className="flex-1 px-3 py-2 border rounded" onKeyDown={(e)=>{if(e.key==='Enter')addExtra()}}/>
          <button onClick={addExtra} className="px-4 py-2 bg-green-500 text-white rounded">Ekle</button>
        </div>
        <div className="space-y-2">
          {(job.cabinetExtras||[]).map((ex, i) => (
            <div key={i} className="flex items-center justify-between bg-gray-50 p-3 rounded">
              <span>{ex}</span>
              <button onClick={() => removeExtra(i)} className="text-red-500 px-2">Sil</button>
            </div>
          ))}
        </div>
      </div>

      <div>
        <label className="font-semibold mb-2 block">Genel Notlar</label>
        <textarea value={job.cabinetNotes || ''} onChange={(e)=>updateJob(prev=>({...prev, cabinetNotes:e.target.value}))} className="w-full px-3 py-2 border rounded" rows="4" placeholder="Genel notlar buraya yazılabilir..."/>
      </div>
    </div>
  );
}

/* ---------- Malzeme Tab ---------- */
function MalzemeTab({ job, updateJob, suppliers, onExportSupplier, onExportAll }) {
  const [collapsedSuppliers, setCollapsedSuppliers] = React.useState(
    suppliers.reduce((acc, s) => ({ ...acc, [s]: true }), {})
  );
  
  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between mb-4">
        <div className="text-sm text-gray-700">Tedarikçi bazlı malzeme listesi</div>
        <button onClick={() => onExportAll(job.supplierOrders || {}, job, 'Malzeme')} className="px-4 py-2 bg-indigo-600 text-white rounded flex items-center gap-2"><Icon type="download" /> Tümünü İndir (Excel)</button>
      </div>

      {suppliers.map(supplier => {
        const items = (job.supplierOrders && job.supplierOrders[supplier]) || [];
        const isCollapsed = collapsedSuppliers[supplier];
        
        return (
          <SupplierSection 
            key={supplier}
            supplier={supplier}
            items={items}
            isCollapsed={isCollapsed}
            onToggle={() => setCollapsedSuppliers(prev => ({ ...prev, [supplier]: !prev[supplier] }))}
            job={job}
            updateJob={updateJob}
            onExport={() => onExportSupplier(supplier, items, job)}
            field="supplierOrders"
          />
        );
      })}
    </div>
  );
}

/* ---------- Lazer Tab ---------- */
function LazerTab({ job, updateJob, lazerSuppliers, onExportSupplier, onExportAll }) {
  const [collapsedSuppliers, setCollapsedSuppliers] = React.useState(
    lazerSuppliers.reduce((acc, s) => ({ ...acc, [s]: true }), {})
  );
  
  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between mb-4">
        <div className="text-sm text-gray-700">Lazer kesim firmaları - malzeme listesi</div>
        <button onClick={() => onExportAll(job.lazerOrders || {}, job, 'Lazer')} className="px-4 py-2 bg-purple-600 text-white rounded flex items-center gap-2"><Icon type="download" /> Tümünü İndir (Excel)</button>
      </div>

      {lazerSuppliers.map(supplier => {
        const items = (job.lazerOrders && job.lazerOrders[supplier]) || [];
        const isCollapsed = collapsedSuppliers[supplier];
        
        return (
          <SupplierSection 
            key={supplier}
            supplier={supplier}
            items={items}
            isCollapsed={isCollapsed}
            onToggle={() => setCollapsedSuppliers(prev => ({ ...prev, [supplier]: !prev[supplier] }))}
            job={job}
            updateJob={updateJob}
            onExport={() => onExportSupplier(supplier, items, job)}
            field="lazerOrders"
          />
        );
      })}
    </div>
  );
}

/* ---------- Supplier Section (Reusable) ---------- */
function SupplierSection({ supplier, items, job, updateJob, onExport, field, isCollapsed, onToggle }) {
  return (
    <div className="bg-white border rounded-lg p-4">
      <div className="flex items-start justify-between mb-3">
        <button 
          onClick={onToggle}
          className="flex-1 text-left flex items-center gap-2 hover:bg-gray-50 -m-2 p-2 rounded"
        >
          <span className="text-gray-400">{isCollapsed ? '▶' : '▼'}</span>
          <div>
            <h4 className="font-semibold text-lg">{supplier}</h4>
            <div className="text-sm text-gray-500">{items.length} ürün</div>
          </div>
        </button>
        {!isCollapsed && (
          <button onClick={onExport} className="px-3 py-2 bg-green-600 text-white rounded flex items-center gap-2"><Icon type="download" /> Excel</button>
        )}
      </div>

      {!isCollapsed && (
        <>
          <div className="space-y-2">
        {items.length === 0 ? (
          <div className="text-sm text-gray-500">Henüz ürün eklenmedi</div>
        ) : items.map((item, idx) => (
          <SupplierItemRow 
            key={idx}
            item={item}
            job={job}
            onUpdate={(newItem) => {
              updateJob(prev => {
                const orders = { ...(prev[field] || {}) };
                orders[supplier] = orders[supplier].map((it, i) => i === idx ? { ...it, ...newItem } : it);
                return { ...prev, [field]: orders };
              });
            }}
            onDelete={() => {
              updateJob(prev => {
                const orders = { ...(prev[field] || {}) };
                orders[supplier] = orders[supplier].filter((_, i) => i !== idx);
                return { ...prev, [field]: orders };
              });
            }}
          />
        ))}
      </div>

          <AddSupplierProduct 
            supplier={supplier}
            job={job}
            onAdd={(prod) => {
              updateJob(prev => {
                const orders = { ...(prev[field] || {}) };
                if(!orders[supplier]) orders[supplier] = [];
                orders[supplier] = [prod, ...orders[supplier]];
                return { ...prev, [field]: orders };
              });
            }}
          />
        </>
      )}
    </div>
  );
}

/* ---------- Supplier Item Row ---------- */
function SupplierItemRow({ item, onUpdate, onDelete, job }) {
  const [editing, setEditing] = useState(false);
  const [local, setLocal] = useState(item);
  const [useStock, setUseStock] = useState(false);
  const [showConfirm, setShowConfirm] = useState(false);
  const [confirmAction, setConfirmAction] = useState(null);
  const [supabaseProduct, setSupabaseProduct] = useState(null);
  const [loadingProduct, setLoadingProduct] = useState(false);
  
  useEffect(() => setLocal(item), [item]);

  // Supabase'den ürün bilgisini yükle
  useEffect(() => {
    const loadProduct = async () => {
      if (!item.product || !job) return;
      setLoadingProduct(true);
      try {
        const data = await supabaseFetch(`products?product_name=eq.${encodeURIComponent(item.product)}&limit=1`);
        if (data && data.length > 0) {
          setSupabaseProduct(data[0]);
          // Eğer item'da useStock bilgisi varsa onu kullan
          if (item.useStock !== undefined) {
            setUseStock(item.useStock);
          }
        }
      } catch (e) {
        console.error('Ürün bilgisi yüklenemedi:', e);
      } finally {
        setLoadingProduct(false);
      }
    };
    loadProduct();
  }, [item.product, job]);

  const handleStockToggle = async (e) => {
    if (!supabaseProduct) return;
    
    const checked = e.target.checked;
    
    if(checked) {
      if(supabaseProduct.quantity < 1) {
        setConfirmAction({
          type: 'insufficient',
          message: `Yetersiz stok!\n\nÜrün: ${supabaseProduct.product_name}\nMevcut Stok: ${supabaseProduct.quantity} adet\n\nStok miktarı yetersiz.`
        });
        setShowConfirm(true);
        return;
      }
      
      setConfirmAction({
        type: 'decrease',
        message: `Stoktan eksiltilecek:\n\nÜrün: ${supabaseProduct.product_name}\nMevcut Stok: ${supabaseProduct.quantity} adet\nEksilecek: 1 adet\nKalan: ${supabaseProduct.quantity - 1} adet\n\nDevam edilsin mi?`,
        onConfirm: async () => {
          const jobName = job.orderNumber || job.customerName || job.id;
          const success = await updateStock(supabaseProduct.id, supabaseProduct.quantity - 1, supabaseProduct.quantity, jobName, 'azaltma');
          
          if(success) {
            setUseStock(true);
            supabaseProduct.quantity = supabaseProduct.quantity - 1;
            setSupabaseProduct({...supabaseProduct});
            onUpdate({...local, useStock: true});
            setConfirmAction({
              type: 'success',
              message: `✅ Stoktan 1 adet düşüldü.\nKalan: ${supabaseProduct.quantity} adet`
            });
            setShowConfirm(true);
          } else {
            setConfirmAction({
              type: 'error',
              message: '❌ Stok güncellenemedi!\n\nİnternet bağlantınızı kontrol edin.'
            });
            setShowConfirm(true);
          }
        }
      });
      setShowConfirm(true);
    } else {
      setConfirmAction({
        type: 'increase',
        message: `Stoğa geri eklenecek:\n\nÜrün: ${supabaseProduct.product_name}\nŞu anki Stok: ${supabaseProduct.quantity} adet\nEklenecek: 1 adet\nYeni Stok: ${supabaseProduct.quantity + 1} adet\n\nDevam edilsin mi?`,
        onConfirm: async () => {
          const jobName = job.orderNumber || job.customerName || job.id;
          const success = await updateStock(supabaseProduct.id, supabaseProduct.quantity + 1, supabaseProduct.quantity, jobName, 'ekleme');
          
          if(success) {
            setUseStock(false);
            supabaseProduct.quantity = supabaseProduct.quantity + 1;
            setSupabaseProduct({...supabaseProduct});
            onUpdate({...local, useStock: false});
            setConfirmAction({
              type: 'success',
              message: `✅ Stoğa 1 adet geri eklendi.\nYeni stok: ${supabaseProduct.quantity} adet`
            });
            setShowConfirm(true);
          } else {
            setConfirmAction({
              type: 'error',
              message: '❌ Stok güncellenemedi!\n\nİnternet bağlantınızı kontrol edin.'
            });
            setShowConfirm(true);
          }
        }
      });
      setShowConfirm(true);
    }
  };

  return (
    <>
      <div className="flex items-center gap-2 bg-gray-50 p-3 rounded">
        {editing ? (
          <div className="flex-1 grid grid-cols-3 gap-2">
            <input className="px-2 py-1 border rounded" placeholder="Ürün" value={local.product || ''} onChange={(e)=>setLocal({...local, product:e.target.value})} />
            <input className="px-2 py-1 border rounded" placeholder="Açıklama" value={local.description || ''} onChange={(e)=>setLocal({...local, description:e.target.value})} />
            <input type="number" className="px-2 py-1 border rounded" placeholder="Adet" value={local.qty || 1} onChange={(e)=>setLocal({...local, qty: Number(e.target.value)})} />
          </div>
        ) : (
          <div className="flex-1 grid grid-cols-3 gap-2 items-center">
            <div className="font-medium">{item.product}</div>
            <div className="text-sm text-gray-600">{item.description}</div>
            <div className="text-sm">Adet: <strong>{item.qty || 1}</strong></div>
          </div>
        )}

        <div className="flex gap-2 items-center">
          {supabaseProduct && !editing && (
            <label className="flex items-center gap-2 text-xs cursor-pointer px-2 py-1 bg-gray-50 rounded border hover:bg-gray-100">
              <input 
                type="checkbox" 
                checked={useStock} 
                onChange={handleStockToggle}
                className="w-4 h-4"
                disabled={loadingProduct}
              />
              <span className={useStock ? 'text-green-600 font-semibold' : 'text-gray-600'}>
                {useStock ? 'Stoktan Kullanıldı' : 'Stoktan Kullan'}
              </span>
            </label>
          )}
          {editing ? (
            <button onClick={() => { setEditing(false); onUpdate(local); }} className="px-3 py-1 bg-green-500 text-white rounded text-sm">Kaydet</button>
          ) : (
            <button onClick={() => setEditing(true)} className="px-2 py-1 bg-yellow-100 rounded text-sm">Düzenle</button>
          )}
          <button onClick={onDelete} className="px-2 py-1 bg-red-100 text-red-600 rounded"><Icon type="trash" className="w-4 h-4"/></button>
        </div>
      </div>

      {showConfirm && (
        <CustomDialog
          message={confirmAction.message}
          onConfirm={() => {
            if(confirmAction.onConfirm) {
              confirmAction.onConfirm();
            } else {
              setShowConfirm(false);
            }
          }}
          onCancel={() => setShowConfirm(false)}
          showCancel={confirmAction.type === 'decrease' || confirmAction.type === 'increase'}
        />
      )}
    </>
  );
}

/* ✅ ---------- UPDATED: Add Supplier Product with Supabase ---------- */
function AddSupplierProduct({ supplier, job, onAdd }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [loading, setLoading] = useState(false);
  const [showForm, setShowForm] = useState(false);
  const [manualProduct, setManualProduct] = useState({ product: '', description: '', qty: 1 });
  const [showSaveToSupabase, setShowSaveToSupabase] = useState(false);
  const [showAllProducts, setShowAllProducts] = useState(false);
  const [allProducts, setAllProducts] = useState([]);
  const [dialog, setDialog] = useState(null); // ✅ Dialog state

  const loadAllProducts = async () => {
    setLoading(true);
    try {
      const data = await supabaseFetch('products?order=product_name.asc&limit=1000');
      setAllProducts(data || []);
      setShowAllProducts(true);
    } catch(e) {
      console.error('Supabase load all error:', e);
      setDialog({
        message: 'Ürünler yüklenemedi!',
        type: 'error',
        showCancel: false,
        onConfirm: () => setDialog(null)
      });
    } finally {
      setLoading(false);
    }
  };

  const addManual = async () => {
    if(!manualProduct.product.trim()) return;
    
    onAdd(manualProduct);
    
    if(showSaveToSupabase) {
      try {
        await supabaseFetch('products', {
          method: 'POST',
          body: JSON.stringify({
            product_name: manualProduct.product.trim(),
            barcode: 'MANUAL-' + Date.now(),
            quantity: 0,
            added_by: 'Web'
          })
        });
        setDialog({
          message: 'Ürün stok listesine eklendi!',
          type: 'success',
          showCancel: false,
          onConfirm: () => setDialog(null)
        });
      } catch(e) {
        setDialog({
          message: 'Supabase\'e eklenirken hata: ' + e.message,
          type: 'error',
          showCancel: false,
          onConfirm: () => setDialog(null)
        });
      }
    }
    
    setManualProduct({ product: '', description: '', qty: 1 });
    setShowForm(false);
    setShowSaveToSupabase(false);
  };

  return (
    <div className="mt-4 border-t pt-4">
      {!showForm ? (
        <div className="flex gap-2">
          <button 
            onClick={loadAllProducts} 
            className="flex-1 px-4 py-2 bg-purple-500 text-white rounded hover:bg-purple-600"
            title="Tüm ürünleri listele"
          >
            📋 Tüm Liste
          </button>
          <button 
            onClick={() => setShowForm(true)} 
            className="flex-1 px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
          >
            + Yeni Ürün Ekle
          </button>
        </div>
      ) : (
        <div className="space-y-2">
          <input placeholder="Ürün adı *" value={manualProduct.product} onChange={e=>setManualProduct({...manualProduct, product:e.target.value})} className="w-full px-3 py-2 border rounded"/>
          <input placeholder="Açıklama" value={manualProduct.description} onChange={e=>setManualProduct({...manualProduct, description:e.target.value})} className="w-full px-3 py-2 border rounded"/>
          <input type="number" placeholder="Adet" value={manualProduct.qty} onChange={e=>setManualProduct({...manualProduct, qty:Number(e.target.value)})} className="w-full px-3 py-2 border rounded"/>
          
          <label className="flex items-center gap-2 text-sm">
            <input type="checkbox" checked={showSaveToSupabase} onChange={e=>setShowSaveToSupabase(e.target.checked)}/>
            <span>Bu ürünü stok listeme de ekle</span>
          </label>

          <div className="flex gap-2">
            <button onClick={addManual} className="flex-1 px-3 py-2 bg-green-500 text-white rounded">Ekle</button>
            <button onClick={()=>{setShowForm(false); setManualProduct({product:'',description:'',qty:1}); setShowSaveToSupabase(false);}} className="px-3 py-2 bg-gray-200 rounded">İptal</button>
          </div>
        </div>
      )}

      {showAllProducts && (
        <div className="fixed inset-0 bg-black/40 flex items-center justify-center z-50 p-4">
          <div className="bg-white rounded-lg shadow-xl max-w-4xl w-full max-h-[80vh] overflow-hidden flex flex-col">
            <div className="p-4 border-b">
              <div className="flex items-center justify-between mb-3">
                <h3 className="font-semibold text-lg">Stok Liste ({allProducts.length})</h3>
                <button 
                  onClick={() => {setShowAllProducts(false); setAllProducts([]); setSearchTerm('');}} 
                  className="text-gray-500 hover:text-gray-700"
                >
                  ✕
                </button>
              </div>
              <div className="relative">
                <input 
                  value={searchTerm} 
                  onChange={e=>setSearchTerm(e.target.value)} 
                  placeholder="Ürün ara..." 
                  className="w-full px-3 py-2 border rounded pl-10"
                />
                <Icon type="search" className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-gray-400"/>
              </div>
            </div>
            
            <div className="flex-1 overflow-auto p-4">
              <div className="grid grid-cols-1 gap-2">
                {allProducts
                  .filter(prod => !searchTerm || prod.product_name.toLowerCase().includes(searchTerm.toLowerCase()))
                  .map(prod => (
                    <SupplierProductItem 
                      key={prod.id}
                      prod={prod}
                      job={job}
                      onAdd={onAdd}
                    />
                  ))}
              </div>
            </div>
          </div>
        </div>
      )}
      {dialog && (
        <CustomDialog
          message={dialog.message}
          type={dialog.type}
          showCancel={dialog.showCancel}
          onConfirm={dialog.onConfirm}
          onCancel={dialog.onCancel || (() => setDialog(null))}
        />
      )}
    </div>
  );
}

/* ---------- Supplier Product Item ---------- */
function SupplierProductItem({ prod, job, onAdd }) {
  const [showConfirm, setShowConfirm] = useState(false);
  const [confirmAction, setConfirmAction] = useState(null);

  const handleAdd = () => {
    onAdd({ product: prod.product_name, description: `Stok: ${prod.quantity}`, qty: 1 });
    setConfirmAction({
      type: 'success',
      message: `✅ Ürün eklendi: ${prod.product_name}`
    });
    setShowConfirm(true);
  };

  return (
    <>
      <div className="border rounded p-3 hover:bg-gray-50">
        <div className="flex items-start justify-between gap-3">
          <div className="flex-1">
            <div className="font-medium">{prod.product_name}</div>
            <div className="text-xs text-gray-500 mt-1">
              Barkod: {prod.barcode}
            </div>
            <div className="text-xs text-gray-500 mt-1">
              Stok: <span className={prod.quantity < 5 ? 'text-red-600 font-semibold' : 'text-green-600'}>{prod.quantity}</span> adet
            </div>
          </div>
          <div className="flex flex-col gap-2">
            <button 
              onClick={handleAdd} 
              className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 text-sm"
            >
              Ekle
            </button>
          </div>
        </div>
      </div>

      {showConfirm && (
        <CustomDialog
          message={confirmAction.message}
          onConfirm={() => {
            if(confirmAction.onConfirm) {
              confirmAction.onConfirm();
            } else {
              setShowConfirm(false);
            }
          }}
          onCancel={() => setShowConfirm(false)}
          showCancel={confirmAction.type === 'decrease' || confirmAction.type === 'increase'}
        />
      )}
    </>
  );
}

/* ---------- Yapılan ve Planlanan İşler Tab ---------- */
function PlanlananIslerTab({ job, updateJob, activeTab }) {
  const [newTask, setNewTask] = useState('');
  
  // ✅ Tüm görevleri SEKME BAZINDA topla
  const tasksByTab = {};
  (job.tabs || []).forEach(tab => {
    if(!tab.id.startsWith('tab-genel') && !tab.id.startsWith('tab-planlanan')) {
      if((tab.tasks || []).length > 0) {
        tasksByTab[tab.title] = (tab.tasks || []).map(task => ({
          text: task.text,
          done: task.done,
          assignedTo: task.assignedTo || []
        }));
      }
    }
  });
  
  // ✅ Siparişleri TEDARİKÇİ BAZINDA topla
  const ordersBySupplier = {};
  
  // Malzeme siparişleri
  if(job.supplierOrders) {
    Object.keys(job.supplierOrders).forEach(supplier => {
      const items = job.supplierOrders[supplier];
      if(items.length > 0) {
        if(!ordersBySupplier[supplier]) ordersBySupplier[supplier] = [];
        items.forEach(item => {
          ordersBySupplier[supplier].push({
            product: item.product,
            description: item.description,
            qty: item.qty,
            type: 'Malzeme'
          });
        });
      }
    });
  }
  
  // Lazer siparişleri
  if(job.lazerOrders) {
    Object.keys(job.lazerOrders).forEach(supplier => {
      const items = job.lazerOrders[supplier];
      if(items.length > 0) {
        if(!ordersBySupplier[supplier]) ordersBySupplier[supplier] = [];
        items.forEach(item => {
          ordersBySupplier[supplier].push({
            product: item.product,
            description: item.description,
            qty: item.qty,
            type: 'Lazer'
          });
        });
      }
    });
  }
  
  // Yapılacak İşler (bu sekmedeki)
  const manualTasks = activeTab.tasks || [];
  
  // Excel export
const exportToExcel = () => {
    const wb = XLSX.utils.book_new();
    
    // Ana başlık ve iş bilgileri
    const data = [
      ['YAPILAN VE PLANLANAN İŞLER'],
      [''],
      ['Müşteri:', job.customerName || '-'],
      ['Ürün:', job.productType || '-'],
      ['Sipariş No:', job.orderNumber || '-'],
      ['Başlangıç:', job.startDate || '-'],
      ['Hedef Tarih:', job.targetDate || '-'],
      [''],
      ['─'.repeat(100)],
      ['']
    ];
    
    // Siparişler - Tedarikçi bazında
    if(Object.keys(ordersBySupplier).length > 0) {
      data.push(['📦 SİPARİŞLER']);
      data.push(['']);
      
      Object.keys(ordersBySupplier).forEach(supplier => {
        data.push([`${supplier}`, '', '', '']);
        data.push(['Ürün', 'Açıklama', 'Adet', 'Tip']);
        
        ordersBySupplier[supplier].forEach(item => {
          data.push([
            item.product || '-',
            item.description || '-',
            item.qty || 1,
            item.type || '-'
          ]);
        });
        data.push(['']);
      });
      
      data.push(['─'.repeat(100)]);
      data.push(['']);
    }
    
    // Sekme bazında görevler
    if(Object.keys(tasksByTab).length > 0) {
      Object.keys(tasksByTab).forEach(tabTitle => {
        data.push([`📋 ${tabTitle.toUpperCase()}`]);
        data.push(['']);
        data.push(['Durum', 'Görev', 'Atanan']);
        
        tasksByTab[tabTitle].forEach(task => {
          data.push([
            task.done ? '✓' : '☐',
            task.text || '-',
            (task.assignedTo || []).join(', ') || '-'
          ]);
        });
        data.push(['']);
        data.push(['─'.repeat(100)]);
        data.push(['']);
      });
    }
    
    // Manuel görevler
    if(manualTasks.length > 0) {
      data.push(['✅ YAPILACAK İŞLER']);
      data.push(['']);
      data.push(['Durum', 'Görev']);
      
      manualTasks.forEach(t => {
        data.push([t.done ? '✓' : '☐', t.text]);
      });
    }
    
    const ws = XLSX.utils.aoa_to_sheet(data);
    
    // Sütun genişlikleri
    ws['!cols'] = [
      {wch: 15},  // Durum/Supplier
      {wch: 40},  // Ürün/Görev
      {wch: 30},  // Açıklama/Atanan
      {wch: 10}   // Adet/Tip
    ];
    
    // Başlık stil
    const titleRef = XLSX.utils.encode_cell({ r: 0, c: 0 });
    if(ws[titleRef]) {
      ws[titleRef].s = {
        font: { bold: true, sz: 16, color: { rgb: "1F4E78" } },
        alignment: { horizontal: "left", vertical: "center" }
      };
    }
    
    // Tüm tablo header'larını bul ve stil uygula
    for(let i = 0; i < data.length; i++) {
      if(Array.isArray(data[i]) && data[i].length > 0) {
        // Siparişler başlığı
        if(data[i][0] && (data[i][0].includes('SİPARİŞLER') || data[i][0].includes('YAPILACAK İŞLER'))) {
          const headerRef = XLSX.utils.encode_cell({ r: i, c: 0 });
          if(ws[headerRef]) {
            ws[headerRef].s = {
              font: { bold: true, sz: 12, color: { rgb: "1F4E78" } },
              alignment: { horizontal: "left", vertical: "center" }
            };
          }
        }
        // Tablo header'ları
        if((data[i][0] === 'Ürün' && data[i][1] === 'Açıklama') || 
           (data[i][0] === 'Durum' && data[i][1] === 'Görev')) {
          const headerCols = data[i].length;
          for(let col = 0; col < headerCols; col++) {
            const cellRef = XLSX.utils.encode_cell({ r: i, c: col });
            if(ws[cellRef]) {
              ws[cellRef].s = {
                font: { bold: true, sz: 11, color: { rgb: "FFFFFF" } },
                fill: { fgColor: { rgb: "4472C4" } },
                alignment: { horizontal: "center", vertical: "center", wrapText: true },
                border: {
                  top: { style: "thin" }, bottom: { style: "thin" },
                  left: { style: "thin" }, right: { style: "thin" }
                }
              };
            }
          }
          // Bu header'dan sonraki veri satırlarına border ekle
          for(let row = i + 1; row < data.length; row++) {
            if(Array.isArray(data[row]) && data[row].length >= headerCols && 
               data[row][0] !== '' && !data[row][0].includes('─') && 
               !data[row][0].includes('SİPARİŞLER') && !data[row][0].includes('YAPILACAK')) {
              for(let col = 0; col < headerCols; col++) {
                const cellRef = XLSX.utils.encode_cell({ r: row, c: col });
                if(ws[cellRef]) {
                  if(!ws[cellRef].s) ws[cellRef].s = {};
                  ws[cellRef].s.border = {
                    top: { style: "thin" }, bottom: { style: "thin" },
                    left: { style: "thin" }, right: { style: "thin" }
                  };
                  ws[cellRef].s.alignment = { vertical: "center", wrapText: true };
                  if(col === headerCols - 1 && headerCols === 4) ws[cellRef].s.alignment.horizontal = "right";
                }
              }
            } else if(data[row][0] === '' || data[row][0].includes('─')) {
              break; // Boş satır veya ayırıcı geldiğinde dur
            }
          }
        }
        // Tedarikçi başlıkları
        if(data[i].length === 4 && data[i][1] === '' && data[i][2] === '' && data[i][3] === '' && 
           data[i][0] !== '' && !data[i][0].includes('─') && !data[i][0].includes('SİPARİŞLER')) {
          const supplierRef = XLSX.utils.encode_cell({ r: i, c: 0 });
          if(ws[supplierRef]) {
            ws[supplierRef].s = {
              font: { bold: true, sz: 11, color: { rgb: "2E75B6" } },
              fill: { fgColor: { rgb: "E7F3FF" } },
              alignment: { horizontal: "left", vertical: "center" }
            };
          }
        }
      }
    }
    
    // Satır yükseklikleri
    ws['!rows'] = [];
    for(let i = 0; i < data.length; i++) {
      ws['!rows'][i] = { hpt: i === 0 ? 25 : 20 };
    }
    
    XLSX.utils.book_append_sheet(wb, ws, 'Planlanan İşler');
    XLSX.writeFile(wb, `Planlanan_Isler_${job.orderNumber || job.id}.xlsx`);
  };
  
  // Word export - Modern HTML formatında
const exportToWord = () => {
    const htmlContent = `
<!DOCTYPE html>
<html lang="tr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Yapılan ve Planlanan İşler - ${job.orderNumber || job.id}</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      line-height: 1.6;
      color: #333;
      padding: 40px;
      background: #f5f5f5;
    }
    .container {
      max-width: 900px;
      margin: 0 auto;
      background: white;
      padding: 40px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    h1 {
      color: #1F4E78;
      font-size: 28px;
      margin-bottom: 30px;
      padding-bottom: 15px;
      border-bottom: 3px solid #4472C4;
    }
    h2 {
      color: #2E75B6;
      font-size: 20px;
      margin-top: 30px;
      margin-bottom: 15px;
      padding: 10px;
      background: #E7F3FF;
      border-left: 4px solid #4472C4;
    }
    .info-grid {
      display: grid;
      grid-template-columns: 150px 1fr;
      gap: 10px;
      margin-bottom: 30px;
      padding: 20px;
      background: #F8F9FA;
      border-radius: 5px;
    }
    .info-label {
      font-weight: bold;
      color: #555;
    }
    .info-value {
      color: #333;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin: 20px 0;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    th {
      background: #4472C4;
      color: white;
      padding: 12px;
      text-align: left;
      font-weight: 600;
      border: 1px solid #2E5A8A;
    }
    td {
      padding: 10px 12px;
      border: 1px solid #ddd;
    }
    tr:nth-child(even) {
      background: #F8F9FA;
    }
    tr:hover {
      background: #E7F3FF;
    }
    .supplier-header {
      background: #E7F3FF;
      font-weight: bold;
      color: #2E75B6;
      padding: 10px;
      margin-top: 20px;
      border-left: 4px solid #4472C4;
    }
    .task-item {
      padding: 8px 0;
      border-bottom: 1px solid #eee;
    }
    .task-done {
      color: #28a745;
      font-weight: bold;
    }
    .task-pending {
      color: #6c757d;
    }
    .badge {
      display: inline-block;
      padding: 4px 8px;
      border-radius: 3px;
      font-size: 12px;
      font-weight: 600;
    }
    .badge-success {
      background: #d4edda;
      color: #155724;
    }
    .badge-info {
      background: #d1ecf1;
      color: #0c5460;
    }
    .footer {
      margin-top: 40px;
      padding-top: 20px;
      border-top: 2px solid #ddd;
      text-align: center;
      color: #666;
      font-size: 12px;
    }
    @media print {
      body { background: white; padding: 20px; }
      .container { box-shadow: none; padding: 20px; }
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>📋 YAPILAN VE PLANLANAN İŞLER</h1>
    
    <div class="info-grid">
      <div class="info-label">Müşteri:</div>
      <div class="info-value">${job.customerName || '-'}</div>
      
      <div class="info-label">Ürün:</div>
      <div class="info-value">${job.productType || '-'}</div>
      
      <div class="info-label">Sipariş No:</div>
      <div class="info-value"><strong>${job.orderNumber || '-'}</strong></div>
      
      <div class="info-label">Başlangıç:</div>
      <div class="info-value">${job.startDate || '-'}</div>
      
      <div class="info-label">Hedef Tarih:</div>
      <div class="info-value">${job.targetDate || '-'}</div>
      
      <div class="info-label">Oluşturulma:</div>
      <div class="info-value">${new Date().toLocaleDateString('tr-TR', { year: 'numeric', month: 'long', day: 'numeric' })}</div>
    </div>
    
    ${Object.keys(ordersBySupplier).length > 0 ? `
    <h2>📦 SİPARİŞLER</h2>
    ${Object.keys(ordersBySupplier).map(supplier => `
      <div class="supplier-header">${supplier}</div>
      <table>
        <thead>
          <tr>
            <th>Ürün</th>
            <th>Açıklama</th>
            <th style="text-align: center;">Adet</th>
            <th>Tip</th>
          </tr>
        </thead>
        <tbody>
          ${ordersBySupplier[supplier].map(item => `
            <tr>
              <td>${item.product || '-'}</td>
              <td>${item.description || '-'}</td>
              <td style="text-align: center;"><strong>${item.qty || 1}</strong></td>
              <td><span class="badge badge-info">${item.type || '-'}</span></td>
            </tr>
          `).join('')}
        </tbody>
      </table>
    `).join('')}
    ` : ''}
    
    ${Object.keys(tasksByTab).length > 0 ? `
    ${Object.keys(tasksByTab).map(tabTitle => `
      <h2>📋 ${tabTitle.toUpperCase()}</h2>
      <table>
        <thead>
          <tr>
            <th style="width: 80px;">Durum</th>
            <th>Görev</th>
            <th>Atanan</th>
          </tr>
        </thead>
        <tbody>
          ${tasksByTab[tabTitle].map(task => `
            <tr>
              <td style="text-align: center;">
                <span class="${task.done ? 'task-done' : 'task-pending'}">
                  ${task.done ? '✓ Tamamlandı' : '☐ Beklemede'}
                </span>
              </td>
              <td>${task.text || '-'}</td>
              <td>${(task.assignedTo || []).length > 0 ? task.assignedTo.join(', ') : '-'}</td>
            </tr>
          `).join('')}
        </tbody>
      </table>
    `).join('')}
    ` : ''}
    
    ${manualTasks.length > 0 ? `
    <h2>✅ YAPILACAK İŞLER</h2>
    <table>
      <thead>
        <tr>
          <th style="width: 80px;">Durum</th>
          <th>Görev</th>
        </tr>
      </thead>
      <tbody>
        ${manualTasks.map(t => `
          <tr>
            <td style="text-align: center;">
              <span class="${t.done ? 'task-done' : 'task-pending'}">
                ${t.done ? '✓' : '☐'}
              </span>
            </td>
            <td>${t.text}</td>
          </tr>
        `).join('')}
      </tbody>
    </table>
    ` : ''}
    
    <div class="footer">
      <p>Bu rapor ${new Date().toLocaleDateString('tr-TR')} tarihinde oluşturulmuştur.</p>
      <p>BEGA Üretim Takip Sistemi</p>
    </div>
  </div>
</body>
</html>
    `;
    
    const blob = new Blob([htmlContent], { type: 'text/html;charset=utf-8' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `Planlanan_Isler_${job.orderNumber || job.id}.html`;
    a.click();
    URL.revokeObjectURL(url);
  };
  
  const addTask = () => {
    if(!newTask.trim()) return;
    const task = { 
      id: 'task-' + Date.now(), 
      text: newTask.trim(), 
      done: false 
    };
    updateJob(prev => ({ 
      ...prev, 
      tabs: prev.tabs.map(t => 
        t.id === activeTab.id 
          ? { ...t, tasks: [...(t.tasks || []), task] } 
          : t
      ) 
    }));
    setNewTask('');
  };
  
  const toggleTask = (taskId) => {
    updateJob(prev => ({ 
      ...prev, 
      tabs: prev.tabs.map(t => 
        t.id === activeTab.id 
          ? { 
              ...t, 
              tasks: t.tasks.map(task => 
                task.id === taskId 
                  ? { ...task, done: !task.done } 
                  : task
              ) 
            } 
          : t
      ) 
    }));
  };
  
  const deleteTask = (taskId) => {
    updateJob(prev => ({ 
      ...prev, 
      tabs: prev.tabs.map(t => 
        t.id === activeTab.id 
          ? { ...t, tasks: t.tasks.filter(task => task.id !== taskId) } 
          : t
      ) 
    }));
  };
  
  return (
    <div className="space-y-6">
      {/* Export Buttons */}
      <div className="flex gap-2 justify-end">
        <button onClick={exportToExcel} className="px-4 py-2 bg-green-600 text-white rounded flex items-center gap-2">
          <Icon type="download" /> Excel İndir
        </button>
        <button onClick={exportToWord} className="px-4 py-2 bg-blue-600 text-white rounded flex items-center gap-2">
          <Icon type="download" /> Word İndir
        </button>
      </div>
      
      {/* Siparişler - Tedarikçi Bazında */}
      {Object.keys(ordersBySupplier).length > 0 && (
        <div className="bg-blue-50 rounded-lg p-4">
          <h4 className="font-semibold mb-3 text-lg">📦 Siparişler</h4>
          <div className="space-y-4">
            {Object.keys(ordersBySupplier).map(supplier => (
              <div key={supplier} className="bg-white rounded-lg p-3">
                <h5 className="font-semibold text-blue-600 mb-2">{supplier}</h5>
                <ul className="space-y-1">
                  {ordersBySupplier[supplier].map((item, idx) => (
                    <li key={idx} className="flex items-start justify-between text-sm">
                      <span className="flex items-start gap-2">
                        <span className="text-blue-600">•</span>
                        <span>{item.product}{item.description ? `: ${item.description}` : ''}</span>
                      </span>
                      <span className="text-gray-600 ml-2 whitespace-nowrap">({item.qty} adet)</span>
                    </li>
                  ))}
                </ul>
              </div>
            ))}
          </div>
        </div>
      )}
      
      {/* ✅ Diğer Sekme Görevleri - Sekme Bazında */}
      {Object.keys(tasksByTab).length > 0 && (
        <div className="bg-purple-50 rounded-lg p-4">
          <h4 className="font-semibold mb-3 text-lg">📋 Diğer Sekme Görevleri</h4>
          <div className="space-y-4">
            {Object.keys(tasksByTab).map(tabTitle => (
              <div key={tabTitle} className="bg-white rounded-lg p-3">
                <h5 className="font-semibold text-purple-600 mb-2">{tabTitle}</h5>
                <div className="space-y-2">
                  {tasksByTab[tabTitle].map((task, idx) => (
                    <div key={idx} className="flex items-start gap-2 text-sm">
                      <span className={task.done ? 'text-green-600' : 'text-gray-400'}>
                        {task.done ? '✓' : '☐'}
                      </span>
                      <div className="flex-1">
                        <span className={task.done ? 'line-through text-gray-400' : ''}>
                          {task.text}
                        </span>
                        {task.assignedTo.length > 0 && (
                          <div className="text-xs text-gray-500 mt-1">
                            Atanan: {task.assignedTo.join(', ')}
                          </div>
                        )}
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            ))}
          </div>
        </div>
      )}
      
      {/* Yapılacak İşler */}
      <div className="bg-white rounded-lg border p-4">
        <h4 className="font-semibold mb-3 text-lg">✅ Yapılacak İşler</h4>
        
        <div className="space-y-2 mb-4">
          {manualTasks.map(task => (
            <div key={task.id} className="flex items-center gap-3 p-3 bg-gray-50 rounded">
              <input 
                type="checkbox" 
                checked={task.done} 
                onChange={() => toggleTask(task.id)}
                className="w-5 h-5"
              />
              <span className={`flex-1 ${task.done ? 'line-through text-gray-400' : ''}`}>
                {task.text}
              </span>
              <button 
                onClick={() => deleteTask(task.id)} 
                className="text-red-500 hover:text-red-700"
              >
                <Icon type="trash" className="w-4 h-4" />
              </button>
            </div>
          ))}
        </div>
        
        <div className="flex gap-2">
          <input 
            value={newTask} 
            onChange={e => setNewTask(e.target.value)}
            onKeyDown={e => e.key === 'Enter' && addTask()}
            placeholder="Yeni manuel görev ekle..."
            className="flex-1 px-3 py-2 border rounded"
          />
          <button 
            onClick={addTask}
            className="px-4 py-2 bg-green-500 text-white rounded"
          >
            <Icon type="plus" />
          </button>
        </div>
      </div>
    </div>
  );
}

/* ---------- Tasks Tab ---------- */
function TasksTab({ activeTab, newTaskText, setNewTaskText, addTaskToTab, toggleTaskDone, deleteTask, updateTask, quickAssign, teamMembers, expandedTaskId, setExpandedTaskId, editingTitleTaskId, setEditingTitleTaskId }) {
  return (
    <div className="space-y-3">
      {(activeTab.tasks || []).map(task => {
        const isExpanded = expandedTaskId === task.id;
        return (
          <div key={task.id} className="bg-white border rounded-lg p-3">
            <div className="flex items-start gap-3">
              <input type="checkbox" checked={task.done || false} onChange={() => toggleTaskDone(activeTab.id, task.id)} className="mt-1"/>
              <div className="flex-1">
                <div className="flex items-center justify-between">
                  <div className="flex-1">
                    {editingTitleTaskId === task.id ? (
                      <div className="flex gap-2">
                        <input className="flex-1 px-2 py-1 border rounded" value={task.text} onChange={(e)=> updateTask(activeTab.id, task.id, { text: e.target.value })} />
                        <button onClick={() => setEditingTitleTaskId(null)} className="px-2 py-1 bg-green-500 text-white rounded text-sm">Kaydet</button>
                      </div>
                    ) : (
                      <button onClick={() => setExpandedTaskId(isExpanded ? null : task.id)} className={`text-left text-sm font-medium ${task.done ? 'line-through text-gray-400':''}`}>{task.text}</button>
                    )}
                  </div>

                  <div className="flex items-center gap-2">
                    {(task.assignedTo || []).length > 0 && <div className="text-xs text-gray-500">{(task.assignedTo||[]).length} kişi</div>}
                    <select onChange={(e)=>{ const val=e.target.value; if(val){ quickAssign(activeTab.id, task.id, val); e.target.value=''; } }} className="text-xs px-2 py-1 border rounded bg-white">
                      <option value="">Ata...</option>
                      {teamMembers.map(m => <option key={m} value={m}>{m}</option>)}
                    </select>
                    <button title="Düzenle" onClick={() => setEditingTitleTaskId(task.id)} className="text-gray-500 hover:text-gray-700"><Icon type="edit" className="w-4 h-4"/></button>
                    <button onClick={() => deleteTask(activeTab.id, task.id)} className="text-red-500"><Icon type="trash" className="w-4 h-4"/></button>
                  </div>
                </div>

                {isExpanded && (
                  <div className="mt-3 space-y-2">
                    <div>
                      <label className="text-xs text-gray-600 block mb-1">Açıklama</label>
                      <textarea value={task.description || ''} onChange={(e)=> updateTask(activeTab.id, task.id, { description: e.target.value })} placeholder="Görev açıklaması..." className="w-full px-2 py-2 border rounded resize-none" rows="3"></textarea>
                    </div>

                    <div className="grid grid-cols-1 md:grid-cols-2 gap-2">
                      <div>
                        <label className="text-xs text-gray-600 block mb-1">Atananlar</label>
                        <select multiple value={task.assignedTo || []} onChange={(e) => {
                          const arr = Array.from(e.target.selectedOptions).map(o => o.value);
                          updateTask(activeTab.id, task.id, { assignedTo: arr });
                        }} className="w-full px-2 py-2 border rounded h-24 overflow-auto text-sm">
                          {teamMembers.map(m => <option key={m} value={m}>{m}</option>)}
                        </select>
                      </div>

                      <div>
                        <label className="text-xs text-gray-600 block mb-1">Öncelik & Tarih</label>
                        <select value={task.priority || 'normal'} onChange={(e)=> updateTask(activeTab.id, task.id, { priority: e.target.value })} className="w-full px-2 py-2 border rounded text-sm mb-2">
                          <option value="low">Düşük</option>
                          <option value="normal">Normal</option>
                          <option value="high">Yüksek</option>
                        </select>
                        <input type="date" value={task.dueDate || ''} onChange={(e)=> updateTask(activeTab.id, task.id, { dueDate: e.target.value })} className="w-full px-2 py-2 border rounded text-sm" />
                      </div>
                    </div>
                  </div>
                )}
              </div>
            </div>
          </div>
        );
      })}

      <div className="flex gap-2 mt-4">
        <input value={newTaskText} onChange={e=>setNewTaskText(e.target.value)} placeholder="Yeni görev ekle..." className="flex-1 px-3 py-2 border rounded" onKeyDown={(e)=>{ if(e.key==='Enter'){ addTaskToTab(activeTab.id, newTaskText); } }}/>
        <button onClick={() => addTaskToTab(activeTab.id, newTaskText)} className="px-4 py-2 bg-green-500 text-white rounded"><Icon type="plus"/></button>
      </div>
    </div>
  );
}

/* ---------- Render ---------- */
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<ProductionTracker />);

// Service Worker kayıt
if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
        navigator.serviceWorker.register('/sw.js')
            .then((registration) => {
                console.log('Service Worker kayıt başarılı:', registration.scope);
            })
            .catch((error) => {
                console.log('Service Worker kayıt hatası:', error);
            });
    });
}
</script>
</body>
</html>
