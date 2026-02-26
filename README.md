import React, { useState, useMemo, useEffect } from 'react';
import { 
  Users, FileText, BookOpen, Plus, Trash2, CheckCircle, ChevronRight, 
  DollarSign, Package, Zap, Save, Tag, Settings, ShoppingCart, 
  TrendingUp, AlertTriangle, Search, BarChart3, PieChart, Menu, X, Truck,
  ClipboardList, Printer, Download, UploadCloud, DownloadCloud
} from 'lucide-react';

// --- Firebase 雲端資料庫整合 ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, deleteDoc, onSnapshot } from 'firebase/firestore';

// 初始化 Firebase 環境變數
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

const HardwareERP = () => {
  const [activeTab, setActiveTab] = useState('dashboard');
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const [user, setUser] = useState(null);

  // --- 系統設定狀態 ---
  const [systemSettings, setSystemSettings] = useState({
    companyName: '大發五金企業有限公司',
    vatNumber: '87654321',
    address: '高雄市三民區建國二路1號',
    phone: '07-7654-3210'
  });

  // --- 雲端/本地資料狀態 ---
  const [customers, setCustomers] = useState([]);
  const [suppliers, setSuppliers] = useState([]);
  const [products, setProducts] = useState([]);
  const [orders, setOrders] = useState([]);
  const [orderItems, setOrderItems] = useState([]);
  const [purchaseOrders, setPurchaseOrders] = useState([]);
  const [journalEntries, setJournalEntries] = useState([]);

  // --- 狀態管理 (UI 互動用) ---
  const [dashboardMonth, setDashboardMonth] = useState('all');
  const [productSearch, setProductSearch] = useState('');
  const [customerSearch, setCustomerSearch] = useState('');
  const [supplierSearch, setSupplierSearch] = useState('');
  
  const [newCustomer, setNewCustomer] = useState({ name: '', phone: '', vatNumber: '', address: '' });
  const [newSupplier, setNewSupplier] = useState({ name: '', phone: '', vatNumber: '', address: '', note: '' });
  const [newProduct, setNewProduct] = useState({ name: '', category: '' });
  const [newOrder, setNewOrder] = useState({ customerId: '', date: new Date().toISOString().split('T')[0] });
  const [selectedOrder, setSelectedOrder] = useState(null);
  const [selectedPurchaseOrder, setSelectedPurchaseOrder] = useState(null);
  const [newItem, setNewItem] = useState({ productId: '', itemName: '', quantity: 1, unitPrice: 0, unitCost: 0 });

  const [quickOrder, setQuickOrder] = useState({
    customerId: '', date: new Date().toISOString().split('T')[0],
    items: [{ localId: Date.now(), productId: '', itemName: '', quantity: 1, unitPrice: 0, unitCost: 0 }]
  });

  const [quickPurchase, setQuickPurchase] = useState({
    supplierId: '', date: new Date().toISOString().split('T')[0],
    items: [{ localId: Date.now(), productId: '', itemName: '', quantity: 1, unitCost: 0 }]
  });

  // --- 資料備份與還原 (存至 Google Cloud/Drive 用) ---
  const handleExportData = () => {
    const exportData = {
      systemSettings,
      customers,
      suppliers,
      products,
      orders,
      orderItems,
      purchaseOrders,
      journalEntries
    };
    
    const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(exportData, null, 2));
    const downloadAnchorNode = document.createElement('a');
    downloadAnchorNode.setAttribute("href", dataStr);
    downloadAnchorNode.setAttribute("download", `五金ERP_資料備份_${new Date().toISOString().split('T')[0]}.json`);
    document.body.appendChild(downloadAnchorNode); 
    downloadAnchorNode.click();
    downloadAnchorNode.remove();
    alert('✅ 資料已成功匯出！您可以將此 .json 檔案上傳至您的 Google Drive 或 Google Cloud 妥善保存。');
  };

  const handleImportData = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    
    if (!window.confirm('⚠️ 警告：匯入資料將會覆蓋您當前系統內的所有相同 ID 資料。確定要繼續嗎？')) {
      e.target.value = '';
      return;
    }

    const reader = new FileReader();
    reader.onload = async (event) => {
      try {
        const importedData = JSON.parse(event.target.result);
        
        if (user) {
          const path = ['artifacts', appId, 'users', user.uid];
          if (importedData.systemSettings) await setDoc(doc(db, ...path, 'settings', 'main'), importedData.systemSettings);
          
          const collections = ['customers', 'suppliers', 'products', 'orders', 'orderItems', 'purchaseOrders', 'journalEntries'];
          for (const col of collections) {
             if (importedData[col] && Array.isArray(importedData[col])) {
                for (const item of importedData[col]) {
                   await setDoc(doc(db, ...path, col, item.id), item);
                }
             }
          }
        } else {
           if(importedData.systemSettings) setSystemSettings(importedData.systemSettings);
           if(importedData.customers) setCustomers(importedData.customers);
           if(importedData.suppliers) setSuppliers(importedData.suppliers);
           if(importedData.products) setProducts(importedData.products);
           if(importedData.orders) setOrders(importedData.orders);
           if(importedData.orderItems) setOrderItems(importedData.orderItems);
           if(importedData.purchaseOrders) setPurchaseOrders(importedData.purchaseOrders);
           if(importedData.journalEntries) setJournalEntries(importedData.journalEntries);
        }
        alert('🎉 資料匯入成功！系統已恢復至備份狀態。');
      } catch (error) {
         console.error(error);
         alert('❌ 檔案格式錯誤或匯入失敗，請確保您選擇的是正確的備份檔。');
      }
      e.target.value = ''; 
    };
    reader.readAsText(file);
  };

  // --- 下載圖片功能 (採購單) ---
  const handleDownloadImage = async (elementId, fileName) => {
    try {
      const btn = document.getElementById('download-btn');
      const originalText = btn.innerHTML;
      btn.innerHTML = '處理中...';

      if (!window.html2canvas) {
        await new Promise((resolve, reject) => {
          const script = document.createElement('script');
          script.src = 'https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js';
          script.onload = resolve;
          script.onerror = reject;
          document.head.appendChild(script);
        });
      }
      
      const element = document.getElementById(elementId);
      const canvas = await window.html2canvas(element, { scale: 2, backgroundColor: '#ffffff', useCORS: true });
      const dataUrl = canvas.toDataURL('image/png');
      
      const link = document.createElement('a');
      link.download = `${fileName}.png`;
      link.href = dataUrl;
      link.click();

      btn.innerHTML = originalText;
    } catch (error) {
      console.error('匯出圖片失敗:', error);
      alert('匯出圖片失敗，請稍後再試。');
    }
  };

  // --- 系統初始化與雲端同步機制 ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("驗證失敗:", error);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const basePath = ['artifacts', appId, 'users', user.uid];

    const unsubCust = onSnapshot(collection(db, ...basePath, 'customers'), snap => setCustomers(snap.docs.map(d => d.data())), e => console.error(e));
    const unsubSupp = onSnapshot(collection(db, ...basePath, 'suppliers'), snap => setSuppliers(snap.docs.map(d => d.data())), e => console.error(e));
    const unsubProd = onSnapshot(collection(db, ...basePath, 'products'), snap => setProducts(snap.docs.map(d => d.data())), e => console.error(e));
    const unsubOrd = onSnapshot(collection(db, ...basePath, 'orders'), snap => setOrders(snap.docs.map(d => d.data())), e => console.error(e));
    const unsubOrdItm = onSnapshot(collection(db, ...basePath, 'orderItems'), snap => setOrderItems(snap.docs.map(d => d.data())), e => console.error(e));
    const unsubPur = onSnapshot(collection(db, ...basePath, 'purchaseOrders'), snap => setPurchaseOrders(snap.docs.map(d => d.data())), e => console.error(e));
    const unsubJe = onSnapshot(collection(db, ...basePath, 'journalEntries'), snap => setJournalEntries(snap.docs.map(d => d.data())), e => console.error(e));
    
    const unsubSet = onSnapshot(doc(db, ...basePath, 'settings', 'main'), snap => {
      if (snap.exists()) setSystemSettings(snap.data());
    }, e => console.error(e));

    return () => {
      unsubCust(); unsubSupp(); unsubProd(); unsubOrd(); unsubOrdItm(); unsubPur(); unsubJe(); unsubSet();
    };
  }, [user]);

  // --- 處理函數 ---
  const handleSaveSettings = async () => {
    if (!user) return alert('目前為離線模式，設定僅保存在當下。建議使用下方的「匯出資料」功能。');
    await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'settings', 'main'), systemSettings);
    alert('設定已儲存！');
  };

  const handleAddCustomer = async () => {
    if (!newCustomer.name) return alert('請輸入客戶名稱');
    const id = `C${Date.now()}`; 
    if (user) await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'customers', id), { ...newCustomer, id });
    else setCustomers([...customers, { ...newCustomer, id }]);
    setNewCustomer({ name: '', phone: '', vatNumber: '', address: '' });
  };
  const handleDeleteCustomer = async (id) => {
    if (window.confirm('確定要刪除此客戶嗎？')) {
      if (user) await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'customers', id));
      else setCustomers(customers.filter(c => c.id !== id));
    }
  };

  const handleAddSupplier = async () => {
    if (!newSupplier.name) return alert('請輸入供應商名稱');
    const id = `S${Date.now()}`;
    if (user) await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'suppliers', id), { ...newSupplier, id });
    else setSuppliers([...suppliers, { ...newSupplier, id }]);
    setNewSupplier({ name: '', phone: '', vatNumber: '', address: '', note: '' });
  };
  const handleDeleteSupplier = async (id) => {
    if (window.confirm('確定要刪除此供應商嗎？')) {
      if (user) await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'suppliers', id));
      else setSuppliers(suppliers.filter(s => s.id !== id));
    }
  };

  const handleAddProduct = async () => {
    if (!newProduct.name || !newProduct.category) return alert('請輸入商品類別與名稱');
    const id = `P${Date.now()}`;
    const prod = { ...newProduct, id, price: 0, cost: 0, stock: 0 };
    if (user) await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'products', id), prod);
    else setProducts([...products, prod]);
    setNewProduct({ name: '', category: '' });
  };
  const handleDeleteProduct = async (id) => {
    if (window.confirm('確定要刪除此商品嗎？')) {
      if (user) await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'products', id));
      else setProducts(products.filter(p => p.id !== id));
    }
  };
  
  const updateProductStock = async (id, newStock) => {
    const p = products.find(x => x.id === id);
    if(p) {
      if (user) await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'products', id), { ...p, stock: parseInt(newStock) || 0 });
      else setProducts(products.map(x => x.id === id ? { ...x, stock: parseInt(newStock) || 0 } : x));
    }
  };
  const updateProductPrice = async (id, newPrice) => {
    const p = products.find(x => x.id === id);
    if(p) {
      if (user) await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'products', id), { ...p, price: parseInt(newPrice) || 0 });
      else setProducts(products.map(x => x.id === id ? { ...x, price: parseInt(newPrice) || 0 } : x));
    }
  };

  const handleAddOrder = async () => {
    if (!newOrder.customerId) return alert('請選擇客戶');
    const id = `ORD-${Date.now()}`;
    const newOrderObj = { ...newOrder, id, totalAmount: 0, totalCost: 0, status: 'Draft' };
    if (user) await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'orders', id), newOrderObj);
    else setOrders([...orders, newOrderObj]);
    setNewOrder({ customerId: '', date: new Date().toISOString().split('T')[0] });
  };

  const handleDeleteOrder = async (orderId) => {
    if (window.confirm('確定要刪除這筆訂單嗎？相關明細與分錄也會被移除。')) {
      if (user) {
        const path = ['artifacts', appId, 'users', user.uid];
        await deleteDoc(doc(db, ...path, 'orders', orderId));
        const relatedItems = orderItems.filter(i => i.orderId === orderId);
        for(const item of relatedItems) await deleteDoc(doc(db, ...path, 'orderItems', item.id));
        const relatedJournals = journalEntries.filter(j => j.refId === orderId);
        for(const j of relatedJournals) await deleteDoc(doc(db, ...path, 'journalEntries', j.id));
      } else {
        setOrders(orders.filter(o => o.id !== orderId));
        setOrderItems(orderItems.filter(i => i.orderId !== orderId));
        setJournalEntries(journalEntries.filter(j => j.refId !== orderId));
      }
      setSelectedOrder(null);
    }
  };

  const handleAddItem = async (orderId) => {
    if (!newItem.productId || newItem.quantity <= 0 || newItem.unitPrice < 0) return alert('請填寫完整明細');
    const id = `ITM-${Date.now()}`;
    const totalPrice = newItem.quantity * newItem.unitPrice;
    const itemTotalCost = newItem.quantity * newItem.unitCost;
    const item = { ...newItem, id, orderId, totalPrice };
    
    const order = orders.find(o => o.id === orderId);
    if(order) {
      if (user) {
        const path = ['artifacts', appId, 'users', user.uid];
        await setDoc(doc(db, ...path, 'orderItems', id), item);
        await setDoc(doc(db, ...path, 'orders', orderId), { ...order, totalAmount: order.totalAmount + totalPrice, totalCost: (order.totalCost||0) + itemTotalCost });
      } else {
        setOrderItems([...orderItems, item]);
        setOrders(orders.map(o => o.id === orderId ? { ...o, totalAmount: o.totalAmount + totalPrice, totalCost: (o.totalCost||0) + itemTotalCost } : o));
      }
    }
    setNewItem({ productId: '', itemName: '', quantity: 1, unitPrice: 0, unitCost: 0 });
  };

  const deleteItem = async (itemId, orderId, subtotal, itemTotalCost) => {
    const order = orders.find(o => o.id === orderId);
    if (order) {
      if (user) {
        const path = ['artifacts', appId, 'users', user.uid];
        await deleteDoc(doc(db, ...path, 'orderItems', itemId));
        await setDoc(doc(db, ...path, 'orders', orderId), { ...order, totalAmount: order.totalAmount - subtotal, totalCost: order.totalCost - (itemTotalCost||0) });
      } else {
        setOrderItems(orderItems.filter(i => i.id !== itemId));
        setOrders(orders.map(o => o.id === orderId ? { ...o, totalAmount: o.totalAmount - subtotal, totalCost: o.totalCost - (itemTotalCost||0) } : o));
      }
    }
  };

  const handleCompleteOrder = async (orderId) => {
    const order = orders.find(o => o.id === orderId);
    if (order.totalAmount <= 0) return alert('訂單金額為0，無法完成');

    const items = orderItems.filter(i => i.orderId === orderId);
    const newEntries = [
      { id: `JE-${Date.now()}-1`, date: new Date().toISOString().split('T')[0], refId: orderId, account: '應收帳款', debit: order.totalAmount, credit: 0, description: `訂單 ${orderId} 銷貨` },
      { id: `JE-${Date.now()}-2`, date: new Date().toISOString().split('T')[0], refId: orderId, account: '銷貨收入', debit: 0, credit: order.totalAmount, description: `訂單 ${orderId} 銷貨` },
      { id: `JE-${Date.now()}-3`, date: new Date().toISOString().split('T')[0], refId: orderId, account: '銷貨成本', debit: order.totalCost || 0, credit: 0, description: `訂單 ${orderId} 成本` },
      { id: `JE-${Date.now()}-4`, date: new Date().toISOString().split('T')[0], refId: orderId, account: '存貨', debit: 0, credit: order.totalCost || 0, description: `訂單 ${orderId} 扣庫存` }
    ];

    if (user) {
      const path = ['artifacts', appId, 'users', user.uid];
      await setDoc(doc(db, ...path, 'orders', orderId), { ...order, status: 'Completed' });
      for(const item of items) {
        if(item.productId) {
          const p = products.find(x => x.id === item.productId);
          if(p) await setDoc(doc(db, ...path, 'products', p.id), { ...p, stock: p.stock - item.quantity });
        }
      }
      for (const je of newEntries) await setDoc(doc(db, ...path, 'journalEntries', je.id), je);
    } else {
      setOrders(orders.map(o => o.id === orderId ? { ...o, status: 'Completed' } : o));
      let updatedProducts = [...products];
      items.forEach(item => {
        if(item.productId) updatedProducts = updatedProducts.map(p => p.id === item.productId ? { ...p, stock: p.stock - item.quantity } : p);
      });
      setProducts(updatedProducts);
      setJournalEntries([...journalEntries, ...newEntries]);
    }
    
    alert('訂單已完成！(如有備份需求請至設定頁匯出檔案)');
    setSelectedOrder(null); 
  };

  const handleQuickOrderSubmit = async () => {
    if (!quickOrder.customerId) return alert('請選擇客戶');
    const validItems = quickOrder.items.filter(i => i.productId !== '' && i.quantity > 0 && i.unitPrice >= 0);
    if (validItems.length === 0) return alert('請至少選擇一筆完整的品項明細');

    const orderId = `ORD-${Date.now()}`;
    let totalAmount = 0; let totalCost = 0;

    let updatedProducts = [...products];
    const newItems = validItems.map((item, index) => {
      const totalPrice = item.quantity * item.unitPrice;
      const itemCost = item.quantity * item.unitCost;
      totalAmount += totalPrice;
      totalCost += itemCost;
      
      const p = products.find(x => x.id === item.productId);
      if(p) updatedProducts = updatedProducts.map(p2 => p2.id === item.productId ? { ...p2, stock: p2.stock - item.quantity } : p2);
      
      return { id: `ITM-${Date.now()}-${index}`, orderId, ...item, totalPrice };
    });

    const newOrderObj = { id: orderId, customerId: quickOrder.customerId, date: quickOrder.date, totalAmount, totalCost, status: 'Completed' };
    const newEntries = [
      { id: `JE-${Date.now()}-1`, date: quickOrder.date, refId: orderId, account: '應收帳款', debit: totalAmount, credit: 0, description: `訂單 ${orderId} 銷貨` },
      { id: `JE-${Date.now()}-2`, date: quickOrder.date, refId: orderId, account: '銷貨收入', debit: 0, credit: totalAmount, description: `訂單 ${orderId} 銷貨` },
      { id: `JE-${Date.now()}-3`, date: quickOrder.date, refId: orderId, account: '銷貨成本', debit: totalCost, credit: 0, description: `訂單 ${orderId} 成本` },
      { id: `JE-${Date.now()}-4`, date: quickOrder.date, refId: orderId, account: '存貨', debit: 0, credit: totalCost, description: `訂單 ${orderId} 扣庫存` }
    ];

    if (user) {
      const path = ['artifacts', appId, 'users', user.uid];
      await setDoc(doc(db, ...path, 'orders', orderId), newOrderObj);
      for (const item of newItems) await setDoc(doc(db, ...path, 'orderItems', item.id), item);
      for (const p of updatedProducts) await setDoc(doc(db, ...path, 'products', p.id), p);
      for (const je of newEntries) await setDoc(doc(db, ...path, 'journalEntries', je.id), je);
    } else {
      setOrders([...orders, newOrderObj]);
      setOrderItems([...orderItems, ...newItems]);
      setJournalEntries([...journalEntries, ...newEntries]);
      setProducts(updatedProducts);
    }

    alert('快速 Key 單成功！');
    setQuickOrder({ customerId: '', date: new Date().toISOString().split('T')[0], items: [{ localId: Date.now(), productId: '', itemName: '', quantity: 1, unitPrice: 0, unitCost: 0 }] });
    setActiveTab('orders'); 
  };

  const handlePurchaseSubmit = async () => {
    if (!quickPurchase.supplierId) return alert('請選擇供應商');
    const validItems = quickPurchase.items.filter(i => i.productId !== '' && i.quantity > 0 && i.unitCost >= 0);
    if (validItems.length === 0) return alert('請至少選擇一筆品項明細');

    const purchaseId = `PUR-${Date.now()}`;
    let totalAmount = 0;
    let updatedProducts = [...products];

    validItems.forEach(item => {
      totalAmount += (item.quantity * item.unitCost);
      const p = products.find(x => x.id === item.productId);
      if(p) updatedProducts = updatedProducts.map(p2 => p2.id === item.productId ? { ...p2, stock: p2.stock + item.quantity, cost: item.unitCost } : p2);
    });

    const newEntries = [
      { id: `JE-${Date.now()}-P1`, date: quickPurchase.date, refId: purchaseId, account: '存貨', debit: totalAmount, credit: 0, description: `進貨 ${purchaseId}` },
      { id: `JE-${Date.now()}-P2`, date: quickPurchase.date, refId: purchaseId, account: '應付帳款', debit: 0, credit: totalAmount, description: `進貨 ${purchaseId}` }
    ];

    const newPurchaseObj = { id: purchaseId, ...quickPurchase, totalAmount, status: 'Completed' };

    if (user) {
      const path = ['artifacts', appId, 'users', user.uid];
      await setDoc(doc(db, ...path, 'purchaseOrders', purchaseId), newPurchaseObj);
      for (const p of updatedProducts) await setDoc(doc(db, ...path, 'products', p.id), p);
      for (const je of newEntries) await setDoc(doc(db, ...path, 'journalEntries', je.id), je);
    } else {
      setPurchaseOrders([...purchaseOrders, newPurchaseObj]);
      setJournalEntries([...journalEntries, ...newEntries]);
      setProducts(updatedProducts);
    }

    alert('採購單建立完成，庫存已增加！');
    setQuickPurchase({ supplierId: '', date: new Date().toISOString().split('T')[0], items: [{ localId: Date.now(), productId: '', itemName: '', quantity: 1, unitCost: 0 }] });
    setActiveTab('purchaseList');
  };

  // --- 衍生資料 ---
  const filteredOrders = useMemo(() => {
    if (dashboardMonth === 'all') return orders.filter(o => o.status === 'Completed');
    return orders.filter(o => o.status === 'Completed' && o.date.split('-')[1] === String(dashboardMonth).padStart(2, '0'));
  }, [orders, dashboardMonth]);

  const dashboardMetrics = useMemo(() => {
    const revenue = filteredOrders.reduce((sum, o) => sum + o.totalAmount, 0);
    const cost = filteredOrders.reduce((sum, o) => sum + (o.totalCost || 0), 0);
    const margin = revenue > 0 ? ((revenue - cost) / revenue * 100).toFixed(1) : 0;
    
    const monthlyData = Array.from({length: 12}, (_, i) => {
      const monthStr = String(i + 1).padStart(2, '0');
      const mOrders = orders.filter(o => o.status === 'Completed' && o.date.split('-')[1] === monthStr);
      const rev = mOrders.reduce((s, o) => s + o.totalAmount, 0);
      const cst = mOrders.reduce((s, o) => s + (o.totalCost || 0), 0);
      return { month: i+1, rev, cst };
    });
    const maxVal = Math.max(...monthlyData.map(d => Math.max(d.rev, d.cst, 1))); 

    return { revenue, cost, margin, orderCount: filteredOrders.length, monthlyData, maxVal };
  }, [filteredOrders, orders]);

  const lowStockProducts = products.filter(p => p.stock < 10);

  // --- 視圖元件 ---
  const renderDashboard = () => (
    <div className="p-4 md:p-6 space-y-6 print:hidden">
      <div className="flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
        <h2 className="text-xl md:text-2xl font-bold text-gray-800 flex items-center">
          <TrendingUp className="mr-2 text-blue-600"/> 營運儀表板 
        </h2>
        <div className="flex items-center gap-2 bg-white p-2 rounded-lg shadow-sm border border-gray-200 w-full md:w-auto">
          <span className="text-sm font-bold text-gray-600 whitespace-nowrap">統計區間：</span>
          <select className="border-none bg-gray-50 p-2 rounded-md outline-none font-bold w-full md:w-auto" value={dashboardMonth} onChange={(e) => setDashboardMonth(e.target.value)}>
            <option value="all">年度總計</option>
            {Array.from({length: 12}, (_, i) => <option key={i} value={i+1}>{i+1} 月</option>)}
          </select>
        </div>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 md:gap-6">
        <div className="bg-white p-6 rounded-xl border-l-4 border-blue-500 shadow-sm flex justify-between items-center">
          <div><p className="text-gray-500 text-sm font-bold">營業收入</p><p className="text-2xl md:text-3xl font-extrabold text-gray-800 mt-1">${dashboardMetrics.revenue.toLocaleString()}</p></div>
          <DollarSign size={36} className="text-blue-100" />
        </div>
        <div className="bg-white p-6 rounded-xl border-l-4 border-green-500 shadow-sm flex justify-between items-center">
          <div><p className="text-gray-500 text-sm font-bold">平均毛利率</p><p className="text-2xl md:text-3xl font-extrabold text-gray-800 mt-1">{dashboardMetrics.margin}%</p></div>
          <PieChart size={36} className="text-green-100" />
        </div>
        <div className="bg-white p-6 rounded-xl border-l-4 border-purple-500 shadow-sm flex justify-between items-center">
          <div><p className="text-gray-500 text-sm font-bold">訂單數</p><p className="text-2xl md:text-3xl font-extrabold text-gray-800 mt-1">{dashboardMetrics.orderCount}</p></div>
          <FileText size={36} className="text-purple-100" />
        </div>
        <div className="bg-white p-6 rounded-xl border-l-4 border-orange-500 shadow-sm flex justify-between items-center">
          <div><p className="text-gray-500 text-sm font-bold">庫存不足</p><p className="text-2xl md:text-3xl font-extrabold text-orange-600 mt-1">{lowStockProducts.length}</p></div>
          <AlertTriangle size={36} className="text-orange-100" />
        </div>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2 bg-white p-6 rounded-xl shadow-sm border border-gray-100 overflow-x-auto">
           <div className="min-w-[500px]">
             <h3 className="text-lg font-bold text-gray-800 mb-6 flex items-center"><BarChart3 className="mr-2 text-gray-500"/> 營收與成本趨勢圖表 (年度)</h3>
             <div className="h-64 flex items-end justify-between gap-1 md:gap-2 border-b border-gray-200 pb-2">
                {dashboardMetrics.monthlyData.map(d => (
                  <div key={d.month} className="flex-1 flex flex-col justify-end items-center group relative h-full">
                    <div className="w-full flex justify-center gap-[2px] md:gap-1 h-full items-end">
                      <div className="w-1/2 md:w-1/3 bg-blue-400 rounded-t-sm transition-all hover:bg-blue-500 relative" style={{ height: `${(d.rev / dashboardMetrics.maxVal) * 100}%`, minHeight: '2px' }}>
                         <div className="opacity-0 group-hover:opacity-100 absolute -top-8 left-1/2 transform -translate-x-1/2 bg-gray-800 text-white text-[10px] md:text-xs py-1 px-2 rounded z-10 whitespace-nowrap">營收: ${d.rev}</div>
                      </div>
                      <div className="w-1/2 md:w-1/3 bg-red-400 rounded-t-sm transition-all hover:bg-red-500 relative" style={{ height: `${(d.cst / dashboardMetrics.maxVal) * 100}%`, minHeight: '2px' }}>
                         <div className="opacity-0 group-hover:opacity-100 absolute -top-8 left-1/2 transform -translate-x-1/2 bg-gray-800 text-white text-[10px] md:text-xs py-1 px-2 rounded z-10 whitespace-nowrap">成本: ${d.cst}</div>
                      </div>
                    </div>
                    <span className="text-[10px] md:text-xs text-gray-500 mt-2 font-bold">{d.month}月</span>
                  </div>
                ))}
             </div>
             <div className="flex justify-center gap-6 mt-4">
               <div className="flex items-center text-xs md:text-sm text-gray-600 font-bold"><div className="w-3 h-3 bg-blue-400 mr-2 rounded-sm"></div> 營業收入</div>
               <div className="flex items-center text-xs md:text-sm text-gray-600 font-bold"><div className="w-3 h-3 bg-red-400 mr-2 rounded-sm"></div> 營業成本</div>
             </div>
           </div>
        </div>

        <div className="bg-white p-6 rounded-xl shadow-sm border border-red-100">
          <div className="flex justify-between items-center mb-4 border-b border-gray-100 pb-2">
            <h3 className="text-lg font-bold text-red-600 flex items-center"><AlertTriangle className="mr-2"/> 庫存警報</h3>
            <button onClick={() => setActiveTab('products')} className="text-blue-500 text-sm font-bold hover:underline">查看全部</button>
          </div>
          {lowStockProducts.length === 0 ? (
            <p className="text-gray-500 text-center py-8">目前庫存皆充足</p>
          ) : (
            <div className="space-y-3 max-h-[300px] overflow-y-auto pr-2">
              {lowStockProducts.map(p => (
                <div key={p.id} className="bg-red-50 p-3 rounded-lg border border-red-100 flex justify-between items-center">
                  <div>
                    <p className="font-bold text-gray-800 text-sm md:text-base">{p.name}</p>
                    <p className="text-xs text-gray-500">{p.category}</p>
                  </div>
                  <div className="text-right">
                    <p className="text-xl md:text-2xl font-extrabold text-red-600">{p.stock}</p>
                    <p className="text-[10px] md:text-xs font-bold text-red-500">需補貨</p>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      </div>
    </div>
  );

  const renderProducts = () => {
    const filteredProducts = products.filter(p => p.name.includes(productSearch) || p.category.includes(productSearch));

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white p-4 md:p-6 rounded-xl shadow-sm border border-gray-100 mb-6">
          <h3 className="text-lg font-bold mb-4 flex items-center"><Plus size={20} className="mr-2"/> 新增商品</h3>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4">
            <input type="text" placeholder="商品類別 (例: 電動工具)" className="border p-2 rounded-lg flex-1" value={newProduct.category} onChange={e => setNewProduct({...newProduct, category: e.target.value})} />
            <input type="text" placeholder="商品名稱" className="border p-2 rounded-lg flex-[2]" value={newProduct.name} onChange={e => setNewProduct({...newProduct, name: e.target.value})} />
            <button onClick={handleAddProduct} className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700 transition font-bold whitespace-nowrap">新增商品</button>
          </div>
        </div>

        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden">
          <div className="p-4 border-b border-gray-100 bg-gray-50 flex flex-col md:flex-row items-start md:items-center justify-between gap-3">
            <h3 className="font-bold text-gray-700 flex items-center"><Tag className="mr-2"/> 商品及庫存總覽</h3>
            <div className="relative w-full md:w-auto">
              <input type="text" placeholder="搜尋品名或類別..." className="border p-2 pl-8 rounded-lg w-full md:w-64 text-sm outline-none focus:border-blue-500" value={productSearch} onChange={e => setProductSearch(e.target.value)} />
              <Search size={16} className="absolute left-2.5 top-3 text-gray-400" />
            </div>
          </div>
          <div className="overflow-x-auto">
            <table className="w-full text-left border-collapse min-w-[700px]">
              <thead>
                <tr className="bg-white border-b border-gray-200">
                  <th className="p-4 font-semibold text-gray-600">編號</th>
                  <th className="p-4 font-semibold text-gray-600">類別</th>
                  <th className="p-4 font-semibold text-gray-600">名稱</th>
                  <th className="p-4 font-semibold text-gray-600 text-right w-32 md:w-40">預設單價 (可編輯)</th>
                  <th className="p-4 font-semibold text-gray-600 text-right w-32 md:w-40">庫存量 (可編輯)</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">狀態</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">操作</th>
                </tr>
              </thead>
              <tbody>
                {filteredProducts.map(p => (
                  <tr key={p.id} className="border-b border-gray-50 hover:bg-gray-50">
                    <td className="p-4 text-gray-500">{p.id}</td>
                    <td className="p-4 text-gray-600"><span className="bg-gray-200 px-2 py-1 rounded text-xs whitespace-nowrap">{p.category}</span></td>
                    <td className="p-4 font-medium text-gray-800">{p.name}</td>
                    <td className="p-4 text-right">
                      <div className="flex items-center justify-end">
                        <span className="text-gray-500 mr-1 text-sm">$</span>
                        <input type="number" className="border p-1.5 w-20 md:w-24 rounded text-right font-bold focus:border-blue-500 outline-none" value={p.price} onChange={(e) => updateProductPrice(p.id, e.target.value)} />
                      </div>
                    </td>
                    <td className="p-4 text-right">
                      <input type="number" className="border p-1.5 w-20 md:w-24 rounded text-right font-bold focus:border-blue-500 outline-none" value={p.stock} onChange={(e) => updateProductStock(p.id, e.target.value)} />
                    </td>
                    <td className="p-4 text-center">
                      {p.stock >= 10 ? 
                        <span className="bg-green-100 text-green-700 px-2 py-1 rounded text-xs font-bold whitespace-nowrap">充足</span> : 
                        <span className="bg-red-100 text-red-700 px-2 py-1 rounded text-xs font-bold animate-pulse whitespace-nowrap">缺貨</span>
                      }
                    </td>
                    <td className="p-4 text-center">
                      <button onClick={() => handleDeleteProduct(p.id)} className="text-red-500 hover:text-red-700 p-1"><Trash2 size={18} /></button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    );
  };

  const renderCustomers = () => {
    const filteredCustomers = customers.filter(c => 
      c.name.includes(customerSearch) || 
      c.phone.includes(customerSearch) || 
      c.vatNumber.includes(customerSearch) || 
      c.address.includes(customerSearch)
    );

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white p-4 md:p-6 rounded-xl shadow-sm border border-gray-100 mb-6">
          <h3 className="text-lg font-bold mb-4 flex items-center"><Plus size={20} className="mr-2"/> 新增客戶檔</h3>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4 mb-3 md:mb-4">
            <input type="text" placeholder="客戶名稱" className="border p-2 rounded-lg flex-[2]" value={newCustomer.name} onChange={e => setNewCustomer({...newCustomer, name: e.target.value})} />
            <input type="text" placeholder="統一編號" className="border p-2 rounded-lg flex-1" value={newCustomer.vatNumber} onChange={e => setNewCustomer({...newCustomer, vatNumber: e.target.value})} />
            <input type="text" placeholder="聯絡電話" className="border p-2 rounded-lg flex-1" value={newCustomer.phone} onChange={e => setNewCustomer({...newCustomer, phone: e.target.value})} />
          </div>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4">
            <input type="text" placeholder="聯絡地址" className="border p-2 rounded-lg flex-[3]" value={newCustomer.address} onChange={e => setNewCustomer({...newCustomer, address: e.target.value})} />
            <button onClick={handleAddCustomer} className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700 transition font-bold flex-1">新增客戶</button>
          </div>
        </div>

        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden">
          <div className="p-4 border-b border-gray-100 bg-gray-50 flex flex-col md:flex-row items-start md:items-center justify-between gap-3">
            <h3 className="font-bold text-gray-700 flex items-center"><Users className="mr-2"/> 客戶總覽</h3>
            <div className="relative w-full md:w-auto">
              <input type="text" placeholder="搜尋名稱、統編、電話..." className="border p-2 pl-8 rounded-lg w-full md:w-64 text-sm outline-none focus:border-blue-500" value={customerSearch} onChange={e => setCustomerSearch(e.target.value)} />
              <Search size={16} className="absolute left-2.5 top-3 text-gray-400" />
            </div>
          </div>
          <div className="overflow-x-auto">
            <table className="w-full text-left border-collapse min-w-[700px]">
              <thead>
                <tr className="bg-white border-b border-gray-200">
                  <th className="p-4 font-semibold text-gray-600">客戶編號</th>
                  <th className="p-4 font-semibold text-gray-600">客戶名稱</th>
                  <th className="p-4 font-semibold text-gray-600">聯絡電話</th>
                  <th className="p-4 font-semibold text-gray-600">聯絡地址</th>
                  <th className="p-4 font-semibold text-gray-600">統一編號</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">操作</th>
                </tr>
              </thead>
              <tbody>
                {filteredCustomers.map(c => (
                  <tr key={c.id} className="border-b border-gray-50 hover:bg-gray-50">
                    <td className="p-4 text-gray-800">{c.id}</td>
                    <td className="p-4 font-medium text-gray-800">{c.name}</td>
                    <td className="p-4 text-gray-600">{c.phone}</td>
                    <td className="p-4 text-gray-600 text-sm truncate max-w-[200px]">{c.address}</td>
                    <td className="p-4 text-gray-600">{c.vatNumber}</td>
                    <td className="p-4 text-center">
                      <button onClick={() => handleDeleteCustomer(c.id)} className="text-red-500 hover:text-red-700 p-1"><Trash2 size={18} /></button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    );
  };

  const renderSuppliers = () => {
    const filteredSuppliers = suppliers.filter(s => 
      s.name.includes(supplierSearch) || 
      s.vatNumber.includes(supplierSearch) || 
      s.phone.includes(supplierSearch) || 
      s.address.includes(supplierSearch) ||
      s.note.includes(supplierSearch)
    );

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white p-4 md:p-6 rounded-xl shadow-sm border border-gray-100 mb-6">
          <h3 className="text-lg font-bold mb-4 flex items-center"><Plus size={20} className="mr-2"/> 新增供應商檔</h3>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4 mb-3 md:mb-4">
            <input type="text" placeholder="供應商名稱" className="border p-2 rounded-lg flex-[2]" value={newSupplier.name} onChange={e => setNewSupplier({...newSupplier, name: e.target.value})} />
            <input type="text" placeholder="統一編號" className="border p-2 rounded-lg flex-1" value={newSupplier.vatNumber} onChange={e => setNewSupplier({...newSupplier, vatNumber: e.target.value})} />
            <input type="text" placeholder="聯絡電話" className="border p-2 rounded-lg flex-1" value={newSupplier.phone} onChange={e => setNewSupplier({...newSupplier, phone: e.target.value})} />
          </div>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4 mb-3 md:mb-4">
            <input type="text" placeholder="聯絡地址" className="border p-2 rounded-lg flex-1" value={newSupplier.address} onChange={e => setNewSupplier({...newSupplier, address: e.target.value})} />
          </div>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4">
            <input type="text" placeholder="備註 (例如: 專供某類商品)" className="border p-2 rounded-lg flex-[3]" value={newSupplier.note} onChange={e => setNewSupplier({...newSupplier, note: e.target.value})} />
            <button onClick={handleAddSupplier} className="bg-indigo-600 text-white px-6 py-2 rounded-lg hover:bg-indigo-700 transition font-bold flex-1">新增供應商</button>
          </div>
        </div>

        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden">
          <div className="p-4 border-b border-gray-100 bg-gray-50 flex flex-col md:flex-row items-start md:items-center justify-between gap-3">
            <h3 className="font-bold text-gray-700 flex items-center"><Truck className="mr-2"/> 供應商總覽</h3>
            <div className="relative w-full md:w-auto">
              <input type="text" placeholder="搜尋名稱、統編、備註..." className="border p-2 pl-8 rounded-lg w-full md:w-64 text-sm outline-none focus:border-indigo-500" value={supplierSearch} onChange={e => setSupplierSearch(e.target.value)} />
              <Search size={16} className="absolute left-2.5 top-3 text-gray-400" />
            </div>
          </div>
          <div className="overflow-x-auto">
            <table className="w-full text-left border-collapse min-w-[800px]">
              <thead>
                <tr className="bg-white border-b border-gray-200">
                  <th className="p-4 font-semibold text-gray-600">編號</th>
                  <th className="p-4 font-semibold text-gray-600">供應商名稱</th>
                  <th className="p-4 font-semibold text-gray-600">統編</th>
                  <th className="p-4 font-semibold text-gray-600">聯絡電話</th>
                  <th className="p-4 font-semibold text-gray-600">地址</th>
                  <th className="p-4 font-semibold text-gray-600">備註</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">操作</th>
                </tr>
              </thead>
              <tbody>
                {filteredSuppliers.map(s => (
                  <tr key={s.id} className="border-b border-gray-50 hover:bg-gray-50">
                    <td className="p-4 text-gray-800">{s.id}</td>
                    <td className="p-4 font-medium text-indigo-700">{s.name}</td>
                    <td className="p-4 text-gray-600">{s.vatNumber}</td>
                    <td className="p-4 text-gray-600">{s.phone}</td>
                    <td className="p-4 text-gray-600 text-sm truncate max-w-[150px]">{s.address}</td>
                    <td className="p-4 text-gray-500 text-sm">{s.note}</td>
                    <td className="p-4 text-center">
                      <button onClick={() => handleDeleteSupplier(s.id)} className="text-red-500 hover:text-red-700 p-1"><Trash2 size={18} /></button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    );
  };

  const renderOrders = () => {
    if (selectedOrder) {
      const order = selectedOrder;
      const items = orderItems.filter(i => i.orderId === order.id);
      const customer = customers.find(c => c.id === order.customerId);

      return (
        <div className="p-4 md:p-6 print:hidden">
          <div className="flex justify-between items-center mb-4">
            <button onClick={() => setSelectedOrder(null)} className="text-gray-500 hover:text-gray-800 font-medium flex items-center text-sm md:text-base">← 返回訂單列表</button>
            <button onClick={() => handleDeleteOrder(order.id)} className="text-red-500 hover:text-red-700 font-bold flex items-center bg-red-50 px-3 py-1.5 md:px-4 md:py-2 rounded-lg transition text-sm md:text-base">
              <Trash2 size={18} className="mr-1 md:mr-2"/> 刪除訂單
            </button>
          </div>
          
          <div className="bg-white p-4 md:p-6 rounded-xl shadow-sm border border-gray-100 mb-6 flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
            <div>
              <h2 className="text-xl md:text-2xl font-bold text-gray-800 flex items-center gap-2"><Package className="text-blue-600"/> 訂單: {order.id}</h2>
              <p className="text-gray-500 mt-1 text-sm md:text-base">客戶: {customer?.name} | 日期: {order.date}</p>
            </div>
            <div className="text-left md:text-right">
              <p className="text-xs md:text-sm text-gray-500 font-bold">訂單總金額</p>
              <p className="text-2xl md:text-3xl font-extrabold text-blue-600">${order.totalAmount.toLocaleString()}</p>
            </div>
          </div>

          {order.status !== 'Completed' && (
            <div className="bg-blue-50 p-4 md:p-6 rounded-xl border border-blue-100 mb-6">
              <h3 className="text-lg font-bold mb-4 flex items-center text-blue-900"><Plus size={20} className="mr-2"/> 新增明細</h3>
              <div className="flex flex-col md:flex-row gap-3 items-end">
                <div className="w-full flex-1">
                  <label className="block text-xs font-bold text-blue-800 mb-1">選擇品項</label>
                  <select 
                    className="border border-blue-200 p-2 rounded-lg w-full bg-white text-sm" 
                    value={newItem.productId}
                    onChange={e => {
                      const p = products.find(x => x.id === e.target.value);
                      if(p) setNewItem({...newItem, productId: p.id, itemName: p.name, unitPrice: p.price, unitCost: p.cost});
                    }}
                  >
                    <option value="" disabled>-- 點擊選擇商品 --</option>
                    {products.map(p => <option key={p.id} value={p.id}>{p.name}</option>)}
                  </select>
                </div>
                <div className="w-full md:w-24 flex gap-2 md:block">
                  <div className="w-1/2 md:w-full">
                     <label className="block text-xs font-bold text-blue-800 mb-1">數量</label>
                     <input type="number" min="1" className="border border-blue-200 p-2 rounded-lg w-full text-sm" value={newItem.quantity} onChange={e => setNewItem({...newItem, quantity: parseInt(e.target.value) || 0})} />
                  </div>
                  <div className="w-1/2 md:w-full">
                     <label className="block text-xs font-bold text-blue-800 mb-1">單價</label>
                     <input type="number" min="0" className="border border-blue-200 p-2 rounded-lg w-full text-sm" value={newItem.unitPrice} onChange={e => setNewItem({...newItem, unitPrice: parseInt(e.target.value) || 0})} />
                  </div>
                </div>
                <button onClick={() => handleAddItem(order.id)} className="w-full md:w-auto bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700 transition font-bold whitespace-nowrap md:h-[38px] mt-2 md:mt-0">加入</button>
              </div>
            </div>
          )}

          <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden mb-6">
            <div className="overflow-x-auto">
              <table className="w-full text-left border-collapse min-w-[600px]">
                <thead>
                  <tr className="bg-gray-50 border-b border-gray-100">
                    <th className="p-4 font-semibold text-gray-600">明細編號</th>
                    <th className="p-4 font-semibold text-gray-600">品項</th>
                    <th className="p-4 font-semibold text-gray-600 text-right">數量</th>
                    <th className="p-4 font-semibold text-gray-600 text-right">單價</th>
                    <th className="p-4 font-semibold text-gray-600 text-right">總價</th>
                    {order.status !== 'Completed' && <th className="p-4 font-semibold text-gray-600 text-center">操作</th>}
                  </tr>
                </thead>
                <tbody>
                  {items.length === 0 ? <tr><td colSpan="6" className="p-8 text-center text-gray-400">尚無明細資料</td></tr> : 
                    items.map(i => (
                      <tr key={i.id} className="border-b border-gray-50 hover:bg-gray-50">
                        <td className="p-4 text-gray-500 text-sm">{i.id}</td>
                        <td className="p-4 font-medium text-gray-800">{i.itemName}</td>
                        <td className="p-4 text-right text-gray-600">{i.quantity}</td>
                        <td className="p-4 text-right text-gray-600">${i.unitPrice.toLocaleString()}</td>
                        <td className="p-4 text-right font-bold text-gray-800">${i.totalPrice.toLocaleString()}</td>
                        {order.status !== 'Completed' && (
                          <td className="p-4 text-center">
                            <button onClick={() => deleteItem(i.id, order.id, i.totalPrice, i.quantity * (i.unitCost||0))} className="text-red-500 hover:text-red-700 p-1"><Trash2 size={18} /></button>
                          </td>
                        )}
                      </tr>
                    ))
                  }
                </tbody>
              </table>
            </div>
          </div>

          {order.status !== 'Completed' && items.length > 0 && (
            <div className="flex justify-end">
              <button onClick={() => handleCompleteOrder(order.id)} className="w-full md:w-auto bg-green-600 text-white px-8 py-3 rounded-xl hover:bg-green-700 transition font-bold flex items-center justify-center text-lg shadow-md">
                <CheckCircle className="mr-2"/> 確認訂單 (完成並扣庫存)
              </button>
            </div>
          )}
        </div>
      );
    }

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white p-4 md:p-6 rounded-xl shadow-sm border border-gray-100 mb-6">
          <h3 className="text-lg font-bold mb-4 flex items-center"><Plus size={20} className="mr-2"/> 建立新訂單</h3>
          <div className="flex flex-col md:flex-row gap-3 md:gap-4">
            <select className="border p-2 rounded-lg flex-1 bg-white" value={newOrder.customerId} onChange={e => setNewOrder({...newOrder, customerId: e.target.value})}>
              <option value="">-- 選擇客戶 --</option>
              {customers.map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
            </select>
            <input type="date" className="border p-2 rounded-lg flex-1 bg-white" value={newOrder.date} onChange={e => setNewOrder({...newOrder, date: e.target.value})} />
            <button onClick={handleAddOrder} className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700 transition font-bold">建立</button>
          </div>
        </div>

        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden">
          <div className="overflow-x-auto">
            <table className="w-full text-left border-collapse min-w-[600px]">
              <thead>
                <tr className="bg-gray-50 border-b border-gray-100">
                  <th className="p-4 font-semibold text-gray-600">訂單編號</th>
                  <th className="p-4 font-semibold text-gray-600">客戶</th>
                  <th className="p-4 font-semibold text-gray-600">建立日期</th>
                  <th className="p-4 font-semibold text-gray-600 text-right">總金額</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">狀態</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">操作</th>
                </tr>
              </thead>
              <tbody>
                {orders.map(o => {
                  const customer = customers.find(c => c.id === o.customerId);
                  return (
                    <tr key={o.id} className="border-b border-gray-50 hover:bg-gray-50 transition">
                      <td className="p-4 font-medium text-blue-600">{o.id}</td>
                      <td className="p-4">{customer ? customer.name : '未知客戶'}</td>
                      <td className="p-4 text-gray-600">{o.date}</td>
                      <td className="p-4 text-right font-bold">${o.totalAmount.toLocaleString()}</td>
                      <td className="p-4 text-center">
                        <span className={`px-3 py-1 rounded-full text-xs font-bold ${o.status === 'Completed' ? 'bg-green-100 text-green-700' : 'bg-yellow-100 text-yellow-700'}`}>
                          {o.status === 'Completed' ? '已完成' : '草稿'}
                        </span>
                      </td>
                      <td className="p-4 text-center">
                        <button onClick={() => setSelectedOrder(o)} className="text-blue-600 hover:text-blue-800 font-medium text-sm flex items-center justify-center w-full">
                          管理明細 <ChevronRight size={16} />
                        </button>
                      </td>
                    </tr>
                  )
                })}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    );
  };

  const renderQuickOrderTab = () => {
    const totalAmount = quickOrder.items.reduce((sum, item) => sum + (item.quantity * item.unitPrice), 0);

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white p-4 md:p-8 rounded-xl shadow-sm border border-gray-100 max-w-5xl mx-auto">
          <h2 className="text-xl md:text-2xl font-bold text-gray-800 flex items-center mb-4 md:mb-6 border-b pb-4">
            <Zap className="mr-2 text-yellow-500" size={28}/> 快速 Key 單 (銷貨)
          </h2>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4 md:gap-6 mb-6 md:mb-8 bg-gray-50 p-4 md:p-6 rounded-lg border border-gray-100">
            <div>
              <label className="block text-sm font-bold text-gray-700 mb-2">客戶名稱</label>
              <select className="w-full border p-3 rounded-lg bg-white" value={quickOrder.customerId} onChange={e => setQuickOrder({...quickOrder, customerId: e.target.value})}>
                <option value="">-- 選擇客戶 --</option>
                {customers.map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
              </select>
            </div>
            <div>
              <label className="block text-sm font-bold text-gray-700 mb-2">訂單日期</label>
              <input type="date" className="w-full border p-3 rounded-lg bg-white" value={quickOrder.date} onChange={e => setQuickOrder({...quickOrder, date: e.target.value})} />
            </div>
          </div>
          
          <div className="mb-8">
            <div className="flex justify-between items-end mb-4">
              <h3 className="text-lg font-bold text-gray-800">訂單明細</h3>
              <button onClick={() => setQuickOrder(p => ({...p, items: [...p.items, { localId: Date.now(), productId: '', itemName: '', quantity: 1, unitPrice: 0, unitCost: 0 }] }))} className="text-blue-600 hover:text-blue-800 font-bold flex items-center bg-blue-50 px-3 py-1.5 rounded-lg transition text-sm">
                <Plus size={16} className="mr-1"/> 新增一列
              </button>
            </div>
            
            <div className="overflow-x-auto">
              <div className="space-y-3 min-w-[600px] pb-2">
                <div className="flex gap-2 md:gap-3 px-2 text-sm font-bold text-gray-600">
                  <div className="w-6"></div>
                  <div className="flex-1">品項</div>
                  <div className="w-24 text-center">數量</div>
                  <div className="w-28 text-center">單價</div>
                  <div className="w-24 text-right">小計</div>
                  <div className="w-8"></div>
                </div>

                {quickOrder.items.map((item, index) => (
                  <div key={item.localId} className="flex gap-2 md:gap-3 items-center bg-gray-50 p-2 md:p-3 rounded-lg border border-gray-100 transition-all hover:bg-white shadow-sm">
                    <span className="w-6 text-center font-bold text-gray-400 text-sm">{index + 1}.</span>
                    
                    <select 
                      className="flex-1 border p-2 rounded-md text-sm text-gray-700 bg-white"
                      value={item.productId}
                      onChange={e => {
                        const p = products.find(x => x.id === e.target.value);
                        if (p) setQuickOrder(prev => ({...prev, items: prev.items.map(i => i.localId === item.localId ? { ...i, productId: p.id, itemName: p.name, unitPrice: p.price, unitCost: p.cost } : i) }));
                      }}
                    >
                      <option value="" disabled>-- 選擇品項 --</option>
                      {products.map(p => <option key={p.id} value={p.id}>{p.name} (庫存:{p.stock})</option>)}
                    </select>

                    <div className="w-24 flex items-center">
                      <input type="number" min="1" className="w-full border p-2 rounded-md text-sm text-center" value={item.quantity} onChange={e => setQuickOrder(prev => ({...prev, items: prev.items.map(i => i.localId === item.localId ? { ...i, quantity: parseInt(e.target.value)||0 } : i) }))} />
                    </div>
                    <div className="w-28 flex items-center">
                      <input type="number" min="0" className="w-full border p-2 rounded-md text-sm text-right" value={item.unitPrice} onChange={e => setQuickOrder(prev => ({...prev, items: prev.items.map(i => i.localId === item.localId ? { ...i, unitPrice: parseInt(e.target.value)||0 } : i) }))} />
                    </div>
                    <div className="w-24 text-right font-extrabold text-gray-700 text-sm">${(item.quantity * item.unitPrice).toLocaleString()}</div>
                    <button onClick={() => setQuickOrder(prev => ({...prev, items: prev.items.filter(i => i.localId !== item.localId) }))} className="w-8 h-8 flex items-center justify-center text-red-500 hover:bg-red-50 rounded-full transition"><Trash2 size={18} /></button>
                  </div>
                ))}
              </div>
            </div>
          </div>
          <div className="flex flex-col md:flex-row justify-between items-center border-t pt-6 gap-4">
            <div className="text-gray-500 font-medium">總計金額 <span className="text-2xl md:text-3xl font-extrabold text-blue-600 ml-2">${totalAmount.toLocaleString()}</span></div>
            <button onClick={handleQuickOrderSubmit} className="w-full md:w-auto bg-blue-600 text-white px-8 py-3 rounded-xl hover:bg-blue-700 transition font-bold flex items-center justify-center shadow-md text-base md:text-lg"><Save className="mr-2" size={20}/> 儲存單據</button>
          </div>
        </div>
      </div>
    );
  };

  const renderPurchase = () => {
    const totalAmount = quickPurchase.items.reduce((sum, item) => sum + (item.quantity * item.unitCost), 0);

    return (
      <div className="p-4 md:p-6 print:hidden">
         <div className="bg-white p-4 md:p-8 rounded-xl shadow-sm border border-gray-100 max-w-5xl mx-auto">
          <h2 className="text-xl md:text-2xl font-bold text-gray-800 flex items-center mb-4 md:mb-6 border-b pb-4">
            <ShoppingCart className="mr-2 text-indigo-500" size={28}/> 採購進貨單 (入庫)
          </h2>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4 md:gap-6 mb-6 md:mb-8 bg-indigo-50 p-4 md:p-6 rounded-lg border border-indigo-100">
            <div>
              <label className="block text-sm font-bold text-indigo-900 mb-2">供應商名稱</label>
              <select className="w-full border p-3 rounded-lg bg-white" value={quickPurchase.supplierId} onChange={e => setQuickPurchase({...quickPurchase, supplierId: e.target.value})}>
                <option value="">-- 選擇供應商 --</option>
                {suppliers.map(s => <option key={s.id} value={s.id}>{s.name}</option>)}
              </select>
            </div>
            <div>
              <label className="block text-sm font-bold text-indigo-900 mb-2">進貨日期</label>
              <input type="date" className="w-full border p-3 rounded-lg bg-white" value={quickPurchase.date} onChange={e => setQuickPurchase({...quickPurchase, date: e.target.value})} />
            </div>
          </div>

          <div className="mb-8">
            <div className="flex justify-between items-end mb-4">
              <h3 className="text-lg font-bold text-gray-800">進貨明細</h3>
              <button onClick={() => setQuickPurchase(p => ({...p, items: [...p.items, { localId: Date.now(), productId: '', itemName: '', quantity: 1, unitCost: 0 }] }))} className="text-indigo-600 hover:text-indigo-800 font-bold flex items-center bg-indigo-50 px-3 py-1.5 rounded-lg transition text-sm">
                <Plus size={16} className="mr-1"/> 新增一列
              </button>
            </div>

            <div className="overflow-x-auto">
              <div className="space-y-3 min-w-[600px] pb-2">
                <div className="flex gap-2 md:gap-3 px-2 text-sm font-bold text-gray-600">
                  <div className="w-6"></div>
                  <div className="flex-1">品項</div>
                  <div className="w-24 text-center">數量</div>
                  <div className="w-28 text-center">單價</div>
                  <div className="w-24 text-right">小計</div>
                  <div className="w-8"></div>
                </div>

                {quickPurchase.items.map((item, index) => (
                  <div key={item.localId} className="flex gap-2 md:gap-3 items-center bg-gray-50 p-2 md:p-3 rounded-lg border border-gray-100 transition-all hover:bg-white shadow-sm">
                    <span className="w-6 text-center font-bold text-gray-400 text-sm">{index + 1}.</span>
                    
                    <select 
                      className="flex-1 border p-2 rounded-md text-sm text-gray-700 bg-white"
                      value={item.productId}
                      onChange={e => {
                        const p = products.find(x => x.id === e.target.value);
                        if (p) {
                           setQuickPurchase(prev => ({...prev, items: prev.items.map(i => i.localId === item.localId ? { ...i, productId: p.id, itemName: p.name, unitCost: p.cost } : i) }));
                        }
                      }}
                    >
                      <option value="" disabled>-- 選擇品項 --</option>
                      {products.map(p => <option key={p.id} value={p.id}>{p.name}</option>)}
                    </select>

                    <div className="w-24 flex items-center">
                      <input type="number" min="1" className="w-full border p-2 rounded-md text-sm text-center" value={item.quantity} onChange={e => setQuickPurchase(prev => ({...prev, items: prev.items.map(i => i.localId === item.localId ? { ...i, quantity: parseInt(e.target.value)||0 } : i) }))} />
                    </div>
                    <div className="w-28 flex items-center">
                      <input type="number" min="0" className="w-full border p-2 rounded-md text-sm text-right" value={item.unitCost} onChange={e => setQuickPurchase(prev => ({...prev, items: prev.items.map(i => i.localId === item.localId ? { ...i, unitCost: parseInt(e.target.value)||0 } : i) }))} />
                    </div>
                    <div className="w-24 text-right font-extrabold text-gray-700 text-sm">${(item.quantity * item.unitCost).toLocaleString()}</div>
                    <button onClick={() => setQuickPurchase(prev => ({...prev, items: prev.items.filter(i => i.localId !== item.localId) }))} className="w-8 h-8 flex items-center justify-center text-red-500 hover:bg-red-50 rounded-full transition"><Trash2 size={18} /></button>
                  </div>
                ))}
              </div>
            </div>
          </div>

          <div className="flex flex-col md:flex-row justify-between items-center border-t pt-6 gap-4">
             <div className="text-gray-500 font-medium">進貨總金額 <span className="text-2xl md:text-3xl font-extrabold text-indigo-600 ml-2">${totalAmount.toLocaleString()}</span></div>
             <button onClick={handlePurchaseSubmit} className="w-full md:w-auto bg-indigo-600 text-white px-8 py-3 rounded-xl hover:bg-indigo-700 transition font-bold flex items-center justify-center shadow-md text-base md:text-lg"><Save className="mr-2" size={20}/> 確認進貨入庫</button>
          </div>
        </div>
      </div>
    );
  };

  const renderPurchaseList = () => {
    if (selectedPurchaseOrder) {
      const order = selectedPurchaseOrder;
      const supplier = suppliers.find(s => s.id === order.supplierId);

      return (
        <div className="p-4 md:p-6 print:p-0">
          <div className="flex justify-between items-center mb-6 print:hidden">
            <button onClick={() => setSelectedPurchaseOrder(null)} className="text-gray-500 hover:text-gray-800 font-medium flex items-center text-sm md:text-base">
              ← 返回採購紀錄
            </button>
            <div className="flex gap-2">
              <button onClick={() => window.print()} className="bg-gray-600 hover:bg-gray-700 text-white font-bold flex items-center px-4 py-2 rounded-lg transition shadow-sm text-sm md:text-base">
                <Printer size={18} className="mr-2"/> 列印 (PDF)
              </button>
              <button id="download-btn" onClick={() => handleDownloadImage('purchase-form-capture', `採購單_${order.id}`)} className="bg-indigo-600 hover:bg-indigo-700 text-white font-bold flex items-center px-4 py-2 rounded-lg transition shadow-sm text-sm md:text-base">
                <Download size={18} className="mr-2"/> 下載圖片
              </button>
            </div>
          </div>

          <div id="purchase-form-capture" className="bg-white p-8 md:p-12 rounded-xl shadow-sm border border-gray-100 max-w-4xl mx-auto print:shadow-none print:border-none print:max-w-none print:w-full print:p-0">
            <div className="text-center mb-8 border-b-2 border-gray-800 pb-6">
              <h1 className="text-3xl font-extrabold text-gray-900 mb-2">{systemSettings.companyName}</h1>
              <h2 className="text-xl font-bold text-gray-600 tracking-widest">採 購 進 貨 單</h2>
            </div>

            <div className="flex flex-col md:flex-row justify-between mb-8 gap-4 text-sm md:text-base">
              <div>
                <p className="text-gray-700 mb-1"><span className="font-bold w-20 inline-block">供應商：</span> {supplier?.name || '未知供應商'}</p>
                <p className="text-gray-700 mb-1"><span className="font-bold w-20 inline-block">統一編號：</span> {supplier?.vatNumber || '無'}</p>
                <p className="text-gray-700"><span className="font-bold w-20 inline-block">聯絡電話：</span> {supplier?.phone || '無'}</p>
              </div>
              <div className="md:text-right">
                <p className="text-gray-700 mb-1"><span className="font-bold">單據號碼：</span> {order.id}</p>
                <p className="text-gray-700"><span className="font-bold">進貨日期：</span> {order.date}</p>
              </div>
            </div>

            <table className="w-full text-left border-collapse mb-8 text-sm md:text-base">
              <thead>
                <tr className="border-b-2 border-gray-800 text-gray-800">
                  <th className="py-3 px-2 font-bold w-12 text-center">項次</th>
                  <th className="py-3 px-2 font-bold">品項名稱</th>
                  <th className="py-3 px-2 font-bold text-right">數量</th>
                  <th className="py-3 px-2 font-bold text-right">單價</th>
                  <th className="py-3 px-2 font-bold text-right">小計金額</th>
                </tr>
              </thead>
              <tbody>
                {order.items.map((item, index) => (
                  <tr key={index} className="border-b border-gray-200">
                    <td className="py-4 px-2 text-center text-gray-500">{index + 1}</td>
                    <td className="py-4 px-2 text-gray-800 font-medium">{item.itemName}</td>
                    <td className="py-4 px-2 text-gray-800 text-right">{item.quantity}</td>
                    <td className="py-4 px-2 text-gray-800 text-right">${item.unitCost?.toLocaleString()}</td>
                    <td className="py-4 px-2 text-gray-800 text-right font-bold">${(item.quantity * item.unitCost).toLocaleString()}</td>
                  </tr>
                ))}
              </tbody>
            </table>

            <div className="flex justify-end border-t-2 border-gray-800 pt-4 mb-16">
              <div className="text-right pr-2">
                <p className="text-sm text-gray-500 font-bold mb-1">採購總金額</p>
                <p className="text-3xl font-extrabold text-indigo-700">${order.totalAmount.toLocaleString()}</p>
              </div>
            </div>

            <div className="mt-8 text-sm text-gray-500 border-t border-gray-200 pt-4 flex flex-col md:flex-row justify-between">
              <div>
                <p>公司地址：{systemSettings.address}</p>
                <p>聯絡電話：{systemSettings.phone}</p>
              </div>
              <div className="mt-2 md:mt-0 md:text-right">
                <p>統一編號：{systemSettings.vatNumber}</p>
                <p>列印日期：{new Date().toISOString().split('T')[0]}</p>
              </div>
            </div>
          </div>
        </div>
      );
    }

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden">
          <div className="p-4 border-b border-gray-100 bg-gray-50 flex flex-col md:flex-row items-start md:items-center justify-between gap-3">
            <h3 className="font-bold text-gray-700 flex items-center"><ClipboardList className="mr-2"/> 採購紀錄總覽</h3>
          </div>
          <div className="overflow-x-auto">
            <table className="w-full text-left border-collapse min-w-[600px]">
              <thead>
                <tr className="bg-white border-b border-gray-200">
                  <th className="p-4 font-semibold text-gray-600">單據號碼</th>
                  <th className="p-4 font-semibold text-gray-600">供應商</th>
                  <th className="p-4 font-semibold text-gray-600">進貨日期</th>
                  <th className="p-4 font-semibold text-gray-600 text-right">總金額</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">狀態</th>
                  <th className="p-4 font-semibold text-gray-600 text-center">操作</th>
                </tr>
              </thead>
              <tbody>
                {purchaseOrders.length === 0 ? (
                  <tr><td colSpan="6" className="p-8 text-center text-gray-400">目前尚無採購紀錄</td></tr>
                ) : (
                  purchaseOrders.map(order => {
                    const supplier = suppliers.find(s => s.id === order.supplierId);
                    return (
                      <tr key={order.id} className="border-b border-gray-50 hover:bg-gray-50 transition">
                        <td className="p-4 font-medium text-indigo-600">{order.id}</td>
                        <td className="p-4">{supplier ? supplier.name : '未知供應商'}</td>
                        <td className="p-4 text-gray-600">{order.date}</td>
                        <td className="p-4 text-right font-bold">${order.totalAmount.toLocaleString()}</td>
                        <td className="p-4 text-center">
                          <span className="px-3 py-1 rounded-full text-xs font-bold bg-green-100 text-green-700">已入庫</span>
                        </td>
                        <td className="p-4 text-center">
                          <button onClick={() => setSelectedPurchaseOrder(order)} className="text-indigo-600 hover:text-indigo-800 font-medium text-sm flex items-center justify-center w-full">
                            表單檢視 <ChevronRight size={16} />
                          </button>
                        </td>
                      </tr>
                    );
                  })
                )}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    );
  };

  const renderFinancials = () => {
    const accountsSummary = journalEntries.reduce((acc, curr) => {
      if(!acc[curr.account]) acc[curr.account] = { debit: 0, credit: 0, type: '' };
      acc[curr.account].debit += curr.debit;
      acc[curr.account].credit += curr.credit;
      
      if(['應收帳款', '存貨'].includes(curr.account)) acc[curr.account].type = 'asset';
      else if(['應付帳款'].includes(curr.account)) acc[curr.account].type = 'liability';
      else if(['銷貨收入'].includes(curr.account)) acc[curr.account].type = 'revenue';
      else if(['銷貨成本'].includes(curr.account)) acc[curr.account].type = 'expense';

      return acc;
    }, {});

    const accountBalances = Object.keys(accountsSummary).map(accName => {
      const data = accountsSummary[accName];
      const balance = ['asset', 'expense'].includes(data.type) ? data.debit - data.credit : data.credit - data.debit;
      return { name: accName, type: data.type, balance };
    });

    const maxBalance = Math.max(...accountBalances.map(a => Math.abs(a.balance)), 1);

    return (
      <div className="p-4 md:p-6 print:hidden">
        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden mb-6 md:mb-8">
           <div className="p-4 md:p-6 border-b border-gray-100 bg-gray-50 flex flex-col md:flex-row justify-between items-start md:items-center gap-2">
             <h2 className="text-lg md:text-xl font-bold text-gray-800 flex items-center"><BookOpen className="mr-2 text-indigo-600"/> 財務報表與會計科目</h2>
           </div>
           
           <div className="p-4 md:p-8 overflow-x-auto">
             <div className="min-w-[500px]">
               <h3 className="text-base md:text-lg font-bold text-gray-700 mb-6 text-center border-b pb-2">會計科目餘額圖表 (T-Account Balances)</h3>
               <div className="space-y-4 md:space-y-6 max-w-4xl mx-auto">
                 {accountBalances.map(acc => {
                   let colorClass = "bg-gray-400";
                   if(acc.type === 'asset') colorClass = "bg-blue-500";
                   if(acc.type === 'liability') colorClass = "bg-orange-500";
                   if(acc.type === 'revenue') colorClass = "bg-green-500";
                   if(acc.type === 'expense') colorClass = "bg-red-500";
                   
                   const widthPct = Math.max(5, (Math.abs(acc.balance) / maxBalance) * 100);

                   return (
                    <div key={acc.name} className="flex items-center gap-2 md:gap-4">
                      <div className="w-24 md:w-32 text-right font-bold text-gray-700 text-sm md:text-base">{acc.name}</div>
                      <div className="flex-1 bg-gray-100 rounded-full h-6 md:h-8 overflow-hidden relative">
                         <div className={`h-full ${colorClass} transition-all duration-1000 flex items-center px-2 md:px-3`} style={{ width: `${widthPct}%` }}>
                           <span className="text-white text-[10px] md:text-xs font-bold drop-shadow-md">
                             ${acc.balance.toLocaleString()}
                           </span>
                         </div>
                      </div>
                    </div>
                   )
                 })}
               </div>
               <div className="flex flex-wrap justify-center gap-4 md:gap-6 mt-8">
                  <span className="text-xs font-bold text-gray-500 flex items-center"><span className="inline-block w-3 h-3 bg-blue-500 mr-1 rounded-sm"></span> 資產</span>
                  <span className="text-xs font-bold text-gray-500 flex items-center"><span className="inline-block w-3 h-3 bg-orange-500 mr-1 rounded-sm"></span> 負債</span>
                  <span className="text-xs font-bold text-gray-500 flex items-center"><span className="inline-block w-3 h-3 bg-green-500 mr-1 rounded-sm"></span> 收入</span>
                  <span className="text-xs font-bold text-gray-500 flex items-center"><span className="inline-block w-3 h-3 bg-red-500 mr-1 rounded-sm"></span> 費用</span>
               </div>
             </div>
           </div>
        </div>

        <div className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden">
          <div className="p-3 md:p-4 bg-gray-800 text-white font-bold text-sm md:text-base">原始會計分錄明細 (Journal Entries)</div>
          <div className="overflow-x-auto">
            <table className="w-full text-left border-collapse min-w-[600px]">
              <thead>
                <tr className="bg-gray-100">
                  <th className="p-3 font-semibold text-xs md:text-sm">日期</th>
                  <th className="p-3 font-semibold text-xs md:text-sm">單據號碼</th>
                  <th className="p-3 font-semibold text-xs md:text-sm">會計科目</th>
                  <th className="p-3 font-semibold text-xs md:text-sm">摘要說明</th>
                  <th className="p-3 font-semibold text-xs md:text-sm text-right">借方 (Debit)</th>
                  <th className="p-3 font-semibold text-xs md:text-sm text-right">貸方 (Credit)</th>
                </tr>
              </thead>
              <tbody>
                {journalEntries.map(j => (
                  <tr key={j.id} className="border-b border-gray-50 hover:bg-orange-50 transition">
                    <td className="p-3 text-gray-600 text-xs md:text-sm">{j.date}</td>
                    <td className="p-3 text-gray-500 text-xs md:text-sm">{j.refId}</td>
                    <td className="p-3 font-bold text-gray-800 text-sm md:text-base">{j.account}</td>
                    <td className="p-3 text-gray-600 text-xs md:text-sm">{j.description}</td>
                    <td className="p-3 text-right font-bold text-blue-600 text-sm md:text-base">{j.debit > 0 ? `$${j.debit.toLocaleString()}` : '-'}</td>
                    <td className="p-3 text-right font-bold text-green-600 text-sm md:text-base">{j.credit > 0 ? `$${j.credit.toLocaleString()}` : '-'}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    );
  };

  const renderSettings = () => (
    <div className="p-4 md:p-6 print:hidden">
      <div className="bg-white p-6 md:p-8 rounded-xl shadow-sm border border-gray-100 max-w-2xl mx-auto">
         <h2 className="text-xl md:text-2xl font-bold text-gray-800 flex items-center mb-4 md:mb-6 border-b pb-4">
            <Settings className="mr-2 text-gray-600" size={28}/> 系統設定
         </h2>
         
         <div className="space-y-4 md:space-y-6">
           <div>
             <label className="block text-sm font-bold text-gray-700 mb-2">公司名稱</label>
             <input type="text" className="w-full border p-3 rounded-lg focus:border-blue-500 outline-none text-sm md:text-base" value={systemSettings.companyName} onChange={e => setSystemSettings({...systemSettings, companyName: e.target.value})} />
           </div>
           <div>
             <label className="block text-sm font-bold text-gray-700 mb-2">公司電話</label>
             <input type="text" className="w-full border p-3 rounded-lg focus:border-blue-500 outline-none text-sm md:text-base" value={systemSettings.phone} onChange={e => setSystemSettings({...systemSettings, phone: e.target.value})} />
           </div>
           <div>
             <label className="block text-sm font-bold text-gray-700 mb-2">統一編號</label>
             <input type="text" className="w-full border p-3 rounded-lg focus:border-blue-500 outline-none text-sm md:text-base" value={systemSettings.vatNumber} onChange={e => setSystemSettings({...systemSettings, vatNumber: e.target.value})} />
           </div>
           <div>
             <label className="block text-sm font-bold text-gray-700 mb-2">聯絡地址</label>
             <input type="text" className="w-full border p-3 rounded-lg focus:border-blue-500 outline-none text-sm md:text-base" value={systemSettings.address} onChange={e => setSystemSettings({...systemSettings, address: e.target.value})} />
           </div>

           <div className="mt-8 flex justify-end pt-6 border-t border-gray-100">
              <button onClick={handleSaveSettings} className="w-full md:w-auto bg-gray-800 text-white px-8 py-3 rounded-xl hover:bg-gray-900 transition font-bold flex items-center justify-center shadow-md">
                <Save className="mr-2" size={20}/> 儲存設定
              </button>
           </div>
         </div>
      </div>

      {/* 資料備份與還原區塊 */}
      <div className="bg-white p-6 md:p-8 rounded-xl shadow-sm border border-gray-100 max-w-2xl mx-auto mt-6 border-t-4 border-t-indigo-500">
         <h2 className="text-xl font-bold text-gray-800 flex items-center mb-6 border-b pb-4">
            <DownloadCloud className="mr-2 text-indigo-600" size={28}/> 資料備份與還原 (可存至 Google Drive)
         </h2>
         <p className="text-sm text-gray-600 mb-6 leading-relaxed">
            透過下方的按鈕，您可以將系統內所有的客戶、商品、訂單與財務資料打包成 <strong>.json</strong> 備份檔下載。您可以將此檔案上傳至您的 <strong>Google Drive / 雲端硬碟</strong> 進行安全備份。若未來需要還原資料，只需重新匯入該檔案即可。
         </p>

         <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div className="bg-indigo-50 p-6 rounded-xl border border-indigo-100 flex flex-col items-center text-center">
               <DownloadCloud className="text-indigo-500 mb-3" size={40} />
               <h3 className="font-bold text-gray-800 mb-2">步驟 1：匯出當前資料</h3>
               <p className="text-xs text-gray-500 mb-4">將所有系統資料下載到您的設備中</p>
               <button onClick={handleExportData} className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2.5 rounded-lg transition shadow flex items-center justify-center">
                 <Download size={18} className="mr-2"/> 匯出資料 (Export)
               </button>
            </div>
            
            <div className="bg-emerald-50 p-6 rounded-xl border border-emerald-100 flex flex-col items-center text-center">
               <UploadCloud className="text-emerald-500 mb-3" size={40} />
               <h3 className="font-bold text-gray-800 mb-2">步驟 2：從備份檔還原</h3>
               <p className="text-xs text-gray-500 mb-4">選擇您之前下載的 .json 備份檔</p>
               <label className="w-full bg-emerald-600 hover:bg-emerald-700 text-white font-bold py-2.5 rounded-lg transition shadow flex items-center justify-center cursor-pointer">
                 <UploadCloud size={18} className="mr-2"/> 匯入資料 (Import)
                 <input type="file" accept=".json" className="hidden" onChange={handleImportData} />
               </label>
            </div>
         </div>
      </div>
    </div>
  );

  const navItems = [
    { id: 'dashboard', icon: TrendingUp, label: '儀表板' },
    { id: 'quickOrder', icon: Zap, label: '快速 Key 單', color: 'text-yellow-400' },
    { id: 'purchases', icon: ShoppingCart, label: '進貨 Key 單' },
    { id: 'purchaseList', icon: ClipboardList, label: '採購作業(表單)' },
    { id: 'orders', icon: FileText, label: '訂單作業' },
    { id: 'products', icon: Tag, label: '商品及庫存' },
    { id: 'customers', icon: Users, label: '客戶檔' },
    { id: 'suppliers', icon: Truck, label: '供應商檔' },
    { id: 'financials', icon: BookOpen, label: '財務報表' },
    { id: 'settings', icon: Settings, label: '系統設定' },
  ];

  return (
    <div className="h-screen w-full bg-gray-100 flex font-sans overflow-hidden">
      
      {isSidebarOpen && (
        <div 
          className="fixed inset-0 bg-black/50 z-20 lg:hidden transition-opacity print:hidden"
          onClick={() => setIsSidebarOpen(false)}
        />
      )}

      <aside className={`
        fixed inset-y-0 left-0 z-30 w-64 bg-gray-900 text-white flex flex-col shadow-2xl print:hidden
        transform transition-transform duration-300 ease-in-out
        ${isSidebarOpen ? 'translate-x-0' : '-translate-x-full'}
        lg:relative lg:translate-x-0 lg:shadow-xl
      `}>
        <div className="p-6 flex items-center justify-between border-b border-gray-800">
          <div className="flex items-center gap-3">
            <Package size={32} className="text-blue-400" />
            <h1 className="text-xl font-extrabold tracking-wide leading-tight">
              五金 ERP<br/>
              <span className="text-xs text-gray-400 font-normal truncate block w-32">{systemSettings.companyName}</span>
            </h1>
          </div>
          <button onClick={() => setIsSidebarOpen(false)} className="lg:hidden text-gray-400 hover:text-white focus:outline-none">
            <X size={24} />
          </button>
        </div>
        <nav className="flex-1 px-4 py-6 space-y-2 overflow-y-auto">
          {navItems.map(item => (
            <button 
              key={item.id} 
              onClick={() => { 
                setActiveTab(item.id); 
                setSelectedOrder(null);
                setSelectedPurchaseOrder(null);
                setIsSidebarOpen(false);
              }} 
              className={`w-full flex items-center px-4 py-3 rounded-xl transition font-bold text-sm md:text-base ${activeTab === item.id ? 'bg-blue-600 text-white shadow-md' : 'text-gray-400 hover:bg-gray-800 hover:text-white'}`}
            >
              <item.icon size={20} className={`mr-3 ${item.color || ''}`} /> {item.label}
            </button>
          ))}
        </nav>
      </aside>

      <div className="flex-1 flex flex-col h-screen overflow-hidden bg-gray-50 relative print:h-auto print:bg-white print:overflow-visible">
        
        <header className="bg-white shadow-sm border-b border-gray-200 p-4 flex items-center lg:hidden z-10 sticky top-0 print:hidden">
          <button 
            onClick={() => setIsSidebarOpen(true)} 
            className="text-gray-600 hover:text-gray-900 focus:outline-none p-1"
          >
            <Menu size={28} />
          </button>
          <h1 className="ml-3 text-lg font-extrabold text-gray-800 tracking-wide truncate">
            {navItems.find(n => n.id === activeTab)?.label || '五金 ERP'}
          </h1>
        </header>

        <main className="flex-1 overflow-y-auto w-full print:overflow-visible print:p-0">
          <div className="pb-10 print:pb-0"> 
            {activeTab === 'dashboard' && renderDashboard()}
            {activeTab === 'quickOrder' && renderQuickOrderTab()}
            {activeTab === 'purchases' && renderPurchase()}
            {activeTab === 'purchaseList' && renderPurchaseList()}
            {activeTab === 'orders' && renderOrders()}
            {activeTab === 'products' && renderProducts()}
            {activeTab === 'customers' && renderCustomers()}
            {activeTab === 'suppliers' && renderSuppliers()}
            {activeTab === 'financials' && renderFinancials()}
            {activeTab === 'settings' && renderSettings()}
          </div>
        </main>
      </div>
    </div>
  );
};

export default HardwareERP;
