/* ============================================================
 * دكانة — store-core.js
 * منطق المتجر النقي (بدون DOM وبدون Firebase): حسابات الأسعار
 * والشحن والخصومات والمخزون وأكواد الطلبات والتقارير.
 * يعمل في المتصفح (window.DokkanaCore) وفي Node (للاختبارات).
 * ============================================================ */
(function (root, factory) {
  const core = factory();
  if (typeof module !== "undefined" && module.exports) module.exports = core;
  root.DokkanaCore = core;
})(typeof self !== "undefined" ? self : globalThis, function () {
  "use strict";

  // محافظات مصر الـ 27
  const GOVS = ["القاهرة","الجيزة","الإسكندرية","الدقهلية","الشرقية","الغربية","المنوفية","القليوبية","البحيرة","كفر الشيخ","دمياط","بورسعيد","الإسماعيلية","السويس","الفيوم","بني سويف","المنيا","أسيوط","سوهاج","قنا","الأقصر","أسوان","البحر الأحمر","مطروح","شمال سيناء","جنوب سيناء","الوادي الجديد"];

  const ORDER_STATUSES = ["new", "confirmed", "courier", "shipped", "delivered", "canceled"];

  // أبجدية أكواد الطلبات: بدون الحروف المتشابهة (0/O و 1/I/L)
  const CODE_ALPHABET = "ABCDEFGHJKMNPQRSTUVWXYZ23456789";

  /* ---------- تنسيق ---------- */

  // تنسيق سعر بالجنيه — تجميع يدوي للأرقام عشان يشتغل بنفس الشكل في كل المتصفحات
  function eg(n) {
    const v = Math.round(Number(n) || 0);
    return String(v).replace(/\B(?=(\d{3})+(?!\d))/g, ",") + " ج.م";
  }

  // رقم موبايل مصري: 11 رقم يبدأ بـ 01
  function validEgyptPhone(p) {
    return /^01[0-9]{9}$/.test(String(p || "").trim());
  }

  /* ---------- السلة والإجماليات ---------- */

  function cartSubtotal(items) {
    return (items || []).reduce((a, i) => a + Number(i.price || 0) * Number(i.qty || 0), 0);
  }

  // مصاريف الشحن: سعر المحافظة لو موجود، وإلا السعر الافتراضي
  function resolveFee(settings, gov) {
    const f = (settings && settings.fees) || {};
    if (gov && f.govs && f.govs[gov] !== undefined && f.govs[gov] !== null && f.govs[gov] !== "") {
      return Number(f.govs[gov]) || 0;
    }
    return Number(f.default) || 0;
  }

  /* ---------- أكواد الخصم ---------- */

  // فحص كود خصم ضد الإجمالي والتاريخ. يرجع {ok, reason?}
  function validatePromo(promo, subtotal, now) {
    now = now || new Date();
    if (!promo) return { ok: false, reason: "not_found" };
    if (promo.active === false) return { ok: false, reason: "inactive" };
    if (promo.expiresAt) {
      const t = promo.expiresAt.toDate ? promo.expiresAt.toDate() : new Date(promo.expiresAt);
      if (t.getTime() < now.getTime()) return { ok: false, reason: "expired" };
    }
    if (promo.minTotal && subtotal < Number(promo.minTotal)) {
      return { ok: false, reason: "min_total", minTotal: Number(promo.minTotal) };
    }
    return { ok: true };
  }

  // قيمة الخصم: نسبة مئوية أو مبلغ ثابت (لا يتجاوز الإجمالي)
  function promoDiscount(promo, subtotal) {
    if (!promo) return 0;
    if (promo.type === "percent") {
      return Math.round(subtotal * (Number(promo.value) || 0) / 100);
    }
    return Math.min(subtotal, Number(promo.value) || 0);
  }

  // الحساب النهائي: المنتجات − الخصم + الشحن (لا يقل عن صفر)
  function calcTotals(items, settings, gov, promo) {
    const subtotal = cartSubtotal(items);
    const discount = Math.min(subtotal, promoDiscount(promo, subtotal));
    const fee = resolveFee(settings, gov);
    return { subtotal, discount, fee, total: Math.max(0, subtotal - discount + fee) };
  }

  /* ---------- أكواد الطلبات ---------- */

  // كود طلب مقروء وغير قابل للتخمين: DK-XXXXXX (729 مليون احتمال)
  function genOrderCode(rand) {
    rand = rand || Math.random;
    let s = "";
    for (let i = 0; i < 6; i++) s += CODE_ALPHABET[Math.floor(rand() * CODE_ALPHABET.length)];
    return "DK-" + s;
  }

  /* ---------- بناء الطلب ---------- */

  // يبني مستند الطلب النهائي من بيانات السلة والعميل — لقطة أسعار وقت الطلب
  function buildOrder(opts) {
    const items = (opts.items || []).map(i => ({
      productId: i.id,
      name: i.name,
      price: Number(i.price) || 0,
      qty: Number(i.qty) || 0,
      size: i.size || "",
      color: i.color || ""
    }));
    const totals = calcTotals(items, opts.settings, opts.customer.governorate, opts.promo);
    return {
      items,
      subtotal: totals.subtotal,
      discount: totals.discount,
      promoCode: opts.promo && opts.promoCode ? opts.promoCode : "",
      deliveryFee: totals.fee,
      total: totals.total,
      customer: {
        name: String(opts.customer.name || "").trim(),
        phone: String(opts.customer.phone || "").trim(),
        governorate: opts.customer.governorate || "",
        address: String(opts.customer.address || "").trim(),
        notes: String(opts.customer.notes || "").trim()
      },
      payment: opts.payment === "vodafone" ? "vodafone" : "cod",
      status: "new"
    };
  }

  /* ---------- المخزون ---------- */

  // يحسب تخفيضات المخزون بعد طلب ناجح.
  // يرجع {ops:[{id,from,to}], insufficient:[{id,name,available,wanted}]}
  // المنتجات بدون رقم مخزون (null) تعتبر غير محدودة ولا عملية لها.
  function stockDecrements(items, productsById) {
    const ops = [], insufficient = [];
    for (const i of items || []) {
      const p = productsById[i.id];
      if (!p || p.stock === null || p.stock === undefined || p.stock === "") continue;
      const from = Number(p.stock) || 0;
      const wanted = Number(i.qty) || 0;
      if (from < wanted) {
        insufficient.push({ id: i.id, name: p.name || i.name, available: from, wanted });
      } else {
        ops.push({ id: i.id, from, to: from - wanted });
      }
    }
    return { ops, insufficient };
  }

  /* ---------- تقارير الأدمن ---------- */

  // إحصائيات الطلبات للوحة المعلومات
  function orderStats(orders, now) {
    now = now || new Date();
    const startOfToday = new Date(now.getFullYear(), now.getMonth(), now.getDate()).getTime();
    // revenue: كل الطلبات غير الملغاة. deliveredRevenue: المسلّمة فقط (الفلوس المحصلة فعليًا في الكاش)
    const s = { total: orders.length, today: 0, newCount: 0, revenue: 0, deliveredRevenue: 0, canceled: 0, byProduct: {} };
    for (const o of orders) {
      const ts = o.createdAt && o.createdAt.toDate ? o.createdAt.toDate().getTime()
        : (o.createdAt ? new Date(o.createdAt).getTime() : 0);
      if (ts >= startOfToday) s.today++;
      if (o.status === "new") s.newCount++;
      if (o.status === "canceled") { s.canceled++; continue; }
      s.revenue += Number(o.total) || 0;
      if (o.status === "delivered") s.deliveredRevenue += Number(o.total) || 0;
      for (const i of o.items || []) {
        s.byProduct[i.name] = (s.byProduct[i.name] || 0) + (Number(i.qty) || 0);
      }
    }
    s.topProducts = Object.entries(s.byProduct).sort((a, b) => b[1] - a[1]).slice(0, 5);
    return s;
  }

  // تصدير الطلبات CSV (مع BOM عشان Excel يقرأ العربي)
  function ordersToCSV(orders) {
    const esc = v => '"' + String(v === null || v === undefined ? "" : v).replace(/"/g, '""') + '"';
    const head = ["كود الطلب","التاريخ","الاسم","الموبايل","المحافظة","العنوان","المنتجات","طريقة الدفع","كود الخصم","الشحن","الخصم","الإجمالي","الحالة"];
    const rows = orders.map(o => [
      o.code || o.id,
      o.createdAt && o.createdAt.toDate ? o.createdAt.toDate().toLocaleString("en-GB") : "",
      o.customer && o.customer.name, o.customer && o.customer.phone,
      o.customer && o.customer.governorate, o.customer && o.customer.address,
      (o.items || []).map(i => i.name + " x" + i.qty + (i.size ? " (" + i.size + ")" : "") + (i.color ? " [" + i.color + "]" : "")).join(" | "),
      o.payment === "vodafone" ? "فودافون كاش" : "كاش عند الاستلام",
      o.promoCode || "", o.deliveryFee || 0, o.discount || 0, o.total || 0, o.status
    ]);
    return "\uFEFF" + [head].concat(rows).map(r => r.map(esc).join(",")).join("\r\n");
  }

  return {
    GOVS, ORDER_STATUSES, CODE_ALPHABET,
    eg, validEgyptPhone,
    cartSubtotal, resolveFee, validatePromo, promoDiscount, calcTotals,
    genOrderCode, buildOrder, stockDecrements,
    orderStats, ordersToCSV
  };
});
