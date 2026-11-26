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

  // يوحّد أشكال التوقيت: Firestore Timestamp (toDate) أو {seconds} أو Date أو نص
  function tsOf(v) {
    if (!v) return null;
    if (v.toDate) return v.toDate();
    if (typeof v.seconds === "number") return new Date(v.seconds * 1000);
    const d = new Date(v);
    return isNaN(d.getTime()) ? null : d;
  }


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
      const t0 = tsOf(o.createdAt);
      const ts = t0 ? t0.getTime() : 0;
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
      tsOf(o.createdAt) ? tsOf(o.createdAt).toLocaleString("en-GB") : "",
      o.customer && o.customer.name, o.customer && o.customer.phone,
      o.customer && o.customer.governorate, o.customer && o.customer.address,
      (o.items || []).map(i => i.name + " x" + i.qty + (i.size ? " (" + i.size + ")" : "") + (i.color ? " [" + i.color + "]" : "")).join(" | "),
      o.payment === "vodafone" ? "فودافون كاش" : "كاش عند الاستلام",
      o.promoCode || "", o.deliveryFee || 0, o.discount || 0, o.total || 0, o.status
    ]);
    return "\uFEFF" + [head].concat(rows).map(r => r.map(esc).join(",")).join("\r\n");
  }


  /* ---------- واتساب ---------- */

  // رقم واتساب بالصيغة الدولية من رقم مصري (01xxxxxxxxx -> 201xxxxxxxxx)
  function waDigits(phone) {
    let d = String(phone || "").replace(/\D/g, "");
    if (d.startsWith("0020")) d = d.slice(2);
    if (d.startsWith("0")) d = "2" + d;
    return d;
  }

  // نص رسالة يبعتها العميل لصاحب المتجر لتأكيد الطلب على واتساب
  function orderWhatsAppText(order, code) {
    const c = order.customer || {};
    const lines = ["طلب جديد من دكانة 🛍️", "الكود: " + code, "الاسم: " + (c.name || "")];
    if (c.governorate) lines.push("المحافظة: " + c.governorate);
    if (c.address) lines.push("العنوان: " + c.address);
    for (const i of order.items || []) {
      lines.push("• " + i.name + " ×" + i.qty + (i.size ? " (" + i.size + ")" : "") + (i.color ? " [" + i.color + "]" : ""));
    }
    lines.push("الإجمالي: " + eg(order.total) + (Number(order.deliveryFee) ? " (شامل شحن " + eg(order.deliveryFee) + ")" : ""));
    lines.push("الدفع: " + (order.payment === "vodafone" ? "فودافون كاش" : "كاش عند الاستلام"));
    return lines.join("\n");
  }

  // لينك wa.me جاهز — فاضي لو مفيش رقم واتساب متسجل
  function orderWhatsAppLink(whatsappNumber, order, code) {
    const w = waDigits(whatsappNumber);
    if (!w) return "";
    return "https://wa.me/" + w + "?text=" + encodeURIComponent(orderWhatsAppText(order, code));
  }

  /* ---------- تقارير إضافية للوحة المعلومات ---------- */

  // إيراد آخر N يوم (بدون الملغي): [{day, label, revenue, count}] من الأقدم للأحدث
  function revenueByDay(orders, days, now) {
    now = now || new Date(); days = Math.max(1, days || 7);
    const key = d => d.getFullYear() + "-" + String(d.getMonth() + 1).padStart(2, "0") + "-" + String(d.getDate()).padStart(2, "0");
    const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
    const buckets = {}, out = [];
    for (let i = days - 1; i >= 0; i--) {
      const d = new Date(today.getTime() - i * 864e5), k = key(d);
      buckets[k] = { day: k, label: d.getDate() + "/" + (d.getMonth() + 1), revenue: 0, count: 0 };
      out.push(buckets[k]);
    }
    for (const o of orders || []) {
      if (o.status === "canceled") continue;
      const t = tsOf(o.createdAt);
      if (!t) continue;
      const b = buckets[key(t)];
      if (!b) continue;
      b.revenue += Number(o.total) || 0; b.count++;
    }
    return out;
  }

  // أكتر المحافظات طلبًا (بدون الملغي): [[محافظة, عدد], ...]
  function topGovernorates(orders, limit) {
    const m = {};
    for (const o of orders || []) {
      if (o.status === "canceled") continue;
      const g = o.customer && o.customer.governorate;
      if (!g) continue;
      m[g] = (m[g] || 0) + 1;
    }
    return Object.entries(m).sort((a, b) => b[1] - a[1]).slice(0, limit || 5);
  }

  /* ---------- بوليصة الشحن ---------- */

  function escHtml(s) {
    return String(s === null || s === undefined ? "" : s).replace(/[&<>"']/g,
      c => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]));
  }

  // بوليصة شحن كاملة قابلة للطباعة (صفحة HTML مستقلة، بتتطبع فور فتحها)
  function shippingLabelHTML(order, code, settings, opts) {
    opts = opts || {};
    const st = settings || {};
    const cust = order.customer || {};
    const dateStr = opts.dateStr || "";
    const items = (order.items || []).map(i =>
      "<tr><td>" + escHtml(i.name) + (i.size ? " (" + escHtml(i.size) + ")" : "") + (i.color ? " [" + escHtml(i.color) + "]" : "") +
      "</td><td>×" + (Number(i.qty) || 0) + "</td><td>" + eg(Number(i.price || 0) * Number(i.qty || 0)) + "</td></tr>").join("");
    const cod = order.payment === "vodafone"
      ? '<div class="cod vf">مدفوع مقدمًا — فودافون كاش ✅<br><small>متحصلش من العميل أي مبلغ</small></div>'
      : '<div class="cod">حصّل من العميل: <b>' + eg(order.total) + '</b><br><small>كاش عند الاستلام</small></div>';
    return '<!DOCTYPE html>\n<html dir="rtl" lang="ar"><head><meta charset="utf-8"><title>بوليصة ' + escHtml(code) + '</title>\n' +
'<style>\n' +
"*{margin:0;box-sizing:border-box;font-family:'Segoe UI',Tahoma,sans-serif}\n" +
"body{background:#fff;color:#111;padding:14px}\n" +
".label{max-width:420px;margin:0 auto;border:2px solid #0b2e59;border-radius:14px;overflow:hidden}\n" +
".head{background:#0b2e59;color:#fff;padding:12px 16px;display:flex;justify-content:space-between;align-items:center}\n" +
".brand{font-size:22px;font-weight:900}.brand i{color:#f2a619;font-style:normal}\n" +
".head span{font-size:13px;opacity:.85}\n" +
".meta{display:flex;justify-content:space-between;padding:10px 16px;border-bottom:1px dashed #ccc;font-size:13px;align-items:center}\n" +
".meta b{font-size:16px}\n" +
".sec{padding:10px 16px;border-bottom:1px dashed #ccc;font-size:13.5px;line-height:1.7}\n" +
".t{font-size:11px;color:#888;font-weight:700;margin-bottom:2px}\n" +
"table{width:100%;border-collapse:collapse;font-size:13px}\n" +
"td{padding:5px 16px;border-bottom:1px solid #eee}\n" +
"tr td:last-child{text-align:left}\n" +
".cod{margin:12px 16px;background:#fff6df;border:1.5px solid #f2a619;border-radius:10px;padding:12px;text-align:center;font-size:15px}\n" +
".cod b{font-size:20px}\n" +
".cod.vf{background:#e8f6ef;border-color:#0e9f6e}\n" +
".sig{display:flex;justify-content:space-between;padding:12px 16px;font-size:12.5px;color:#555}\n" +
".foot{text-align:center;font-size:11.5px;color:#999;padding:8px}\n" +
"@media print{body{padding:0}.label{border-radius:0;max-width:none}}\n" +
"</style></head><body>\n" +
'<div class="label">\n' +
'  <div class="head"><span class="brand">دكانة<i>.</i></span><span>بوليصة شحن</span></div>\n' +
'  <div class="meta"><b>' + escHtml(code) + '</b><span>' + escHtml(dateStr) + '</span></div>\n' +
'  <div class="sec"><div class="t">من</div>متجر دكانة' + (st.courierName ? " — عن طريق " + escHtml(st.courierName) : "") + (st.vodafoneNumber ? "<br>تليفون المتجر: " + escHtml(st.vodafoneNumber) : "") + '</div>\n' +
'  <div class="sec"><div class="t">إلى</div><b>' + escHtml(cust.name) + '</b> — <b dir="ltr">' + escHtml(cust.phone) + '</b><br>' +
escHtml(cust.governorate) + (cust.address ? " — " + escHtml(cust.address) : "") + (cust.notes ? "<br>ملاحظات: " + escHtml(cust.notes) : "") + '</div>\n' +
'  <div class="sec" style="padding:4px 0 8px"><div class="t" style="padding:0 16px">المنتجات</div><table>' + items + '</table></div>\n' +
cod + '\n' +
'  <div class="sig"><span>رقم التتبع: ' + (order.trackingNumber ? escHtml(order.trackingNumber) : "…………") + '</span><span>توقيع المستلم: ……………</span></div>\n' +
'  <div class="foot">شكرًا لتسوقك من دكانة' + (st.whatsappNumber ? " — واتساب المتجر: " + escHtml(st.whatsappNumber) : "") + '</div>\n' +
'</div>\n' +
'<script>window.onload=function(){window.print()}<\/script>\n' +
'</body></html>';
  }

  return {
    GOVS, ORDER_STATUSES, CODE_ALPHABET,
    eg, validEgyptPhone,
    cartSubtotal, resolveFee, validatePromo, promoDiscount, calcTotals,
    genOrderCode, buildOrder, stockDecrements,
    tsOf, orderStats, ordersToCSV,
    waDigits, orderWhatsAppText, orderWhatsAppLink,
    revenueByDay, topGovernorates, escHtml, shippingLabelHTML
  };
});
